# Chapter 18 — Transactions & MVCC

## 18.1 The problem: concurrent readers and writers without blocking, without corruption

Every real database has to answer the same question: what happens when one
session is in the middle of writing a row while another session tries to read
it? Two families of answer exist, and it's worth being explicit about both
before looking at which one DuckDB picked.

**Lock-based concurrency control** answers by making the writer and the reader
take turns. A writer that wants to change a row takes an exclusive lock on it;
any reader that wants to see the row has to wait for that lock to be released
before it can even look. This is simple to reason about and it is genuinely
correct, but it has an operational cost that shows up immediately under
concurrency: a long-running writer (an `UPDATE` touching a million rows, say)
stalls every reader that wants any of those rows, for as long as the writer's
transaction stays open. Readers block writers too, in the other direction, in
systems that also take read locks. This is the traditional relational-database
default, and it is why "why is my `SELECT` hanging" is such a common first
encounter with transactions for anyone coming from a lock-based system.

**Multi-Version Concurrency Control (MVCC)** answers differently: instead of
making a writer's in-progress change invisible by *blocking* readers from
looking, it makes the writer's in-progress change invisible by *versioning* —
a reader that started before the writer's change simply never sees a version
of the row newer than the one that existed when the reader's transaction
began. Both the reader and the writer proceed concurrently, on separate copies
of the truth, and nobody blocks anybody for the sake of visibility. The
correctness property this needs to deliver is **snapshot isolation**: every
transaction sees the database as a single consistent snapshot, frozen at some
well-defined point in time (usually thought of as "the moment my transaction
started"), no matter how much writing happens to the underlying tables while
that transaction is still open. A transaction never sees a half-committed
row — the classic case a lock-based system's read locks exist specifically to
prevent — and it never sees `some columns from before an UPDATE, some columns
from after` for the same row.

DuckDB's specific flavor of this, confirmed by the source examined in this
chapter, is **snapshot isolation via row-versioning**: a concurrent write
never overwrites a row that a still-open transaction might need to see.
Instead, it creates a new, timestamped version of the affected data and links
it into the existing chain of versions for that row (or, for a brand-new row,
simply marks the new row as visible only from a particular point onward).
Readers walk this chain and pick out whichever version is visible to their own
snapshot. The rest of this chapter is about exactly how that chain is built,
walked, made permanent, and eventually discarded — and about the two-layer
picture that makes the whole thing durable across a crash, not just correct
while the process is alive.

## 18.2 DuckTransaction: "what am I allowed to see?"

Every open transaction in DuckDB's own storage engine is represented by a
`DuckTransaction` object, and the two fields that matter most sit right at the
top of the class:

```cpp
// src/include/duckdb/transaction/duck_transaction.hpp:36
class DuckTransaction : public Transaction {
public:
	DuckTransaction(DuckTransactionManager &manager, ClientContext &context, transaction_t start_time,
	                SnapshotView view, idx_t catalog_version);
	~DuckTransaction() override;

	//! The start timestamp of this transaction
	transaction_t start_time;
	//! What this transaction sees: its own writes, and everything before its visibility bound
	SnapshotView view;
	//! The commit id of this transaction, if it has successfully been committed
	transaction_t commit_id;
	// ...
};
```

`start_time` and `view` are, put plainly, the transaction's entire answer to
the question "what am I allowed to see?" — its own uncommitted writes, plus
everything that was committed before its visibility bound. `SnapshotView`
spells this out as a two-line method:

```cpp
// src/include/duckdb/transaction/transaction_data.hpp:23-37
struct SnapshotView {
	SnapshotView(transaction_t transaction_id_p, VisibilityBound visibility_bound_p)
	    : transaction_id(transaction_id_p), visibility_bound(visibility_bound_p) {
	}

	//! The reading transaction, so that its own writes are visible to it
	transaction_t transaction_id;
	//! Exclusive: timestamps below it are visible
	VisibilityBound visibility_bound;

	//! Below the bound, or written by this transaction
	bool Sees(transaction_t timestamp) const {
		return timestamp < visibility_bound || timestamp == transaction_id;
	}
};
```

`Sees()` is the entire visibility check, and it is worth staring at because it
is the single most load-bearing one-liner in this chapter: given some row
version's stamped timestamp, a transaction can see it if that timestamp is
below the transaction's visibility bound, *or* if the timestamp is literally
this transaction's own ID (so a transaction always sees its own uncommitted
writes, even though nobody else does yet). Every MVCC visibility decision in
DuckDB's storage layer reduces to this comparison — one branch, two integer
comparisons, no locks, no I/O.

### Why this is cheap: transaction IDs and commit timestamps share one counter space, split in two

That `Sees()` can be a bare integer comparison is not an accident — it's the
payoff of a specific numbering scheme. `transaction_t` itself is nothing
exotic:

```cpp
// src/include/duckdb/common/typedefs.hpp:32
//! Type used for transaction timestamps
typedef idx_t transaction_t;
```

— a 64-bit unsigned integer, used for both "the timestamp a change committed
at" and "the ID of a transaction that hasn't committed yet." The trick is that
these two meanings live in two disjoint numeric ranges, so a single comparison
against a fixed constant tells you which one you're looking at:

```cpp
// src/common/constants.cpp:18-23
const transaction_t SYSTEM_TRANSACTION_TIMESTAMP = 1;
const transaction_t TRANSACTION_ID_START = 4611686018427388000ULL;                // 2^62
const transaction_t MAX_COMMIT_ID = TRANSACTION_ID_START - 1;                     // 2^62 - 1
const transaction_t MAX_TRANSACTION_ID = NumericLimits<transaction_t>::Maximum(); // 2^63
const transaction_t NOT_DELETED_ID = NumericLimits<transaction_t>::Maximum() - 1; // 2^64 - 1
const transaction_t MAXIMUM_QUERY_ID = NumericLimits<transaction_t>::Maximum();   // 2^64
```

```cpp
// src/include/duckdb/transaction/transaction_data.hpp:18-20
inline bool IsCommitted(transaction_t timestamp) {
	return timestamp <= MAX_COMMIT_ID;
}
```

Commit timestamps are handed out starting at 2 and counting up (real, small
numbers). Transaction IDs — the placeholder stamp a row version gets the
instant it's written, before anyone knows whether the transaction will ever
commit — start at `2^62` and count up from there, a range that ordinary commit
timestamps will never reach in the lifetime of a database. So `IsCommitted`
is one comparison, and `Sees()` never has to ask a catalog, a lock manager, or
anything else "is this thing committed and if so when" — the number itself
already encodes the answer, or encodes "not committed yet, and belongs to
transaction ID `N`."

`DuckTransactionManager::StartTransaction` is where both numbers get drawn,
from two counters that live on the manager and are protected by one lock:

```cpp
// src/transaction/duck_transaction_manager.cpp:78-101 (elided)
Transaction &DuckTransactionManager::StartTransaction(ClientContext &context) {
	// ...
	lock_guard<mutex> lock(transaction_lock);
	// ...
	// obtain the start time and transaction ID of this transaction
	transaction_t start_time = current_start_timestamp++;
	transaction_t transaction_id = current_transaction_id++;
	// the transaction sees its own writes, and every commit before its start time
	SnapshotView view(transaction_id, VisibilityBound::Before(start_time));
	// ...
	auto transaction = make_uniq<DuckTransaction>(*this, context, start_time, view, last_committed_version);
	// ...
}
```

Both `current_start_timestamp` and `current_transaction_id` are plain
monotonically-increasing counters (`current_start_timestamp` seeded one past
`SYSTEM_TRANSACTION_TIMESTAMP`, `current_transaction_id` seeded at
`TRANSACTION_ID_START`, per the constructor at
`src/transaction/duck_transaction_manager.cpp:49-65`). There is no clock
involved, no wall-clock timestamp, no coordination across machines — a
transaction's "time" is simply "how many transactions have started before me,"
and a commit's "time" is "how many transactions have committed before this
one." That is all `snapshot isolation via row-versioning` needs: a strict,
total, cheaply-comparable order over events, not real time.

