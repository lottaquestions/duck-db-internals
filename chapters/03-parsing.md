# Chapter 3 — Parsing

## 3.0 Scope note (read this first)

This chapter documents the parser in this book's source checkout: branch
`v2.0-cyanoptera`, commit `cb5673685153` (`git describe --tags`:
`v2.0.0-1041-gcb567368515`). That parser is a **hand-written, native PEG
(Parsing Expression Grammar) parser**. It replaced DuckDB's older parser. The
older parser was built on a vendored fork of `libpg_query`, a tool that takes
PostgreSQL's bison/flex grammar and turns it into a standalone C library.

The switch happened on the main development line in April 2026. First,
`libpg_query` was deleted (commit `b791d0b5f74` "remove libpg_query",
2026-04-20). Then PR #22194 ("Replace PostgreSQL parser with PEG-based
parser") was merged on 2026-04-25. That merge moved the PEG parser out of the
`autocomplete` extension and into `src/parser/peg/`:

```
$ git log -1 --format='%h %ci %s' 8140496820f
8140496820f 2026-04-25 13:11:08 +0000 [WIP] Replace PostgreSQL parser with PEG-based parser (#22194)
```

v2.0.0 (tagged 2026-10-20) is the first stable release that ships the PEG
parser. So this chapter describes the parser of a released DuckDB, not only
of a development branch.

Which parser does a given DuckDB version use?

```
 v1.5.0 … v1.5.6   (stable 1.5 releases, last one 2026-09-27)  → libpg_query (Appendix A)
 v2.0.0 and later  (first tagged 2026-10-20)                   → native PEG parser (this chapter)
```

The 1.5 releases were cut from a separate release branch, so they kept
`libpg_query` even after the main line dropped it. v1.5.0 to v1.5.2 came out
before the April 2026 merge. v1.5.3 to v1.5.6 came out after it, but on the
release branch. At this checkout there is
no `libpg_query` code left in the tree at all. The only trace is one stale
sentence in `src/README.md`, which still says "DuckDB uses the parser of
Postgres". Appendix A covers the old parser, and most older external material
about "how DuckDB parses SQL" describes that old parser. Nothing downstream of
the parser — binder, logical plan, optimizer, physical plan, execution —
changed because of the rewrite. The AST types this chapter ends with (§3.6)
are exactly the types Chapter 4 (the Binder) consumes, whichever parser built
them.

## 3.1 What a PEG parser is, and why DuckDB switched to one

Almost every SQL-parsing explanation you've read assumes a **context-free
grammar (CFG)** parsed by a **bison/flex-style LALR parser** — the approach
`libpg_query` used, and the approach Postgres itself uses. It's worth being
precise about what that approach actually guarantees, because the PEG approach
DuckDB now uses trades away exactly one of those guarantees, and gains something
else in exchange.

**A CFG can be genuinely ambiguous.** A grammar rule like
`Expr -> Expr '+' Expr | Expr '*' Expr | NUMBER` admits more than one parse tree
for `1 + 2 * 3` unless you separately impose precedence and associativity rules.
An LALR parser generator (bison/yacc) resolves this at *grammar-compile time*: it
builds a finite-state automaton with a stack, and where the grammar is ambiguous,
you tell the generator how to break ties (`%left '+'`, `%left '*'`, and so on —
precedence declarations bison uses to resolve shift/reduce conflicts). The
parser that results is a **table-driven state machine**: at every point it has
consumed some prefix of the input, is in exactly one automaton state, and does
exactly one of "shift the next token" or "reduce a handful of stack symbols to a
nonterminal" — no backtracking, no trying one alternative and rewinding to try
another. This is what makes LALR parsing **linear time** (technically, linear in
the number of tokens, with a bounded lookahead): each token is consumed once, the
parser state transition is an O(1) table lookup, and the parser never revisits
input it has already shifted.

**A PEG cannot be ambiguous by construction, but it can require backtracking.**
A Parsing Expression Grammar replaces CFG's undirected `|` (alternation, "either
of these can match, and if both can, that's ambiguous") with **ordered choice**,
written `/`: `A / B` means "try to match `A`; if `A` fails, try `B` instead — and
whichever one succeeds first wins, permanently." There is never a question of
which alternative was "the" match, because the grammar itself specifies a
priority order. This is a real trade: you get determinism at the level of *which
rule wins*, but you pay for it with a parsing strategy that must, in general, be
willing to attempt a rule, fail partway through, **discard everything it
matched**, rewind the input position, and try the next alternative — this is
backtracking, and it is fundamentally different from what an LALR parser does.
Naively implemented, a PEG matcher can be exponential-time on pathological
grammars (nested optional/repeat constructs can force the same subexpression to
be re-matched at the same input position many times, once per calling context
that tries it). The classic fix — and the reason PEG parsers are usable in
production at all — is **packrat parsing**: memoize the result of matching every
`(rule, input position)` pair the first time it's computed, so that if the same
rule is tried again at the same position (which happens constantly, because
sibling alternatives and repeated backtracking revisit the same prefixes), the
matcher returns the cached answer instead of re-deriving it. With memoization,
PEG parsing is linear time in the size of the input, at the cost of memory
proportional to `(number of grammar rules) × (number of tokens)` for the memo
table. This is a genuine, well-known technique (Bryan Ford's packrat
parsing paper [18]), not something DuckDB invented — but it's worth knowing *why*
DuckDB's PEG engine has a component called `parser_packrat.cpp` (§3.4 below):
without it, ordered-choice-with-backtracking would not be a defensible
production parsing strategy.

Two more consequences of the ordered-choice design show up constantly in the
actual `.gram` files (§3.2):

- **More specific alternatives must be listed before general ones.** If
  `Interval <- YearKeyword / YearToMonth` were written in that order, `YEAR TO
  MONTH` would never match: the matcher tries `YearKeyword` first, matches just
  the `YEAR` token, declares success, and never backtracks to look at what
  follows — ordered choice commits to the first alternative that succeeds, even
  if a later alternative would have consumed more input and been the "obviously
  intended" parse. This is exactly backwards from how a human reads the grammar
  rule, and it is a real source of subtle grammar bugs; the actual grammar
  handles it correctly (§3.2 below shows the real rule, with the compound form
  listed first).
- **Negative lookahead (`!`) lets a rule assert "not this" without consuming
  input** — used to keep reserved keywords out of the identifier grammar (§3.2).
  This has no CFG equivalent at all; it's a PEG-specific primitive that falls
  out naturally once you think of a grammar rule as a small program that either
  succeeds (consuming some input) or fails (consuming nothing), rather than as a
  set of production rules to be matched against.

The other big structural difference — and the one that most changes what
"parsing" *feels like* while reading the source — is that DuckDB's PEG tokenizer
is **hand-written** (`base_tokenizer.cpp`, 694 lines), not generated by a
lexer-generator like flex. Historically, `libpg_query`'s lexer was a
flex-generated scanner transpiled alongside the bison grammar. The PEG engine
instead tokenizes the whole query up front into a flat token stream (§3.4), and
the grammar-matching phase operates purely over that pre-tokenized stream — the
grammar never sees raw characters, only classified tokens (keywords, literals,
identifiers, operators).

## 3.2 The grammar: files, keywords, and the PEG operators

The grammar's source of truth in this checkout is a tree of plain-text `.gram`
files:

```
src/parser/peg/grammar/
├── statements/            (41 files, one per statement kind + shared/expression rules)
│   ├── select.gram        SelectStatement, TableRef, JoinClause, WhereClause, ...
│   ├── expression.gram    the 16-level operator-precedence expression grammar
│   ├── create_table.gram  CreateStatement, column/constraint grammar, Identifier/PlainIdentifier
│   ├── insert.gram, update.gram, delete.gram, merge_into.gram, ...
│   └── common.gram        Program/TopLevelStatement/Statement, List(D)/Parens(D) macros, Type grammar
└── keywords/
    ├── reserved_keyword.list        cannot be used as an identifier (SELECT, FROM, WHERE, ...)
    ├── unreserved_keyword.list      can be used as an identifier (ABORT, ABSOLUTE, ...)
    ├── column_name_keyword.list     allowed specifically as column names
    ├── func_name_keyword.list       allowed specifically as function names
    ├── type_name_keyword.list       allowed specifically as type names
    └── bare_label_keyword.list      keywords allowed as a column alias without AS
```

