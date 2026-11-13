# Appendix A — The Historical `libpg_query` Parser

## A.0 Scope note (read this first)

Chapter 3 documents the parser in this book's source checkout: branch
`v2.0-cyanoptera`, commit `cb5673685153` (`v2.0.0-1041-gcb567368515`). That is
a hand-written PEG parser. This appendix documents the parser it replaced: a
vendored, modified fork of PostgreSQL's own **bison/flex** grammar, wrapped and
repackaged by a tool called **`libpg_query`** [38].

Which DuckDB versions run which parser:

```
 v1.5.0 … v1.5.6   stable 1.5 line; v1.5.6 (2026-09-27) is the last one  → libpg_query (this appendix)
 v2.0.0 and later  v2.0.0 tagged 2026-10-20                              → native PEG parser (Chapter 3)
```

So this appendix describes **the parser of every stable DuckDB release up to
and including v1.5.6**. It is also, almost certainly, the parser that most
existing blog posts, conference talks, and Stack Overflow answers about "how
DuckDB parses SQL" describe.

On the main development line, `libpg_query` was deleted on 2026-04-20
(commit `b791d0b5f74` "remove libpg_query"). The PEG parser moved into core
in PR #22194, merged on 2026-04-25 as commit `8140496820f`. The 1.5 releases
were cut from a separate release branch. v1.5.0 to v1.5.2 came out before
that merge. v1.5.3 to v1.5.6 came out after it, but on the release branch, so
all of them kept `libpg_query`. v2.0.0 (2026-10-20) is the first release that
ships the PEG parser.

**A note on sourcing, up front, because this appendix's epistemics are
genuinely different from every other chapter's.** Every other chapter in this
book reads code that is in the DuckDB repository (https://github.com/duckdb/duckdb) at commit
`cb5673685153`. This appendix cannot do
that. At that commit there is no `libpg_query` code left at all: no
`third_party/libpg_query/` directory, no `PostgresParser` class, no
`src/parser/transform/` directory. (The one leftover is a stale sentence in
`src/README.md` that still says "DuckDB uses the parser of Postgres
(libpg_query)".) Everything code-level in this appendix instead comes from
**`git show <tag>:<path>`** against the very same local repository's history —
reading a file exactly as it existed at a named, public, tagged release,
without checking out or modifying the working tree. Concretely, this appendix
reads tag **`v1.5.5`** (2026-07-21), a patch release on the 1.5 release
branch. Every code excerpt below is cited as `git show v1.5.5:path/to/file`,
not as a live file in the current checkout. The later and final 1.5 release,
`v1.5.6`, has byte-identical copies of every source file this appendix quotes
(`postgres_parser.cpp`, `select.y`, `parser.cpp`, `transform_expression.cpp`,
`transform_columnref.cpp`, `transform_subquery.cpp`; checked with
`git diff v1.5.5 v1.5.6 -- <file>`), so every excerpt holds for v1.5.6 too.
This is a primary source — it's the actual historical compiler, not a
reconstruction — but it is a *different* primary source than the rest of the
book uses, and every citation says so explicitly. Where this appendix instead
draws on well-known public background about `libpg_query` as a project (who
wrote it, why it exists as a standalone thing), that's flagged as secondary,
general knowledge, separately from the verified-via-`git-show` claims.

## A.1 What `libpg_query` is, and why DuckDB adopted it

PostgreSQL's own SQL grammar is, by a wide margin, one of the most
battle-tested pieces of parsing code in existence: it has been extended,
fuzzed, and fixed against real-world SQL for over two decades, and it correctly
handles an enormous amount of genuinely gnarly SQL surface area — nested CTEs,
window function edge cases, the full `INTERVAL` literal grammar, dollar-quoted
strings, every historical operator-precedence quirk anyone has ever
complained about on a mailing list. Writing a new SQL grammar from scratch that
matches that coverage is a multi-year undertaking even before you get to
DuckDB's own dialect extensions.

`libpg_query` [38] (secondary/background knowledge: a project by
Lukas Fittl of pganalyze, originally built for SQL-analysis tooling — query fingerprinting,
`pg_stat_statements`-style normalization — that needs to actually parse
arbitrary Postgres SQL without linking against a running Postgres server) is a
tool and library that extracts Postgres's own bison/flex grammar and its
supporting C parse-tree code out of the Postgres server source tree and
repackages it as a standalone, embeddable library, decoupled from
`postmaster`, the executor, the catalog, and everything else that makes
Postgres a database rather than just a grammar. DuckDB forked this — the
vendored copy's own README, read directly from the historical source (primary,
verified via `git show`), says so in its own words:

```
# Modified Postgres Parser (as of tag v1.5.5, third_party/libpg_query/README.md:1-5)

This directory holds the core parser that is used by DuckDB. It is based on
the [Postgres parser](https://github.com/pganalyze/libpg_query), but has been
stripped down and generally cleaned up to make it easier to modify and extend.
```

The trade this bought DuckDB is exactly the one you'd expect: a mature,
extensively-exercised SQL grammar *for free*, at the cost of (a) being
structurally constrained to accept whatever shape Postgres's own grammar rules
allow, even where DuckDB's dialect diverges from Postgres's, and (b) needing a
whole extra layer of code whose only job is converting Postgres's own internal
parse-tree representation into DuckDB's own AST — because `libpg_query`
produces a Postgres parse tree, not a DuckDB one, and nothing downstream of the
parser in DuckDB has ever spoken Postgres's internal node types.

## A.2 The pipeline: five stages, one of which downstream code never sees

At the highest level, here is the shape of the whole historical pipeline, and
it's worth holding this diagram next to Chapter 3's own runtime-flow diagram
(§3.4) while reading, because the point of this whole appendix is really made
in one line at the bottom of it:

```
 SQL text
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│ third_party/libpg_query/scan.l  (flex-generated lexer)             │
│  transpiled at dev-time into src_backend_parser_scan.cpp,          │
│  checked into the tree as generated C++ — NOT regenerated at        │
│  every build                                                        │
└───────────────────────────┬───────────────────────────────────────┘
                             │ token stream (Postgres's own token types)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ third_party/libpg_query/grammar/*.y  (bison LALR grammar,           │
│  Postgres's own rules, lightly modified for DuckDB syntax)           │
│  transpiled at dev-time into src_backend_parser_gram.cpp,           │
│  checked into the tree as generated C++ — NOT regenerated at        │
│  every build either                                                  │
└───────────────────────────┬───────────────────────────────────────┘
                             │ PGNode*/PGList* tree — Postgres's OWN
                             │ internal AST representation (duckdb_
                             │ libpgquery::PGSelectStmt, PGColumnRef,
                             │ PGAExpr, PGJoinExpr, ...), not DuckDB's
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ PostgresParser::Parse(query)  (postgres_parser.cpp)                 │
│  thin C++ wrapper around pg_parser_init/pg_parser_parse;            │
│  success/parse_tree or error_message/error_location                 │
└───────────────────────────┬───────────────────────────────────────┘
                             │ (on success) PGList* of statements
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Transformer::TransformParseTree  (src/parser/transformer.cpp,       │
│  src/parser/transform/**/*.cpp — one file roughly per Postgres      │
│  node kind: transform_columnref.cpp, transform_select_node.cpp,     │
│  transform_join.cpp, transform_subquery.cpp, ...)                    │
│  walks the PGNode tree, dispatching on node.type, and hand-builds    │
│  the equivalent DuckDB AST node at every step                        │
└───────────────────────────┬───────────────────────────────────────┘
                             │
                             ▼
              unique_ptr<SQLStatement>  ◄── IDENTICAL TARGET TYPE to
        (pushed onto Parser::statements)     Chapter 3 §3.6's AST hierarchy
```

The last line is the entire point of this appendix, said plainly: **the old
parser and the new parser produce the exact same target AST.**
`SQLStatement`/`QueryNode`/`ParsedExpression`/`TableRef` and every one of their
subclasses in Chapter 3's §3.6 hierarchy (`SelectStatement`, `SelectNode`,
`ColumnRefExpression`, `FunctionExpression`, `JoinRef`, `SubqueryExpression`,
and the rest) are the *same C++ types*, defined in the *same*
`src/parser/expression/`, `src/parser/tableref/`, `src/parser/query_node/`
directories, whether they were built by this appendix's Transformer or Chapter
3's PEG transformer. Only the front end that produces them changed — which is
exactly why the Binder, the logical planner, the optimizer, the physical
planner, and the execution engine did not need one line changed when DuckDB
swapped parsers. (You can check this: the merge that swapped them, commit
`8140496820f`, changes no file under `src/planner/`, `src/optimizer/` or
`src/execution/` — `git diff --stat 8140496820f^1 8140496820f -- src/planner
src/optimizer src/execution` prints nothing.) Chapter 3 makes this same claim from the new-parser side
(§3.6); this appendix is where it gets verified from the old-parser side, with
real code from both ends of the pipeline.