The constructor also sets `last_commit = SYSTEM_TRANSACTION_TIMESTAMP`
(line 53). `last_commit` is the newest commit timestamp. A checkpoint that runs
with no client transaction (for example at shutdown) reads at
`VisibilityBound::Through(last_commit)`. So `last_commit` must have a value
even before the first commit. If a database were reopened and then shut down at
once with `last_commit == 0`, that bound would hide everything stamped with
timestamp 1, which is how catalog entries loaded from disk are stamped. The
checkpoint would leave those tables out, and they would vanish after the next
restart. `test/sql/storage/checkpoint/checkpoint_without_transactions.test`
guards against this bug.

## 18.3 UndoBuffer: the actual MVCC mechanism

Everything in §18.2 explains how a transaction decides what it's allowed to
see. `UndoBuffer` is the structure that makes there be something *to* decide
among — it is where every "what did this row look like before I changed it"
fact lives while a transaction is open, and its own doc comment states its
job plainly:

```cpp
// src/include/duckdb/transaction/undo_buffer.hpp:34-38
//! The undo buffer of a transaction is used to hold previous versions of tuples
//! that might be required in the future (because of rollbacks or previous
//! transactions accessing them)
class UndoBuffer {
	// ...
};
```

Every open transaction owns exactly one `UndoBuffer`. It is an
append-only log, in memory, of small typed entries — one entry per row (or
per contiguous run of rows) that this transaction has touched. The entry
types are an enum, and there are exactly six of them:

```cpp
// src/include/duckdb/common/enums/undo_flags.hpp:15-23
enum class UndoFlags : uint32_t { // far too big but aligned (TM)
	EMPTY_ENTRY = 0,
	CATALOG_ENTRY = 1,
	INSERT_TUPLE = 2,
	DELETE_TUPLE = 3,
	UPDATE_TUPLE = 4,
	SEQUENCE_VALUE = 5,
	ATTACHED_DATABASE = 6
};
```

`INSERT_TUPLE`, `DELETE_TUPLE`, and `UPDATE_TUPLE` are the row-data cases this
chapter is mostly about. `CATALOG_ENTRY` is the same mechanism applied one
level up — a `CREATE TABLE`, `ALTER TABLE`, or `DROP TABLE` pushes an undo
entry that points at the old `CatalogEntry`, so schema changes get exactly the
same commit/rollback/cleanup treatment as row changes. `SEQUENCE_VALUE` tracks
a transaction's use of a `SEQUENCE` (so a rolled-back transaction's sequence
consumption can be reasoned about), and `ATTACHED_DATABASE` tracks an `ATTACH`
issued inside the transaction (so a rolled-back `ATTACH` can be detached
again — see `RollbackState::RollbackEntry`'s `ATTACHED_DATABASE` case in
§18.3.3). Each entry is a tiny self-describing record — a type tag, a length,
and a payload — appended to a per-transaction buffer:

```cpp
// src/transaction/undo_buffer.cpp:28-39
UndoBufferReference UndoBuffer::CreateEntry(UndoFlags type, idx_t len) {
	idx_t alloc_len = AlignValue<idx_t>(len + UNDO_ENTRY_HEADER_SIZE);
	auto handle = allocator.Allocate(alloc_len);
	auto data = handle.GetDataMutable();
	// write the undo entry metadata
	Store<UndoFlags>(type, data);
	data += sizeof(UndoFlags);
	Store<uint32_t>(UnsafeNumericCast<uint32_t>(alloc_len - UNDO_ENTRY_HEADER_SIZE), data);
	// increment the position of the header past the undo entry metadata
	handle.position += UNDO_ENTRY_HEADER_SIZE;
	return handle;
}
```

Three things then walk this log, one per moment in a transaction's life —
commit, rollback, and (much later) garbage collection — and each walk visits
the entries in a different order for a reason:

- **Commit** (`commit_state.cpp`) walks **forward**, oldest entry to newest
  (insertion order), via `UndoBuffer::IterateEntries`.
- **Rollback** (`rollback_state.cpp`) walks **backward**, newest entry to
  oldest, via `UndoBuffer::ReverseIterateEntries`.
- **Cleanup** (`cleanup_state.cpp`) walks forward again, once no transaction
  can possibly still need any of these entries.

Rollback's reverse order matters concretely: if a single transaction issues
two `UPDATE`s to the same row (say, `status = 'shipped'` and then, moments
later inside the same transaction, `status = 'cancelled'`), undoing must peel
the second change off before the first, exactly like popping a stack —
otherwise the row would end up in neither its original state nor any state
the transaction actually produced.

### 18.3.1 Running example: `UPDATE orders SET status = 'shipped' WHERE orderkey = 42`

Follow one statement through the whole apparatus. Assume `orders` is an
ordinary, already-committed table and row `orderkey = 42` already exists with
`status = 'pending'`; some session opens `BEGIN;` and runs the `UPDATE` above,
but has not yet said `COMMIT` or `ROLLBACK`.

**On UPDATE, DuckDB does not do "append a whole new row."** It also does not
do a bare in-place overwrite with nothing recorded. What it does — confirmed
in `src/storage/table/update_segment.cpp` — is a **column-level version
chain**: for the vector (batch of up to `STANDARD_VECTOR_SIZE` rows) that
contains row 42, there is a small linked list of `UpdateInfo` structs, one
link per still-relevant version:

```cpp
// src/include/duckdb/transaction/update_info.hpp:44-49 (elided)
struct UpdateInfo {
	// ...
	//! The version number
	atomic<transaction_t> version_number;
	// ...
	//! The previous update info (or nullptr if it is the base)
	UndoBufferPointer prev;
	//! The next update info in the chain (or nullptr if it is the last)
	UndoBufferPointer next;
	// ...
};
```