(This lives under `src/parser/peg/` because the grammar was "moved into core"
from the `autocomplete` extension, where it started, in April 2026 (commit
`3d2ef2153f6` "Moving PEG into core", merged with PR #22194; see §3.0).
Older documentation, and this book's own research notes, cite the pre-move
path `extension/autocomplete/grammar/`, which no longer exists on disk. Even
`src/parser/peg/README.md` at this checkout still names the old path in its
first section. If you're reading this against a different DuckDB checkout,
check both.) A
keyword must appear in exactly one of the five *category* lists (reserved,
unreserved, column name, function name, type name). The build script (§3.3)
checks this and fails the build otherwise.

There is a
sixth list, `bare_label_keyword.list` (358 entries). It is not a sixth
category. Every keyword in it must also be in one of the five category lists.
It answers one narrow question: can this keyword be used as a column alias
*without* `AS`? The idea and the name come from PostgreSQL's own
`bare_label_keyword` list. Keywords that can follow an expression are left
out on purpose. `DAY` is one: in `SELECT d + 1 day`, the word `day` could be
an interval unit, so the grammar does not read it as an alias, and the query
is a syntax error. `test/sql/parser/bare_column_alias.test` shows both sides:

```
query IIII
SELECT a foo, b type, a name, b value FROM t ORDER BY a;
----
1	2	1	2
3	4	3	4
...
statement error
INSERT INTO t VALUES (7, 8) RETURNING a day;
----
<REGEX>:Parser Error: syntax error.*
```

`type`, `name` and `value` are bare-label keywords, so they work as aliases.
`day` is not, so the last statement fails.

`src/parser/peg/README.md` (254 lines) is the grammar-syntax reference the
DuckDB team itself maintains; the operators it documents, with a real rule from
the actual `.gram` files illustrating each one:

| Operator | Meaning | Real example |
|---|---|---|
| `<-` | rule definition | `RuleName <- RuleDefinition` |
| `/` | ordered choice — first success wins | `ComparisonOperator <- OperatorEqual / OperatorNotEqual / OperatorLessThan / ...` |
| (space) | sequence — all elements must match, in order | `CastExpression <- CastOrTryCast Parens(CastArguments)` |
| `?` | optional (zero or one) | `SimpleSelect <- SelectFrom WhereClause? GroupByClause? HavingClause? ...` |
| `*` | repeat, zero or more | `TableRef <- InnerTableRef JoinOrPivot*` |
| `+` | repeat, one or more | `CaseExpression <- 'CASE' Expression? CaseWhenThen+ CaseElse? 'END'` |
| `( )` | grouping, no new named rule | `IntervalToIntervalAsType <- (YearKeyword 'TO' MonthKeyword) / (DayKeyword 'TO' HourKeyword) / ...` (`common.gram:88`) |
| `Name(Param)` | parameterized rule (macro) | `List(D) <- D (',' D)* ','?` |
| `!` | negative lookahead — consumed by nothing | `PlainIdentifier <- !ReservedKeyword <[a-z_]i[a-z0-9_]i*>` |
| `<...>` | raw character-class/regex match | the `[a-z_]i[a-z0-9_]i*` inside the rule above |

Two of these deserve a closer look because they show up constantly and because
the book's example queries depend on them.

**The `List`/`Parens` macros**, defined once in `common.gram` and used
everywhere a comma-separated or parenthesized construct appears:

```
# src/parser/peg/grammar/statements/common.gram:193-194 (tail of file)
List(D) <- D (',' D)* ','?
Parens(D) <- '(' D ')'
```

`List(D)` is a parameterized rule: `D` is a placeholder for *any other rule*,
substituted at the call site. `List(Expression)` expands to "one `Expression`,
then zero or more of `,` followed by another `Expression`, then an optional
trailing comma" — this is exactly SQL's comma-separated-list-with-optional-
trailing-comma shape, and it's why you'll see calls like
`TargetList <- List(AliasedExpression)` (the `SELECT` list) and
`GroupByList <- List(GroupByExpression)` (the `GROUP BY` list) instead of each
clause spelling out its own comma-handling. `Parens(D)` is even simpler — wrap
`D` in literal `(` and `)` — and it's used for everything from function-call
argument lists (`FunctionExpressionArguments <- Parens(FunctionExpressionArgumentList)`)
to the query's own `IN (SELECT ...)` (`InSelectStatement <- Parens(SelectStatementInternal)`).
Because these macros are just ordinary PEG rules with a substitution parameter,
nothing about matching them is special-cased — the packrat cache memoizes
`List(Expression)` matched at a given token position exactly like any other
rule.

**Negative lookahead and identifiers**, from `create_table.gram` (where the
core `Identifier` rule actually lives in this checkout):

```
# src/parser/peg/grammar/statements/create_table.gram:76-77,80
PlainIdentifier <- !ReservedKeyword <[a-z_]i[a-z0-9_]i*>
QuotedIdentifier <- ('"' [^"]* '"') / ('`' [^`]* '`')
Identifier <- QuotedIdentifier / PlainIdentifier
```

A quoted
identifier can use backticks as well as double quotes, so
`` `my table` `` means the same as `"my table"`. Many other SQL dialects
quote names with backticks, and people paste that SQL into DuckDB. The real
work happens in the tokenizer: `Tokenizer::IsQuotedIdentifierDelimiter`
(`src/parser/peg/tokenizer/base_tokenizer.cpp:42-46`) accepts both
characters. A backtick-quoted name follows the same rules as a
double-quoted one: it keeps its case, and you escape the quote character by
writing it twice. `ToString()` always prints names with double quotes. The
test is `test/sql/parser/backtick_identifiers.test`.

Line by line: `PlainIdentifier` must **not** be `ReservedKeyword` (`!` asserts
this without consuming any input — if `ReservedKeyword` *would* match here, the
whole `PlainIdentifier` rule fails immediately, before the character-class match
even runs), and only then does the rule try to consume a run of characters
matching `[a-z_][a-z0-9_]*` (letters/underscore, then letters/digits/underscore,
case-insensitive per the `i` suffix — this is the regex-like `<...>` raw-match
syntax). `Identifier` is then "a quoted identifier, or else a plain one" — so
`"select"` (quoted) is a valid identifier even though bare `select` is not
(it would fail the `!ReservedKeyword` check and get parsed as the `SELECT`
keyword instead). One honest caveat, stated directly in the grammar's own
README and worth repeating because it's the kind of detail that's easy to miss
by reading `.gram` files alone: **negative lookahead is parsed into the compiled
grammar but currently ignored by the matcher** — there's a literal
`TODO(Dtenwolde): Implement negative lookahead` in `src/parser/peg/README.md`.
In practice this particular rule still works today because keyword-vs-identifier
disambiguation happens earlier, in the tokenizer's keyword classification
(§3.4) — but it means `!` as a *general-purpose* grammar construct is aspirational
in this checkout, not yet load-bearing everywhere it appears.

**A grammar rule you'll recognize** — the `WHERE`/`GROUP BY`/`ORDER BY`/`LIMIT`
clauses that Query A uses, verbatim from `select.gram`:

```
# src/parser/peg/grammar/statements/select.gram:150-152,198,203
FromClause      <- 'FROM' List(TableRef)
WhereClause     <- 'WHERE' Expression
GroupByClause   <- 'GROUP' 'BY' GroupByExpressions
OrderByClause   <- 'ORDER' 'BY' OrderByExpressions
LimitClause     <- 'LIMIT' LimitValue
```

Every one of these is a plain sequence: a literal keyword (or keyword pair, for
the two-word `GROUP BY`/`ORDER BY`) followed by whatever sub-rule matches that
clause's payload. There's no clever precedence handling needed at this level —
that's all pushed down into `Expression`, which is the one part of the grammar
that *does* need to think carefully about operator precedence, and does so the
way PEGs conventionally do: as an explicit chain of 16 numbered precedence
levels, each one a rule that calls the next-tighter-binding level and then
optionally wraps it in its own operator:

```
# src/parser/peg/grammar/statements/expression.gram:153-300 (excerpted; 16 levels total)
# LEVEL 1 (Lowest)
Expression <- LogicalOrExpression
ColumnDefaultExpr <- ColDefOrExpr
LogicalOrExpression <- LogicalAndExpression LogicalOrExpressionTail*
LogicalOrExpressionTail <- 'OR' LogicalAndExpression
# ...
# LEVEL 2
LogicalAndExpression <- LogicalNotExpression LogicalAndExpressionTail*
LogicalAndExpressionTail <- 'AND' LogicalNotExpression
# ...
# LEVEL 6
ComparisonExpression <- BetweenInLikeExpression ComparisonExpressionTail*
ComparisonExpressionTail <- ComparisonOperator ComparisonOperand
ComparisonOperand <- NotComparisonOperand / BetweenInLikeExpression
# NOT after a comparison operator binds everything up to the IS level: a < NOT b IS NULL is a < (NOT (b IS NULL))
NotComparisonOperand <- NotExpression IsExpression
# ...
# LEVEL 8
OtherOperatorExpression <- InfixOtherOperatorExpression
InfixOtherOperatorExpression <- BitwiseExpression OtherOperatorTail*
OtherOperatorTail <- AnyAllSubqueryTail / OtherOperatorExpressionTail / SingleArrowTail
# ...
# LEVEL 10
AdditiveExpression <- MultiplicativeExpression AdditiveExpressionTail*
AdditiveExpressionTail <- Term MultiplicativeExpression
Term <- '+' / '-'
# LEVEL 11
MultiplicativeExpression <- ExponentiationExpression MultiplicativeExpressionTail*
# ...
# LEVEL 16 (Highest)
BaseExpression <- SingleExpression IndirectionList?
```

This "left-operand, then zero-or-more (operator, next-level-down)" shape at
every level is the standard PEG encoding of left-associative binary operator
precedence — it's doing by hand, in the grammar text, what a bison
`%left`/`%right` precedence table does implicitly for an LALR grammar. `AND`
binds tighter than `OR` because `LogicalAndExpression` is nested one level
*below* `LogicalOrExpression` in this chain (matched first, closer to the
leaves); `+`/`-` bind tighter than comparisons because `AdditiveExpression`
sits at level 10, well below `ComparisonExpression` at level 6. (Because
`operator-*` grammars like this can't natively express iterated left-recursion
without care, DuckDB also has a dedicated `operator_matcher.cpp` for the pieces
of the `Expression` grammar that need real operator-precedence climbing rather
than this fixed-level-chain encoding — e.g. user-defined infix operators. The
16-level chain above handles the built-in operators.)

The arrow operator `->` needs care. It has two jobs in DuckDB: it is the
lambda arrow (`x -> x + 1`) and the JSON "extract" operator (`j -> 'a'`). Low
precedence suits a lambda, whose body should reach as far as possible. But
low precedence breaks JSON. An early version of the PEG grammar gave `->` its
own level below `OR`. In `j->'a'->>'b' = 'x'`, the comparison `= 'x'` then
ended up *inside* the arrow's right side, and the query failed with "No
function matches ... json_extract(JSON, BOOLEAN)". DuckDB v1.5.6 (with the
old bison grammar) ran it fine. PR #28240 (commit `2926f4dfe79`) fixed this.
`->` is an operator tail at level 8, next to the other operators:

```
# src/parser/peg/grammar/statements/expression.gram:244-251
# '->' is the lambda arrow or the JSON extract operator. Its operand extends as far as possible (the lambda body),
# unless another arrow follows directly: then the arrows chain to the left, like in PostgreSQL.
SingleArrowTail <- '->' SingleArrowBody
SingleArrowBody <- ArrowChainBody / ArrowFullBody
ArrowChainBody <- BitwiseExpression ArrowChainTail+
ArrowChainTail <- DoubleArrowTail / SingleArrowTail
DoubleArrowTail <- '->>' BitwiseExpression
ArrowFullBody <- LogicalOrExpression
```