One honest qualification: "the same
C++ types" does not mean "the same fields". The *classes* are the same, but
some of their *fields* changed between v1.5.x and v2.0, for reasons that have
nothing to do with the parser swap. The clearest example is names. In
v1.5.5, `ColumnRefExpression` stores `vector<string> column_names`
(`git show v1.5.5:src/include/duckdb/parser/expression/columnref_expression.hpp`,
line 34). In v2.0.0 and at this checkout it stores
`vector<Identifier> column_names`
(`src/include/duckdb/parser/expression/columnref_expression.hpp:60`).
`Identifier` (`src/include/duckdb/common/identifier.hpp:26`) is a small
wrapper around a string that keeps the original spelling but compares,
hashes and sorts *case-insensitively*. So the two parsers fill the same
kinds of node, in the same shape, but a v2.0 node is not byte-for-byte a
v1.5 node.

## A.3 The vendored grammar: what's checked in, and what gets (re)generated

Structurally, `third_party/libpg_query/` as of tag `v1.5.5` looks like this
(verified via `git show v1.5.5:third_party/libpg_query`):

```
third_party/libpg_query/
├── CMakeLists.txt
├── README.md                        the "Modified Postgres Parser" note quoted in §A.1
├── grammar/
│   ├── grammar.y                       top-level bison entry point
│   ├── statements/                     42 .y files — select.y, insert.y, create.y, ...
│   ├── types/                          return-type declarations per rule family
│   └── keywords/
├── include/                         Postgres's own node type headers, ported to C++
│   (nodes/parsenodes.hpp, nodes/pg_list.hpp, ...)
├── pg_functions.cpp                 helper/utility functions
├── postgres_parser.cpp              the thin C++ wrapper DuckDB calls into
├── scan.l                           flex lexer source (a modified copy of
│                                     Postgres's own src/backend/parser/scan.l)
├── src_backend_parser_gram.cpp      GENERATED from grammar.y by bison, checked in
├── src_backend_parser_scan.cpp      GENERATED from scan.l by flex, checked in
├── src_backend_parser_parser.cpp
├── src_backend_parser_scansup.cpp
├── src_backend_nodes_list.cpp
├── src_backend_nodes_makefuncs.cpp
├── src_backend_nodes_value.cpp
└── src_common_keywords.cpp
```