The **head of the chain** (called `root` / `base_info` in the code) always
holds the *current* value — `status = 'pending'` before our `UPDATE` runs, and
`status = 'shipped'` the instant it does, whether or not that write has
committed yet. `UpdateSegment::Update` creates a brand-new `UpdateInfo` for
this transaction (via `DuckTransaction::CreateUpdateInfo`, which allocates it
*inside this transaction's `UndoBuffer` as an `UPDATE_TUPLE` entry*) and
splices it in immediately after the head:

```cpp
// src/storage/table/update_segment.cpp:1394 (elided to the splice + merge)
void UpdateSegment::Update(TransactionData transaction, DuckTableEntry &table_entry, idx_t column_index,
                           Vector &update_p, row_t *ids, idx_t count, Vector &base_data, idx_t row_group_start) {
	// ...
	// insert the new node into the chain
	node->next = base_info.next;
	// ...
	node->prev = root_pointer;
	base_info.next = /* new node */;
	// ...
	// now we are going to perform the merge
	merge_update_function(base_info, base_data, *node, update_format, ids, count, sel, row_group_start);
	// ...
}
```

and the merge step (`InitializeUpdateData` / its "already has updates"
sibling `MergeUpdateInfo`) is where the split happens: the *new* value goes
into the head of the chain (so anyone who reads the column right now sees
`'shipped'`), and the **old, pre-update value is copied into this
transaction's own `UpdateInfo`** — the very entry that just became an
`UPDATE_TUPLE` record inside the transaction's `UndoBuffer`:

```cpp
// src/storage/table/update_segment.cpp:785-807 (elided)
static void InitializeUpdateData(UpdateInfo &base_info, Vector &base_data, UpdateInfo &update_info,
                                 UnifiedVectorFormat &update, const SelectionVector &sel) {
	auto update_data = update.GetData<T>(update);
	auto tuple_data = update_info.GetData<T>();
	for (idx_t i = 0; i < update_info.N; i++) {
		auto idx = update.sel->get_index(sel.get_index(i));
		tuple_data[i] = update_data[idx];   // new value -> current/root slot
	}

	auto base_array_data = FlatVector::GetData<T>(base_data);
	// ...
	auto base_tuple_data = base_info.GetData<T>();
	for (idx_t i = 0; i < base_info.N; i++) {
		auto base_idx = base_tuples[i];
		// ...
		base_tuple_data[i] = base_array_data[base_idx]; // old value -> this transaction's undo entry
	}
}
```

(The two calls in the caller are named confusingly close to identically —
`base_info`/`update_info` swap roles depending on whether this is the first
update ever made to this vector or a subsequent one — but the invariant is
constant: **head of chain = current value, freshly-linked transaction entry =
pre-image**.) So, concretely, after the `UPDATE` runs but before `COMMIT`:

```
UpdateSegment (vector holding row 42)

   root/head ─────────────► [ status = 'shipped' ]   version_number = MAX_COMMIT_ID
                                     │ next
                                     ▼
                             [ status = 'pending' ]   version_number = <this txn's transaction_id, e.g. 2^62 + 7>
                             (an UPDATE_TUPLE entry, living inside
                              this transaction's UndoBuffer)
```

Any other transaction that reads row 42 right now walks this chain
(`UpdateInfo::UpdatesForTransaction`) and asks, link by link, "does my
snapshot see this link's `version_number`?" via exactly the `Sees()` check
from §18.2:

```cpp
// src/include/duckdb/transaction/update_info.hpp:61-70 (elided)
bool AppliesToTransaction(const SnapshotView &view) {
	if (version_number == MAX_COMMIT_ID) {
		return true; // dummy transaction number for the root element - should always match
	}
	return !view.Sees(version_number.load());
}
```

Read that last line carefully: it is `!view.Sees(...)`, because the value
sitting *in this link* is the value to substitute *if the reader's snapshot
does **not** yet see the change that produced this link* — i.e., if the
change is still in the future relative to the reader. A transaction whose
snapshot predates our `UPDATE` walks past the head and lands on the
`'pending'` link; a transaction whose snapshot postdates a *committed*
`UPDATE` never even needs to look past the head.

### 18.3.2 On COMMIT: entries get a real timestamp, they don't get deleted

`commit_state.cpp`'s `CommitState::CommitEntry` is what a `COMMIT` actually
runs, once per entry in the transaction's `UndoBuffer`, walked forward
(oldest first) via `UndoBuffer::Commit`:

```cpp
// src/transaction/undo_buffer.cpp:193-199
void UndoBuffer::Commit(UndoBuffer::IteratorState &iterator_state, CommitInfo &info) {
	active_transaction_state = info.active_transactions;
	CommitState state(transaction, info.commit_id, active_transaction_state, CommitMode::COMMIT);
	IterateEntries(iterator_state, [&](UndoFlags type, data_ptr_t data) { state.CommitEntry(type, data, info); });
	state.Verify();
}
```

For our `UPDATE_TUPLE` entry, committing is a single field write:

```cpp
// src/transaction/commit_state.cpp:383-395
case UndoFlags::UPDATE_TUPLE: {
	// update:
	auto info = reinterpret_cast<UpdateInfo *>(data);
	// ...
	info->version_number = commit_id;
	break;
}
```

The placeholder `transaction_id` (`2^62 + 7` in the running example) is
overwritten with the real, small, permanent `commit_id` handed out by
`DuckTransactionManager::GetCommitTimestamp()`. Nothing is deleted, nothing
moves in memory, no row is copied anywhere. The link that used to say "not
committed yet, belongs to transaction `2^62+7`" now says "committed at time
`117`" (say), and from that instant on, any transaction whose visibility
bound has advanced past `117` will see straight through to the head of the
chain — where `'shipped'` has been sitting the whole time — instead of
falling through to this link's old `'pending'` value. `INSERT_TUPLE` and
`DELETE_TUPLE` entries go through the same pattern: `CommitAppend(commit_id,
...)` and `CommitDelete(commit_id, ...)` respectively stamp the real commit
timestamp onto rows that were, until this instant, only visible to their own
inserting/deleting transaction.

### 18.3.3 On ROLLBACK: the undo entries physically undo the writes

`rollback_state.cpp`'s `RollbackState::RollbackEntry` is what a `ROLLBACK`
runs — walked in **reverse** order via `UndoBuffer::Rollback`:

```cpp
// src/transaction/undo_buffer.cpp:208-212
void UndoBuffer::Rollback() {
	// rollback needs to be performed in reverse
	RollbackState state(transaction);
	ReverseIterateEntries([&](UndoFlags type, data_ptr_t data) { state.RollbackEntry(type, data); });
}
```

```cpp
// src/transaction/rollback_state.cpp:23-71 (elided)
void RollbackState::RollbackEntry(UndoFlags type, data_ptr_t data) {
	switch (type) {
	case UndoFlags::CATALOG_ENTRY: {
		auto catalog_entry = Load<CatalogEntry *>(data);
		catalog_entry->set->Undo(*catalog_entry);
		break;
	}
	case UndoFlags::INSERT_TUPLE: {
		auto info = reinterpret_cast<AppendInfo *>(data);
		info->table->GetStorage().RevertAppend(transaction, info->start_row, info->count);
		break;
	}
	case UndoFlags::DELETE_TUPLE: {
		auto info = reinterpret_cast<DeleteInfo *>(data);
		// reset the deleted flag on rollback
		info->version_info->CommitDelete(info->vector_idx, NOT_DELETED_ID, *info);
		break;
	}
	case UndoFlags::UPDATE_TUPLE: {
		auto info = reinterpret_cast<UpdateInfo *>(data);
		info->segment->RollbackUpdate(*info);
		break;
	}
	case UndoFlags::ATTACHED_DATABASE: {
		auto db = Load<AttachedDatabase *>(data);
		auto &db_manager = DatabaseManager::Get(db->GetDatabase());
		// after this transaction detached it, another connection can have attached a database under the name
		auto attached_db = db_manager.DetachInternal(db->name, db);
		// ... detach the database this transaction ATTACHed, since it never really happened
	}
	// ...
	}
}
```

Each case is a literal physical undo of that entry's effect: a rolled-back
`INSERT` gets its row range reverted out of the table; a rolled-back `DELETE`
gets its tombstone timestamp reset to `NOT_DELETED_ID` (the row was never
really gone); a rolled-back `ATTACH` gets detached again. For our running
example, `UpdateSegment::RollbackUpdate` copies the pre-image sitting in this
`UndoBuffer` entry straight back into the head of the chain — undoing exactly
the write §18.3.1 made — and then unsplices this entry from the chain:

```cpp
// src/storage/table/update_segment.cpp:571-585
template <class T>
static void RollbackUpdate(UpdateInfo &base_info, UpdateInfo &rollback_info) {
	auto base_data = base_info.GetData<T>();
	auto base_tuples = base_info.GetTuples();
	auto rollback_data = rollback_info.GetData<T>();
	auto rollback_tuples = rollback_info.GetTuples();
	idx_t base_offset = 0;
	for (idx_t i = 0; i < rollback_info.N; i++) {
		auto id = rollback_tuples[i];
		while (base_tuples[base_offset] < id) {
			base_offset++;
		}
		base_data[base_offset] = rollback_data[i]; // old value written back into the head
	}
}
```

After this runs for the running example, the head of the chain reads
`status = 'pending'` again, exactly as if the `UPDATE` had never happened —
because, from every other transaction's point of view, it never committed,
and now it never will.

Two more details of rollback are worth knowing.

- **`DETACH` inside a transaction.** A transaction may `DETACH` a database it
  has already used, and its `COMMIT` still commits the work done there.
  Because of this, a name can be freed and reused while a transaction is open.
  So the `ATTACHED_DATABASE` rollback above passes the database object as well
  as the name (`DetachInternal(db->name, db)`). It only detaches the database
  this transaction attached, not a different database that another connection
  has since attached under the same name.
  `test/sql/attach/attach_detach_in_transaction.test` runs
  `ATTACH ... AS other_db`, `CREATE TABLE`, `INSERT`, `DETACH other_db` and
  `COMMIT` in one transaction, then attaches the file again and finds the row.
- **Half-built update nodes.** Pinning a block of the undo buffer goes through
  the buffer manager and can fail with out-of-memory, for example when the
  buffer was evicted to disk. `UpdateSegment::Update` pins the next node
  *before* it changes any links, and `RollbackUpdate` skips a node that was
  never linked into the chain. Otherwise a failure in the middle could leave a
  half-linked node whose rollback wrote stale values into the head of the
  chain.

### 18.3.4 Eventually: cleanup — garbage-collecting entries nobody can need anymore

Committing does not remove an undo entry (§18.3.2); it only stamps it with a
real timestamp and leaves it in place, because some *other*, still-open
transaction, with a snapshot older than this commit, might still need to walk
past it to reconstruct the pre-update value. Only once **every** currently
active transaction's snapshot has advanced past a given commit can that
commit's old-value link be thrown away. `UndoBuffer::Cleanup`'s own comment
states the precondition exactly:

```cpp
// src/transaction/undo_buffer.cpp:166-185
void UndoBuffer::Cleanup(VisibilityBound lowest_visibility_bound) {
	// garbage collect everything in the Undo Chunk
	// this should only happen if
	//  (1) the transaction this UndoBuffer belongs to has successfully
	//  committed
	//      (on Rollback the Rollback() function should be called, that clears
	//      the chunks)
	//  (2) there is no active transaction with start_id < commit_id of this
	//  transaction
	CleanupState state(transaction, lowest_visibility_bound, active_transaction_state);
	UndoBuffer::IteratorState iterator_state;
	idx_t entry_idx = 0;
	IterateEntries(iterator_state, [&](UndoFlags type, data_ptr_t data) {
		if (entry_idx++ < cleaned_up_entries) {
			return;
		}
		state.CleanupEntry(type, data);
		cleaned_up_entries++;
	});
}
```

The `cleaned_up_entries` counter is there because cleanup can throw part-way,
for example when it runs out of memory while pinning a block. It is then
retried. A retry that started again at the first entry would clean up some
entries twice: a delete would be removed from an index a second time (a fatal
error), and an update node would be unlinked again from neighbours that may
have changed. So the undo buffer counts the entries it has finished, and a
retry skips them. `CleanupUpdateInternal` also pins both neighbours of a node
before it changes either link
(`test/sql/transactions/test_commit_cleanup_oom_index_retry.test`).

`DuckTransactionManager` maintains exactly the number this precondition
needs — the *minimum snapshot bound across every currently active
transaction* — as `lowest_visibility_bound`:

```cpp
// src/include/duckdb/transaction/duck_transaction_manager.hpp:134-136 (elided)
//! The lowest bound any active transaction reads at. A version preceding it is visible to
//! every active transaction, so whatever it supersedes can be cleaned up or compacted
atomic<VisibilityBound> lowest_visibility_bound;
```

Every time a transaction starts or finishes, `UpdateLowestVisibilityBound`
recomputes this minimum over `active_transactions`. `DuckCleanupInfo` batches
transactions that share the same `lowest_visibility_bound` together so they
can be cleaned up in one pass, off the hot transaction-management path:

```cpp
// src/include/duckdb/transaction/duck_transaction_manager.hpp:21-30
//! CleanupInfo collects transactions awaiting cleanup.
//! This ensures we can clean up after releasing the transaction lock.
struct DuckCleanupInfo {
	//! All transactions in a cleanup info share the same lowest_visibility_bound.
	VisibilityBound lowest_visibility_bound;
	vector<unique_ptr<DuckTransaction>> transactions;

	void Cleanup();
	bool ScheduleCleanup() noexcept;
};
```

`cleanup_state.cpp`'s `CleanupEntry` finally does the physical work per entry
type — for `UPDATE_TUPLE`, unsplicing the link out of the version chain
entirely (`UpdateSegment::CleanupUpdateInternal`, which just relinks `prev`
and `next` around the discarded node); for `DELETE_TUPLE`, finishing the
removal from any indexes that were deferred while other transactions might
still need to see the "deleted" row. Once this runs, the pre-image is gone
for good — but only because nothing left in the system could possibly still
need it.

### Diagram: one row's version chain across UPDATE → COMMIT → cleanup

```
 Time ──────────────────────────────────────────────────────────────►

 (1) before UPDATE           (2) after UPDATE,           (3) after COMMIT           (4) after cleanup
     — no chain                  before COMMIT               (commit_id = 117)          (no other txn needs
                                                                                          the old value anymore)

  ┌───────────────┐         ┌───────────────┐          ┌───────────────┐          ┌───────────────┐
  │ status=pending│         │ status=shipped│◄─ head    │ status=shipped│◄─ head    │ status=shipped│◄─ head
  │ (base data,   │         │ ver=MAX_COMMIT│          │ ver=MAX_COMMIT│          │ ver=MAX_COMMIT│
  │  no chain)    │         └───────┬───────┘          └───────┬───────┘          └───────────────┘
  └───────────────┘                 │ next                     │ next                (chain collapsed:
                                     ▼                          ▼                      no other txn's
                            ┌───────────────┐          ┌───────────────┐              snapshot predates
                            │ status=pending│          │ status=pending│              commit 117 anymore)
                            │ ver=2^62+7    │          │ ver=117       │
                            │ (UNDO_TUPLE   │          │ (same entry,  │
                            │  entry, this  │          │  now a real   │
                            │  txn's        │          │  commit       │
                            │  UndoBuffer)  │          │  timestamp)   │
                            └───────────────┘          └───────────────┘

  ROLLBACK at step (2) would instead: copy 'pending' back into the head,
  unsplice the entry — same end state as (1), nothing ever became visible.
```

### 18.3.5 Inserted rows at commit: moving local rows into the table

The running example was an `UPDATE`. An `INSERT` takes a different path.
New rows first go into the transaction's own `LocalStorage` (Chapter 17,
§17.2.3), where no other transaction can see them. At commit,
`LocalStorage::Flush` (`src/storage/local_storage.cpp:618-689`) moves them
into the shared table in three steps:

1. add the new keys to the table's shared indexes;
2. add the rows to the table, either by moving whole row groups over (a
   *bulk append*, used when there is at least one full row group and no
   deletes) or by appending them row by row;
3. record an `INSERT_TUPLE` entry in the undo buffer (`PushAppend`), so that
   commit can stamp the rows and rollback can remove them.

Three rules keep this step safe.

**Rule 1: bulk-inserted rows stay invisible until commit.** Every row carries
an insert ID, and `Sees()` (§18.2) decides from it whether a reader sees the
row. In a bulk append, step 2 moves row groups into the table *before* the
commit stamps the inserts and deletes. If those row groups were built with
`TransactionData::Unversioned()` (insert ID 0, which every transaction sees),
another reader could see the new rows during that short window. A transaction
that replaced all 250,000 rows of a table (delete all, insert all) could then
let a concurrent reader count 500,000 rows with only 250,000 distinct IDs. So
parallel and batch inserts use the inserting transaction's own ID, just like
small single-threaded inserts:

```cpp
// src/execution/operator/persistent/physical_insert.cpp:671-672 (PhysicalInsert::Combine)
// the transaction id keeps the rows invisible to other transactions until the commit
TransactionData tdata(DuckTransaction::Get(context.client, table.catalog));
```

The rows stay invisible until the commit stamps them with a commit timestamp.
`test/sql/parallelism/interquery/concurrent_reads_during_bulk_replace.test_slow`
runs such replacements while other threads count rows.