Read it as two cases. If another `->` or `->>` follows, the right side is a
short operand and the arrows group to the left: `(j->'a')->>'b'`. If not,
the right side is a whole `LogicalOrExpression`, so `x -> x + 1 > 2` is still
a lambda with body `x + 1 > 2`. Ordered choice does the work here:
`ArrowChainBody` is tried first and only succeeds when a second arrow is
there. `test/sql/json/scalar/test_json_arrow_chain_precedence.test` guards
against the old bug. It checks many shapes, for example:

```
query I
SELECT j->'a'->>'b' = 'x' FROM t;
----
true

query I
SELECT 'x' = j->'a'->>'b' FROM t;
----
true
```

A few more details of the levels. At level 4, an `IS` test can be followed
by more comparisons (for example chained `IS` checks). `NOT` directly after a
comparison operator binds down to the `IS` level (the comment in the excerpt
above gives the example). `IN` has its own repeatable `InPredicate` rule.
`BETWEEN ASYMMETRIC` is supported, and `SOME` is a synonym of `ANY`.

## 3.3 The build pipeline: from `.gram` text to a compiled-in grammar

None of the `.gram` files or `.list` files are read at runtime. They are
transpiled, at build time, into C++ that gets compiled directly into the DuckDB
binary. The pipeline has two independent halves — one turns grammar text into a
matcher, the other turns a type-mapping file into transformer dispatch code —
and they're driven by two separate generator scripts. (One shell script,
`build_grammar.sh`, runs both of them in turn.)

```
 src/parser/peg/grammar/statements/*.gram  ─┐
 src/parser/peg/grammar/keywords/*.list     ─┼──► scripts/parser/build_grammar.sh
                                             │      (runs inline_grammar.py twice)
                                             │
                                             ├─► src/include/duckdb/parser/peg/
                                             │     inlined_grammar.gram
                                             │     (all .gram files + keywords merged
                                             │      into one file, for human debugging)
                                             │
                                             └─► src/include/duckdb/parser/peg/
                                                   inlined_grammar.hpp
                                                   const unsigned char INLINED_PEG_GRAMMAR[] = {
                                                       80, 114, 111, ...   (one row per grammar line)
                                                   (+ keyword_map.cpp: keyword lookup tables)
                                                          │
                                                          ▼
                                              compiled into the duckdb binary;
                                              read on first use by peg_parser.cpp's
                                              own tiny PEG-of-PEGs parser into a
                                              ParsedGrammar, then built into a
                                              CompiledGrammar object

 scripts/parser/grammar_types.yml            ─────► scripts/parser/
 (maps grammar rule name → C++ return type)          generate_transformer_trampoline.py
                                                          │
                                                          ├─► src/include/duckdb/parser/peg/
                                                          │     transformer/peg_transformer.hpp
                                                          │     (transform-process declarations +
                                                          │      DUCKDB_REGISTER_TRANSFORM_RESULT_TYPE
                                                          │      entries for every result type)
                                                          │
                                                          ├─► src/parser/peg/transformer/
                                                          │     transform_generated_trampoline.cpp
                                                          │     (per-rule "frame plan" data, Finalize*
                                                          │      trampolines, and one table of
                                                          │      {rule name, Initialize, Finalize, plan})
                                                          │
                                                          └─► src/include/duckdb/parser/peg/peg_rule_id.hpp
                                                                + src/parser/peg/peg_rule_id.cpp
                                                                (PEGRuleId enum: one number per
                                                                 built-in rule, plus name lookups)
```

`scripts/parser/build_grammar.sh` is the top-level entry point; it invokes
`inline_grammar.py` (which reads `statements_dir = .../grammar/statements`,
`keywords_dir = .../grammar/keywords`, per `scripts/parser/inline_grammar.py`
lines 7–10) once with `--grammar-file` to produce the human-readable merged
`inlined_grammar.gram`, and once by default to produce the actual compiled
artifact, `inlined_grammar.hpp`. That file holds the entire merged grammar as a
C++ byte array named `INLINED_PEG_GRAMMAR` (an `unsigned char` array of
character codes, not a string literal). Each row of the array holds exactly
one line of grammar. So a one-line grammar change is a one-row change in the
generated header, and two branches that touch different grammar lines do not
conflict there.

The first time a grammar is needed, `ParsedGrammar::CreateDefault`
(`src/parser/peg/parsed_grammar.cpp:41`) hands that array to `peg_parser.cpp`.
`peg_parser.cpp` holds a small parser for the PEG *notation itself* — literally
a hand-written recursive-descent parser (`PEGExpressionParser::ParseChoice` /
`ParseSequence` / `ParsePostfix`, `src/parser/peg/peg_parser.cpp:31-144`). Its
choice/sequence/repeat/optional grammar mirrors exactly the operators in the
table in §3.2. The result is a `ParsedGrammar`: rules as data. Then
`CompiledGrammar::Create` (`src/parser/peg/compiled_grammar.cpp:141-178`)
applies any grammar extensions (end of §3.4), checks that every referenced rule
exists, and builds a `CompiledGrammar` object: a graph of `Matcher` instances,
one per named rule. Each matcher is allocated once and given a stable
`packrat_id` (`MatcherAllocator::Allocate`, `src/parser/peg/matcher.cpp:56-61`)
that the packrat cache uses as half of its memoization key (§3.4). Two more
steps run at the end of `Create`: `ComputeFirstSets` (§3.4) and
`InitializeListChains`. The plain built-in grammar is compiled once per
process and kept in a static (`CompiledGrammar::DefaultGrammar`,
`compiled_grammar.cpp:180-183`) — not once per query.

The
generated code names rules by number. `peg_rule_id.hpp` defines an enum
`PEGRuleId` with one constant per built-in rule (`INVALID = 0`, then
`AlterStatement`, `AlterOptions`, ... up to `BUILTIN_COUNT`). When
`CompiledGrammar` is built, it fills a `rules_by_id` vector so a rule can be
found by number instead of by a string lookup
(`compiled_grammar.cpp:17-36`). Rules that a grammar extension adds at
runtime are not in the enum. They get numbers after `BUILTIN_COUNT`, sorted
by name, so the same set of rules always gets the same numbers.

The second half of the pipeline is independent: `grammar_types.yml` is a plain
mapping from grammar rule name to the C++ type its transform should produce
(`unique_ptr<QueryNode>`, `ShowType`, `QualifiedName`, and so on — the "Parse
Result Types" table in the README, §3.5, lists the raw parse-tree shapes these
get built from). `generate_transformer_trampoline.py --write` reads that file
and, for every rule it knows the type of, emits two things into
`transform_generated_trampoline.cpp`. The first is a small *plan*: plain data
that says which children of the rule's parse result to transform, and in which
result slots to put them. A shared function, `InitializePlannedTrampoline`,
reads the plan and schedules those children. The second is a
`Finalize*Trampoline` that receives the already-transformed, correctly-typed
children and calls into a hand-written `Transform*` function that builds the
actual AST node. §3.5 walks through a concrete instance.