The `CMakeLists.txt` (verified, `git show v1.5.5:third_party/libpg_query/CMakeLists.txt`)
compiles a static library, `duckdb_pg_query`, directly out of the *generated*
`.cpp` files — `src_backend_parser_gram.cpp` and `src_backend_parser_scan.cpp`
are ordinary source files as far as the build is concerned, no different from
any hand-written `.cpp`. Regenerating them from the `.y`/`.l` sources is a
separate, deliberate, developer-invoked step — `scripts/generate_grammar.py`
(bison) and `scripts/generate_flex.py` (flex) — not something CMake runs on
every build. The vendored README documents the exact toolchain versions this
was tested against (`bison (GNU Bison) 2.3`, a specific flex build), which is
itself a hint at one of the tradeoffs covered in §A.6: pinning to specific
bison/flex versions is exactly the kind of fragile, environment-sensitive
build dependency a from-scratch hand-written parser doesn't have.

**This "checked-in generated code, regenerated only by an explicit script" shape
is structurally the same pattern Chapter 3 describes for the PEG parser's own
build pipeline** (§3.3: `.gram`/`.list` files → `build_grammar.sh` →
`inlined_grammar.hpp`, a C++ byte array compiled into the binary). Both
parsers keep their actual grammar-authoring format (bison `.y` grammar files
here; PEG `.gram` files there) out of the hot compile path, and both check in a
generated artifact that the ordinary build just compiles. The generators
differ (`bison`/`flex`, external, versioned tools vs. `inline_grammar.py`, a
DuckDB-owned Python script) but the shape — author grammar in one format,
generate a compiled-in artifact, never hand-edit the generated file — is the
same engineering instinct on both sides of the rewrite.

A real rule from the vendored grammar, to make "bison LALR grammar" concrete
rather than abstract — the `FROM` clause and a `JOIN ... ON` clause, verbatim
(as of tag `v1.5.5`, `third_party/libpg_query/grammar/statements/select.y`):

```yacc
# third_party/libpg_query/grammar/statements/select.y:1049-1062
from_clause:
			FROM from_list_opt_comma							{ $$ = $2; }
			| /*EMPTY*/								{ $$ = NIL; }
		;

from_list:
			table_ref								{ $$ = list_make1($1); }
			| from_list ',' table_ref				{ $$ = lappend($1, $3); }
		;

from_list_opt_comma:
			from_list								{ $$ = $1; }
			| from_list ','							{ $$ = $1; }
		;
```

```yacc
# third_party/libpg_query/grammar/statements/select.y:1310-1323
			| table_ref join_type JOIN table_ref join_qual
				{
					PGJoinExpr *n = makeNode(PGJoinExpr);
					n->jointype = $2;
					n->joinreftype = PG_JOIN_REGULAR;
					n->larg = $1;
					n->rarg = $4;
					if ($5 != NULL && IsA($5, PGList))
						n->usingClause = (PGList *) $5; /* USING clause */
					else
						n->quals = $5; /* ON clause */
					n->location = @2;
					$$ = n;
				}
```

The very next alternative in the same rule is the one a plain `JOIN` (with
no `INNER`/`LEFT`/... keyword) actually matches. Its comment explains why it
exists at all:

```yacc
# third_party/libpg_query/grammar/statements/select.y:1324-1338
			| table_ref JOIN table_ref join_qual
				{
					/* letting join_type reduce to empty doesn't work */
					PGJoinExpr *n = makeNode(PGJoinExpr);
					n->jointype = PG_JOIN_INNER;
					n->joinreftype = PG_JOIN_REGULAR;
					n->larg = $1;
					n->rarg = $3;
					if ($4 != NULL && IsA($4, PGList))
						n->usingClause = (PGList *) $4; /* USING clause */
					else
						n->quals = $4; /* ON clause */
					n->location = @2;
					$$ = n;
				}
```

(Query A's `JOIN` does not go through the `join_type` alternative with a
default value. `join_type` cannot be empty, so bison needs this separate
alternative, which hard-codes `PG_JOIN_INNER`. The PEG grammar in Chapter 3 says the same thing more
directly with an optional element, `JoinType?`.)

Read this the way Chapter 3's own README-quoted PEG grammar wants to be read
(§3.2), but notice how differently a bison rule *feels*: there's no ordered
choice, no packrat memoization concern, no worrying about which alternative
must be listed first. The grammar declares that `table_ref join_type JOIN
table_ref join_qual` is *a* valid `joined_table`, and bison's LALR table
generator — offline, at grammar-compile time, the same `generate_grammar.py`
step from §A.3's intro — resolves any ambiguity against every other rule in
the file using the shift/reduce precedence machinery Chapter 3's §3.1
describes in the abstract. The inline C code (`{ ... }`) inside the rule is
executed the moment the rule reduces, at parse time, and builds a `PGJoinExpr`
— note: a **Postgres** node type, `n->jointype`/`n->joinreftype`/`n->quals`
are all fields Postgres itself defined, decades before DuckDB existed. This is
the Postgres-native parse tree from §A.2's diagram; turning `PGJoinExpr` into
DuckDB's own `JoinRef` is a separate, later step (§A.5).

## A.4 Runtime parse flow: `Parser::ParseQuery`, historical version

Chapter 3 (§3.4) reproduces the current `Parser::ParseQuery` verbatim and
walks through it step by step. Here is the same function, same file, same
class, as it existed at tag `v1.5.5` — worth reading side by side with
Chapter 3's version, because the *shape* of the control flow changed
noticeably more than a casual "same function, different parser" description
would suggest:

```cpp
// as of tag v1.5.5, src/parser/parser.cpp, starting at line 221 (verified via git show;
// condensed: some comments and the extension retry loop are elided)
void Parser::ParseQuery(const string &query) {
	Transformer transformer(options);
	string parser_error;
	optional_idx parser_error_location;
	{
		// check if there are any unicode spaces in the string
		string new_query;
		if (StripUnicodeSpaces(query, new_query)) {
			// there are - strip the unicode spaces and re-run the query
			ParseQuery(new_query);
			return;
		}
	}
	{
		if (options.extensions) {
			// ... parser_override extension loop, essentially identical in
			// shape to the current version Chapter 3 §3.4 quotes ...
		}
		PostgresParser::SetPreserveIdentifierCase(options.preserve_identifier_case);
		bool parsing_succeed = false;
		{
			PostgresParser parser;
			parser.Parse(query);
			if (parser.success) {
				if (!parser.parse_tree) {
					// empty statement
					return;
				}
				// if it succeeded, we transform the Postgres parse tree into a list of
				// SQLStatements
				transformer.TransformParseTree(parser.parse_tree, statements);
				parsing_succeed = true;
			} else {
				parser_error = parser.error_message;
				if (parser.error_location > 0) {
					parser_error_location = NumericCast<idx_t>(parser.error_location - 1);
				}
			}
		}
		if (parsing_succeed) {
			// no-op — falls through to shared wrap-up code
		} else if (!options.extensions || !options.extensions->HasParserExtensions()) {
			throw ParserException::SyntaxError(query, parser_error, parser_error_location);
		} else {
			// split sql string into statements and re-parse using extension
			auto queries = SplitQueries(query);
			// ... per-statement retry loop, described below ...
		}
	}
	// ... assigns per-statement stmt_location/length ...
}
```

Two structural differences from Chapter 3's version are worth calling out
explicitly, because they're exactly the kind of detail that's easy to get
subtly wrong if you only remember "the old parser used Postgres's grammar" and
reconstruct the control flow from vibes:

1. **The old parser hands the *entire* query string to `PostgresParser::Parse`
   in one call, and gets back either total success (a `PGList*` of every
   top-level statement) or total failure (one `error_message`/`error_location`
   pair) — there's no per-statement loop on the success path.** This is a
   direct consequence of bison being a single state machine that consumes the
   whole token stream for the whole input in one pass; `pg_parser_parse`
   internally handles the `;`-separated multi-statement case (Postgres's own
   grammar has always supported a `stmtmulti` top-level production), and hands
   back a single `PGList` already split into per-statement `PGNode`s. The new
   PEG parser, by contrast, explicitly loops
   `while (token_iterator.Current())`, peeling one `TopLevelStatement` off the
   shared token stream at a time (Chapter 3 §3.4) — a deliberate design choice
   in the PEG world, needed because the new parser also wants the
   `TryParseExtensionStatement` fallback to be retriable *per statement*, not
   just once for the whole query.
2. **The old parser's extension-fallback strategy is coarser-grained: on
   *any* parse failure, it doesn't retry per-statement automatically — it
   re-splits the *entire original query string* into individual statements
   (`SplitQueries(query)`, a semicolon-aware string-splitting utility, not a
   token-stream operation) and re-parses each piece from scratch with a fresh
   `PostgresParser`, so that a parser extension gets a chance to claim
   whichever *individual* statement DuckDB's own grammar couldn't parse.** This
   is a real cost of the single-shot bison approach: because one bad
   statement fails the *entire* one-shot parse (bison doesn't checkpoint
   partial progress across statement boundaries the way the new parser's
   explicit token-iterator loop does), recovering enough information to let an
   extension take over means re-parsing from character position zero, one
   statement at a time, rather than resuming a shared token cursor exactly
   where the failure occurred. The new PEG parser's `TryParseExtensionStatement`
   (Chapter 3 §3.4, step 4) is comparatively cheap by contrast — it operates on
   the *same* already-tokenized stream, from exactly the point of failure,
   with no re-lexing.

Everything else about the control flow — the parser-extension override
protocol taking priority when installed, `StripUnicodeSpaces` running before
any parsing at all, the eventual `ParserException::SyntaxError` on total
failure — is the same idea in both versions, differing mainly in mechanism
rather than intent.