**Rule 2: a failed append removes what it added.** If step 2 fails (for
example out of memory), the rows and index keys already added are removed
again: `LocalTableStorage::AppendToTable` catches the error, calls
`DataTable::RevertFailedAppend`, and then `RevertAppendToIndexes`
(`src/storage/local_storage.cpp:199-223`). It returns the error instead of
throwing it. The commit then fails cleanly and can be retried
(`test/sql/storage/commit_out_of_memory_reverts_append.test`).

**Rule 3: when it cannot be undone, the database is invalidated.** Some
failures cannot be cleaned up. An out-of-memory error in the *middle* of an
index insert is one example. Rollback cannot see these half-written keys,
because no undo entry exists yet, and a later checkpoint would write them to
disk. So `Flush` splits errors into two kinds:

```cpp
// src/storage/local_storage.cpp:633-680 (elided)
table.AppendLock(transaction, append_state);
// until PushAppend, a returned error means the shared state was restored; anything thrown invalidates
ErrorData error;
try {
	// ... steps 1 and 2 ...
	if (!error.HasError()) {
		// ...
		transaction.PushAppend(*storage.table_entry, NumericCast<idx_t>(append_state.row_start), append_count);
	}
} catch (std::exception &ex) {
	// the shared table or indexes may be partially modified: no checkpoint may persist them
	ValidChecker::Invalidate(table.db.GetDatabase(), "Failed to flush table " + table.GetTableName() +
	                                                     " during commit: " + ErrorData(ex).RawMessage());
	throw;
}
```

An error that is *returned*, such as a primary-key conflict, means the shared
state was put back, so the commit just fails. An error that is *thrown*
invalidates the whole database: every later query fails until the database is
reopened, and the on-disk file keeps its last good state. To make this safe,
`ART::Append` returns errors from evaluating index expressions instead of
throwing them (`src/execution/index/art/art.cpp:624-638`). And `StorageManager::WALStartCheckpoint` checks
again for invalidation after it takes the commit lock
(`src/storage/storage_manager.cpp:265-268`), so a checkpoint that was waiting
cannot write the bad state. `test/sql/transactions/test_commit_oom_ghost_keys.test`
makes a `COMMIT` run out of memory while it inserts 20,000 long keys into a
primary-key index. The commit fails, and after a restart the keys are gone.

## 18.4 The Write-Ahead Log: durability, not isolation — a distinct mechanism

Everything in §18.3 is an **in-memory** mechanism. `UndoBuffer` gives a
transaction isolation and rollback while the `duckdb` process is running; none
of it, by itself, survives the process being killed. That is a completely
separate problem — **durability**, "once I've told the client `COMMIT`
succeeded, that change must survive a crash" — and DuckDB solves it with a
second, on-disk mechanism: the **Write-Ahead Log (WAL)**. It is easy, and a
real mistake, to conflate the two: the undo buffer is about *what different
concurrent transactions are allowed to see right now*; the WAL is about *what
gets reconstructed if the process dies before the main file is rewritten to
reflect a commit*. A transaction that never touches the WAL at all (nothing
committed yet) is still fully covered by `UndoBuffer`'s isolation guarantees;
conversely, once something *is* in the WAL, `UndoBuffer`'s job for that change
is done and durability is `wal_replay.cpp`'s problem, not the version chain's.

`WriteAheadLog` is the class, and its own doc comment says exactly this:

```cpp
// src/include/duckdb/storage/write_ahead_log.hpp:38-48 (elided)
enum class WALInitState { NO_WAL, UNINITIALIZED, UNINITIALIZED_REQUIRES_TRUNCATE, INITIALIZED };

//! The WriteAheadLog (WAL) is a log that is used to provide durability. Prior
class WriteAheadLog {
public:
	//! Initialize the WAL in the specified directory
	explicit WriteAheadLog(StorageManager &storage_manager, const string &wal_path, idx_t wal_size = 0ULL,
	                       WALInitState state = WALInitState::NO_WAL,
	                       optional_idx checkpoint_iteration = optional_idx());
	// ...
};
```

Mechanically, each `DuckTransaction::WriteToWAL` call (invoked while
committing, before the in-memory undo-buffer commit even finishes — see the
`storage->Commit(...)` / `undo_buffer.WriteToWAL(*wal, ...)` pairing in
`src/transaction/duck_transaction.cpp:234-271`) serializes every entry in that
transaction's `UndoBuffer` out to the log file, using the same
forward-iteration machinery as an ordinary commit
(`UndoBuffer::WriteToWAL` at `src/transaction/undo_buffer.cpp:187-191`, driven
by per-entry logic in `src/transaction/wal_write_state.cpp`), with the entry
formats themselves defined in `src/storage/serialization/serialize_wal.cpp`
(research file §7b). So the WAL entries and the undo-buffer entries describe
the same underlying facts (an insert of these rows, an update to this column,
a catalog change) — they are just two different serializations of the same
transaction, aimed at two different failure modes: process-still-alive
concurrent visibility (`UndoBuffer`) versus process-restarted durability
(the WAL, replayed from disk).

### `WALInitState`'s four states

The four-state enum captures exactly the lifecycle a WAL file goes through
around startup and replay, confirmed by tracing every assignment site in
`write_ahead_log.cpp` and `wal_replay.cpp`:

- **`NO_WAL`** — there is no WAL backing this object at all. This is the state
  a freshly-created `WriteAheadLog` starts in by default (it's the default
  constructor argument). It is also the state of the new WAL that a
  checkpoint creates. `StorageManager::WALStartCheckpoint` closes the main
  WAL and opens an empty second WAL file (`<db>.wal.checkpoint`) in the
  `NO_WAL` state (`src/storage/storage_manager.cpp:309`). Commits that happen
  while the checkpoint runs go to this second file. When the checkpoint is
  done, `WALFinishCheckpoint` either deletes the old WAL (nothing was
  written to the second file) or moves the second file over it
  (`src/storage/storage_manager.cpp:313-342`). The old WAL's *contents* are
  gone either way, because they are now durably in the main file (see the
  cross-reference below).
- **`UNINITIALIZED`** — a WAL file exists on disk and `wal_replay.cpp`
  successfully replayed every entry in it up to a clean end-of-file (its
  `all_succeeded` flag was true, see `src/storage/wal_replay.cpp:704`). The
  file's on-disk bytes are exactly right as they are; nothing needs to be
  trimmed before the writer can start appending new entries to it — it just
  hasn't been opened for writing yet (lazily, on first actual write).
- **`UNINITIALIZED_REQUIRES_TRUNCATE`** — either (a) replay found a partial,
  truncated entry at the tail of the WAL file — the classic signature of a
  crash that happened mid-write, before the process could even finish
  appending one log record — and the file needs to be trimmed back to the
  last fully-written entry before anything new can be appended after it; or
  (b) `Truncate()` was called before the underlying file writer had ever
  actually been opened, so the truncation itself has to be deferred and
  remembered until `Initialize()` finally opens the file
  (`src/storage/write_ahead_log.cpp:77-89`).

  Case (a) covers more than a torn last entry. A transaction writes its WAL
  entries and then a `WAL_FLUSH` marker. Only entries before a `WAL_FLUSH`
  belong to transactions that finished committing. Replay notes where the
  last `WAL_FLUSH` ends (`last_wal_flush_end`) and stops replaying there, even
  if complete entries follow (`src/storage/wal_replay.cpp:660-677`). Such
  leftovers come from a commit that crashed before its flush. If the WAL holds
  no `WAL_FLUSH` at all, replay rolls back and returns
  `UNINITIALIZED_REQUIRES_TRUNCATE` at once (lines 641-645). Replay also drops
  the row-group blocks that those unfinished transactions had already written
  optimistically, so their rows do not come back after the crash (line 535;
  `test/sql/storage/wal/wal_torn_uncommitted_row_group.test`).