Why plans instead of one generated function per rule? Such functions would
differ only in rule ids, child positions and slot numbers. So the generator
emits static `TransformFramePlan` records, and three shared trampolines
(`InitializePlannedTrampoline`, `InitializePlannedChoiceTrampoline`,
`FinalizePassthroughTrampoline`) read them. This keeps the code small. The
commit that made this change (`e74995506bf`, PR #27849) reports that the
transformer object file shrank from 844 KB to 723 KB.

**Both generated files are explicitly
off-limits for hand-editing** — the README says so directly, and the convention
in `transform_*.cpp` files is to add hand-written semantic logic in a
same-named-but-not-generated file (e.g. `transform_describe.cpp` holds the
hand-written `TransformShowTables`, while `transform_generated_trampoline.cpp`
holds the generated plan and `FinalizeShowTablesTrampoline` that call it).

## 3.4 Runtime parse flow: tokenizer → packrat-memoized matching → transform

With the grammar compiled into the binary once, here is what happens for one
query, end to end. The engine lives in `src/parser/peg/`:

```
 query text
     │
     ▼
┌───────────────────────────────────────────────────────────────┐
│ tokenizer/base_tokenizer.cpp  (694 lines, hand-written —        │
│                                 NOT flex-generated)              │
│  whole query text  →  flat vector<MatcherToken>                 │
│  classifies each token via the grammar's own PEGKeywordHelper    │
│  (ParsedGrammarKeywordHelper; knows extension keywords too)      │
└───────────────────────────┬─────────────────────────────────────┘
                             │  TokenIterator over MatcherToken stream
                             ▼
┌───────────────────────────────────────────────────────────────┐
│ per top-level statement:                                        │
│   PEGTransformerFactory::TransformTopLevelStatement(...)          │
│   (src/parser/peg/transformer/peg_transformer_factory.cpp:51)     │
│                                                                   │
│   fresh ParserPackratCache + arenas for this statement            │
│        │                                                          │
│        ▼                                                          │
│   grammar.TopLevelStatementMatcher().MatchParseResult(state)       │
│        │  matcher_stack.cpp: one loop drives ListMatchFrame,       │
│        │  ChoiceMatchFrame, OptionalMatchFrame, RepeatMatchFrame — │
│        │  skips children whose FIRST set rules them out,           │
│        │  memoizes in the packrat cache by (matcher_id,            │
│        │  token_index)                                              │
│        ▼                                                          │
│   success → ParseResult tree (ListParseResult/ChoiceParseResult/   │
│              OptionalParseResult/RepeatParseResult/leaf results)    │
│   failure → ParserException "syntax error at or near \"...\""      │
└───────────────────────────┬─────────────────────────────────────┘
                             │ (on success)
                             ▼
┌───────────────────────────────────────────────────────────────┐
│ transformer.Transform<unique_ptr<SQLStatement>>(stmt_pr)          │
│  (PEGTransformer, transformer/*.cpp)                              │
│  walks the ParseResult tree directly into DuckDB AST nodes —      │
│  there is no separate untyped-parse-tree stage in between          │
└───────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
                 unique_ptr<SQLStatement>
        (pushed onto Parser::statements, one per top-level statement)
```

**The tokenizer is hand-written, not flex-generated.** `base_tokenizer.cpp`
walks the raw query text once, classifying runs of characters into
`MatcherToken`s (keywords, string/number literals, operator symbols,
identifiers). To decide whether a word is a keyword, it asks the
`PEGKeywordHelper` it was built with (`base_tokenizer.cpp:35`, and the lookup
at line 551). Each `CompiledGrammar` makes its own tokenizer with its own
helper, a `ParsedGrammarKeywordHelper` built from that grammar's rules
(`compiled_grammar.cpp:156-157`). So a grammar extension that adds a keyword
also changes how the tokenizer classifies that word. (The global
`DuckDBKeywordHelper::Instance()` is only used by the syntax-highlighting
entry point `Parser::Tokenize`.)
This differs from the historical `libpg_query` pipeline,
where the lexer was itself flex-generated from a `.l` file transpiled alongside
the bison grammar. Tokenizing happens exactly once per query, up front — the PEG
matcher that runs afterward never looks at raw characters again, only at the
token stream (there's also a separate, much smaller `highlight_tokenizer.cpp`
used purely for syntax-highlighting purposes, outside the parse path).

A dot needs care. A `.` can start a number (`.5`) or join two names
(`t.a`). The tokenizer starts a number at a dot only when a digit follows it
(`base_tokenizer.cpp:444-446`). This guards against a bug (issue #27439,
fixed in commit `04e1fd57fa0`) where any dot started a number, so in
`select r.e100` the `.` was read as a number literal.
`test/sql/function/tokenize/simple_tokenize.test` expects `8	OPERATOR	.`
for that query. For `SELECT users..id FROM users;`,
`test/sql/peg_parser/peg_syntax_error.test` expects
`syntax error at or near "."`.

**Matching is not naive recursive-descent — it's built as an explicit resumable
state machine, not raw C++ recursion.** This detail is easy to miss reading only
the grammar files. The contract is the `MatchProcess` class
(`src/include/duckdb/parser/peg/matcher.hpp:247-253`). A process has a
`Resume(optional<MatcherResult> child_result)` method. It is called once at the
start and again each time a child result becomes available. It returns either
`MatchStep::Child(...)` ("go match this next child, come back to me when you're
done") or `MatchStep::Complete(...)` ("I'm done, here's my result")
(`src/parser/peg/matcher_process.cpp:6-12`). This is a coroutine written by hand
in pre-C++20 style. An explicit `MatchStack` (`matcher_stack.cpp`) drives it,
not the C++ call stack. This is a deliberate choice: it lets the ordered
choice/backtracking machinery unwind and retry alternatives without risking a
C++ stack overflow on deeply nested expressions, and it makes packrat
memoization easy to add — each frame is checked against the packrat cache at
`(matcher, token position)` before it runs.

The four "structural" matchers — sequence, choice, optional and repeat — do
not have their own `MatchProcess` classes. `matcher_process.cpp` is 60 lines
and holds only `AtomicMatchProcess`. The four structural cases are frame
types inside `matcher_stack.cpp`: `ListMatchFrame`, `ChoiceMatchFrame`,
`OptionalMatchFrame` and `RepeatMatchFrame` (lines 36-224). One loop,
`MatchStack::ExecuteInternal` (from line 343), handles all of them in a
`switch` on the frame kind.
Any other matcher, such as one an extension supplies, runs through its
own `MatchProcess`, wrapped in an `OpaqueMatchFrame`. Frames are allocated in
an arena and freed by rewinding the arena when the frame is popped
(`MatchStack::DestroyTopFrame`, lines 331-338).

**FIRST sets: skipping alternatives that cannot match.** (Commit
`a84368623af` added this.) The "FIRST set" of a grammar rule is the
set of tokens that the rule's match can start with. For example, the FIRST set
of `CaseExpression <- 'CASE' ...` is just `{CASE}`. When the next token is
`SELECT`, there is no point trying `CaseExpression` at all. DuckDB
computes an over-approximation of each matcher's FIRST set once, when the
grammar is compiled (`ComputeFirstSets`, called from
`compiled_grammar.cpp:171`; code in `src/parser/peg/matcher_first_set.cpp`).
"Over-approximation" means the set may contain extra tokens but never misses
one. The set is stored in a small struct:

```cpp
// src/include/duckdb/parser/peg/matcher.hpp:269-292
//! Over-approximation of the tokens a matcher can start with (its FIRST set), computed once per grammar. Lets the
//! matcher skip sub-matchers that certainly cannot match at the current token.
struct MatcherFirstSet {
    bool computed = false;
    //! the matcher can succeed without consuming a token
    bool nullable = false;
    //! no information: never skip
    bool any = false;
    //! MatcherTokenClass bits
    uint8_t class_mask = 0;
    //! bitset over grammar literal ids (keywords and symbols)
    vector<uint64_t> literals;
    optional_ptr<const GrammarLiteralTable> table;
    // ...
    //! False only if the matcher certainly cannot match at the current token
    DUCKDB_API bool MightMatch(MatchState &state) const;
};
```

The matcher stack calls `MightMatch` before it pushes a frame. A choice frame
skips alternatives that cannot start here (`matcher_stack.cpp:446-454`). An
optional whose child cannot start here resolves straight to "matched nothing"
without pushing anything (lines 537-541). A skipped attempt would have failed
anyway without using up a token, so the result, the token positions and the
syntax-error position do not change. The commit message reports the median
parse time per statement on the test corpus fell from 65.7 µs to 29.8 µs.

Two more speed-ups are worth knowing. The first one also
changes the shape of the parse tree that the transformer sees:

- **Collapsible rules** (commit `8e9df4df2ee`). Each precedence level has the
  shape `X <- Y Tail*`. For the literal `42`, every one of the 16 levels would
  build a `ListParseResult` with a single child, and the transformer would
  then unwrap each one. To avoid this, rules listed as collapsible hand their only child's
  result straight up when no tail matched. The list is generated into the
  `PEGTransformerFactory` constructor (`peg_transformer_factory.cpp:117`; the
  first entry, `"Expression"`, is at line 122). The collapsing itself happens
  when a list frame completes (`ListMatchFrame::CompleteList`,
  `matcher_stack.cpp:93-110`). The commit reports an 18% gain on the parser
  benchmarks.
- **Arena allocation.** Parse results, match frames, the packrat cache and the
  transformer's own work items are carved out of `ArenaAllocator`s that
  live for one statement, instead of one `malloc`/`free` per matched rule.
  `TransformTopLevelStatement` sets up a `ParseResultAllocator` and separate
  arenas for match processes, the packrat cache and the transformer
  (`peg_transformer_factory.cpp:58-63,111`).

**Packrat memoization, concretely.** `ParserPackratCache` (`parser_packrat.cpp`,
81 lines total) is small, but it has two storage paths:

```cpp
// src/parser/peg/parser_packrat.cpp:7-9,29-54
size_t ParserPackratKeyHash::operator()(const ParserPackratKey &key) const {
    return std::hash<idx_t>()(key.matcher_id) ^ (std::hash<idx_t>()(key.token_index) << 1);
}
// ...
optional_ptr<const ParserPackratEntry> ParserPackratCache::Lookup(const Matcher &matcher, idx_t token_index) const {
    D_ASSERT(matcher.IsPackratMemoized());
    auto packrat_id = matcher.GetPackratId();
    D_ASSERT(packrat_id.IsValid());
    auto matcher_id = packrat_id.GetIndex();
    if (IsIndexed(token_index)) {
        auto index = token_index - token_start;
        if (index >= token_entries.size()) {
            return nullptr;
        }
        auto node = token_entries[index];
        while (node) {
            if (node->matcher_id == matcher_id) {
                return &node->value;
            }
            node = node->next;
        }
        return nullptr;
    }
    ParserPackratKey key {matcher_id, token_index};
    auto entry = entries.find(key);
    if (entry == entries.end()) {
        return nullptr;
    }
    return optional_ptr<const ParserPackratEntry>(&entry->second);
}
```

The cache key is exactly `(matcher_id, token_index)` — which rule, at
which position in the token stream — precisely the theoretical packrat-parsing
key. The interesting part is how it is stored.

The lookup has two paths. When the
cache is created, it is told which token positions the current statement
covers (`token_start` up to `token_limit`). For a position in that range,
the cache uses a plain array indexed by position. Each array slot holds a
short linked list of `(matcher_id, result)` nodes for that position. Finding
an entry is then an array access plus a short list walk, with no hashing.
Positions outside the range use a hash map (`entries`). The nodes live in
the cache's arena (`Store`, lines 56-79).

To make the array idea concrete, picture the statement `SELECT 1` as tokens
`[0:SELECT] [1:1] [2:<end>]`:

```
token_entries[0] → (rule A, success, next token 2) → (rule B, failure) → ...
token_entries[1] → (rule C, success, next token 2) → ...
token_entries[2] → (rule D, success, next token 2)
```

Each node records whether the rule matched at that position and, if it did,
where the match ended (`ParserPackratEntry`: `success`, `token_index_after`,
`max_token_index_seen`, `result`). Rules A to D stand for whichever memoized
rules the parser happened to try there; this is a picture of the layout, not a
dump of a real run.

Every `Matcher` gets a stable `packrat_id` when it's allocated at
grammar-compile time (`MatcherAllocator::Allocate`, `matcher.cpp:56-61`); not
every matcher opts into memoization (`IsPackratMemoized()` — some very cheap
atomic matchers aren't worth caching), but the ones that are looked up
repeatedly across sibling alternatives and backtracking attempts are. The
lookup and the store are done by the matcher stack, around each frame
(`PackratMatchState::TryLoadCachedResult` and `StoreResult`,
`matcher_stack.cpp:235-263`). One implementation detail worth flagging
honestly: the cache in `TransformTopLevelStatement` is constructed *fresh for
each top-level statement* (a local `ParserPackratCache packrat_cache(...)`,
`peg_transformer_factory.cpp:63`) — memoization is scoped per statement, not
shared across an entire multi-statement query string. That is a reasonable
trade: grammar-rule/token-position pairs from one statement are essentially
never useful for parsing the next one.

**The transformer pattern turns a matched parse tree directly into typed DuckDB
AST — there is no untyped intermediate parse-tree stage.** Once
`MatchParseResult` succeeds, the resulting `ParseResult` tree (a tree of
`ListParseResult`/`ChoiceParseResult`/`OptionalParseResult`/`RepeatParseResult`
nodes, each still shaped exactly like the grammar rule that produced it, plus
leaf `IdentifierParseResult`/`StringLiteralParseResult`/`NumberParseResult`/
`KeywordParseResult` nodes) is walked exactly once, by
`transformer.Transform<unique_ptr<SQLStatement>>(stmt_pr)`
(`peg_transformer_factory.cpp:28`, inside the helper
`ExtractAndTransformStatement`), and every node in that walk produces a real
`SQLStatement`/`QueryNode`/`ParsedExpression`/`TableRef` — never a generic,
untyped "parse node" that gets reinterpreted later. §3.5 shows exactly how one
rule's `ParseResult` becomes one AST node.

The
transformer counts how deeply expressions are nested while it builds
them, and throws a `ParserException` once the count passes the
`max_expression_depth` setting (`PEGTransformer::EnterExpression`,
`src/parser/peg/transformer/peg_transformer.cpp:277-285`). The code comment gives the reason: "the transformer does not
recurse, but binding the resulting expressions does - reject them early".
The binder walks expressions with ordinary C++ recursion, so a very deep
tree could overflow the stack there. `test/sql/parser/test_deep_expression_nesting.test`
feeds in very deeply nested lists (`SELECT [[[[...1]]]]`)
and expects `Parser Error: Max expression depth limit of 1000 exceeded`
(1000 is the default). Not everything counts: long `AND`/`OR` chains do not
add depth, but stacked casts and nested types do.

**`Parser::ParseQuery`, the real control flow**, verbatim, with the extension
fallback protocol intact:

```cpp
// src/parser/parser.cpp:253-324
void Parser::ParseQuery(const string &query_p) {
    const string query = NormalizeSQLString(query_p);
    if (options.extensions) {
        bool has_strict_extension_error = false;
        ErrorData last_strict_extension_error;
        for (auto &ext : options.extensions->ParserExtensions()) {
            if (!ext.parser_override) {
                continue;
            }
            if (options.parser_override_setting == AllowParserOverride::DEFAULT_OVERRIDE) {
                continue;
            }
            auto result = ext.parser_override(ext.parser_info.get(), query, options);
            if (result.type == ParserExtensionResultType::PARSE_SUCCESSFUL) {
                statements = std::move(result.statements);
                return;
            }
            if (options.parser_override_setting == AllowParserOverride::STRICT_OVERRIDE) {
                if (result.type == ParserExtensionResultType::DISPLAY_EXTENSION_ERROR) {
                    has_strict_extension_error = true;
                    last_strict_extension_error = std::move(result.error);
                } else {
                    has_strict_extension_error = false;
                }
                continue;
            }
        }
        if (options.parser_override_setting == AllowParserOverride::STRICT_OVERRIDE && has_strict_extension_error) {
            last_strict_extension_error.Throw();
        }
    }
    // PEG parser: tokenize, then peel one TopLevelStatement at a time. On per-statement PEG
    // failure, hand the rest of the query to parse_function extensions; the extension reports
    // how many bytes it consumed and we advance the token cursor past them.
    ParserTokenizerBehavior behavior(query);
    auto &tokenizer = GetGrammar().GetTokenizer();
    tokenizer.TokenizeInput(behavior);
    TokenIterator token_iterator(behavior);
    // A statement's text starts where the previous one ended, so comments and whitespace before its
    // first token belong to it; the last statement runs to the end of the input.
    idx_t stmt_start = 0;
    while (token_iterator.Current()) {
        unique_ptr<SQLStatement> stmt;
        try {
            stmt = ParseTopLevelStatement(token_iterator);
        } catch (ParserException &e) {
            stmt = TryParseExtensionStatement(token_iterator, query);
            if (!stmt) {
                throw;
            }
        }
        auto stmt_end = token_iterator.ConsumedEndOffset();
        if (stmt) {
            stmt->stmt_location = QueryLocation(stmt_start, stmt_end - stmt_start);
            statements.push_back(std::move(stmt));
        }
        stmt_start = stmt_end;
    }

    if (!statements.empty()) {
        auto &last_location = statements.back()->stmt_location;
        last_location = QueryLocation(last_location.offset, query.size() - last_location.offset);
        for (auto &statement : statements) {
            statement->query = query.substr(statement->stmt_location.offset, statement->stmt_location.length);
            statement->stmt_location = QueryLocation(0, statement->query.size());
            if (statement->type == StatementType::CREATE_STATEMENT) {
                auto &create = statement->Cast<CreateStatement>();
                create.info->sql = statement->query;
            }
        }
    }
}
```

Reading this top to bottom:

1. **`NormalizeSQLString(query_p)` (`parser.cpp:241-251`).** This runs before
   a single token is produced. It first calls `ValidateUTF8Query`, which throws
   immediately on invalid UTF-8 — the code comment is explicit about why this
   ordering matters: *"Validate before strip: StripUnicodeSpaces walks
   multi-byte sequences and assumes valid UTF-8"* — i.e. the tokenizer must
   never be handed bytes that could make a multi-byte-aware scan run off the
   end of a malformed sequence (this guards against a real oss-fuzz-class bug
   class). If the query is plain ASCII, it is returned as is: there can be no
   Unicode spaces in it. Otherwise it calls `StripUnicodeSpaces`, which
   normalizes away Unicode space look-alikes (NBSP, the U+2000-range spaces,
   ideographic space, byte-order-mark) that could otherwise be used to smuggle
   characters past a naive tokenizer that only recognizes ASCII whitespace as a
   token separator.

   Each
   Unicode space is replaced by *as many* ASCII spaces as it had bytes
   (`ReplaceUnicodeSpaces`, `parser.cpp:51-65`). A non-breaking space is two
   bytes (`C2 A0`), so it becomes two spaces. This keeps every byte offset
   after it unchanged, so error positions are byte offsets into the
   *original* text. `test/sql/json/test_json_serialize_sql.test`
   checks this: for `'SELECT' || chr(160) || ')'` the error position is 8,
   and for `chr(8195)` (a three-byte space) it is 9.

2. **The parser-extension override protocol runs first, before the core PEG
   grammar even sees the query**, but only if the caller has installed
   extensions with a `parser_override` hook and *not* set
   `AllowParserOverride::DEFAULT_OVERRIDE` (which explicitly skips this loop and
   goes straight to the core parser). Under `STRICT_OVERRIDE`, if every
   extension either doesn't apply or reports an error, that error is what gets
   thrown — the core PEG grammar isn't consulted at all in that mode. This is
   the mechanism a custom-dialect extension uses to claim syntax the core
   grammar doesn't recognize *before* the core grammar gets a chance to reject
   it.
3. **Tokenize once, for the whole query string**, then loop, peeling one
   `TopLevelStatement` at a time off the front of the token stream via
   `ParseTopLevelStatement` (which, as shown in §3.4's diagram, is a thin
   wrapper around `PEGTransformerFactory::TransformTopLevelStatement`). This is
   why a multi-statement string like `SELECT 1; SELECT 2;` produces two
   `SQLStatement`s: the `while (token_iterator.Current())` loop keeps consuming
   the token stream, one statement at a time, advancing the shared
   `token_iterator` after each success.

   The
   loop also decides which slice of the text belongs to each statement.
   A statement's text starts where the previous one ended, right after its
   semicolons. It does not start at its own first token. So in
   `/* leading */ SELECT current_query();` the comment is part of the
   statement's text, and `current_query()` and the query log show it again.
   (In the v2.0.0 release a leading comment was dropped, and a comment
   between two statements was glued to the *earlier* one. v1.5.6 behaved
   correctly. PR #28597 fixed it.) The test
   `test/sql/parser/statement_query_text.test` runs
   `SELECT 'first'; /* between */ SELECT 'second'` and then reads the
   query log. It expects these two texts:

   ```
   [SELECT 'first';]
   [ /* between */ SELECT 'second']
   ```

   The comment, and the space before it, belong to the second statement.
4. **The second, orthogonal extension fallback: `parse_function`
   extensions.** If a *single statement's* PEG match fails partway through
   (`ParseTopLevelStatement` throws `ParserException`), the loop doesn't
   propagate the exception immediately — it first calls
   `TryParseExtensionStatement(token_iterator, query)`, which hands the
   remaining token stream (as plain `SimpleToken`s) to any registered
   parser extensions. An extension reports back how many tokens (equivalently,
   how many bytes) of the remainder it consumed, and the shared token cursor
   advances past exactly that much — allowing, for example, a trailing clause
   the core grammar has no rule for to be recognized by an extension while the
   rest of the statement was already parsed natively. Only if no extension
   claims the failure does the original `ParserException` propagate to the
   caller — this is the path Query error example in §3.7 exercises: no
   extension is installed in that test, so the `ParserException` reaches the
   caller unmodified.

Two smaller, related entry points worth knowing exist, both in
`src/parser/parser.cpp` and both explicitly *not* part of the parse path
described above: **`Parser::Tokenize`** (line 377), a public tokenization-only
entry point that classifies tokens into `SimplifiedTokenType` for SQL syntax
highlighting, and **`Parser::TokenizeError`** (line 423), a separate, more
elaborate tokenizer used purely to *format* a parser error message (locating the
`LINE n:` marker and the `^` caret under the offending token) — a
diagnostics/pretty-printing utility layered on top of the parse path, not a step
inside it. `Parser::ParseQuery` also has several convenience siblings built by
constructing a small mock query string and asserting on the shape of the
resulting statement — `ParseExpressionList` prefixes `"SELECT "`,
`ParseGroupByList` wraps in `"SELECT 42 GROUP BY %s"`, `ParseOrderList` wraps in
`"SELECT * FROM tbl ORDER BY ..."`, and so on (`parser.cpp:578-726`) — these
exist so call sites elsewhere in DuckDB that need to parse a bare expression or
column list can reuse the exact same grammar and transformer machinery instead
of duplicating any part of it. Two of these helpers check for exactly one
item: `ParseSelectNode` (line 578) parses exactly one `SELECT`, and
`ParseSingleExpression` (line 623) parses exactly one expression.

### Changing the grammar at runtime: grammar extensions, `ParseIterator`, dialects

The grammar is not fixed once DuckDB is built. An extension can change it
while DuckDB runs, and a user can switch those changes on and off with SQL.

**A grammar extension is a list of edits to the grammar.** An extension
returns a `vector<GrammarChange>`. The kinds of edit are listed in an enum:

```cpp
// src/include/duckdb/parser/grammar_change.hpp:16-25
enum class GrammarChangeType : uint8_t {
    ADD_RULE,
    ADD_CHOICE,
    PREPEND_CHOICE,
    REMOVE_CHOICE,
    REPLACE_CHOICE,
    REPLACE_RULE,
    SET_TRANSFORM,
    ADD_TERMINAL_RULE_OVERRIDE
};
```

`CompiledGrammar::Create` starts from the built-in grammar and applies every
change of every active extension, in order, before it builds the matchers
(`compiled_grammar.cpp:141-148`). Then it checks that the result still has
the root rules `Program` and `TopLevelStatement`, and that every rule a rule
refers to exists (lines 39-69). So a broken extension fails when its grammar
is compiled, not in the middle of a query.

The DuckDB shell is a real, small example. In the shell you can type a bare
expression such as `1 + 1`, or a bare table name, as a whole statement. The
core grammar has no rule for that. The shell adds one, called
`ExpressionStatement`, through a grammar extension:

```cpp
// tools/shell/shell_grammar_extension.cpp:80-90
vector<GrammarChange> ShellGrammarExtension::GetChanges() const {
    vector<GrammarChange> changes;
    // suppress suggestions so a bare expression does not pollute statement-level completion
    changes.push_back(GrammarChange::AddRule("ExpressionStatement <- List(ExpressionAlias)",
                                             StartExpressionStatementTransform, true));
    // no implicit aliases ("1 + 1 foo") - those are too easily confused with a typo'd statement
    changes.push_back(GrammarChange::AddRule("ExpressionAlias <- ColIdExpression / ExpressionAsCollabel / Expression"));
    // last alternative so every real statement is tried first
    changes.push_back(GrammarChange::AddChoice("Statement", "ExpressionStatement"));
    return changes;
}
```

Three edits: add two rules, and add `ExpressionStatement` as the *last*
alternative of `Statement`. "Last" matters because of ordered choice (§3.1):
every real statement kind is tried first. The new rule also brings its own
transform function (`StartExpressionStatementTransform`), so the transformer
knows how to turn a match into a `SelectStatement`. Outside the shell — for
example in a test or a client library — a bare `override` is a syntax
error rather than an attempt to read a table called `override`.
`test/extension/loadable_parser_override.test` expects
`Parser Error: syntax error at or near "override"`.

**Switching extensions on and off.** Each connection has a setting,
`active_grammar_extensions`, holding a list of extension names. SQL functions
manage it: `add_grammar_extension`,
`remove_grammar_extension`, and the table function `duckdb_grammar_extensions()`
(`src/function/table/system/duckdb_grammar_extensions.cpp`).
`test/extension/test_add_grammar_extension.test` uses a demo extension that
adds the "pipe" query syntax (`|>`):

```
query TI
SELECT name, active FROM duckdb_grammar_extensions() WHERE name = 'pipe_query_syntax';
----
pipe_query_syntax	false

statement error
FROM produce |> WHERE item != 'bananas';
----
Parser Error

statement ok
CALL add_grammar_extension('pipe_query_syntax');

query TI
FROM produce |> WHERE item != 'bananas';
----
apples	10
```

Before the call, `|>` is a syntax error. After it, the same text parses and
runs. `remove_grammar_extension` and `RESET active_grammar_extensions` turn it
off again.

**Which compiled grammar does a query use?** `CompiledGrammar::Get`
(`src/main/parser_context.cpp:20-39`) decides, in this order:

```
 1. CONNECT-ed to another database?  → that connection's grammar
 2. a dialect is selected?            → the dialect extension's grammar
 3. grammar extensions are active?    → the connection's cached grammar for them
 4. otherwise                         → the database's built-in grammar
```

Compiling a grammar is not free, so compiled grammars are shared. The
database keeps a `ParserCache` (`src/include/duckdb/parser/peg/compiled_grammar.hpp`).
It holds the built-in grammar, plus up to `MAX_CACHED_EXTENSION_MATCHERS = 16`
grammars compiled for different lists of active extensions. Every connection
with the same list gets the same compiled grammar.
When the cache is full, the least recently used grammar is dropped
(`ParserCache::GetMatcher`, `parser_context.cpp:68-99`).
A connection that still holds a dropped grammar keeps its own reference, so
nothing breaks under it.

**`ParseIterator`: one statement at a time, with fresh options.** A query
string sent through `ClientContext` is usually not parsed with
`Parser::ParseQuery` in one go. It is read through `ParseIterator`
(`src/main/parse_iterator.cpp`, used by `ClientContext::IterateStatements`,
`src/main/client_context.cpp:460-462`). The iterator tokenizes the whole
string once, but parses only one statement per `Peek()`. So the caller can run
a statement before the next one is parsed. This
matters when a statement changes how later statements parse. The iterator
builds a new `Parser` for every statement, with the options as they are at
that moment:

```cpp
// src/main/parse_iterator.cpp:116-118
// parser options can change between statements of the same input (SET statements such as
// integer_division or active_grammar_extensions), so each statement gets a parser with fresh options
Parser parser(options);
```

A fresh parser is cheap: a `Parser` holds only a copy of the options and a
shared pointer to the compiled grammar, and the expensive matcher data lives
in the shared `CompiledGrammar`. Two tests show the effect.
In `test/api/test_parse_statement_iterator.cpp`, `SET integer_division=true;
SELECT 2/4` returns `0`, not `0.5`. In
`test/extension/test_loadable_grammar_extension.test`, one string loads the
demo extension, sets `active_grammar_extensions`, and then uses `|>` in a
`CREATE TABLE ... AS` — all as three statements of one input.

**Dialects and `CONNECT`.** Two related mechanisms replace the grammar
completely instead of editing it. A *dialect extension*
(`src/parser/peg/dialect_extension.cpp`) supplies its own compiled grammar,
selected per connection (step 2 above). And while a client is `CONNECT`-ed to
another database (step 1), DuckDB uses the small "passthrough" grammar from
`src/include/duckdb/parser/peg/passthrough_dialect.hpp`:

```
//!   Statement            <- DisconnectStatement / PassthroughStatement
//!   PassthroughStatement <- PassthroughToken+
```

`PassthroughToken` matches any single token that is not a `;` or the end of
input, without trying to understand it. So every statement except
`DISCONNECT` becomes a `PassthroughStatement`
(`src/include/duckdb/parser/statement/passthrough_statement.hpp`) that carries
the raw SQL text to the remote database. DuckDB's tokenizer still decides
where one statement ends and the next begins.

## 3.5 The transformer pattern, in detail: `TransformShowTables`

Take one grammar rule end to end, because the two-file generated/hand-written
split described in §3.3 is easiest to understand with a worked example. The
rule:

```
# src/parser/peg/grammar/statements/describe.gram:8
ShowTables <- ShowOrDescribe 'TABLES' 'FROM' QualifiedName
```

Two files cooperate to turn a match of this rule into an AST node. The
generated half, in `transform_generated_trampoline.cpp`, does the mechanical
work of pulling the rule's typed children out of the raw `ParseResult` tree.
That half is mostly *data*. First, a plan that lists the
children to transform:

```cpp
// src/parser/peg/transformer/transform_generated_trampoline.cpp:2294-2308
static const TransformChildPlan FRAME_CHILDREN_160[] = {{PEGRuleId::QualifiedName,
                                                         TransformChildKind::REQUIRED,
                                                         3,
                                                         1,
                                                         false,
                                                         {0, TRANSFORM_PATH_END, 0},
                                                         {0, TRANSFORM_PATH_END, 0}},
                                                        {PEGRuleId::ShowOrDescribe,
                                                         TransformChildKind::REQUIRED,
                                                         0,
                                                         0,
                                                         false,
                                                         {0, TRANSFORM_PATH_END, 0},
                                                         {0, TRANSFORM_PATH_END, 0}}};
static const TransformFramePlan FRAME_PLAN_173 = {FRAME_CHILDREN_160, 2, 2, false, 0, {}};
```

Each `TransformChildPlan` entry reads: *which rule* to transform the child
with, whether it is required or optional, *which child* of the rule's
`ListParseResult` it is (`parse_child`), and *which result slot* to put it in
(`slot`) (field meanings in
`src/include/duckdb/parser/peg/transformer/peg_transformer.hpp:290-303`). So
the first entry says "child 3 is a `QualifiedName`, put it in slot 1", and the
second says "child 0 is a `ShowOrDescribe`, put it in slot 0". The plan itself
says: two children, two slots.

Second, a typed finalize function, generated one per rule:

```cpp
// src/parser/peg/transformer/transform_generated_trampoline.cpp:12365-12371
arena_ptr<TransformResultValue>
PEGTransformerFactory::FinalizeShowTablesTrampoline(PEGTransformer &transformer, GeneratedTransformProcess &process) {
    auto show_or_describe = process.TakeResult<ShowType>(0);
    auto qualified_name = process.TakeResult<QualifiedName>(1);
    auto result = TransformShowTables(transformer, show_or_describe, qualified_name);
    return transformer.MakeResult<unique_ptr<QueryNode>>(std::move(result));
}
```

The plan is read by one shared function, `InitializePlannedTrampoline`
(`src/parser/peg/transformer/transform_frame_plan.cpp:76`). For a plain
required child it does this:

```cpp
// src/parser/peg/transformer/transform_frame_plan.cpp:109-111
case TransformChildKind::REQUIRED:
    process.PushChild({transformer.GetRule(child.rule), PlannedChild(list_pr, child)}, slot);
    break;
```

So the shared initializer schedules two of the rule's four sequence children
onto the transformer's explicit stack — child 0 (`ShowOrDescribe`) and child 3
(`QualifiedName`); the literal keywords `'TABLES'`/`'FROM'` at children 1 and 2
need no transformation, so they're simply skipped. Note that the rule is
found by number (`PEGRuleId::QualifiedName`, §3.3), not by the string
`"QualifiedName"`. Once both scheduled children have themselves been
transformed (recursively, by the same machinery, all the way down to leaf
tokens), `FinalizeShowTablesTrampoline` retrieves them by slot index with the
correct static types (`ShowType`, `QualifiedName` — exactly the types
`grammar_types.yml` maps those rule names to) and calls the one hand-written
function that actually knows what a `SHOW TABLES FROM ...` statement means:

```cpp
// src/parser/peg/transformer/transform_describe.cpp:51-67
unique_ptr<QueryNode> PEGTransformerFactory::TransformShowTables(PEGTransformer &transformer,
                                                                 const ShowType &show_or_describe,
                                                                 const QualifiedName &qualified_name) {
    auto showref = make_uniq<ShowRef>();
    showref->show_type = ShowType::SHOW_FROM;
    if (!IsInvalidCatalog(qualified_name.Catalog())) {
        throw ParserException("Expected \"SHOW TABLES FROM database\", \"SHOW TABLES FROM schema\", or "
                              "\"SHOW TABLES FROM database.schema\"");
    }
    if (IsInvalidSchema(qualified_name.Schema())) {
        showref->SetSchemaName(qualified_name.Name());
    } else {
        showref->SetCatalogName(qualified_name.Schema());
        showref->SetSchemaName(qualified_name.Name());
    }
    return WrapShowRef(std::move(showref));
}
```

This is where the actual DuckDB AST node — a `ShowRef` (a `TableRef` subclass,
§3.6) wrapped into a `QueryNode` — gets constructed, and it's also where a
semantic check that has nothing to do with grammar shape lives (rejecting a
qualified name with more parts than `catalog.schema` makes sense of). This
hand-written/generated split is the whole point of the two-script build
pipeline in §3.3: `generate_transformer_trampoline.py` regenerates the
mechanical plan and `Finalize*` function whenever the grammar or
`grammar_types.yml` changes, without ever touching the hand-written semantic
body in `transform_describe.cpp`.

Finally, each rule is entered under its grammar rule name in one static
array, so the generic transform-dispatch loop can find its functions and its
plan at runtime:

```cpp
// src/parser/peg/transformer/transform_generated_trampoline.cpp:6545, 7356-7357
static const TransformFrameOps GENERATED_FRAME_OPS[] = {
    // ... one entry per generated rule ...
    {"ShowTables", &PEGTransformerFactory::InitializePlannedTrampoline,
     &PEGTransformerFactory::FinalizeShowTablesTrampoline, &FRAME_PLAN_173},
```

Each entry is a `TransformFrameOps`: a name, an initialize function, a finalize
function, and a plan pointer (`peg_transformer.hpp:337-343`). Rules that still
have hand-written trampolines carry `nullptr` as their plan.

(One honest note on naming: the grammar's own `README.md` illustrates this
registration step with a `REGISTER_TRANSFORM(TransformShowTables)` macro call
inside a `PEGTransformerFactory::RegisterDescribe()` method — a clean,
readable stand-in for "this rule's transform is registered." What's actually
generated into `transform_generated_trampoline.cpp` in this exact checkout is
the `TransformFrameOps` static-array form shown just above — the same idea
(rule name maps to its transform functions), expressed as generated
data rather than a macro-annotated call site. If you go looking for a literal
`REGISTER_TRANSFORM` macro invocation for a specific rule and don't find one,
this is why — trust the generated file over the README's illustration for the
current mechanism, per this book's general policy on source vs. secondary
description.)

## 3.6 The parser → binder handoff: the AST type hierarchy

Whatever grammar rule matched, and whichever `Transform*` function built it,
every parse ultimately bottoms out in one of four small class hierarchies. This
is the contract Chapter 4 (the Binder) is written against — nothing about the
binder cares whether these types were built by the PEG parser or (on an older
release) `libpg_query`'s transform layer; only the parser side changed.

```
Parser::statements : vector<unique_ptr<SQLStatement>>   ← what ParseQuery populates,
                                                            and what gets handed to Binder

SQLStatement (src/parser/statement/*.cpp)
├── SelectStatement            (holds a `unique_ptr<QueryNode> node`)
├── InsertStatement, UpdateStatement, DeleteStatement, MergeIntoStatement
├── CreateStatement, AlterStatement, DropStatement
├── CopyStatement, ExplainStatement, PragmaStatement
├── ExecuteStatement, PrepareStatement, TransactionStatement
├── AttachStatement, DetachStatement, VacuumStatement
├── PassthroughStatement        (new: raw SQL for a CONNECT-ed database, end of §3.4)
└── ExtensionStatement          (the escape hatch TryParseExtensionStatement uses)

QueryNode (src/parser/query_node/)
├── SelectNode      a single SELECT ... FROM ... WHERE ... GROUP BY ... block
│     select_list : vector<unique_ptr<ParsedExpression>>
│     from_table   : unique_ptr<TableRef>
│     where_clause : unique_ptr<ParsedExpression>
│     groups       : GroupByNode
│     having, qualify : unique_ptr<ParsedExpression>
│     (ResultModifiers — ORDER BY / LIMIT — attach to the QueryNode base, not SelectNode itself)
├── SetOperationNode   UNION / EXCEPT / INTERSECT
└── RecursiveCTENode

ParsedExpression (src/parser/expression/)
├── ColumnRefExpression      column_names : vector<Identifier>  — a raw dotted name
│                            chain like ["o", "custkey"]; NOT yet resolved to a table
├── ConstantExpression, CastExpression, CaseExpression
├── FunctionExpression       function name + vector<FunctionArgument> — covers both
│                            scalar and aggregate calls; nothing distinguishes
│                            "aggregate" from "scalar" until the binder looks the
│                            name up in the catalog
├── ComparisonExpression, ConjunctionExpression, OperatorExpression
├── SubqueryExpression       subquery : unique_ptr<SelectStatement>,
│                            subquery_type : SubqueryType (SCALAR/EXISTS/IN/ANY),
│                            child : unique_ptr<ParsedExpression> (for IN/ANY forms)
├── WindowExpression, StarExpression, BetweenExpression
└── ParameterExpression, PositionalReferenceExpression, LambdaExpression

TableRef (src/parser/tableref/)
├── BaseTableRef        a bare table name (+ optional alias)
├── JoinRef             left, right : unique_ptr<TableRef>;
│                       condition : unique_ptr<ParsedExpression>; type : JoinType
├── SubqueryRef, TableFunctionRef, PivotRef
├── ExpressionListRef   VALUES (...)
└── ShowRef, EmptyTableRef, ColumnDataRef, DelimGetRef
```

The detail worth internalizing from the field names above, because it's the
clearest possible statement of "parsing is purely syntactic": a
`ColumnRefExpression` stores a raw `vector<Identifier>` — for `o.custkey`,
literally the two strings `"o"` and `"custkey"`, in that order, with no
indication yet of whether `"o"` is a table alias, a struct field access, or
something else. The parser cannot tell, and doesn't try — that disambiguation
happens only once the Binder has a `BindContext` populated with what tables
and columns are actually in scope (Chapter 4). Likewise, a `FunctionExpression`
for `SUM(l.extendedprice)` and one for, say, `LENGTH(x)` are exactly the same
C++ type at this stage; nothing in the parser's output distinguishes an
aggregate function from a scalar one, because that distinction lives in the
catalog's function registry, which the parser never consults.

## 3.7 Tracing Query A through the parser

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

The tokenizer first reduces this to a flat stream of `MatcherToken`s (keywords
`SELECT`/`FROM`/`JOIN`/`ON`/`WHERE`/`GROUP`/`BY`/`ORDER`/`DESC`/`LIMIT`, as
classified by the grammar's keyword helper; identifiers `o`, `custkey`, `l`,
`extendedprice`, `orders`, `lineitem`, `orderkey`, `shipdate`, `total`; the
string literal `'1998-01-01'`; the number literal `10`; operator tokens `.`,
`=`, `<`). Each `.` in `o.custkey` is an operator token because a letter
follows it; only a dot followed by a digit starts a number (the tokenizer fix
in §3.4). The PEG
matcher then walks that stream against the compiled grammar, and here's which
named rule from §3.2 is responsible for which piece:

| Query fragment | Grammar rule(s) matched | AST node produced |
|---|---|---|
| `SELECT o.custkey, SUM(l.extendedprice) AS total` | `SelectClause <- 'SELECT' DistinctClause? TargetList?`, `TargetList <- List(AliasedExpression)`, `AliasedExpression <- ... / ExpressionAsCollabel / ...` | `select_list` = `[ColumnRefExpression(["o","custkey"]), FunctionExpression("sum", [ColumnRefExpression(["l","extendedprice"])])` with an alias `"total"` attached] |
| `FROM orders o JOIN lineitem l ON o.orderkey = l.orderkey` | `FromClause <- 'FROM' List(TableRef)`, `TableRef <- InnerTableRef JoinOrPivot*`, `RegularJoinClause <- Asof? JoinType? 'JOIN' TableRef JoinQualifier`, `OnClause <- 'ON' Expression` | `from_table` = a `JoinRef` with `left` = `BaseTableRef("orders", alias "o")`, `right` = `BaseTableRef("lineitem", alias "l")`, `type` = `JoinType::INNER` (no explicit `JoinType` keyword present, so the default applies), `condition` = a `ComparisonExpression` |
| `o.orderkey = l.orderkey` | `ComparisonExpression <- BetweenInLikeExpression ComparisonExpressionTail*`, `ComparisonOperator <- OperatorEqual / ...` | `ComparisonExpression(COMPARE_EQUAL, ColumnRefExpression(["o","orderkey"]), ColumnRefExpression(["l","orderkey"]))` — this becomes the `JoinRef`'s `condition` |
| `WHERE l.shipdate < DATE '1998-01-01'` | `WhereClause <- 'WHERE' Expression`, down through the precedence chain to `ComparisonExpression`, with `DATE '1998-01-01'` matching `TypeLiteral <- Type StringLiteral` | `where_clause` = `ComparisonExpression(COMPARE_LESSTHAN, ColumnRefExpression(["l","shipdate"]), ConstantExpression(Value::DATE("1998-01-01")))` |
| `SUM(l.extendedprice)` | `FunctionExpression <- FunctionIdentifier FunctionExpressionArguments ...` | `FunctionExpression("sum", [ColumnRefExpression(["l","extendedprice"])])` — nothing here marks this as an aggregate yet; that's a catalog lookup, deferred to the binder |
| `GROUP BY o.custkey` | `GroupByClause <- 'GROUP' 'BY' GroupByExpressions`, `GroupByList <- List(GroupByExpression)` | `groups` populated with one `GroupByBaseExpression` wrapping `ColumnRefExpression(["o","custkey"])` |
| `ORDER BY total DESC` | `OrderByClause <- 'ORDER' 'BY' OrderByExpressions`, `OrderByExpression <- Expression DescOrAsc? NullsFirstOrLast?` | an `OrderByNode` referencing the `total` alias, with `OrderType::DESCENDING` |
| `LIMIT 10` | `LimitClause <- 'LIMIT' LimitValue`, `LimitExpression <- Expression '%'?` | a `LimitModifier` holding `ConstantExpression(10)` |

The whole statement is a `SelectStatement` whose `node` is one `SelectNode`
built up exactly this way — which is precisely the shallow preview Chapter 1
gave (§1.3, Stage 1); this section is that preview made exhaustive. Nothing in
this tree has touched the catalog: the parser doesn't know `orders` or
`lineitem` exist as tables, doesn't know `custkey`/`orderkey`/`extendedprice`/
`shipdate` are real columns of those tables, and doesn't know `sum` is a
registered aggregate function rather than, say, a typo for a scalar function
that doesn't exist. All of that is Chapter 4's job.

## 3.8 Tracing Query B through the parser: a subquery is just syntax

Query B, the book's running correlated-subquery example:

```sql
SELECT c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey) AS n_orders
FROM customer c;
```

The outer query parses exactly like any other `SimpleSelect`: `select_list`
gets a `ColumnRefExpression(["c","name"])` and one more expression for the
`(SELECT ...) AS n_orders` term; `from_table` becomes a `BaseTableRef("customer",
alias "c")`. The interesting part is what that second `select_list` entry
becomes. The grammar rule doing the work is:

```
# src/parser/peg/grammar/statements/expression.gram:62
SubqueryExpression <- SubqueryNot? SubqueryExists? SubqueryReference
```

```
# src/parser/peg/grammar/statements/select.gram:189
SubqueryReference <- Parens(SelectStatementInternal)
```

`(SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey)` matches
`SubqueryReference` — a parenthesized full `SelectStatementInternal`, which
means the entire inner query is parsed *recursively*, using exactly the same
`SimpleSelect` machinery §3.7 just walked through for Query A, producing its own
complete `SelectNode`: `select_list` = `[FunctionExpression("count", [StarExpression])]`,
`from_table` = `BaseTableRef("orders", alias "o")`, `where_clause` =
`ComparisonExpression(COMPARE_EQUAL, ColumnRefExpression(["o","custkey"]),
ColumnRefExpression(["c","custkey"]))`. Because neither `SubqueryNot` nor
`SubqueryExists` matched (this isn't a `NOT EXISTS`/`EXISTS` form), the
transformer produces a plain `SubqueryExpression` wrapping that inner
`SelectStatement`, with `subquery_type` set to `SubqueryType::SCALAR` (a bare
`(SELECT ...)` used where a single value is expected, as opposed to
`SubqueryType::EXISTS` or `SubqueryType::IN`), and `child` left null (that field
is only populated for the `IN`/`ANY` subquery forms — see the `TYPE(D)`-style
usages in `InClause`/`OtherOperator` in §3.2's precedence table).

The one point this book keeps returning to at this stage, because it matters
for Chapter 6: **this `SubqueryExpression` is not, and cannot yet be, marked
"correlated."** The parser's job ends the moment it has produced a syntactically
valid tree — a `ColumnRefExpression(["c","custkey"])` sitting inside the inner
query's `where_clause` looks, to the parser, exactly like any other column
reference: a bare two-part dotted name. Nothing in `ColumnRefExpression`,
`SubqueryExpression`, or any type in §3.6's hierarchy has a "this references an
outer scope" flag, because *scope* is a binding-time concept — it requires
knowing which tables are in context at which nesting level, which is exactly
the `BindContext` machinery Chapter 4 builds and Chapter 6 exploits to detect
that `c.custkey` inside the subquery actually resolves to the *outer* query's
`customer c`, not anything inside the subquery's own `FROM orders o`. At the
parser stage, both queries in this pair are just nested syntax; the discovery
that one of them is correlated is a downstream, semantic fact, not a syntactic
one.

## 3.9 The error path: a deliberately malformed query

DuckDB's own test suite includes a query built specifically to exercise the PEG
matcher's failure path, `test/sql/peg_parser/enable_peg_parser.test` (full file
reproduced, verified against this checkout):

```
# name: test/sql/peg_parser/enable_peg_parser.test
# description: Test enabling and disabling the PEG Parser
# group: [peg_parser]

statement error
select xvariable a = 1;
----
Parser Error: syntax error at or near "="

# Here we should get the error from the parser override
statement error
select xvariable a = 1;
----
Parser Error: syntax error at or near "="

# Just check that it works properly
query I
SELECT * FROM values(1);
----
1
```

Walking `select xvariable a = 1;` through the machinery in §3.4: the
`Statement` alternative that gets furthest is `SelectStatement`. Its
`SelectClause <- 'SELECT' DistinctClause? TargetList?` takes `select`, and
`TargetList <- List(AliasedExpression)` takes `xvariable a` as one item: the
expression `xvariable` with the alias `a`, written without `AS`. (`a` is a
plain identifier, so the alias-without-`AS` alternative accepts it.) After
that, a `SELECT` may continue with `,`, `FROM`, `WHERE` and other clauses, or
end with `;`. The next token is `=`, which none of them accept, so the match
fails at the `=` token.

The core grammar has no `ExpressionStatement <- List(ExpressionAlias)` rule
(it lives in the shell's grammar extension, end of §3.4), so `SelectStatement`
is the one that gets furthest. The furthest token any alternative reaches is
`=`.

That failure surfaces through `TransformTopLevelStatement`'s own error path —
verified directly against this checkout's source, and worth showing because it
is *literally* where the exact test's error text comes from:

```cpp
// src/parser/peg/transformer/peg_transformer_factory.cpp:67-86
auto match_result = grammar.TopLevelStatementMatcher().MatchParseResult(state);
process_allocator.FreeAll();
if (!match_result.IsSuccess()) {
    // syntax error — surface as a parser exception in the same shape as Transform()
    auto token_stream = token_iterator.ToString();
    idx_t error_token_idx = state.GetMaxTokenIndex();
    if (error_token_idx >= token_iterator.Size()) {
        error_token_idx = token_iterator.Size() - 1;
    }
    // Walk back past the EOI sentinel so the error message names a real token.
    if (error_token_idx > 0 &&
        (token_iterator.GetToken(error_token_idx).type == TokenType::END_OF_INPUT ||
         token_iterator.GetToken(error_token_idx).type == TokenType::END_OF_INPUT_AUTOCOMPLETE)) {
        error_token_idx--;
    }
    auto &error_token = token_iterator.GetToken(error_token_idx);
    auto error_message = "syntax error at or near \"" + error_token.text + "\"";
    throw ParserException::SyntaxError(token_stream, error_message,
                                       QueryLocation(error_token.offset, error_token.length));
}
```

`state.GetMaxTokenIndex()` tracks the furthest position into the token stream
any matcher attempt reached before the whole match gave up — the standard PEG
technique for producing a *useful* error location despite backtracking (a naive
implementation would report wherever the *last* alternative gave up, which is
often much earlier than the point a human would call "where it broke"; tracking
the maximum position reached across every attempted alternative gives the
furthest-forward, most informative failure point instead). For this query, that
furthest point is the `=` token — hence `syntax error at or near "="`, matching
the test's expected output exactly. Because this particular test doesn't
register any parser extensions, `Parser::ParseQuery`'s `TryParseExtensionStatement`
fallback (§3.4, step 4) finds nothing to claim the failure, and the
`ParserException` propagates unmodified to the caller as
`Parser Error: syntax error at or near "="` — precisely the two identical
`statement error` blocks the test asserts (the second comment, *"Here we should
get the error from the parser override"*, is a holdover from when this test
exercised an actual override toggle; in this checkout both blocks take the same
code path and produce the same error). The test's final block,
`SELECT * FROM values(1);`, exists purely to confirm that the engine still works
normally after two consecutive parse failures — nothing about a failed
`ParserException` is left mutating shared state that would corrupt a
subsequent, well-formed parse.

## 3.10 What crosses the boundary into Chapter 4

By the end of this stage, `Parser::statements` holds one `unique_ptr<SQLStatement>`
per top-level statement in the query string — for a single `SELECT`, that's one
`SelectStatement` wrapping one `SelectNode`, built entirely out of the
`ParsedExpression`/`TableRef` vocabulary in §3.6. Everything about this tree is
purely syntactic: names are raw strings or dotted-name chains, functions are
undifferentiated by kind, and subqueries carry no notion of which of their
column references reach outside their own scope. Chapter 4 (the Binder) is
where every one of those open questions gets closed — and, per the
architectural note in Chapter 1 (§1.1, "Difference #1"), it closes them by
building the logical plan directly as a byproduct of resolving names, not as a
separate step afterward.