One small behavior of the old parser became a reference point for the new
one. According to the message of commit `a15352e5994` (PR #28597, Chapter 3 §3.4), in
v1.5.6 the text of each statement in a multi-statement string started right
after the previous statement's `;`. So a comment *before* a statement belonged
to that statement. v2.0.0 lost this: a leading comment was dropped from a
statement's text. That commit restores the v1.5.6 behavior on purpose.

**`PostgresParser` itself**, the thin wrapper that actually calls into the
generated bison/flex code (verified, `git show v1.5.5:third_party/libpg_query/postgres_parser.cpp`):

```cpp
// as of tag v1.5.5, third_party/libpg_query/postgres_parser.cpp (full file, minus namespace close)
void PostgresParser::Parse(const std::string &query) {
	duckdb_libpgquery::pg_parser_init();
	duckdb_libpgquery::parse_result res;
	pg_parser_parse(query.c_str(), &res);
	success = res.success;

	if (success) {
		parse_tree = res.parse_tree;
	} else {
		error_message = std::string(res.error_message);
		error_location = res.error_location;
	}
}
```

`pg_parser_init()`/`pg_parser_parse()` are the actual generated-bison/flex
entry points — this is the boundary where DuckDB's own C++ hands off to code
that is, structurally, still Postgres's own parser, wrapped in the
`duckdb_libpgquery` namespace (per the vendored README's own description,
§A.1). `parse_result` (a plain struct: `success`/`parse_tree`/`error_message`/
`error_location`) is `libpg_query`'s own return-value convention, not
something DuckDB invented — it's the shape `libpg_query` as a general-purpose
library was designed to hand back to *any* caller, DuckDB included.

## A.5 The transform layer: one dispatch switch, then hand-written functions per node kind

Where Chapter 3's transformer is table-driven — a generated rule-name-keyed
lookup table dispatching to `Initialize*`/`Finalize*` trampoline pairs (§3.5) —
the historical transformer is a **hand-written C++ `switch` statement over
Postgres's own `PGNodeTag` enum**, one `case` per Postgres node type DuckDB's
grammar can produce, calling into one `Transform*` function per case. This is
the direct structural consequence of "the front end hands back a
Postgres-native tree": every node in that tree carries a Postgres node-type
tag, and the only way to know which `Transform*` function applies is to switch
on it.

The expression-level dispatch, verbatim (as of tag `v1.5.5`,
`src/parser/transform/expression/transform_expression.cpp:27-86`):

```cpp
unique_ptr<ParsedExpression> Transformer::TransformExpression(duckdb_libpgquery::PGNode &node) {
	auto stack_checker = StackCheck();

	switch (node.type) {
	case duckdb_libpgquery::T_PGColumnRef:
		return TransformColumnRef(PGCast<duckdb_libpgquery::PGColumnRef>(node));
	case duckdb_libpgquery::T_PGAConst:
		return TransformConstant(PGCast<duckdb_libpgquery::PGAConst>(node));
	case duckdb_libpgquery::T_PGAExpr:
		return TransformAExpr(PGCast<duckdb_libpgquery::PGAExpr>(node));
	case duckdb_libpgquery::T_PGFuncCall:
		return TransformFuncCall(PGCast<duckdb_libpgquery::PGFuncCall>(node));
	case duckdb_libpgquery::T_PGBoolExpr:
		return TransformBoolExpr(PGCast<duckdb_libpgquery::PGBoolExpr>(node));
	case duckdb_libpgquery::T_PGTypeCast:
		return TransformTypeCast(PGCast<duckdb_libpgquery::PGTypeCast>(node));
	case duckdb_libpgquery::T_PGCaseExpr:
		return TransformCase(PGCast<duckdb_libpgquery::PGCaseExpr>(node));
	case duckdb_libpgquery::T_PGSubLink:
		return TransformSubquery(PGCast<duckdb_libpgquery::PGSubLink>(node));
	// ... T_PGCoalesceExpr, T_PGNullTest, T_PGResTarget, T_PGParamRef,
	//     T_PGNamedArgExpr, T_PGSQLValueFunction, T_PGSetToDefault,
	//     T_PGCollateClause, T_PGIntervalConstant, T_PGLambdaFunction,
	//     T_PGSingleArrowFunction, T_PGAIndirection, T_PGPositionalReference,
	//     T_PGGroupingFunc, T_PGAStar, T_PGBooleanTest, T_PGMultiAssignRef,
	//     T_PGString cases elided — same shape, one line each ...
	default:
		throw NotImplementedException("Expression type %s (%d)", NodetypeToString(node.type), (int)node.type);
	}
}
```

Contrast this directly with Chapter 3 §3.5's `TransformShowTables` example:
there, the dispatch key is a **grammar rule name** (`"ShowTables"`), looked up
in a table the build script generated from `grammar_types.yml`; here, the
dispatch key is a **Postgres C struct tag** (`T_PGColumnRef`), matched in a
hand-written `switch` that a human wrote and must keep in sync by hand every
time DuckDB's grammar starts producing a Postgres node type it didn't
previously handle. Both approaches solve the same problem — "given a matched
piece of syntax, find the function that knows how to build the right AST node
for it" — but one is generated from a declarative mapping file and one is
maintained by hand as a growing `switch` statement. This hand-maintenance
burden (adding a new `case` by hand, in a file that by `v1.5.5` already has
dozens of them, every time the grammar grows) is one of the concrete costs
§A.6 attributes to the old architecture.

One concrete `Transform*` function, chosen because Chapter 3 doesn't have a
direct analogue for it and because it's short enough to show in full — this
is where a `PGColumnRef` (Postgres's parse-tree representation of something
like `o.custkey`) becomes DuckDB's own `ColumnRefExpression` (as of tag
`v1.5.5`, `src/parser/transform/expression/transform_columnref.cpp:114-136`):

```cpp
unique_ptr<ParsedExpression> Transformer::TransformColumnRef(duckdb_libpgquery::PGColumnRef &root) {
	auto fields = root.fields;
	auto head_node = PGPointerCast<duckdb_libpgquery::PGNode>(fields->head->data.ptr_value);
	switch (head_node->type) {
	case duckdb_libpgquery::T_PGString: {
		if (fields->length < 1) {
			throw InternalException("Unexpected field length");
		}
		vector<string> column_names;
		for (auto node = fields->head; node; node = node->next) {
			column_names.emplace_back(PGPointerCast<duckdb_libpgquery::PGValue>(node->data.ptr_value)->val.str);
		}
		auto colref = make_uniq<ColumnRefExpression>(std::move(column_names));
		SetQueryLocation(*colref, root.location);
		return std::move(colref);
	}
	case duckdb_libpgquery::T_PGAStar: {
		return TransformStarExpression(PGCast<duckdb_libpgquery::PGAStar>(*head_node));
	}
	default:
		throw NotImplementedException("ColumnRef not implemented!");
	}
}
```

Two things worth pointing at directly. First: `root.fields` is a Postgres
`PGList` — a Postgres-native, hand-rolled linked list of `PGString` value
nodes, one per dotted-name segment (`o` then `custkey`) — and the function's
entire job is walking that Postgres-native list structure and re-packaging it
as a plain `vector<string> column_names`. Second, and this is the whole
appendix's thesis made concrete in eleven lines of real code: the type being
constructed at the end, `ColumnRefExpression`, constructed from exactly a
`vector<string>` of dotted-name segments, with no notion yet of which segment
is a table alias versus a struct field — **is the same class, filled in the
same way, that Chapter 3 §3.6 describes the PEG parser building for the exact
same SQL fragment.** A `ColumnRefExpression` built here in `v1.5.5` and one
built by the PEG transformer in this book's `v2.0-cyanoptera` checkout are
instances of the same C++ class, holding the same list of name parts in the
same order. The one difference is the element type of that list: a plain
`string` in v1.5.5, a case-insensitive `Identifier` in v2.0 (see the callout
in §A.2). That change came from a separate refactor, not from the parser
swap.