- **`INITIALIZED`** — the underlying `BufferedFileWriter` has been opened
  (lazily, on the first real write via `Initialize()`,
  `src/storage/write_ahead_log.cpp:50-66`) and the log is ready to have new
  entries appended.

```cpp
// src/storage/write_ahead_log.cpp:50-66
BufferedFileWriter &WriteAheadLog::Initialize() {
	if (Initialized()) {
		return *writer;
	}
	lock_guard<mutex> lock(wal_lock);
	if (!writer) {
		writer = make_uniq<BufferedFileWriter>(/* ... open for append ... */);
		if (init_state == WALInitState::UNINITIALIZED_REQUIRES_TRUNCATE) {
			writer->Truncate(storage_manager.GetWALSize());
		} else {
			storage_manager.SetWALSize(writer->GetFileSize());
		}
		init_state = WALInitState::INITIALIZED;
	}
	return *writer;
}
```

### Startup: `wal_replay.cpp`'s role

On opening a database file, the storage manager checks whether a WAL file
sits alongside it and, if so, hands it to `wal_replay.cpp` before the database
is considered open for queries. This is precisely the scenario durability
exists for: the process (or the machine) died at some point after a
transaction told its client `COMMIT` succeeded, but before a `CHECKPOINT` had
rewritten that change into the main `.duckdb` file. Replay opens a real
internal connection, replays each serialized WAL entry as if it were being
executed live (`WriteAheadLogDeserializer::ReplayEntry`, dispatching on the
same `WALType` tags `serialize_wal.cpp` wrote), and — critically — commits
that replay as its own transaction, so the replayed changes go through the
exact same `DuckTransaction::Commit` / `UndoBuffer::Commit` path described in
§18.3.2 that a live `COMMIT` would use. By the time replay finishes, the
in-memory state is exactly what it would have been had the process never
crashed; the `WALInitState` the replay hands back to the newly-constructed
`WriteAheadLog` (`UNINITIALIZED` or `UNINITIALIZED_REQUIRES_TRUNCATE`,
depending on whether the tail entry was intact) is exactly the bookkeeping
described just above.

Replay does not blindly trust the WAL file. The WAL is a separate file next to
the database, and it is replayed as soon as the database opens, before any
query runs. So a damaged or crafted WAL must not be able to crash the process.
One example is the `ALTER` entry. Its payload is read as a general `ParseInfo`,
and the file itself decides which subclass is built. Casting it to `AlterInfo`
without a check would, in a release build, let a WAL that holds some other
`ParseInfo` subclass make replay read and write the wrong memory. So
`ReplayAlter` checks the type first:

```cpp
// src/storage/wal_replay.cpp:975-980
void WriteAheadLogDeserializer::ReplayAlter() {
	auto info = deserializer.ReadProperty<unique_ptr<ParseInfo>>(101, "info");
	if (!info || info->info_type != ParseInfoType::ALTER_INFO) {
		throw DataCorruptionException("Corrupt WAL file: alter entry does not carry alter information");
	}
	auto &alter_info = info->Cast<AlterInfo>();
```

It also checks that an `ADD CONSTRAINT` entry really carries its constraint
(lines 981-989), and it validates the allocation sizes of index data in the
WAL. The test is the C++ file `test/sql/storage/wal_replay_invalid_info.cpp`.

### Cross-reference: this is what a CHECKPOINT is *for*

A `CHECKPOINT` — the subject of the checkpoint-manager discussion elsewhere in
this book's storage chapters — is the operation that lets a WAL be reset. Once
a checkpoint has rewritten every committed change into fresh blocks in the
main file, the WAL's accumulated entries no longer describe anything that
isn't *already* durably present in the main file on disk, so they can be
discarded rather than replayed on the next startup — which is exactly why a
fresh post-checkpoint `WriteAheadLog` starts life in the `NO_WAL` state
(`src/storage/storage_manager.cpp:309`, cited above) instead of carrying
forward whatever was in the old one. The relationship, stated plainly: the
undo buffer is what makes concurrent transactions correct *right now*, in
memory; the WAL is what makes a committed transaction survive a crash *before
the next checkpoint*; the checkpoint is what lets the WAL be thrown away once
its job is done.

Two rules make this hand-over safe.

**The WAL is always rotated at checkpoint start.** `WALStartCheckpoint` opens
the second WAL even when the main WAL is empty. A forced checkpoint
(`PRAGMA force_checkpoint`) can start with an empty WAL. If the rotation were
skipped and a transaction committed while that checkpoint ran, its entries
would go into the *old* WAL file, which is stamped with the old checkpoint
iteration number, while the database header gets the new number. On restart,
replay would see the mismatch ("file iteration 2, WAL iteration 1") and refuse
to open the database, and the only way out would be to delete the WAL and lose
the committed rows. If nothing commits meanwhile, `WALFinishCheckpoint` just
deletes the empty files
(`test/sql/storage/checkpoint/empty_wal_forced_checkpoint_generation.test_slow`).

**Indexes with buffered replays are bound before the checkpoint.** After a
restart, an index is loaded *unbound*: its key expressions are not bound until
a query uses it. WAL replay cannot bind indexes, so it *buffers* its index
inserts and deletes inside the unbound index. A checkpoint that wrote such an
index to disk without applying those buffered operations would lose index
entries. So `SingleFileStorageManager::CreateCheckpoint` first calls
`BindIndexesForCheckpoint` (`src/storage/storage_manager.cpp:816-877`):

- It binds every index that has buffered replays, applying the buffered
  operations. An explicit `CHECKPOINT` binds all indexes, so that their tables
  can also be vacuumed.
- If a bind fails (for example, the index uses a macro that no longer exists),
  an automatic checkpoint is skipped, the WAL is kept, and a warning is
  logged. An explicit `CHECKPOINT` fails with "Cannot CHECKPOINT: an index on
  table ... has buffered operations and cannot be bound".
- Binding happens *before* the checkpoint starts, because a failure after that
  point would invalidate the database.

The commit path cooperates. A large commit may normally skip writing its WAL
entries, because it is about to trigger a checkpoint that writes the data
directly. That shortcut is off while any unbound index has buffered replays,
since the checkpoint might be skipped
(`src/transaction/duck_transaction_manager.cpp:341-345`). On the index side,
`UnboundIndex::Bind` keeps owning its on-disk blocks until the buffered replays
have been applied. A bind that fails part-way leaves the unbound index whole,
so it can be retried (`src/execution/index/unbound_index.cpp:70-85`).

## 18.5 `meta_transaction.cpp`: coordinating across ATTACHed databases

DuckDB lets a single connection `ATTACH` several independent database files at
once and query across all of them in one `BEGIN ... COMMIT` block. Each
attached database has its own, fully independent `DuckTransactionManager` — its
own transaction-ID counter, its own visibility bound, its own undo-buffer
garbage collection. Something above all of them has to make one `BEGIN` /
`COMMIT` typed by the user correspond to the right set of per-database
transactions, and that something is `MetaTransaction`
(`src/transaction/meta_transaction.cpp`): one `MetaTransaction` per logical,
user-facing transaction, holding a map from `AttachedDatabase` to the
`DuckTransaction` (or other `Transaction` subclass — see §18.6) that this
logical transaction opened against it, lazily, the first time that database
is actually touched:

```cpp
// src/transaction/meta_transaction.cpp:60-86 (elided)
Transaction &MetaTransaction::GetTransaction(AttachedDatabase &db) {
	// ...
	auto entry = transactions.find(db);
	if (entry == transactions.end()) {
		auto &new_transaction = db.GetTransactionManager().StartTransaction(context);
		// ... record it, remember db in all_transactions (in touch-order) ...
		return new_transaction;
	} else {
		return entry->second.transaction;
	}
}
```

At `COMMIT` time, `MetaTransaction::Commit()` walks every per-database
transaction it opened and commits each one in turn:

```cpp
// src/transaction/meta_transaction.cpp:119-165 (elided)
ErrorData MetaTransaction::Commit() {
	ErrorData error;
	// ...
	// commit transactions in reverse order
	for (idx_t i = all_transactions.size(); i > 0; i--) {
		auto &db = all_transactions[i - 1].get();
		auto entry = transactions.find(db);
		// ...
		auto &transaction_manager = db.GetTransactionManager();
		auto &transaction_ref = entry->second;
		if (ValidChecker::IsInvalidated(db)) {
			error.Merge(ErrorData(IOException("%s", ValidChecker::InvalidatedMessage(db))));
			continue;
		}
		if (transaction_ref.state != TransactionState::UNCOMMITTED) {
			continue;
		}
		auto &transaction = transaction_ref.transaction;
		try {
			if (!error.HasError()) {
				// Commit the transaction.
				error = transaction_manager.CommitTransaction(context, transaction);
				transaction_ref.state = error.HasError() ? TransactionState::ROLLED_BACK : TransactionState::COMMITTED;
			} else {
				// Rollback due to previous error.
				transaction_manager.RollbackTransaction(transaction);
				transaction_ref.state = TransactionState::ROLLED_BACK;
			}
		} catch (std::exception &ex) {
			error.Merge(ErrorData(ex));
			transaction_ref.state = TransactionState::ROLLED_BACK;
		}
	}
	return error;
}
```

(The loop also skips two kinds of entry. A database that has been
invalidated, for example by a failed commit as in §18.3.5, adds an error
instead of committing. A transaction that is no longer `UNCOMMITTED` is left
alone.)

This is worth reading carefully, because it answers the "is this real
two-phase commit" question the source actually settles rather than leaving
ambiguous: **it is not.** There is no prepare phase, no vote, no durable
"intent to commit" record written before any database's `CommitTransaction`
actually runs. The loop simply commits each attached database's transaction
one at a time, in the reverse of the order those databases were first
touched, and *if a later commit in the loop fails*, everything from that point
onward in the loop gets rolled back instead of committed — but anything
**already committed earlier in the same loop stays committed.** A crash or a
commit failure partway through a multi-database transaction can leave some
attached databases with the transaction's changes durably committed and
others rolled back. This is a real, sourced limitation, not a guess.

What keeps this from being a routine correctness hazard in practice is a
separate, simpler rule enforced at write time, in `ModifyDatabase`:

```cpp
// src/transaction/meta_transaction.cpp:259-284 (elided)
void MetaTransaction::ModifyDatabase(AttachedDatabase &db, DatabaseModificationType modification) {
	// Temporary objects are private to the connection, so a read-only transaction may still write them
	if (IsReadOnly() && !db.IsTemporary()) {
		throw TransactionException("Cannot write to database %s - transaction is launched in read-only mode",
		                           db.GetName());
	}
	// ...
	if (db.IsSystem() || db.IsTemporary()) {
		// we can always modify the system and temp databases
		return;
	}
	if (!modified_database) {
		modified_database = &db;
		return;
	}
	if (&db != modified_database.get()) {
		throw TransactionException(
		    "Attempting to write to database %s in a transaction that has already modified database %s - a "
		    "single transaction can only write to a single attached database.",
		    db.GetName(), modified_database->GetName());
	}
}
```

**A single logical transaction can write to at most one non-system,
non-temporary attached database.** This means the obvious example of a
cross-database transaction —

```sql
ATTACH 'other.duckdb';
BEGIN;
INSERT INTO other.t ...;
INSERT INTO main.t ...;   -- writes to a SECOND attached database
COMMIT;
```

— is not actually a scenario DuckDB permits: the second `INSERT`, against a
different attached database than the first, throws the `TransactionException`
above the moment it's attempted, well before `COMMIT` is ever reached.
This matters, because it changes what "coordination" even
means here: `MetaTransaction` does coordinate reading consistently across
*any number* of attached databases in one transaction — each gets its own
snapshot, opened independently, and a query joining `main.orders` against
`other.customers` inside one transaction sees a coherent picture of both — but
the one place genuine cross-database atomicity would be needed (writing to
two databases and having both commits succeed or both fail together) is
exactly the place DuckDB sidesteps the problem by disallowing it outright,
rather than by implementing real two-phase commit for it. The reverse-order,
best-effort-abort-the-rest commit loop above is real and is what runs — it
just, in the common case the write restriction leaves standing, only ever
has one non-system database's `CommitTransaction` in it to actually run.

`MetaTransaction::Rollback()` is simpler and has no analogous subtlety: it
walks the same list in the same reverse order and unconditionally rolls every
per-database transaction back, since there is nothing to sequence around a
partial failure when the whole point is "undo everything."

Two smaller rules in `meta_transaction.cpp` complete the picture.