## A.6 Old vs. new: a balanced tradeoff, not a strict improvement in every axis

Chapter 3 §3.1 explains, from first principles, what a PEG parser trades away
(guaranteed non-ambiguity of "which alternative wins," at the cost of
backtracking and the packrat machinery needed to make that affordable) and
what it gains (no dependency on an external grammar-compiler toolchain, a
grammar that can also drive autocomplete and SQL formatting). Framed as an
explicit list, here is the same tradeoff from the old architecture's side:

**What `libpg_query` bought DuckDB, genuinely, for free:**
- Decades of Postgres's own grammar maturity and real-world dialect coverage,
  inherited rather than re-derived — the exact argument §A.1 opens with.
- A grammar that had already been fuzzed, extended, and battle-tested against
  an enormous body of real SQL, long before DuckDB's own SQL surface area grew
  to the size it's at today.
- LALR parsing's genuine algorithmic guarantee: linear time, no backtracking,
  by construction — a property Chapter 3 (§3.1) is explicit that the new PEG
  approach only recovers via the *separate* engineering effort of packrat
  memoization, not for free the way an LALR table gives it.

**What it cost, and — this next part is the book's own reasoning, not a
DuckDB team statement found in the source, and is flagged as such — what
plausibly motivated the eventual rewrite:**
- **Grammar-shape lock-in.** A forked bison grammar is still fundamentally
  Postgres's grammar, with DuckDB-specific rules patched in around the edges.
  Where DuckDB's own SQL dialect diverges meaningfully from Postgres's (its own
  `PIVOT`/`UNPIVOT`, `ASOF JOIN`, `POSITIONAL JOIN`, dot-indirection into
  structs and lists, and more — several of which do appear as bolted-on
  productions in the `select.y` excerpts in §A.3, e.g. `PG_JOIN_ASOF`/
  `PG_JOIN_POSITION` as extra alternatives spliced into `joined_table`), every
  addition has to be expressed as *another* alternative wedged into an
  LALR grammar that was never designed with that syntax in mind, and checked
  against Postgres's own precedence/shift-reduce table for new conflicts.
  A grammar designed DuckDB-first, rather than retrofitted onto Postgres's,
  has no such constraint.
- **Maintenance burden of a vendored fork.** `third_party/libpg_query/` is,
  concretely, a second copy of a large piece of someone else's parser (itself
  extracted from a third project, Postgres), living inside DuckDB's tree,
  requiring its own README (quoted in §A.1) explaining what's been changed and
  why. Every one of those three parties' changes over time is a potential
  source of drift to track by hand.
- **A heavier, less modern build dependency.** The vendored README's own
  build instructions (§A.3) pin specific `bison`/`flex` versions tested on a
  specific developer's Macbook — exactly the kind of environment-sensitive,
  external-tool build step that's easy to get right once and painful to keep
  reproducible across contributors' machines and CI matrices indefinitely, in
  a way a self-contained Python-script-driven grammar generator (§3.3's
  `build_grammar.sh`/`inline_grammar.py`) is not.
- **One grammar, multiple consumers.** Chapter 3 notes (§3.2/§3.3) that the
  new PEG grammar started life *inside* the `autocomplete` extension before
  being "moved into core" — meaning the same `.gram` grammar that parses a
  query is also positioned to drive autocomplete suggestions and, plausibly,
  SQL-formatting tooling, from one grammar definition. A vendored bison
  grammar, by contrast, has no comparable story: nothing about
  `libpg_query`'s output is naturally reusable for "what could come next in
  this partially-typed query," so autocomplete-style features would need
  separate, duplicated logic rather than being able to walk the same compiled
  grammar object the query parser itself uses.

None of these four points is drawn from a DuckDB blog post or commit message
stating "this is why we rewrote the parser" — no such source was found or
consulted for this appendix (recall §A.0's sourcing note: this appendix's
primary source is `git show`-read historical code, not DuckDB's own stated
rationale for retiring it). They are this book's own reasoning about what the
historical architecture's own shape, read directly, made structurally
awkward — offered as informed inference, not as a confirmed institutional
motive. (The merge commit itself was read: `git show -s
8140496820f`. Its message lists *what* the change did — remove the
`libpg_query` library, remove the transformer for `PGNode`s, move the PEG
parser and its transformer from the `autocomplete` extension into
`src/parser/peg/` — but it does not say *why*.)

## A.7 Tracing Query A through the historical pipeline

Query A, the book's running join/aggregate/order/limit example:

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

Fed to `PostgresParser::Parse`, this reduces in one bison/flex pass to a
single Postgres `PGSelectStmt` node (`stmt.op == PG_SETOP_NONE`, per the
`TransformSelectInternal` switch shown below), with its clauses populated as
separate `PGList*`/`PGNode*` fields — `targetList`, `fromClause`,
`whereClause`, `groupClause`, and the sort/limit fields consumed by
`TransformModifiers`. Here's which historical grammar rule and which
`Transform*` function is responsible for each fragment, deliberately laid out
the same way Chapter 3 §3.7's table is, for direct comparison:

| Query fragment | Historical grammar rule(s) | `Transform*` function | AST node produced |
|---|---|---|---|
| `SELECT o.custkey, SUM(l.extendedprice) AS total` | `target_list` (select.y), each item a `ResTarget` wrapping an `a_expr` | `TransformExpressionList` over `stmt.targetList`, each entry via `TransformResTarget` → `TransformExpression` | `select_list` = `[ColumnRefExpression(["o","custkey"]), FunctionExpression("sum", [ColumnRefExpression(["l","extendedprice"])])` aliased `"total"`] — **identical to Chapter 3 §3.7's row for the same fragment** |
| `FROM orders o JOIN lineitem l ON o.orderkey = l.orderkey` | `from_clause <- FROM from_list_opt_comma`; `joined_table <- table_ref JOIN table_ref join_qual` (§A.3, verbatim, `select.y:1324-1338`) | `TransformFrom` → `TransformTableRefNode` → `TransformJoin` (`src/parser/transform/tableref/transform_join.cpp`, verified) | `from_table` = a `JoinRef`, `type = JoinType::INNER` (no explicit `join_type` token, so the separate no-`join_type` alternative sets `PG_JOIN_INNER`), `left`/`right` = `BaseTableRef`s for `orders`/`lineitem` |
| `o.orderkey = l.orderkey` (the `ON` qualifier) | `join_qual <- ON a_expr`; the `a_expr` itself matches `a_expr '=' a_expr` | `TransformJoin` sets `result->condition = TransformExpression(root.quals)`; that expression transforms via `TransformAExpr` → `TransformBinaryOperator` (`transform_operator.cpp`, verified) | `ComparisonExpression(COMPARE_EQUAL, ColumnRefExpression(["o","orderkey"]), ColumnRefExpression(["l","orderkey"]))` — this becomes the `JoinRef`'s `condition` |
| `WHERE l.shipdate < DATE '1998-01-01'` | `where_clause <- WHERE a_expr`; `DATE '...'` matches the `ConstTypename`/typed-literal family of rules (`select.y` has several `makeTypeCast(...)` call sites for this — e.g. line 2264, `a_expr TYPECAST Typename`) | `TransformSelectInternal` sets `result.where_clause = TransformExpression(stmt.whereClause)`, dispatching through `TransformAExpr` for the `<` and through `TransformTypeCast`/`TransformConstant` for the date literal | `ComparisonExpression(COMPARE_LESSTHAN, ColumnRefExpression(["l","shipdate"]), ConstantExpression(Value::DATE("1998-01-01")))` |
| `SUM(l.extendedprice)` | `func_expr`/`func_application` (bison's function-call family of rules) | `TransformFuncCall` (`transform_function.cpp`) | `FunctionExpression("sum", [ColumnRefExpression(["l","extendedprice"])])` — nothing here marks it aggregate vs. scalar; exactly as Chapter 3 §3.7 notes for the new parser, that's a catalog lookup deferred to the Binder |
| `GROUP BY o.custkey` | `group_clause <- GROUP_P BY group_by_list_opt_comma` (verbatim, §A.3-adjacent excerpt in `select.y:893-900`) | `TransformGroupBy` (`src/parser/transform/helpers/transform_groupby.cpp`, verified — builds the `GroupByNode`/`GroupingSet` machinery, deduplicating repeated expressions via a `GroupingExpressionMap`) | `groups` populated with one group-by expression wrapping `ColumnRefExpression(["o","custkey"])` |
| `ORDER BY total DESC` | `sort_clause <- ORDER BY sortby_list` | `TransformOrderBy` (`src/parser/transform/helpers/transform_orderby.cpp`, verified in full above) | an `OrderByNode` referencing the `total` alias, `OrderType::DESCENDING` |
| `LIMIT 10` | `limit_clause <- LIMIT select_limit_value` | `TransformModifiers` (`src/parser/transform/statement/transform_select_node.cpp`, verified) builds a `LimitModifier` from `stmt.limitCount` | a `LimitModifier` holding `ConstantExpression(10)` |

The resulting tree — one `SelectStatement` wrapping one `SelectNode`, with
`select_list`/`from_table`/`where_clause`/`groups`/`modifiers` populated
exactly as above — **is the identical AST shape Chapter 3 §3.7 derives for
this same query**, down to the individual field values. Nothing in this
appendix's trace disagrees with Chapter 3's; it's the same destination reached
by a structurally different route (a single bison reduction pass plus a
hand-dispatched `switch`-driven transform walk, versus a packrat-memoized PEG
match plus a generated-trampoline transform walk).

## A.8 Tracing Query B through the historical pipeline: the subquery, again just syntax

Query B, the book's running correlated-subquery example:

```sql
SELECT c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey) AS n_orders
FROM customer c;
```

The outer query parses the same way Query A's outer shape did: `select_list`
gets a `ColumnRefExpression(["c","name"])` and one more `ResTarget` for the
`(SELECT ...) AS n_orders` expression; `from_table` becomes a
`BaseTableRef("customer", alias "c")`. The interesting part, exactly as in
Chapter 3 §3.8, is the second `select_list` entry. In Postgres's own grammar, a
parenthesized subquery used as a value expression is a **`SubLink`** node
(`PGSubLink`) — Postgres's umbrella node type for every "a query used where an
expression is expected" form: scalar subqueries, `EXISTS (...)`, `x IN
(SELECT ...)`, and `x = ANY (SELECT ...)` are all `PGSubLink`s, distinguished
by a `subLinkType` tag.

`Transformer::TransformExpression`'s dispatch switch (§A.5) routes
`T_PGSubLink` to `TransformSubquery`
(`src/parser/transform/expression/transform_subquery.cpp`, verified in full
above). For a bare `(SELECT ...)` used as a scalar value — exactly Query B's
shape — the relevant branch is:

```cpp
// as of tag v1.5.5, src/parser/transform/expression/transform_subquery.cpp:66-72
case duckdb_libpgquery::PG_EXPR_SUBLINK: {
	// return a single scalar value from the subquery
	// no child expression to compare to
	subquery_expr->subquery_type = SubqueryType::SCALAR;
	subquery_expr->subquery = TransformSelectStmt(*root.subselect);
	D_ASSERT(subquery_expr->subquery);
	break;
}
```

`root.subselect` is itself a full nested `PGSelectStmt` — the entire inner
query `SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey`, parsed by
the *same* single bison pass that parsed the outer query (bison's grammar is
recursive by nature: `a_expr` productions that admit a parenthesized
`SelectStmt` are just more grammar rules, resolved as part of the very same
LALR table-driven parse, not a separate invocation). `TransformSelectStmt`
recurses into exactly the same `TransformSelectInternal` machinery §A.7 just
walked through for Query A's outer query, producing its own complete
`SelectNode`: `select_list = [FunctionExpression("count", [StarExpression])]`,
`from_table = BaseTableRef("orders", alias "o")`, `where_clause =
ComparisonExpression(COMPARE_EQUAL, ColumnRefExpression(["o","custkey"]),
ColumnRefExpression(["c","custkey"]))`. The outer `SubqueryExpression` wraps
that as `subquery`, with `subquery_type = SubqueryType::SCALAR` and `child`
left null — **the identical `SubqueryExpression` shape Chapter 3 §3.8 derives
for the same query**, right down to which fields are populated and which are
left null.

And the same conclusion Chapter 3 §3.8 draws applies here with equal force,
for the identical reason: **nothing in this transform pass — old or new —
marks `c.custkey` inside the subquery's `where_clause` as a reference to the
*outer* query's scope.** `TransformColumnRef` (§A.5) built exactly the same
`ColumnRefExpression(["c","custkey"])` it would have built for any other
two-part dotted name; the fact that this particular occurrence needs to escape
the inner `SELECT`'s own `FROM orders o` and resolve against the outer
`FROM customer c` is a fact about *scope*, and scope is something neither
version of the parser has any representation for. That discovery is
Chapter 4's `BindContext` machinery, and Chapter 6's decorrelation logic built
on top of it — both entirely unaffected by which parser produced the tree
they operate on.

## A.9 The error path, historically

The old parser surfaces syntax errors through the same top-level exception
type the new parser does — `ParserException::SyntaxError`, carrying a message
and (when available) a query location — but the *source* of that message and
location is Postgres's own error-reporting convention (`ereport`,
`errposition()`) rather than a hand-tracked "furthest token index reached"
counter (Chapter 3 §3.9's `state.GetMaxTokenIndex()`). The vendored bison
grammar's own error productions call into Postgres's `scanner_yyerror`, which
formats a message and records a byte offset via `errposition()`; `pg_parser_parse`
propagates that offset out through `parse_result.error_location`, and
`PostgresParser::Parse` copies it into `error_message`/`error_location`
(§A.4's verbatim listing).

The wording is, reassuringly, the *same* wording — DuckDB's own real test
suite confirms this directly (verified, `git show
v1.5.5:test/sql/aggregate/grouping_sets/grouping.test:189`, among many other
`.test` files across the `v1.5.5` tree using the identical phrase):

```
Parser Error: syntax error at or near ")"
```

This is the same `"syntax error at or near \"...\""` phrasing Chapter 3 §3.9's
`enable_peg_parser.test` example checks for the new parser — strong evidence
(from the test suite itself, not from any design document) that preserving the
exact wording of parser error messages was treated as a compatibility
requirement across the rewrite, presumably because a large number of existing
`.test` files across the whole suite assert on this exact string and could not
all be rewritten as part of swapping the underlying engine.

One genuinely old-parser-specific wrinkle, found directly in the historical
test suite rather than reconstructed, is worth flagging because it has no
clean equivalent in the new PEG error path: `test/sql/parser/pg_err_pos_reset.test`
(verified, `git show v1.5.5:test/sql/parser/pg_err_pos_reset.test`) is a
regression test for a bug where Postgres's own global `pg_err_pos` state
(the position `errposition()` last recorded) could leak stale data from one
parse into an unrelated later parse's error message, if the later error was
thrown via `errhint()` before `errposition()` ran again in that later parse.
The fix was to reset `pg_err_pos` inside `pg_parser_init`. This class of bug —
global, cross-invocation state inside a wrapped C library leaking between
logically unrelated parses — is a direct structural cost of the "wrap
somebody else's C parser, which was never designed to be re-entered cleanly
call after call" architecture; it has no real analogue in a parser (the new
PEG one) written from scratch in modern C++ with no vestigial global state to
reset between invocations.

## A.10 What crosses the boundary into Chapter 4 — and the reassurance this appendix exists to give

By the end of this pipeline, `Parser::statements` holds the exact same thing
Chapter 3 §3.10 describes: one `unique_ptr<SQLStatement>` per top-level
statement, built entirely out of the `ParsedExpression`/`TableRef`/`QueryNode`
vocabulary in Chapter 3 §3.6 — because, as this whole appendix has tried to
show with real, verbatim, `git show`-verified code rather than by assertion,
that vocabulary never changed in kind (only small details, such as names
becoming `Identifier`s, §A.2). Names are still raw strings or dotted-name
chains; functions are still undifferentiated by kind; subqueries still carry
no notion of which of their column references reach outside their own scope.
Every one of those open questions is still Chapter 4's job to close, exactly
as before.

**If you are working against a stable DuckDB release from `v1.5.6` back —
any `v1.5.x` or earlier — this appendix replaces Chapter 3 for you.** If you
run `v2.0.0` or later, Chapter 3 is the one that applies, and this appendix is
history. For a 1.5 release, the rest of the book applies to its overall
architecture, but not to every detail: the book's code is from the 2.0
development line, so smaller things (field types, optimizer passes, settings)
can differ from what your release has. Chapter 3's specific parser
mechanics (the PEG grammar files, the packrat cache, the generated-trampoline
transform pattern) are the one *whole-architecture* difference; substitute
this appendix for that chapter and every subsequent chapter —
the Binder (4), the logical plan (5), subquery decorrelation (6), the
optimizer (7) through join ordering (8), physical plan generation (9), vectors
and execution (10–14), sorting (15), the ART index (16), storage (17), and
transactions (18) — describes the same architecture your release actually
runs, because none of it has ever depended on which parser produced the
`SQLStatement` it starts from.