- A read-only transaction may write to the *temporary* database, since
  temporary objects are private to the connection. Other databases reject the
  write ("Cannot write to database ... - transaction is launched in read-only
  mode"); see the start of `ModifyDatabase` above.
- Since `DETACH` is allowed inside a transaction (§18.3.3), a name can be
  detached and then attached again to a *different* database file in the same
  transaction. Using that name again in the same transaction raises a
  `TransactionException` ("Database name ... was already used by a different
  database in this transaction - commit or roll back the transaction before
  using the name for another database"). This is a user-facing error, not an
  `InternalException` (the kind of error meant for bugs inside DuckDB itself).

## 18.6 Why `Transaction`/`TransactionManager` are abstract base classes

`DuckTransaction` and `DuckTransactionManager` — everything described in
§18.2–18.4 — are DuckDB's *own* storage engine's implementation of two
abstract interfaces:

```cpp
// src/include/duckdb/transaction/transaction.hpp (elided)
class Transaction {
public:
	DUCKDB_API Transaction(TransactionManager &manager, ClientContext &context);
	DUCKDB_API virtual ~Transaction();
	// ...
	virtual bool IsDuckTransaction() const { return false; }
	//! What this transaction sees. A transaction without MVCC state of its own sees everything.
	virtual SnapshotView GetSnapshotView() const {
		return SnapshotView(transaction_t(-1), VisibilityBound::IncludingUncommitted());
	}
};
```

```cpp
// src/include/duckdb/transaction/transaction_manager.hpp (elided)
class TransactionManager {
public:
	explicit TransactionManager(AttachedDatabase &db);
	virtual ~TransactionManager();

	//! Start a new transaction
	virtual Transaction &StartTransaction(ClientContext &context) = 0;
	//! Commit the given transaction. Returns a non-empty error message on failure.
	virtual ErrorData CommitTransaction(ClientContext &context, Transaction &transaction) = 0;
	//! Rollback the given transaction
	virtual void RollbackTransaction(Transaction &transaction) = 0;
	virtual void Checkpoint(ClientContext &context, bool force = false) = 0;
	// ...
};
```

Every place this chapter has been talking about "the transaction manager" or
"the transaction," the code is actually calling through these virtual
interfaces — `MetaTransaction::GetTransaction` calls
`db.GetTransactionManager().StartTransaction(context)`, not
`DuckTransactionManager::StartTransaction` directly. The reason this
indirection exists, rather than DuckDB's core simply hard-coding
`DuckTransaction` everywhere, is `ATTACH`: DuckDB's extension mechanism lets a
third-party extension register an entirely different storage engine behind
the same `ATTACH` syntax — a SQLite file, a Postgres connection, an Iceberg
catalog — each showing up to the rest of the system as an `AttachedDatabase`
with its own `Catalog` *and its own `TransactionManager` subclass*, which is
free to implement whatever transaction semantics that underlying engine
actually has (which may have no row-versioning at all, or may delegate
entirely to the remote engine's own transaction protocol). `MetaTransaction`,
the physical planner, and every query that reads or writes through an
attached database only ever touch the abstract `Transaction`/
`TransactionManager` interface, so none of that code needs a single `if
(IsDuckTransactionManager())` branch to work correctly against a completely
foreign storage engine attached alongside DuckDB's own tables in the same
query. `Transaction::GetSnapshotView()`'s default implementation — "a
transaction without MVCC state of its own sees everything" — is exactly the
fallback such a foreign engine gets for free if it doesn't want to implement
its own row-versioning at all.

## 18.7 Worked example: two sessions, one uncommitted write

Take a fresh angle from Query A and Query B (neither of which writes
anything) and follow three sessions against the same `orders` table.

**Session 1** begins a transaction and updates a row, but does not commit yet:

```sql
-- Session 1
BEGIN;
UPDATE orders SET status = 'shipped' WHERE orderkey = 42;
-- (not yet committed)
```

Per §18.2, `DuckTransactionManager::StartTransaction` handed Session 1's
`DuckTransaction` a fresh `start_time` (say `500`) and a fresh, high-range
`transaction_id` (say `2^62 + 7`), with `view.visibility_bound =
VisibilityBound::Before(500)`. The `UPDATE` runs exactly as walked through in
§18.3.1: the head of row 42's update chain now reads `'shipped'`, and a new
`UPDATE_TUPLE` entry, stamped with `version_number = 2^62 + 7`, sits in
Session 1's `UndoBuffer` holding the pre-image `'pending'`.

**Session 2**, a completely separate connection, now runs:

```sql
-- Session 2
SELECT status FROM orders WHERE orderkey = 42;
```

Session 2's own `DuckTransaction` (started fresh for this single statement,
if it's running outside an explicit `BEGIN`) gets some `start_time` greater
than `500` — it started after Session 1 did — but its `visibility_bound` is
`VisibilityBound::Before(<its own start_time>)`, and critically, **Session
1's write has not committed**, so it has no commit timestamp below anyone's
visibility bound at all; it only has the placeholder `transaction_id =
2^62+7`, and `2^62+7` is nowhere close to being `< visibility_bound` for any
real transaction (visibility bounds only ever compare against *commit*
timestamps in the low range, by construction — see §18.2's disjoint-range
argument). Walking row 42's update chain, `AppliesToTransaction` evaluates
`!view.Sees(2^62+7)`: Session 2's `transaction_id` is not `2^62+7` (that's
Session 1's, not Session 2's), so `Sees()` returns false, and
`AppliesToTransaction` returns true — Session 2 falls through past the head
and reads the link holding `'pending'`. **Session 2 sees `status = 'pending'`
— the value from before Session 1's still-open `UPDATE`** — not because
anything blocked it, but because the version chain handed it exactly the
version that existed at the moment its own snapshot was taken.

Now **Session 1 commits**:

```sql
-- Session 1
COMMIT;
```

Per §18.3.2, `DuckTransactionManager::GetCommitTimestamp()` hands out a real
commit timestamp (say `117` — deliberately less than 500 in the example
numbering only in the informal sense that it's drawn from the same small,
low counter; the actual value is whatever the manager's running commit
counter has reached), and `CommitState::CommitEntry` overwrites the undo
entry's `version_number` from `2^62+7` to `117`. Nothing else changes: the
head of the chain already said `'shipped'`.

**Session 3**, a brand-new connection that starts *after* Session 1's
`COMMIT` returned, runs:

```sql
-- Session 3
SELECT status FROM orders WHERE orderkey = 42;
```

Session 3's `visibility_bound` is `Before(<its start_time>)`, and its
start_time was assigned strictly after Session 1's commit finished, so its
visibility bound is comfortably past `117`. Walking the chain,
`AppliesToTransaction` for the link now stamped `117` evaluates
`!view.Sees(117)` — and `Sees(117)` is true (`117 < visibility_bound`), so
`AppliesToTransaction` returns **false**: Session 3 never even looks past the
head. **Session 3 sees `status = 'shipped'`.**

**What if Session 1 had instead rolled back?**

```sql
-- Session 1, alternate ending
ROLLBACK;
```

Per §18.3.3, `RollbackState::RollbackEntry`'s `UPDATE_TUPLE` case would run
`UpdateSegment::RollbackUpdate`, copying `'pending'` (the pre-image still
sitting in the undo entry) back into the head of the chain, and then
unsplicing the now-dead entry out of the list entirely. In that world,
*every* subsequent session — Session 2 during the transaction, and a
hypothetical Session 3 afterward — would see `status = 'pending'`, because as
far as the version chain is concerned, nothing ever happened: an aborted
transaction leaves no trace for anyone else to walk past.

### Diagram: three transactions against one shared visibility timeline

```
 commit-timestamp / transaction-id axis  ──────────────────────────────►

                         Session 1 (writer)
                         start_time=500, txn_id=2^62+7
                         ├── UPDATE row 42 (chain head: pending → shipped;
                         │   undo entry stamped 2^62+7, holds 'pending')
                         │
                         ▼ COMMIT — undo entry restamped 2^62+7 → 117
   ═══════════◄──────────╬═══════════════════════════════════════════►
   committed commit_ids  │  Session 2 (reader)
   ... 116, 117 ...      │  start_time=510, sees only commits < 510's bound
                         │  AND its own txn_id — 2^62+7 is neither ⇒ falls
                         │  through to 'pending' link, reads 'pending'
                         │
                         │                    Session 3 (reader, starts later)
                         │                    start_time=900, sees commit 117
                         │                    (117 < 900's bound) ⇒ stops at
                         │                    head, reads 'shipped'

   lowest_visibility_bound = min(start_time of every still-active txn)
   Once Session 2 (and anyone else predating commit 117) has finished,
   the 'pending' undo entry is no longer reachable by any active snapshot —
   cleanup_state.cpp / UpdateSegment::CleanupUpdateInternal unsplice it.
```

## 18.8 Summary

DuckDB gives every transaction a private, consistent snapshot rather than
making readers and writers wait on locks for one another. That snapshot is
two cheap integers — a `start_time` and a `transaction_id` drawn from a
disjoint high range — compared against row-version stamps with a single
inequality (`SnapshotView::Sees`). The mechanism that actually makes old
versions available to reconstruct is `UndoBuffer`: a per-transaction,
append-only log of typed entries (`INSERT_TUPLE`, `DELETE_TUPLE`,
`UPDATE_TUPLE`, `CATALOG_ENTRY`, `SEQUENCE_VALUE`, `ATTACHED_DATABASE`) that
gets walked forward to commit (stamp a real timestamp, change nothing else),
walked backward to roll back (physically undo each entry's effect, newest
first), and eventually walked forward once more, long after the fact, to
garbage-collect entries no active transaction's `lowest_visibility_bound`
could possibly still need. At commit, a transaction's inserted rows move from
its local storage into the shared table in a way that stays invisible until
the commit stamp, and that either reverts cleanly or invalidates the database
when it fails (§18.3.5). This in-memory mechanism is isolation and
rollback; it is entirely separate from the on-disk Write-Ahead Log, which
exists purely so a committed-but-not-yet-checkpointed change survives a
crash, and whose four `WALInitState`s track exactly how much of a prior WAL
file's tail is trustworthy at startup. `MetaTransaction` coordinates one
logical transaction across every attached database it touches, but — a
correction worth keeping over a plausible-sounding guess — it does this
without real two-phase commit, relying instead on a hard rule that a single
transaction may write to only one non-system attached database at a time.
And underneath all of `DuckTransaction`/`DuckTransactionManager`, an abstract
`Transaction`/`TransactionManager` pair exists specifically so an extension
can attach a wholly different storage engine and participate in this same
framework on its own terms.
