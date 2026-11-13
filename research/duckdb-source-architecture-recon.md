# DuckDB Internals — Architectural Research Map

**Purpose:** raw research/reconnaissance findings used as source material for a multi-chapter internals document on DuckDB (SQL → Parser → Binder/Planner → Logical Plan → Optimizer → Physical Plan → Execution → storage/transactions/indexes). This file is findings only; the chapters were written from it.

**Source examined:** https://github.com/duckdb/duckdb, branch `v2.0-cyanoptera`, commit `cb5673685153` (2026-11-12; `git describe --tags` prints `v2.0.0-1041-gcb567368515`). The tag `v2.0.0` (commit `d54cbbe3df1`, 2026-10-20) lies on this branch. The 1,041 commits after it are mostly fixes for the v2.0.1 patch release. The stable releases before v2.0.0 are the `v1.5.x` line. One finding below (the parser rewrite) is branch-load-bearing — call it out explicitly wherever the doc discusses the parser, and decide which DuckDB version the chapter targets.

All file paths below are relative to the root of https://github.com/duckdb/duckdb at commit `cb5673685153` (https://github.com/duckdb/duckdb/tree/cb5673685153159d22bf1042ab444273c9ad268a). External works are cited with numbers like `[31]` from the reference list in `chapters/appendix-c-references.md`. Source and test files keep the inline `path:line` form.

---

## 0. Headline finding: the parser has been rewritten (read this before section 1)

The common assumption — "DuckDB's parser is derived/transpiled from PostgreSQL's grammar via `libpg_query`" — is **correct for v1.5.x and earlier, but false for v2.0**. Exhaustive greps confirm zero trace of the historical pipeline in this branch:

```
$ find . -iname "*libpg_query*" -not -path "./.git/*"
(no output)

$ grep -rl "PostgresParser" --include=*.cpp --include=*.hpp . 2>/dev/null | grep -v .git
(no output)
```

**Historical (v1.5.x and earlier, NOT what's in this checkout):** DuckDB vendored a fork of `libpg_query` (itself a tool that transpiles PostgreSQL's bison/flex grammar into a standalone C library) at `third_party/libpg_query`, wrapped it in a `PostgresParser` class, and converted the resulting Postgres `List*`/`Node*` parse tree into DuckDB's own AST via a `src/parser/transform/*.cpp` directory. **None of this exists in this checkout** — `third_party/` no longer contains a pg-query directory at all (its current contents are listed in §1 below).

**Current (v2.0, branch `v2.0-cyanoptera`):** DuckDB has replaced the entire Postgres-derived pipeline with a **hand-written, native PEG (Parsing Expression Grammar) parser**, "moved into core" from the `autocomplete` extension:

```
$ git log --oneline --diff-filter=A -- src/parser/peg | tail -5
d967531296 Auto generate detach
0c7c697e3b Add first version of generated file as well
056c08546f Add generated Internal wrapper, fix up written case
60ac23238b Fix CLAUDE.md for parser
3d2ef2153f Moving PEG into core
```

A Postgres-style parser is available as an option: `SET enable_legacy_parser = true` autoloads the out-of-tree `legacy_parser` extension, which provides a Postgres-style parser with v1.5 quirks, for example `~` as a full regex match (`72d330366ac`, PR #26479).

The PEG parser was merged into core in September 2026, and v2.0.0 (tagged 2026-10-20) ships it, so it is the parser of a released DuckDB. This is a major architectural change and should be flagged prominently, because most outside material still describes the v1.5.x line. Its main internal parts are a planned (table-driven) transformer, FIRST-set pruning in the matcher, a `PEGRuleId` enum, a `bare_label_keyword.list`, and a passthrough grammar for `CONNECT` (all in §1).

---

## 1. Parser

### 1a. Grammar definition and transpilation tooling

- **Grammar source (`.gram` files):** `src/parser/peg/grammar/statements/` — one file per statement type, 41 files total:
  ```
  alter.gram             create_table.gram         external_resource_source.gram
  analyze.gram           create_trigger.gram       insert.gram
  attach.gram            create_type.gram          load.gram
  call.gram              create_view.gram          merge_into.gram
  checkpoint.gram        deallocate.gram           pivot.gram
  comment.gram           delete.gram               pragma.gram
  common.gram            describe.gram             prepare.gram
  connect.gram           detach.gram               select.gram
  copy.gram              drop.gram                 set.gram
  create_index.gram      execute.gram              transaction.gram
  create_macro.gram      explain.gram              update.gram
  create_schema.gram     export.gram               use.gram
  create_secret.gram     expression.gram           vacuum.gram
  create_sequence.gram   external_resource.gram
  ```
- **Keyword lists:** `src/parser/peg/grammar/keywords/` (in core, not under `extension/autocomplete/`) — `reserved_keyword.list` (cannot be used as identifiers, e.g. `SELECT`, `FROM`), `unreserved_keyword.list` (can be used as identifiers, e.g. `ABORT`, `ABSOLUTE`), `column_name_keyword.list`, `func_name_keyword.list`, `type_name_keyword.list`. A keyword must appear in exactly one of these five category lists; the build script validates this.

  There is a sixth list, `bare_label_keyword.list`. It is not a category. It says which keywords may be used as a column alias without `AS`, as in `SELECT 1 name`. It follows Postgres's `bare_label_keyword` idea. Every entry must also be in a category list. Keywords that can come right after an expression are left out on purpose (interval units such as `DAY`, `FILTER`, `OVER`, `VARYING`, `EXCLUDE`). So `SELECT d + 1 day` is a syntax error, not a column called `day` (`src/parser/peg/README.md`, "Keywords" section).
- **Small syntax facts worth knowing.** A backtick-quoted name such as `` `my col` `` is a quoted identifier with the same rules as `"my col"` (no case folding; escape by doubling; PR #26519). The JSON arrow `->` sits in the operator precedence level and arrow chains associate to the left, as in PostgreSQL, so `j->'a'->>'b' = 'x'` works (PR #28240). `DESTROY`, `REGISTER` and `RESOURCES` are unreserved keywords. `PRAGMA name=value` used to change a setting logs a deprecation warning that points to `SET name=value` (PR #27280).
- **Grammar syntax (PEG operators)**, documented in `src/parser/peg/README.md` (254 lines):
  - Rules: `RuleName <- RuleDefinition`
  - Ordered choice: `A / B` (PEG semantics — first successful match wins, no ambiguity; more-specific alternatives must be listed before general ones, e.g. compound interval types like `YEAR TO MONTH` must precede bare `YEAR`)
  - Sequences: space-separated elements, e.g. `CastExpression <- CastOrTryCast Parens(Expression 'AS' Type)`
  - Optional: `?` (e.g. `CreateStatement <- 'CREATE' OrReplace? Temporary? CreateStatementVariation`)
  - Repeat: `*` (zero or more), `+` (one or more)
  - Grouping: parentheses without creating a named rule
  - Parameterized rules (macros), e.g. built into `common.gram`:
    ```
    List(D) <- D (',' D)* ','?
    Parens(D) <- '(' D ')'
    ```
  - Negative lookahead: `!` (e.g. `PlainIdentifier <- !ReservedKeyword <[a-z_]i[a-z0-9_]i*>`, `src/parser/peg/grammar/statements/create_table.gram:76`) — parsed but **ignored**. The README has the line `TODO(Dtenwolde): Implement negative lookahead` (`src/parser/peg/README.md:103`), and the grammar reader simply skips the `!` and reads what follows (`src/parser/peg/peg_parser.cpp:128-131`):
    ```cpp
    		if (IsOp('!')) {
    			// FIXME: NOT ignored when parsing
    			Advance();
    			return ParsePrimary();
    		}
    ```
  - Special matcher-handled tokens (not grammar-expanded): `Identifier`/`ReservedIdentifier`, `TableName`/`ColumnName`/`SchemaName` (context-aware identifier matching), `StringLiteral`, `NumberLiteral`, `OperatorLiteral` — full list in `matcher.cpp`.
- **Build/transpilation tooling:**
  - `scripts/parser/build_grammar.sh` — the top-level regeneration script. It runs `scripts/parser/inline_grammar.py` twice: once with `--grammar-file` to combine all `.gram` files + keyword lists into a single `src/include/duckdb/parser/peg/inlined_grammar.gram` (for debugging), and once by default to generate `src/include/duckdb/parser/peg/inlined_grammar.hpp` and `src/parser/peg/keyword_map.cpp` (keyword lookup tables). The header embeds the whole grammar as a byte array, `const unsigned char INLINED_PEG_GRAMMAR[] = { 80, 114, ... }` (`inlined_grammar.hpp:7`), not as a string literal. The script then runs `scripts/parser/generate_transformer_trampoline.py --write` and finally `make format-parser-grammar` (`scripts/parser/build_grammar.sh`). Note: the README still says the inliner is `extension/autocomplete/inline_grammar.py`; that path is stale. The real script is `scripts/parser/inline_grammar.py`.
  - `scripts/parser/generate_transformer_trampoline.py` — reads `scripts/parser/grammar_types.yml` (maps grammar rule names to C++ return types) and regenerates: `peg_transformer.hpp` (transformer process declarations + transform result types), `transform_generated_trampoline.cpp` (transformer process implementations + a registration table), matcher rules in `matcher.cpp` / `compiled_grammar.cpp`, and `src/include/duckdb/parser/peg/peg_rule_id.hpp` / `src/parser/peg/peg_rule_id.cpp` (a `PEGRuleId` enum with one constant per built-in grammar rule). Helper modules sit next to it: `scripts/parser/grammar_types.py`, `transformer_plan.py`, `transformer_trampoline.yml`, `transformer_trampoline_config.py`. It can be run standalone with `--write` to regenerate transformer processes without rebuilding the inlined grammar.
  - **Do not hand-edit generated files** (`inlined_grammar.gram`, `inlined_grammar.hpp`, `keyword_map.cpp`, `transform_generated_trampoline.cpp`, `peg_rule_id.*`) — the README is explicit about this.
  - Transformer implementation pattern. **Warning:** the README example (`src/parser/peg/README.md:216-226`) shows an older hand-written style, where a transform takes a raw `ParseResult` and picks children by index, and is registered with a `REGISTER_TRANSFORM(...)` macro in a `RegisterDescribe()` method. The code does not use that style (`REGISTER_TRANSFORM(` appears only in the README). The real code is split in two:
    1. A **hand-written semantic body** that receives already-transformed, typed children. From `src/parser/peg/transformer/transform_describe.cpp:51-67`:
       ```cpp
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
    2. A **generated trampoline** in `src/parser/peg/transformer/transform_generated_trampoline.cpp` (18,706 lines). The registration table maps a rule name to an initialize function, a finalize function and a static *frame plan* (`transform_generated_trampoline.cpp:7356-7357`):
       ```cpp
           {"ShowTables", &PEGTransformerFactory::InitializePlannedTrampoline,
            &PEGTransformerFactory::FinalizeShowTablesTrampoline, &FRAME_PLAN_173},
       ```
       The frame plan says which parse-tree children to transform and into which result slot. For `ShowTables`, child 3 (`QualifiedName`) goes to slot 1 and child 0 (`ShowOrDescribe`) to slot 0 (`transform_generated_trampoline.cpp:2294-2308`). The finalize step then takes the typed results out of the slots and calls the semantic body (`transform_generated_trampoline.cpp:12366-12371`):
       ```cpp
       arena_ptr<TransformResultValue>
       PEGTransformerFactory::FinalizeShowTablesTrampoline(PEGTransformer &transformer, GeneratedTransformProcess &process) {
       	auto show_or_describe = process.TakeResult<ShowType>(0);
       	auto qualified_name = process.TakeResult<QualifiedName>(1);
       	auto result = TransformShowTables(transformer, show_or_describe, qualified_name);
       	return transformer.MakeResult<unique_ptr<QueryNode>>(std::move(result));
       }
       ```
    Most rules share one generic `InitializePlannedTrampoline` (or `InitializePlannedChoiceTrampoline` for choice rules, and `FinalizePassthroughTrampoline` for rules that pass a child through). Rules differ only in a static data table, the `TransformFramePlan` (`src/include/duckdb/parser/peg/transformer/peg_transformer.hpp:325-335`), so the lookup table is one static array. Each planned child has a `TransformChildKind`: `REQUIRED`, `OPTIONAL`, `LIST`, `OPTIONAL_LIST`, `REPEAT` or `OPTIONAL_REPEAT` (`peg_transformer.hpp:264-277`). The generic code that walks these plans lives in `src/parser/peg/transformer/transform_frame_plan.cpp`. Children are named by `PEGRuleId`, not by string. The README says transformers "schedule child transformations on the explicit transform stack", that is, the transformer does not recurse on the C++ call stack. One owning heap vector holds the transform frames.
  - **Expression depth limit.** While it transforms an expression, the transformer rejects nesting deeper than the `max_expression_depth` setting. Operator chains, nested types and nested FROM-clause subqueries count towards the depth; plain `AND`/`OR` chains do not (PR #26771 and follow-ups).
  - Parse-result types consumed by transformers: `ListParseResult` (sequence `A B C`, access via `list_pr.Child<T>(index)` / `GetChild(index)`), `ChoiceParseResult` (choice `A / B`, access via `.result`), `OptionalParseResult` (`A?`, via `.HasResult()`/`.optional_result`), `RepeatParseResult` (`A+`/`A*`, iterable), `IdentifierParseResult`/`StringLiteralParseResult`/`NumberParseResult`/`KeywordParseResult` (leaf tokens, `.result` is the raw string; `StringLiteralParseResult` also has `.ToExpression()`). (Table at `src/parser/peg/README.md`, "Parse Result Types".)

### 1b. Runtime parser engine (`src/parser/peg/`)

- `peg_parser.cpp` — despite the name, this is the reader for the *grammar text* itself (`struct PEGParser`, `PEGRule`, `PEGExpression` in `src/include/duckdb/parser/peg/peg_parser.hpp:10-48`). It turns the inlined `.gram` text into rule objects. It is not the SQL entry point; that is `Parser::ParseQuery` → `PEGTransformerFactory::TransformTopLevelStatement` (§1c).
- `matcher.cpp`, `matcher_factory.cpp`, `matcher_process.cpp`, `matcher_stack.cpp` — the PEG matching machinery: sequence/choice/optional/repeat combinators over the compiled grammar.
- `matcher_first_set.cpp` (`d58ce106373`, PR #26451) — computes a FIRST set for each matcher: the set of tokens that a rule can start with (keyword and symbol ids plus token classes such as word, operator, string and number), and whether the rule can match nothing (`MatcherFirstSet::Merge`, `src/parser/peg/matcher_first_set.cpp:10`). A choice can then skip any alternative that cannot start at the current token, instead of trying it and backtracking. The PR measured the median parse time per statement at 29.8 µs with FIRST sets against 65.7 µs without. Autocomplete mode does not use this pruning.
- **Collapsible rules.** Each precedence level (`Expression` → `LogicalOrExpression` → `LogicalAndExpression` → …) has the shape `X <- Y Tail*`. Rules listed as *collapsible* hand out their only child directly when exactly one child produced a result, so a plain literal like `42` is not wrapped in one `ListParseResult` per level (PR #26090; the transformer keeps the list in `collapsible_rules`).
- `peg_rule_id.cpp` (generated) — names for the `PEGRuleId` enum, one id per built-in rule; rules added at run time by grammar extensions get ids past `PEGRuleId::BUILTIN_COUNT` (`src/parser/peg/README.md`).
- `passthrough_dialect.cpp` — a small grammar used while a session is `CONNECT`-ed to another database. Its `PassthroughTokenMatcher` accepts any single token that is not a statement end, and the rule `PassthroughStatement <- PassthroughToken+` turns the whole statement into an unparsed `PassthroughStatement` whose text is forwarded as-is (`src/parser/peg/passthrough_dialect.cpp`, `src/include/duckdb/parser/peg/passthrough_dialect.hpp`, `src/include/duckdb/parser/statement/passthrough_statement.hpp`).
- `parser_packrat.cpp` — packrat memoization (the classic PEG technique that keeps parsing linear-time despite the grammar's unbounded backtracking/ordered-choice semantics). Packrat entries are indexed by token position. Parse results, their children, the packrat cache and the transformer's processes all live in arenas.
- `compiled_grammar.cpp` / `parsed_grammar.cpp` — the compiled/loaded grammar object, exposed as `CompiledGrammar` (see `Parser::GetGrammar()` below). The default grammar is cached in `CompiledGrammar::DefaultGrammar`. Connections that load the same grammar extensions share one compiled grammar, from a cache with LRU eviction. A `Parser` is cheap: one is built per statement, and the warm state lives in the shared `CompiledGrammar`. `Parser` has no default constructor; callers pass options (`Parser::GetBuiltinParser()` at `src/parser/parser.cpp:36` uses `ParserOptions::Builtin()`).
- `tokenizer/` — `base_tokenizer.cpp` (694 lines; the hand-written lexer — not flex-generated), `parser_tokenizer.cpp` (47 lines), `highlight_tokenizer.cpp` (19 lines, used for `Parser::Tokenize` / syntax highlighting).
- `keyword_helper/` — `duckdb_keyword_helper.cpp`, `default_keyword_maps.cpp`, `parsed_grammar_keyword_helper.cpp`, `grammar_literal_table.cpp` — keyword classification (`DuckDBKeywordHelper::Instance().GetKeywordCategory(...)`, called from `src/parser/keyword_helper.cpp:9-13`).
- `transformer/` — one file per statement kind, replacing the old `src/parser/transform/` directory (which no longer exists): `transform_select.cpp`, `transform_create_table.cpp`, `transform_expression.cpp`, `transform_merge_into.cpp`, `transform_alter.cpp`, `transform_call.cpp`, `transform_insert.cpp`, `transform_update.cpp`, `transform_delete.cpp`, `transform_copy.cpp`, `transform_pivot.cpp`, `transform_attach.cpp`, `transform_detach.cpp`, `transform_connect.cpp`, `transform_pragma.cpp`, `transform_prepare.cpp`, `transform_execute.cpp`, `transform_explain.cpp`, `transform_export.cpp`, `transform_vacuum.cpp`, `transform_set.cpp`, `transform_load.cpp`, `transform_analyze.cpp`, `transform_checkpoint.cpp`, `transform_comment.cpp`, `transform_create_view.cpp`, `transform_create_schema.cpp`, `transform_create_sequence.cpp`, `transform_create_macro.cpp`, `transform_create_type.cpp`, `transform_create_index.cpp`, `transform_create_secret.cpp`, `transform_create_trigger.cpp`, `transform_drop.cpp`, `transform_use.cpp`, `transform_deallocate.cpp`, `transform_transaction.cpp`, `transform_describe.cpp`, `transform_generic_copy_option.cpp`, `transform_external_resource.cpp`, `transform_external_resource_source.cpp`, `transform_common.cpp`, `transform_frame_plan.cpp` (the generic code that walks the static frame plans, see §1a), `transform_generated_trampoline.cpp` (generated), plus `peg_transformer.cpp` and `peg_transformer_factory.cpp` (the dispatch/registration core).
- `autocomplete_core.cpp` (287 lines) and `sql_formatter.cpp` (1690 lines) — shared infrastructure: the same grammar drives SQL autocompletion suggestions and a SQL formatter, in addition to parsing. (The formatter also formats and highlights the output of `EXPLAIN (SQL)`; see §2a.)
- `grammar_change.cpp` / `dialect_extension.cpp` — hooks for grammar dialect variation (e.g. extensions altering accepted syntax).
- `operator_matcher.cpp` (30 lines) — a small helper, `OperatorMatcher::HasSpecialPrecedence`, that says which operator symbols (`=`, `<`, `>`, `~`, `->`, `<=`, `>=`, `!=`, `<>`, ...) have their own precedence level; the matcher class itself is declared in `src/include/duckdb/parser/peg/matcher/operator_matcher.hpp`. Precedence levels are otherwise written as a ladder of rules in `expression.gram` (`Expression <- LogicalOrExpression`, ...).

### 1c. Entry point / control flow

`src/parser/parser.cpp`, `Parser::ParseQuery(const string &query_p)` (lines 253-324; the extension-override loop is shortened here):

```cpp
void Parser::ParseQuery(const string &query_p) {
	const string query = NormalizeSQLString(query_p);
	if (options.extensions) {
		// ... parser_override extension hooks (STRICT_OVERRIDE / DEFAULT_OVERRIDE) ...
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
	// ... last statement runs to query.size(); per-statement `query` substrings; CREATE info->sql ...
}
```

Two details of this loop matter. (1) `ParserTokenizerBehavior` owns the token vector, and `TokenIterator` is built from it. The iterator points at the normalized query, so a grammar-extension transform can call `transformer.token_iterator.Source()` and cut out the original text, comments included. (2) Each statement's text starts where the previous statement ended (after its semicolons). So a comment in front of a statement belongs to that statement. Example: for `/* leading */ SELECT current_query();`, `current_query()` returns the comment too. This is `a15352e5994` "Add leading comments in SQL" (PR #28597). It fixed a v2.0.0 bug where a leading comment was lost and a comment between two statements was attached to the one before; v1.5.6 already behaved this way.

- `NormalizeSQLString` (`src/parser/parser.cpp:241-251`) first validates UTF-8 (`ValidateUTF8Query`, line 79 — throws before `StripUnicodeSpaces` walks multi-byte sequences, "so the tokenizer never reads past bad bytes"; the code comment at line 78 cites an oss-fuzz bug: "see ossfuzz clusterfuzz-test-24"), then strips Unicode space look-alikes (NBSP, various U+2000-range spaces, ideographic space, BOM; `Parser::StripUnicodeSpaces`, line 95) that could otherwise be used to smuggle characters past naive tokenizers. There is a fast path: if `ValidateUTF8Query` reports the query is pure ASCII, it is returned unchanged and the strip step is skipped (line 243).
- `Parser::ParseTopLevelStatement` (line 369) just calls `PEGTransformerFactory::TransformTopLevelStatement(token_iterator, options, compiled_grammar)` (`src/parser/peg/transformer/peg_transformer_factory.cpp:51`) — i.e. the whole "parse" step is: tokenize once for the entire query, then match one top-level statement's worth of tokens and transform it into DuckDB AST. Strictly, there *is* a short-lived untyped parse result: `TransformTopLevelStatement` first runs `grammar.TopLevelStatementMatcher().MatchParseResult(state)` with `MatchMode::BUILD_PARSE_RESULT` and a per-statement packrat cache (`ParserPackratCache`), building a `ParseResult` tree in an arena; the transformer then turns that tree into AST and the arena is freed. There is no persistent intermediate stage.
- `Parser::ThrowParserOverrideError` (line 228) / the `options.extensions->ParserExtensions()` loop (lines 255-282) and `TryParseExtensionStatement` (line 326) implement a **parser-extension fallback protocol**: on a native PEG parse failure, registered extensions get the remaining token stream (as `SimpleToken`s) and report back how many tokens they consumed; this lets e.g. a custom-dialect extension parse a trailing clause the core grammar doesn't understand, with `AllowParserOverride::STRICT_OVERRIDE`/`DEFAULT_OVERRIDE` settings controlling precedence.
- `Parser::Tokenize` (line 377) — public tokenization entry point used for SQL syntax highlighting, distinct from the full parse path; classifies tokens into `SimplifiedTokenType` (`SIMPLIFIED_TOKEN_IDENTIFIER`, `_NUMERIC_CONSTANT`, `_STRING_CONSTANT`, `_OPERATOR`, `_KEYWORD`, `_COMMENT`, plus the error-display types `_ERROR`, `_ERROR_EMPHASIS`, `_ERROR_SUGGESTION`; `src/include/duckdb/parser/simplified_token.hpp:18-26`).
- `Parser::TokenizeError` (line 423) — a separate, elaborate tokenizer specifically for **formatting parser error messages** (locating `Error:`, the `LINE n:` marker, the `^` caret, and re-tokenizing the quoted query snippet to emphasize the erroring token) — this is purely a diagnostics/pretty-printing utility, not part of the parse path itself.
- Convenience parsers built on top of `ParseQuery` by constructing a mock query string and asserting on the resulting statement shape: `Parser::ParseExpressionList` (prefixes `"SELECT "`), `Parser::ParseGroupByList` (`"SELECT 42 GROUP BY %s"`), `Parser::ParseOrderList` (`"SELECT * FROM tbl ORDER BY " + ...`), `Parser::ParseUpdateList` (`"UPDATE tbl SET " + ...`), `Parser::ParseValuesList` (`"VALUES " + ...`), `Parser::ParseColumnList`/`ParseColumnDefinition` (`"CREATE TABLE tbl (" + ... + ")"`) — all in `src/parser/parser.cpp`, lines 578-724 (`ParseSelectNode` at 578 and `ParseSingleExpression` at 623 are in the same block).

### 1d. Third-party directory contents (confirms no libpg_query)

```
$ find third_party -maxdepth 1 -type d
third_party/httplib        third_party/re2            third_party/pdqsort
third_party/jaro_winkler    third_party/vergesort       third_party/jemalloc
third_party/fastpforlib     third_party/catch           third_party/imdb
third_party/fmt             third_party/fast_float      third_party/parquet
third_party/yyjson          third_party/pcg             third_party/ska_sort
third_party/thrift          third_party/brotli          third_party/miniz
third_party/hyperloglog     third_party/mbedtls
third_party/tdigest         third_party/skiplist
third_party/lz4             third_party/snappy
third_party/zstd            third_party/concurrentqueue
third_party/fsst            third_party/utf8proc
```
No pg-query, no bison/flex-generated grammar, no LLVM/asmjit (relevant again for §6). There are 28 directories. `third_party/utf8proc` is used by `ValidateUTF8Query` via `Utf8Proc::Analyze` (`src/parser/parser.cpp:82`).

### 1e. AST node types produced (parser → binder handoff)

Regardless of front-end (PEG vs. historical libpg_query), the **target AST type hierarchy is unchanged** — this is the parser/binder boundary:

- `SQLStatement` subclasses — `src/parser/statement/*.cpp` (headers `src/include/duckdb/parser/statement/`): `SelectStatement`, `InsertStatement`, `UpdateStatement`, `DeleteStatement`, `CreateStatement`, `AlterStatement`, `DropStatement`, `CopyStatement`, `TransactionStatement`, `ExplainStatement`, `PragmaStatement`, `ExecuteStatement`, `PrepareStatement`, `AttachStatement`/`DetachStatement`, `VacuumStatement`, `MergeIntoStatement`, `ExtensionStatement` (the escape hatch used by `TryParseExtensionStatement`), `CallStatement`, `SetStatement`, `LoadStatement`, `ConnectStatement`/`DisconnectStatement`, `MultiStatement`, `RelationStatement`, `ExternalResourceStatement`, etc. Also `PassthroughStatement` (`src/include/duckdb/parser/statement/passthrough_statement.hpp`, `StatementType::PASSTHROUGH_STATEMENT`) — "Unparsed SQL forwarded as-is to the CONNECT-ed database; the text lives in `query`". It is produced by the passthrough grammar in `src/parser/peg/passthrough_dialect.cpp` (§1b).
- `QueryNode` subclasses — `src/parser/query_node/`: `SelectNode` (a single `SELECT ... FROM ... WHERE ... GROUP BY ...` block), `SetOperationNode` (UNION/EXCEPT/INTERSECT), `RecursiveCTENode`, `CTENode`, `StatementNode`, and the DML query nodes `InsertQueryNode`, `UpdateQueryNode`, `DeleteQueryNode`, `MergeQueryNode`, `CopyQueryNode`. The shared base is `src/parser/query_node.cpp` (one level up, not inside `query_node/`).
- `ParsedExpression` subclasses — `src/parser/expression/`: `ColumnRefExpression`, `FunctionExpression`, `ComparisonExpression`, `ConjunctionExpression`, `OperatorExpression`, `CastExpression`, `CaseExpression`, `SubqueryExpression`, `WindowExpression`, `StarExpression`, `CollateExpression`, `BetweenExpression`, `ParameterExpression`, `PositionalReferenceExpression`, `LambdaExpression`, `LambdaRefExpression`, `ConstantExpression`, `DefaultExpression`, `TypeExpression` (plus `value_expression.cpp` in the same directory).
- `TableRef` subclasses — `src/parser/tableref/`: `BaseTableRef`, `JoinRef`, `SubqueryRef`, `TableFunctionRef`, `PivotRef`, `ExpressionListRef` (`VALUES (...)`), `DelimGetRef`, `EmptyTableRef`, `ShowRef`, `ColumnDataRef`, `BoundRefWrapper`. (`at_clause.cpp` in the same directory holds the time-travel `AT (...)` clause, which is not a `TableRef`.)
- **Handoff object:** `Parser::statements` — a `vector<unique_ptr<SQLStatement>>` — is what `ParseQuery` ultimately populates; this is what gets handed to the `Binder`.

### 1f. Example queries to trace this stage

1. `SELECT xvariable a = 1;` — deliberately malformed; exercises the PEG parser's error path and the extension-fallback protocol. Used verbatim (lower-case `select`) in `test/sql/peg_parser/enable_peg_parser.test:6` and `:12`, which assert the exact error text `Parser Error: syntax error at or near "="`. (`peg_syntax_error.test` covers other cases, such as unbalanced parentheses: `SELECT (1 + (2 * 3)` → `Parser Error: syntax error at or near ")"`.)
2. `SELECT a, (SELECT b FROM t2 WHERE t2.x = t1.x) AS sub FROM t1 JOIN t3 ON t1.id = t3.id GROUP BY a ORDER BY count(*) OVER (PARTITION BY a) DESC` — exercises `select.gram` + `expression.gram` in one parse (scalar subquery, join, aggregate, window function all present), landing in `transform_select.cpp`.

### 1g. Representative test files

- `test/sql/peg_parser/enable_peg_parser.test` — full contents:
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
  (Note: despite the filename, there is no visible runtime toggle between "old" and "new" parser in this test — it appears to be a legacy name from when the PEG parser was optional/experimental in the `autocomplete` extension.)
- `test/sql/peg_parser/peg_syntax_error.test` — 131 non-empty lines of syntax-error cases. Some cases guard against fixed bugs: expression syntax the PEG grammar once accepted by accident (`7f9d5319ec5`), and a tokenizer bug that returned the dot in a qualified name like `t.a` as a numeric constant (PR #27453, `04e1fd57fa0`, `605e5cb0c33`).
- `test/sql/peg_parser/transformer/peg_transformer.test`
- Other files in `test/sql/peg_parser/`: `binary_type.test`, `deep_nesting.test`, `escape_string.test`, `identifier.test`, `qualified_operator.test`, `unterminated_literals.test`, and the folders `fuzzer/`, `parser/`, `transformer/` (17 transformer tests, e.g. `between_asymmetric.test`, `is_predicate_operators.test`, `statement_length.test`).

---

## 2. Binder / Planner (name resolution, typing, `Bind` objects)

### 2a. Location

- Top-level driver: `src/planner/binder.cpp` (class `Binder`, header `src/include/duckdb/planner/binder.hpp`).
- `src/planner/binder/expression/` — one file per `ParsedExpression` subtype: `bind_columnref_expression.cpp`, `bind_function_expression.cpp`, `bind_subquery_expression.cpp`, `bind_window_expression.cpp`, `bind_aggregate_expression.cpp`, `bind_cast_expression.cpp`, `bind_case_expression.cpp`, `bind_comparison_expression.cpp`, `bind_conjunction_expression.cpp`, `bind_operator_expression.cpp`, `bind_between_expression.cpp`, `bind_lambda.cpp`, `bind_macro_expression.cpp`, `bind_parameter_expression.cpp`, `bind_positional_reference_expression.cpp`, `bind_star_expression.cpp`, `bind_unpacked_star_expression.cpp`, `bind_unnest_expression.cpp`, `bind_constant_expression.cpp`, `bind_collate_expression.cpp`, `bind_type_expression.cpp`.
- `src/planner/binder/query_node/` — `bind_select_node.cpp`, `bind_setop_node.cpp`, `bind_cte_node.cpp`, `bind_recursive_cte_node.cpp`, `bind_table_macro_node.cpp`, `bind_statement_node.cpp`, `bind_trigger_expansion.cpp`, `plan_select_node.cpp`, `plan_query_node.cpp`, `plan_subquery.cpp`, `plan_setop.cpp`.
- `src/planner/binder/statement/` — `bind_select.cpp`, `bind_insert.cpp`, `bind_create.cpp`, `bind_create_table.cpp`, `bind_update.cpp`, `bind_update_extensions.cpp`, `bind_delete.cpp`, `bind_merge_into.cpp`, `bind_call.cpp`, `bind_copy.cpp`, `bind_copy_database.cpp`, `bind_pragma.cpp`, `bind_prepare.cpp`, `bind_execute.cpp`, `bind_explain.cpp`, `bind_export.cpp`, `bind_summarize.cpp`, `bind_alter.cpp`, `bind_drop.cpp`, `bind_attach.cpp`, `bind_detach.cpp`, `bind_connect.cpp`, `bind_vacuum.cpp`, `bind_load.cpp`, `bind_set.cpp`, `bind_transaction.cpp`, `bind_extension.cpp`, `bind_external_resource.cpp`, `bind_relation.cpp`, `bind_logical_plan.cpp`.
- `src/planner/binder/tableref/` — `bind_basetableref.cpp`, `bind_joinref.cpp`, `plan_joinref.cpp`, `bind_table_function.cpp`, `bind_pivot.cpp`, `bind_subqueryref.cpp`, `bind_expressionlistref.cpp`, `bind_emptytableref.cpp`, `bind_showref.cpp`, `bind_bound_table_ref.cpp`, `bind_column_data_ref.cpp`, `bind_delimgetref.cpp`. (There is no separate named-parameter binder file: since PR #26091 table and pragma functions hold a `FunctionSignature`, like scalar, aggregate and window functions. Named parameters of table and pragma functions are checked in `src/function/function_binder.cpp`, against a declared `FunctionSignature`.)
- Non-binder-subdirectory planner files (`src/planner/*.cpp`): `bind_context.cpp`, `binder.cpp`, `binding_alias.cpp`, `bound_parameter_map.cpp`, `bound_result_modifier.cpp`, `collation_binding.cpp`, `column_binding.cpp`, `column_qualifier.cpp`, `expression.cpp`, `expression_binder.cpp`, `expression_iterator.cpp`, `expression_nullability.cpp`, `joinside.cpp`, `logical_operator.cpp`, `logical_operator_deep_copy.cpp`, `logical_operator_repeatability.cpp`, `logical_operator_visitor.cpp`, `logical_plan_verification_result.cpp`, `logical_plan_verifier.cpp`, `planner.cpp`, `statement_preprocessor.cpp`, `table_binding.cpp`, `table_filter_set.cpp`, `table_filter_state.cpp`.

- `src/planner/sql_export/` — 16 `.cpp` files (`bound_expression_sql_exporter.cpp`, `logical_plan_sql_exporter.cpp`, `sql_export_relations.cpp`, `sql_export_joins.cpp`, `sql_export_window.cpp`, `sql_export_cte.cpp`, `sql_export_dml.cpp`, `sql_export_pivot.cpp`, `sql_export_limit.cpp`, `sql_export_values.cpp`, `sql_export_functions.cpp`, `sql_export_scope.cpp`, `sql_export_composition.cpp`, `sql_export_constants.cpp`, `sql_export_sources.cpp`, `table_function_sql_export.cpp`) and a `README.md`. This is the `EXPLAIN (SQL)` feature (PR #26062, "Export optimized logical plans as executable SQL with EXPLAIN (SQL)"). It turns the *optimized logical plan* back into runnable DuckDB SQL. For example `EXPLAIN (SQL) SELECT category, sum(amount) AS total FROM sales WHERE amount > 0 GROUP BY category;` returns one row with key `sql` and the generated query as its value. The README warns that the output is generated SQL, not the original text: optimizer rewrites show up, and values seen while planning (for example `now()` or a `count(*)` answered from statistics) may be folded into literals. It supports SELECT/VALUES/WITH and INSERT/UPDATE/DELETE/MERGE INTO, but not DDL, CALL or `EXPLAIN EXECUTE` (`src/planner/sql_export/README.md`). MARK joins are written with the DuckDB syntax `JOIN BY (TYPE MARK)`. The output is formatted and highlighted with the PEG grammar's SQL formatter (PR #26288). The file `src/planner/expression/window_range_info.cpp` (`d7c8b288cd2`, same PR) keeps a record of how the binder lowered a window `RANGE` frame, so the exporter can write it back.

### 2b. Key class: `Binder`

`src/include/duckdb/planner/binder.hpp` — one `Binder` per lexical scope. Nested/child binders are created for subqueries, CTEs, and macros (see `ExtraBoundInfo::child_binders` below). The header pulls in `bind_context.hpp`, `bound_statement.hpp`, `bound_tokens.hpp`, `expression/bound_columnref_expression.hpp`, `joinside.hpp`, `bound_constraint.hpp`, `logical_operator.hpp` — i.e. `Binder` is the single class that spans parsed-AST input and logical-plan output.

### 2c. Column/table resolution

`BindContext` (`src/planner/bind_context.cpp`, header `src/include/duckdb/planner/bind_context.hpp:41-43`) — "keeps track of all the tables and columns that are encountered during the binding process" (header doc comment). Key API surface:

```cpp
class BindContext {
public:
    explicit BindContext(Binder &binder);
public:
    //! Given a column name, find the matching table it belongs to. Throws an
    //! exception if no table has a column of the given name.
    optional_ptr<Binding> GetMatchingBinding(const Identifier &column_name,
                                             QueryErrorContext context = QueryErrorContext());
    //! Like GetMatchingBinding, but instead of throwing an error if multiple tables have the same
    //! binding it will return a list of all the matching ones
    vector<reference<Binding>> GetMatchingBindings(const Identifier &column_name);
    //! Like GetMatchingBindings, but returns the top 3 most similar bindings (in levenshtein distance)
    //! instead of the matching ones
    vector<Identifier> GetSimilarBindings(const Identifier &column_name);

    optional_ptr<CTEBinding> GetCTEBinding(const BindingAlias &ctename);
    // ...
};
```

- Every table/CTE/subquery in scope is registered as a `Binding` object (`src/include/duckdb/planner/table_binding.hpp`, impl `src/planner/table_binding.cpp`), keyed by a `BindingAlias`.
- `bind_columnref_expression.cpp` is the concrete column-resolution entry point: it turns a `ColumnRefExpression` (parser AST) into a `BoundColumnRefExpression` (`src/include/duckdb/planner/expression/bound_columnref_expression.hpp`), which carries a `ColumnBinding` — a `{table_index, column_index}` pair (`src/include/duckdb/planner/column_binding.hpp:22-27`, impl `src/planner/column_binding.cpp`). The two fields are strong index types, not plain integers: `TableIndex table_index;` and `ProjectionIndex column_index;`. **`ColumnBinding` is the currency of the entire logical-plan layer** — every logical operator's `GetColumnBindings()` returns a `vector<ColumnBinding>` (`virtual vector<ColumnBinding> GetColumnBindings();`, `src/include/duckdb/planner/logical_operator.hpp:58`), and expressions reference upstream columns purely by this pair, not by name.
- Ambiguous/unqualified column references (`USING` joins, `NATURAL JOIN`) go through `UsingColumnSet` (`bind_context.hpp:34-37`): `{ BindingAlias primary_binding; vector<BindingAlias> bindings; }`.
- `GetSimilarBindings` is what powers DuckDB's well-known "did you mean 'foo'?" column-not-found suggestions, using Levenshtein distance (it returns `StringsToIdentifiers(StringUtil::TopNStrings(scores))`, `src/planner/bind_context.cpp:89`).

### 2d. Type resolution / implicit casts

- `ExpressionBinder` (`src/planner/expression_binder.cpp`, base header `src/include/duckdb/planner/expression_binder.hpp`) is the base class; clause-specific subclasses (declared under `src/include/duckdb/planner/expression_binder/`) enforce context-sensitive rules — e.g. a `WhereBinder` rejects aggregate expressions in a `WHERE` clause, an `OrderBinder` allows `ORDER BY` to reference `SELECT`-list aliases/positional integers that a `WhereBinder` would not.
- Casts are resolved through the cast-function catalog machinery invoked from `bind_cast_expression.cpp`, using `FunctionBinder` (`src/include/duckdb/function/function_binder.hpp`) to pick implicit vs. explicit cast rules and resolve overloaded function calls against argument types (this is also where `bind_function_expression.cpp` picks a concrete function overload given bound-argument types).
- `CollationBinding` (`src/planner/collation_binding.cpp`) handles collation-aware string comparison/ordering semantics threaded through comparison and order-by binding. Collations also apply inside structs and tuples: `('a' COLLATE NOCASE, 0) < ('B', 0)` is `true` (PR #25427).
- A scalar function binds in two steps: `resolve_types` chooses argument and return types, and `bind` builds the `FunctionData` used at run time (PR #26548).
- **Constant folding while binding.** The function binder keeps the values of foldable constant arguments it evaluates, so binding a deeply nested constant expression is linear, not quadratic (PR #26795). A constant that throws when folded does not make planning fail: the binder and optimizer use `TryEvaluateScalar`, and on error keep the expression for run time, where it may never be reached (PR #28356). Example: `SELECT CASE WHEN i::VARCHAR = 'x' THEN (i > 0 OR CAST('a' AS BOOLEAN)) ELSE true END FROM range(3) t(i);` runs.
- **Statements that expand into several.** A dynamic `PIVOT`, `ALTER ... ADD COLUMN` with a volatile default/NOT NULL/UNIQUE, `IMPORT DATABASE` and `PRAGMA copy_database` run as one query, in one transaction, with one result (PR #27108). `ExtractStatements` only splits a script into statements.

### 2e. The "Bind" object

`BoundStatement` (`src/include/duckdb/planner/bound_statement.hpp:25-37`) — the actual struct produced by binding:

```cpp
struct ExtraBoundInfo {
    SetOperationType setop_type = SetOperationType::NONE;
    vector<shared_ptr<Binder>> child_binders;
    vector<BoundStatement> bound_children;
    vector<unique_ptr<ParsedExpression>> original_expressions;
};

struct BoundStatement {
    unique_ptr<LogicalOperator> plan;
    vector<LogicalType> types;
    vector<Identifier> names;
    ExtraBoundInfo extra_info;
};
```

This is produced by `Binder::Bind(SQLStatement&)` (dispatching into `bind_select.cpp`, `bind_insert.cpp`, etc.) and is what the rest of the system (optimizer, physical planner) ultimately consumes.

**Important architectural point:** binding in DuckDB is *fused* with initial logical-plan construction — `BoundStatement` already contains a fully-formed `unique_ptr<LogicalOperator> plan`. The `plan_*.cpp` files inside `binder/query_node/` and `binder/tableref/` (`plan_select_node.cpp`, `plan_query_node.cpp`, `plan_joinref.cpp`, `plan_subquery.cpp`, `plan_setop.cpp`) build `LogicalGet`/`LogicalJoin`/`LogicalProjection`/etc. nodes as a direct byproduct of binding, *before* the optimizer ever runs. There is no separate "unoptimized bound-tree IR" distinct from `LogicalOperator` — the logical plan *is* the binder's output representation.

### 2f. Subquery decorrelation is planner-level, not optimizer-level

`src/planner/subquery/`:
- `flatten_dependent_join.cpp` — implements `FlattenDependentJoins`, the correlated-subquery decorrelation algorithm (the "unnesting arbitrary queries" technique of Neumann and Kemper [12] — see §4 for the full mechanics).
- `rewrite_correlated_expressions.cpp` — rewrites correlated-column references once a dependent join has been introduced/flattened.
- `pair_dependent_join.cpp` / `pair_dependent_full_outer_join.cpp` — pairing logic for `LogicalDependentJoin` construction.
- `delim_join_cte_rewriter.cpp` — interaction between decorrelation and CTEs.

A correlated subquery first becomes a `LogicalDependentJoin` (`src/planner/operator/logical_dependent_join.cpp`) during binding; it is later flattened into a plain join tree (`LogicalComparisonJoin`/`LogicalDelimJoin`) by `FlattenDependentJoins` (declared at `src/include/duckdb/planner/subquery/flatten_dependent_join.hpp:30`). When decorrelation creates a correlation CTE, equality restrictions that come from joins are pushed into that CTE as a semi-join, so nested lateral queries do not compute partitions for unrelated keys (PR #26158; see `CTE_FILTER_PUSHER` in §4b). The optimizer's `unnest_rewriter.cpp` and `deliminator.cpp` (§4) do the *follow-up* cleanup of the resulting delim-join structure — they do not perform the core decorrelation themselves.

### 2g. End-to-end handoff summary (parser → binder → optimizer → physical plan)

```
SQLStatement (parser AST)
   │  Binder::Bind()
   ▼
BoundStatement { plan: LogicalOperator tree, types, names }
   │  Optimizer::Optimize(unique_ptr<LogicalOperator>)
   ▼
optimized LogicalOperator tree
   │  PhysicalPlanGenerator::Plan(unique_ptr<LogicalOperator>)
   ▼
PhysicalOperator tree (owned by a PhysicalPlan arena)
```

### 2h. Example queries for this stage

1. `SELECT c.name, (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS n FROM customers c` — correlated scalar subquery; exercises `bind_subquery_expression.cpp` → `LogicalDependentJoin` → `FlattenDependentJoins`.
2. `SELECT a.*, b.val FROM t1 a JOIN t2 b USING (id)` — exercises `bind_joinref.cpp`'s `USING`-column resolution and `UsingColumnSet` in `bind_context.hpp`.

### 2i. Representative test files

- `test/sql/subquery/scalar/test_correlated_aggregate_subquery.test_slow`
- `test/sql/subquery/complex/correlated_list_aggregate.test_slow`

---

## 3. Logical Plan

### 3a. Base class: `LogicalOperator`

`src/include/duckdb/planner/logical_operator.hpp:35` (impl `src/planner/logical_operator.cpp`):

```cpp
//! LogicalOperator is the base class of the logical operators present in the
//! logical query tree
class LogicalOperator {
    friend class LogicalPlanVerifier;
public:
    explicit LogicalOperator(LogicalOperatorType type);
    LogicalOperator(LogicalOperatorType type, vector<unique_ptr<Expression>> expressions);
    virtual ~LogicalOperator();

    //! The type of the logical operator
    LogicalOperatorType type;
    //! The set of children of the operator
    vector<unique_ptr<LogicalOperator>> children;
    //! The set of expressions contained within the operator, if any
    vector<unique_ptr<Expression>> expressions;
    //! The types returned by this logical operator. Set by calling LogicalOperator::ResolveTypes.
    vector<LogicalType> types;
    //! Estimated Cardinality
    idx_t estimated_cardinality;
    bool has_estimated_cardinality;

public:
    virtual LogicalPlanSQLExportResult ToSQL(LogicalPlanSQLExportContext &context,
                                             const LogicalPlanVerificationPath &path);
    virtual vector<ColumnBinding> GetColumnBindings();
    virtual TableIndex GetRootIndex();
    static string ColumnBindingsToString(const vector<ColumnBinding> &bindings);
    void PrintColumnBindings();
    static vector<ColumnBinding> GenerateColumnBindings(TableIndex table_idx, idx_t column_count);
    static vector<LogicalType> MapTypes(const vector<LogicalType> &types,
                                        const vector<ProjectionIndex> &projection_map);
    static vector<ColumnBinding> MapBindings(const vector<ColumnBinding> &types,
                                             const vector<ProjectionIndex> &projection_map);

    //! Resolve the types of the logical operator and its children
    void ResolveOperatorTypes();

    //! Returns true if this operator or any of its descendants has side effects
    //! Used to prevent inlining or elimination of side-effecting CTEs.
    bool HasSideEffects() const;
    //! Returns true if this operator or any of its descendants contains a volatile expression
    bool HasVolatileExpressions() const;

    virtual string GetName() const;
    virtual InsertionOrderPreservingMap<string> ParamsToString() const;
    virtual string ToString(optional_ptr<ClientContext> context = nullptr,
                            const ProfilerPrintFormat &format = ProfilerPrintFormat::Default()) const;
    DUCKDB_API void Print();
    //! Debug method: verify that the integrity of expressions & child nodes are maintained
    virtual void Verify(ClientContext &context);

    void AddChild(unique_ptr<LogicalOperator> child);
    virtual idx_t EstimateCardinality(ClientContext &context);
    void SetEstimatedCardinality(idx_t _estimated_cardinality);
    void SetParamsEstimatedCardinality(InsertionOrderPreservingMap<string> &result) const;

    virtual void Serialize(Serializer &serializer) const;
    static unique_ptr<LogicalOperator> Deserialize(Deserializer &deserializer);
    virtual unique_ptr<LogicalOperator> Copy(ClientContext &context) const;
    virtual bool RequireOptimizer() const { return true; }

    //! Allows LogicalOperators to opt out of serialization
    virtual bool SupportSerialization() const { return true; };
    virtual bool HasProjectionMap() const { return false; }
    //! Returns the set of table indexes of this operator
    virtual vector<TableIndex> GetTableIndex() const;
    // ... (ResolveTypes() = 0, Cast<T>() helpers)
};
```

Notable virtuals: `ToSQL` (turns an optimized logical plan back into SQL text that DuckDB can run; it drives `EXPLAIN (SQL)`, commit `09eb7f7004c`, PR #26062; operators that cannot be printed as SQL report that instead of failing; the code lives in `src/planner/sql_export/`, §2a), `ToString`/`ParamsToString` (drive `EXPLAIN` output), `Serialize`/`Deserialize` (plan serialization — used by prepared statements and `EXPLAIN (FORMAT json)`), `Copy` (deep copy, needed for CTE re-planning where a CTE body may be planned more than once), `EstimateCardinality` (feeds the optimizer's cost model), `RequireOptimizer` (lets some operator types opt out of the optimizer pipeline).

### 3b. Enum: `LogicalOperatorType`

`src/include/duckdb/common/enums/logical_operator_type.hpp:18` — grouped by numeric range with explicit comments dividing sections:

```cpp
enum class LogicalOperatorType : uint8_t {
    LOGICAL_INVALID = 0,
    LOGICAL_PROJECTION = 1,
    LOGICAL_FILTER = 2,
    LOGICAL_AGGREGATE_AND_GROUP_BY = 3,
    LOGICAL_WINDOW = 4,
    LOGICAL_UNNEST = 5,
    LOGICAL_LIMIT = 6,
    LOGICAL_ORDER_BY = 7,
    LOGICAL_TOP_N = 8,
    LOGICAL_COPY_TO_FILE = 10,
    LOGICAL_DISTINCT = 11,
    LOGICAL_SAMPLE = 12,
    LOGICAL_PIVOT = 14,
    LOGICAL_COPY_DATABASE = 15,
    LOGICAL_SECURE_VIEW = 16,

    // -----------------------------
    // Data sources
    // -----------------------------
    LOGICAL_GET = 25,
    LOGICAL_CHUNK_GET = 26,
    LOGICAL_DELIM_GET = 27,
    LOGICAL_EXPRESSION_GET = 28,
    LOGICAL_DUMMY_SCAN = 29,
    LOGICAL_EMPTY_RESULT = 30,
    LOGICAL_CTE_REF = 31,
    // -----------------------------
    // Joins
    // -----------------------------
    LOGICAL_JOIN = 50,
    LOGICAL_DELIM_JOIN = 51,
    LOGICAL_COMPARISON_JOIN = 52,
    LOGICAL_ANY_JOIN = 53,
    LOGICAL_CROSS_PRODUCT = 54,
    LOGICAL_POSITIONAL_JOIN = 55,
    LOGICAL_ASOF_JOIN = 56,
    LOGICAL_DEPENDENT_JOIN = 57,
    // -----------------------------
    // SetOps
    // -----------------------------
    LOGICAL_UNION = 75,
    LOGICAL_EXCEPT = 76,
    LOGICAL_INTERSECT = 77,
    LOGICAL_RECURSIVE_CTE = 78,
    LOGICAL_MATERIALIZED_CTE = 79,

    // -----------------------------
    // Updates
    // -----------------------------
    LOGICAL_INSERT = 100,
    LOGICAL_DELETE = 101,
    LOGICAL_UPDATE = 102,
    LOGICAL_MERGE_INTO = 103,
    LOGICAL_TRIGGER = 104,

    // -----------------------------
    // Schema
    // -----------------------------
    LOGICAL_ALTER = 125,
    LOGICAL_CREATE_TABLE = 126,
    LOGICAL_CREATE_INDEX = 127,
    LOGICAL_CREATE_SEQUENCE = 128,
    LOGICAL_CREATE_VIEW = 129,
    LOGICAL_CREATE_SCHEMA = 130,
    LOGICAL_CREATE_MACRO = 131,
    LOGICAL_DROP = 132,
    LOGICAL_PRAGMA = 133,
    LOGICAL_TRANSACTION = 134,
    LOGICAL_CREATE_TYPE = 135,
    LOGICAL_ATTACH = 136,
    LOGICAL_DETACH = 137,
    LOGICAL_CREATE_TRIGGER = 138,

    // -----------------------------
    // Explain
    // -----------------------------
    LOGICAL_EXPLAIN = 150,

    // -----------------------------
    // Helpers
    // -----------------------------
    LOGICAL_PREPARE = 175,
    LOGICAL_EXECUTE = 176,
    LOGICAL_EXPORT = 177,
    LOGICAL_VACUUM = 178,
    LOGICAL_SET = 179,
    LOGICAL_LOAD = 180,
    LOGICAL_RESET = 181,
    LOGICAL_UPDATE_EXTENSIONS = 182,
    LOGICAL_CONNECT = 183,
    LOGICAL_DISCONNECT = 184,
    LOGICAL_EXTERNAL_RESOURCE = 185,

    // -----------------------------
    // Secrets
    // -----------------------------
    LOGICAL_CREATE_SECRET = 190,

    LOGICAL_EXTENSION_OPERATOR = 255
};
```

(Numeric gaps are deliberate — they reserve room to insert new operators within a category without renumbering everything else, which matters for plan serialization stability.)

### 3c. Concrete subclasses

`src/planner/operator/*.cpp` (headers under `src/include/duckdb/planner/operator/`) — roughly one file per enum value, plus shared base classes such as `logical_join.cpp`, `logical_create.cpp` and `logical_unconditional_join.cpp`. There are 57 `.cpp` files:

```
logical_aggregate.cpp          logical_distinct.cpp            logical_positional_join.cpp
logical_alter.cpp              logical_drop.cpp                logical_pragma.cpp
logical_any_join.cpp           logical_dummy_scan.cpp          logical_prepare.cpp
logical_attach.cpp             logical_empty_result.cpp        logical_projection.cpp
logical_column_data_get.cpp    logical_explain.cpp             logical_recursive_cte.cpp
logical_comparison_join.cpp    logical_export.cpp              logical_reset.cpp
logical_connect.cpp            logical_expression_get.cpp      logical_sample.cpp
logical_copy_database.cpp      logical_extension_operator.cpp  logical_secure_view.cpp
logical_copy_to_file.cpp       logical_external_resource.cpp   logical_set.cpp
logical_create.cpp             logical_filter.cpp              logical_set_operation.cpp
logical_create_index.cpp       logical_get.cpp                 logical_top_n.cpp
logical_create_table.cpp       logical_insert.cpp              logical_transaction.cpp
logical_cross_product.cpp      logical_join.cpp                logical_trigger.cpp
logical_cteref.cpp             logical_limit.cpp               logical_unconditional_join.cpp
logical_delete.cpp             logical_load.cpp                logical_unnest.cpp
logical_delim_get.cpp          logical_materialized_cte.cpp    logical_update.cpp
logical_dependent_join.cpp     logical_merge_into.cpp          logical_update_extensions.cpp
logical_detach.cpp             logical_order.cpp               logical_vacuum.cpp
logical_disconnect.cpp         logical_pivot.cpp               logical_window.cpp
```

Many operator files carry `ToSQL` support (for example `logical_explain.cpp` and `logical_get.cpp`).

Key ones for query-processing chapters: `LogicalGet` (`logical_get.cpp` — the scan/data-source node, wraps a `TableFunction` + bind data; every base-table scan, table-function call, and even `read_csv`/`read_parquet` calls become a `LogicalGet`), `LogicalFilter` (`logical_filter.cpp`), `LogicalProjection` (`logical_projection.cpp`), `LogicalJoin`/`LogicalComparisonJoin`/`LogicalAnyJoin`/`LogicalCrossProduct`/`LogicalPositionalJoin` (`logical_join.cpp`, `logical_comparison_join.cpp`, `logical_any_join.cpp`, `logical_cross_product.cpp`, `logical_positional_join.cpp`), `LogicalDependentJoin` (`logical_dependent_join.cpp` — the not-yet-decorrelated correlated-subquery marker node from §2f), `LogicalAggregate` (`logical_aggregate.cpp` — covers both `GROUP BY` and grouping sets), `LogicalWindow` (`logical_window.cpp`), `LogicalUnnest` (`logical_unnest.cpp`), `LogicalOrder`/`LogicalTopN` (`logical_order.cpp`, `logical_top_n.cpp` — the latter is the fused ORDER BY + LIMIT node produced by the `TOP_N` optimizer pass, see §4), `LogicalDistinct` (`logical_distinct.cpp`), `LogicalSetOperation` (`logical_set_operation.cpp` — UNION/EXCEPT/INTERSECT), `LogicalRecursiveCTE`/`LogicalMaterializedCTE`/`LogicalCTERef` (`logical_recursive_cte.cpp`, `logical_materialized_cte.cpp`, `logical_cteref.cpp`), `LogicalInsert`/`LogicalUpdate`/`LogicalDelete`/`LogicalMergeInto` (DML), `LogicalCreateTable`/`LogicalCreateIndex`/`LogicalAlter`/`LogicalDrop` (DDL), `LogicalExtensionOperator` (`logical_extension_operator.cpp` — the extension hook for third-party logical operators).

### 3d. Supporting visitor infrastructure

- `LogicalOperatorVisitor` (`src/planner/logical_operator_visitor.cpp`, header `src/include/duckdb/planner/logical_operator_visitor.hpp`) — the base tree-walker class most optimizer passes derive from (visitor pattern over the `LogicalOperator` tree and its contained `Expression`s).
- `LogicalPlanVerifier` (`src/planner/logical_plan_verifier.cpp`) — runs after every optimizer pass (when verification is enabled) to assert plan sanity: called from `Optimizer::Verify(LogicalOperator&)` (`src/optimizer/optimizer.cpp:155`), itself invoked inside `Optimizer::RunOptimizer` after each pass's callback.
- `ColumnBindingResolver` (`src/execution/column_binding_resolver.cpp`) — technically runs *after* optimization, right before physical planning (see §5), converting `ColumnBinding{table_index, column_index}` pairs into flat physical chunk-column indices.

### 3e. Handoff to next stage

The optimizer takes and returns a raw `unique_ptr<LogicalOperator>`:
```cpp
unique_ptr<LogicalOperator> Optimizer::Optimize(unique_ptr<LogicalOperator> plan_p);
```
(`src/include/duckdb/optimizer/optimizer.hpp:28`, impl `src/optimizer/optimizer.cpp:577`) — rewriting the tree in place / returning a new tree built from the same node-type vocabulary described above.

---

## 4. Optimizer

### 4a. Driver class and construction

`Optimizer` — `src/optimizer/optimizer.cpp` (constructor at line 73, `RunOptimizer` at line 137, the pass sequence in `RunBuiltInOptimizers()`). Constructed with a fixed list of expression-rewrite rules (an `ExpressionRewriter rewriter` member, populated in the constructor — see §4c), then `Optimizer::Optimize()` runs a **fixed, hardcoded sequence** of `RunOptimizer(OptimizerType::X, [&]{ ... })` calls. Each pass is individually toggleable at runtime via `disabled_optimizers` (settable through `PRAGMA disabled_optimizers` / the `EnableOptimizerSetting`), and `RunOptimizer` triggers `LogicalPlanVerifier::Verify()` after every pass when verification is enabled:

```cpp
void Optimizer::RunOptimizer(OptimizerType type, const std::function<void()> &callback) {
    context.InterruptCheck();

    if (OptimizerDisabled(type)) {
        // optimizer is marked as disabled: skip
        return;
    }
    auto &profiler = QueryProfiler::Get(context);
    {
        auto optimizer_timer = profiler.StartTimerInternal("optimizer." + StringUtil::Lower(EnumUtil::ToString(type)));
        callback();
    }
    if (plan) {
        Verify(*plan);
    }
}
```

(`context.InterruptCheck()` throws if the query was interrupted. `OptimizerDisabled` asks `config.IsOptimizerDisabled(type)`, `optimizer.cpp:134`.)

### 4b. Full pipeline order (as it appears in `src/optimizer/optimizer.cpp`, with line numbers)

Two of the passes have the highest `OptimizerType` values: `SELF_JOIN_AGGREGATE` (step 4, value 47; commit `32c8a25387a`, PR #28259) rewrites a self-join into a `GROUP BY`, and `REDUNDANT_ORDER_KEYS` (step 22, value 46; commit `e091a79d435`, PR #25878) drops `ORDER BY` keys that come after a unique prefix (`src/include/duckdb/common/enums/optimizer_type.hpp`). The second `UNUSED_COLUMNS` run (step 39) runs when statistics propagation removed expressions or when `TOP_N_WINDOW_ELIMINATION` used late materialization.

1. **`REMOTE_PUSHDOWN`** (line 223) — `src/optimizer/remote_pushdown_optimizer.cpp`
2. **`EXPRESSION_REWRITER`** (253) — `src/optimizer/expression_rewriter.cpp`, runs the fixed rule list from §4c
3. **`CTE_INLINING`** (260, run again at 333) — `src/optimizer/cte_inlining.cpp`
4. **`SELF_JOIN_AGGREGATE`** (268) — `src/optimizer/self_join_aggregate.cpp`. A self-join of one table, on equal keys plus exactly one `<>`, whose parent only asks "which keys exist?" (for example through `IN (...)`), becomes `GROUP BY keys HAVING min(x) <> max(x)` over one scan. The motivating case is TPC-DS q95. It must run before filter pushdown, which would move the join filters into the scans and hide the pattern. Test: `test/sql/optimizer/self_join_aggregate.test`.
5. **`AGGREGATE_FUNCTION_REWRITER`** (274) — `src/optimizer/aggregate_function_rewriter.cpp`
6. **`FILTER_PULLUP`** (280) — `src/optimizer/filter_pullup.cpp`
7. **`FILTER_PUSHDOWN`** (286) — `src/optimizer/filter_pushdown.cpp` (uses `src/optimizer/filter_combiner.cpp` plus per-operator handlers under `src/optimizer/pushdown/`)
8. **`CTE_FILTER_PUSHER`** (299) — `src/optimizer/cte_filter_pusher.cpp`. It also calls `CTEJoinFilterPusher` (`src/optimizer/cte_join_filter_pusher.cpp`, with `cte_join_filter_collector.cpp`), which pushes key restrictions that come from joins into correlation CTEs (PR #26158).
9. **`REGEX_RANGE`** (305) — `src/optimizer/regex_range_filter.cpp`
10. **`IN_CLAUSE`** (310) — `src/optimizer/in_clause_rewriter.cpp`
11. **`DELIMINATOR`** (316) — `src/optimizer/deliminator.cpp` (removes unnecessary `DelimJoin`/`DelimGet` pairs left over from decorrelation, §2f)
12. **`GROUPING_SETS`** (322) — `src/optimizer/grouping_sets_optimizer.cpp`. Right after it (line 328), a `MultiStageAggregateRewriter` runs with policy `UNCONDITIONAL`; it is not wrapped in `RunOptimizer`.
13. **`CTE_INLINING`** again (333)
14. **`EMPTY_RESULT_PULLUP`** (339) — `src/optimizer/empty_result_pullup.cpp`
15. **`WINDOW_SELF_JOIN`** (345) — `src/optimizer/window_self_join.cpp`
16. **`PROJECTION_PULLUP`** (351) — `src/optimizer/projection_pullup.cpp`
17. **`OUTER_JOIN_SIMPLIFICATION`** (357) — `src/optimizer/outer_join_simplification.cpp`
18. **`JOIN_ORDER`** (364) — `src/optimizer/join_order/` (DPhyp-based reorderer, §4d)
19. **`AGGREGATE_REUSE`** (370) — `src/optimizer/aggregate_reuse.cpp` (+ `materialized_aggregate_reuse.cpp`)
20. **`PARTIAL_AGGREGATE_PUSHDOWN`** (379) — `src/optimizer/partial_aggregate_pushdown.cpp`
21. **`JOIN_ELIMINATION`** (384) — `src/optimizer/join_elimination.cpp`
22. **`REDUNDANT_ORDER_KEYS`** (389) — `src/optimizer/remove_redundant_order_keys.cpp`. In `ORDER BY id, payload`, if `id` is already unique (for example a primary key), `payload` can never change the order, so the pass removes it. It works out unique column sets once per plan, from primary keys, unique constraints on NOT NULL columns, and `GROUP BY` outputs. The freed column can then be pruned by `UNUSED_COLUMNS`, so the pass must run before it. Test: `test/optimizer/remove_redundant_order_keys.test` (issue #24000).
23. **`UNNEST_REWRITER`** (395) — `src/optimizer/unnest_rewriter.cpp` (post-decorrelation cleanup of the join tree; §4e).
24. **`UNUSED_COLUMNS`** (401, again at 517) — `src/optimizer/remove_unused_columns.cpp` (+ `src/optimizer/column_binding_replacer.cpp`)
25. **`DUPLICATE_GROUPS`** (407) — `src/optimizer/remove_duplicate_groups.cpp`
26. **`COMMON_SUBEXPRESSIONS`** (413) — `src/optimizer/cse_optimizer.cpp` (CSE — see §4f)
27. **`COLUMN_LIFETIME`** (419, again at 525) — `src/optimizer/column_lifetime_analyzer.cpp`
28. **`BUILD_SIDE_PROBE_SIDE`** (426) — `src/optimizer/build_probe_side_optimizer.cpp` (decides hash-join build vs. probe side using cardinality estimates)
29. **`COMMON_SUBPLAN`** (435) — `src/optimizer/common_subplan_optimizer.cpp` (skipped when a CTE contains DML)
30. **`LIMIT_PUSHDOWN`** (442) — `src/optimizer/limit_pushdown.cpp`
31. **`ROW_GROUP_PRUNER`** (447) — `src/optimizer/row_group_pruner.cpp`
32. **`SAMPLING_PUSHDOWN`** (453) — `src/optimizer/sampling_pushdown.cpp`
33. **`TOP_N`** (459) — `src/optimizer/topn_optimizer.cpp` (fuses `ORDER BY` + `LIMIT` into `LogicalTopN`)
34. **`LATE_MATERIALIZATION`** (465) — `src/optimizer/late_materialization.cpp` (+ `late_materialization_helper.cpp`)
35. **`STATISTICS_PROPAGATION`** (476) — `src/optimizer/statistics_propagator.cpp` (+ `src/optimizer/statistics/`, `src/optimizer/relation_statistics/`, `src/optimizer/filter_statistics.cpp`). A cost-based `MultiStageAggregateRewriter` follows at line 485 and re-runs statistics propagation if it changed the plan. The propagator takes the plan by reference to its owning pointer (`StatisticsPropagator propagator(*this, plan)`), so it can safely remove or replace the root (PRs #26245, #26421).
36. **`PROJECTION_PLACEMENT`** (493) — `src/optimizer/projection_placement.cpp`
37. **`TOP_N_WINDOW_ELIMINATION`** (501) — `src/optimizer/topn_window_elimination.cpp`
38. **`COMMON_AGGREGATE`** (508) — `src/optimizer/common_aggregate_optimizer.cpp`
39. **`UNUSED_COLUMNS`** again (517) — only if statistics propagation removed expressions or step 37 used late materialization
40. **`COLUMN_LIFETIME`** again (525)
41. **`REORDER_FILTER`** (531) — `src/optimizer/expression_heuristics.cpp`
42. **`PARTITIONED_EXECUTION`** (537) — `src/optimizer/partitioned_execution.cpp`
43. **`JOIN_FILTER_PUSHDOWN`** (543) — `src/optimizer/join_filter_pushdown_optimizer.cpp` (dynamic/runtime semi-join filter pushdown; see also `src/optimizer/runtime_filter_cast.cpp`)
44. **`ROW_NUMBER_REWRITER`** (549) — `src/optimizer/row_number_rewriter.cpp`
45. **`TYPE_PUSHDOWN`** (555) — `src/optimizer/type_pushdown.cpp`
46. **`SCALAR_FN_PUSHDOWN`** (561) — `src/optimizer/scalar_fn_pushdown.cpp`
47. A final `MultiStageAggregateRewriter` with policy `MANDATORY` (line 571), not wrapped in `RunOptimizer`.
48. **`EXTENSION`** (595 for pre-optimizer extensions, 607 for post-optimizer extensions, both in `Optimizer::Optimize` at line 577) — third-party `OptimizerExtension` hooks

Additional files present in `src/optimizer/` not explicitly named as a distinct `OptimizerType` pass above (either invoked as helpers by one of the above, or specialized sub-passes): `compressed_materialization.cpp` (+ `src/optimizer/compressed_materialization/`), `constant_or_null_simplification.cpp`, `expression_heuristics.cpp` (cost heuristics used to decide expression evaluation order; it is the `REORDER_FILTER` pass), `multi_stage_aggregate_rewriter.cpp`, `aggregate_rewrite.cpp` / `aggregate_rewrite_helper.cpp`, `cte_join_filter_collector.cpp` / `cte_join_filter_pusher.cpp` (used by `CTE_FILTER_PUSHER`), the `pullup/`, `pushdown/`, `statistics/` and `relation_statistics/` subdirectories, and matcher helpers under `src/optimizer/matcher/`.

### 4c. Expression-rewrite rule set (run inside step 2, `EXPRESSION_REWRITER`)

Populated in the `Optimizer::Optimizer(Binder&, ClientContext&)` constructor (`src/optimizer/optimizer.cpp:73-118`, right after `rewriter(context)` initialization) as a fixed, ordered list of `rewriter.rules.push_back(make_uniq<...Rule>(rewriter))` calls. Each rule is a small pattern-matching class under `src/optimizer/rule/`:

```cpp
Optimizer::Optimizer(Binder &binder, ClientContext &context) : context(context), binder(binder), rewriter(context) {
    rewriter.rules.push_back(make_uniq<ConstantOrderNormalizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ConstantFoldingRule>(rewriter));
    rewriter.rules.push_back(make_uniq<StructExtractStructPackFoldingRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DistributivityRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ArithmeticSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<LeastGreatestSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<CaseSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ConjunctionSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DatePartSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DateTruncSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ComparisonSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<RowComparisonSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<InClauseSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<InEnumSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EnumCompareSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EqualOrNullSimplification>(rewriter));
    rewriter.rules.push_back(make_uniq<MoveConstantsRule>(rewriter));
    rewriter.rules.push_back(make_uniq<MoveUnaryMinusRule>(rewriter));
    rewriter.rules.push_back(make_uniq<MonotonePreimageRule>(rewriter));
    rewriter.rules.push_back(make_uniq<LikeOptimizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<StringPrefixRule>(rewriter));
    rewriter.rules.push_back(make_uniq<InstrPrefixRule>(rewriter));
    rewriter.rules.push_back(make_uniq<OrderedAggregateOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<DistinctAggregateOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<DistinctWindowedOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<RegexOptimizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<RegexpReplaceExtractRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EmptyNeedleRemovalRule>(rewriter));
    rewriter.rules.push_back(make_uniq<NoopReplaceRemovalRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EnumComparisonRule>(rewriter));
    rewriter.rules.push_back(make_uniq<JoinDependentFilterRule>(rewriter));
    rewriter.rules.push_back(make_uniq<TimeStampComparison>(rewriter));
    rewriter.rules.push_back(make_uniq<PredicateFactoringRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ListComprehensionRewriteRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ContainsToInClauseRule>(rewriter));
    rewriter.rules.push_back(make_uniq<NotComparisonSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<NotConjunctionSimplificationRule>(rewriter));
}
```

The list has 37 rules. Rules that fold constants do not turn a run-time error into a planning error: the empty-needle rule and the LEFT JOIN rewrite leave errors from constant arguments to execution time instead of folding them away (commit `5bf940c1318`, PR #27421). Other passes follow the same idea: filter pullup keeps fallible predicates below projections, projection pullup keeps projections below throwing filters, and partial aggregate pushdown does not push aggregates whose arguments can throw below a join (tests `test/optimizer/constant_argument_errors*.test`, `test/optimizer/case_simplification_errors.test`).

File-to-rule mapping (`src/optimizer/rule/*.cpp`, 29 files): `predicate_factoring.cpp` (`PredicateFactoringRule`), `monotone_preimage.cpp` (`MonotonePreimageRule`), `distributivity.cpp` (`DistributivityRule`), `ordered_aggregate_optimizer.cpp` (`OrderedAggregateOptimizer`), `like_optimizations.cpp` (`LikeOptimizationRule`), `date_trunc_simplification.cpp` (`DateTruncSimplificationRule`), `list_comprehension_rewrite.cpp` (`ListComprehensionRewriteRule`), `regex_optimizations.cpp` (`RegexOptimizationRule`, `RegexpReplaceExtractRule`), `empty_needle_removal.cpp` (`EmptyNeedleRemovalRule`, `NoopReplaceRemovalRule`), `string_prefix.cpp` (`StringPrefixRule`, `InstrPrefixRule`), `constant_folding.cpp` (`ConstantFoldingRule`), `constant_order_normalization.cpp` (`ConstantOrderNormalizationRule`), `date_part_simplification.cpp` (`DatePartSimplificationRule`), `distinct_aggregate_optimizer.cpp` (`DistinctAggregateOptimizer`, `DistinctWindowedOptimizer`), `comparison_simplification.cpp` (`ComparisonSimplificationRule`, `RowComparisonSimplificationRule`), `struct_extract_struct_pack_folding.cpp` (`StructExtractStructPackFoldingRule`), `not_comparison_simplification.cpp` (`NotComparisonSimplificationRule`), `join_dependent_filter.cpp` (`JoinDependentFilterRule`), `in_clause_simplification_rule.cpp` (`InClauseSimplificationRule`, `InEnumSimplificationRule`, `EnumCompareSimplificationRule`), `case_simplification.cpp` (`CaseSimplificationRule`), `not_conjunction_simplification.cpp` (`NotConjunctionSimplificationRule`), `conjunction_simplification.cpp` (`ConjunctionSimplificationRule`), `move_constants.cpp` (`MoveConstantsRule`, `MoveUnaryMinusRule`), `least_greatest_simplification.cpp` (`LeastGreatestSimplificationRule`), `enum_comparison.cpp` (`EnumComparisonRule`), `equal_or_null_simplification.cpp` (`EqualOrNullSimplification`), `contains_to_in_clause.cpp` (`ContainsToInClauseRule`), `arithmetic_simplification.cpp` (`ArithmeticSimplificationRule`), `timestamp_comparison.cpp` (`TimeStampComparison`).

Note the constructor also has a debug-only assertion block (`optimizer.cpp:112-117`):
```cpp
#ifdef DEBUG
for (auto &rule : rewriter.rules) {
    D_ASSERT(rule->root);  // root not defined in rule
}
#endif
```

### 4d. Join order optimizer — confirmed DPhyp / dynamic programming

`src/optimizer/join_order/`:
- `join_order_optimizer.cpp` — top-level driver, invoked as `OptimizerType::JOIN_ORDER` (pipeline step 18 above).
- `query_graph_manager.cpp` / `query_graph.cpp` — build the join **hypergraph**. Header comment at `src/include/duckdb/optimizer/join_order/query_graph_manager.hpp:59`: "Get a reference to the QueryGraphEdges structure that stores edges between nodes and hypernodes"; `CreateHyperGraphEdges()` declared at line 69.
- `relation_manager.cpp` / `join_relation_set.cpp` / `join_node.cpp` — relation-set bitmask bookkeeping (each base relation gets a bit; a `JoinRelationSet` is a bitmask of relations already joined together in one DP sub-plan).
- `plan_enumerator.cpp` — **the actual search algorithm**. Explicit paper citation:
  ```
  src/optimizer/join_order/plan_enumerator.cpp:600:
  // Moerkotte and Thomas Neumannn, see that paper for additional info/documentation bonus slides:
  ```
  This is the DPhyp paper by Moerkotte and Neumann [20]. The enumerator performs exact dynamic programming over the join hypergraph, with an explicit escape hatch to a greedy heuristic when the search space is too large:
  ```
  plan_enumerator.cpp:214:  // the new plan costs less than the old plan. Update our DP table.
  plan_enumerator.cpp:244:  // when the amount of pairs gets too large we exit the dynamic programming and resort to a greedy algorithm
  plan_enumerator.cpp:386:  // now we perform the actual dynamic programming to compute the final result
  plan_enumerator.cpp:421:  // at this point, we exited the dynamic programming but did not compute the final join order because it took too [long]
  ```
- `cost_model.cpp` + `cardinality_estimator.cpp` — cost/cardinality estimates feeding the DP table.
- `filter_info.cpp` / `join_predicate.cpp` — maps join predicates onto hypergraph edges.
- `join_order_operator.cpp` — turns the winning DP plan back into a tree of `LogicalComparisonJoin`/`LogicalCrossProduct` nodes.

The greedy fallback evaluates each candidate pair of relations once, not once per round (commit `91e650acc22`, PR #26965), and tries both orientations of each pair (PR #28231). It also keeps the join order the query was written in as one more candidate, so the greedy result is never worse than the input order by the cost model (commit `c68bb26acd9`, PR #28362). Recursive-CTE join-order limits are kept through projections (PR #26094). The cardinality estimator treats a join on `<>` or `IS DISTINCT FROM` as nearly a cross product (commit `a725d696d8a`; test `test/optimizer/joins/not_equal_join_cardinality_estimation.test`), and counts each table filter once in scan estimates (PR #25193). A secure view does not block join reordering: it is treated as a pass-through with its own frozen estimate (PR #27924).

### 4e. Subquery unnesting (Neumann and Kemper, "unnesting arbitrary queries" [12]) — split across planner + optimizer

As established in §2f, the **core decorrelation algorithm lives at the planner level**: `FlattenDependentJoins` in `src/planner/subquery/flatten_dependent_join.cpp`. The optimizer's contribution is the *follow-up cleanup* of the join tree that decorrelation leaves behind:
- `src/optimizer/deliminator.cpp` — removes redundant `DelimJoin`/`DelimGet` pairs (`OptimizerType::DELIMINATOR`, pipeline step 11).
- `src/optimizer/unnest_rewriter.cpp` — further simplifies the resulting join tree (`OptimizerType::UNNEST_REWRITER`, pipeline step 23). It casts a fixed-size `ARRAY` to `LIST` when it builds a `LogicalUnnest` (PR #26845).
- `src/optimizer/pushdown/pushdown_unnest.cpp` — filter-pushdown-specific handling for `LogicalUnnest` nodes (part of the `FILTER_PUSHDOWN` pass).
- `src/planner/operator/logical_dependent_join.cpp` — the intermediate node type representing an as-yet-unflattened correlated subquery (produced during binding, consumed by `FlattenDependentJoins`).

### 4f. CSE / common-subplan reuse

- `src/optimizer/cse_optimizer.cpp` — classic per-operator common-subexpression elimination *within expression trees* (e.g. `a+1` appearing twice in a projection list gets computed once).
- `src/optimizer/common_aggregate_optimizer.cpp` — dedupes identical aggregate computations (e.g. `SUM(x)` requested twice).
- `src/optimizer/common_subplan_optimizer.cpp` — dedupes identical *subplans* (e.g. the same scan+filter subtree appearing in two branches of a query, such as both sides of a self-join or two CTE-less duplicated references).

### 4g. Statistics propagation

- `src/optimizer/statistics_propagator.cpp` — bottom-up propagation of column statistics through the plan (min/max, distinct-count, null-fraction), used both for cardinality estimation (feeding `cardinality_estimator.cpp` in the join-order optimizer) and for constant-folding-style simplifications (e.g. proving `WHERE x > 1000000` is always `FALSE` given a column's known max value).
- `src/optimizer/statistics/` — per-operator and per-expression propagation code (`statistics/operator/*.cpp`, `statistics/expression/*.cpp`).
- `src/optimizer/relation_statistics/` — per-relation statistics structures.
- `src/optimizer/filter_statistics.cpp` — statistics specific to filter predicates.

### 4h. Example queries for this stage

1. `SELECT c.id FROM customer c, orders o, lineitem l WHERE c.id=o.custkey AND o.orderkey=l.orderkey AND c.nation='FR'` — 3-way join with no explicit join syntax (implicit cross-join + WHERE-clause predicates), a classic DPhyp reordering target; TPC-H-style.
2. `SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.a = t1.a AND t2.b > 10)` — correlated `EXISTS`; exercises `FlattenDependentJoins` → `deliminator` → `unnest_rewriter` end-to-end.
3. `SELECT a, a+1, a+1 FROM t` — trivial but directly exercises CSE (`a+1` should be computed once, not twice).

### 4i. Representative test files

- `test/sql/optimizer/expression/test_cse.test`
- `test/sql/optimizer/plan/test_recursive_join_order_guard.test`
- `test/sql/aggregate/distinct/rewrite/distinct_aggregate_rewrite_join_order.test`
- `test/sql/subquery/scalar/test_unnest_subquery.test` — full contents excerpt (uncorrelated → correlated → correlated-with-aggregate progression, ideal for a walkthrough):
  ```
  # name: test/sql/subquery/scalar/test_unnest_subquery.test
  # description: Test unnest in subqueries
  # group: [scalar]

  # uncorrelated
  query I
  SELECT (SELECT UNNEST([1]))
  ----
  1
  ...
  # correlated
  query I
  SELECT (SELECT UNNEST(i)) FROM (VALUES ([1])) tbl(i);
  ----
  1
  ...
  # now with an aggregate
  query I
  SELECT (SELECT SUM(k) FROM (SELECT UNNEST(i)) tbl(k)) FROM (VALUES ([1, 2, 3])) tbl(i);
  ----
  ```
- `test/sql/subquery/lateral/lateral_unnest.test_slow`, `test/sql/subquery/lateral/lateral_unnest_large.test_slow`
- `test/sql/subquery/complex/nested_unnest_subquery.test`
- `test/sql/subquery/scalar/test_correlated_aggregate_subquery.test_slow`
- `test/sql/subquery/complex/correlated_list_aggregate.test_slow`

---

## 5. Physical Plan

### 5a. Base class: `PhysicalOperator`

`src/include/duckdb/execution/physical_operator.hpp` (fields at lines 58-72). Fields: `ArenaLinkedList<reference<PhysicalOperator>> children` (note: children are *reference-wrapped*, not `unique_ptr`-owned — operators are owned by a `PhysicalPlan` arena instead, so the tree can share/reference nodes cheaply), `PhysicalOperatorType type`, `vector<LogicalType> types`, `idx_t estimated_cardinality`, plus the crucial mutable execution state: `unique_ptr<GlobalSinkState> sink_state`, `unique_ptr<GlobalOperatorState> op_state`, and a `mutex lock` guarding (re)sets of either state (with a comment noting the global sink state is "published under `lock` by `Pipeline::ResetSink` on a worker; a reader racing pipeline initialization must hold `lock` to observe it safely").

`PhysicalOperator` has a virtual method `InitializeOutputChunk(Allocator &, DataChunk &)` (`physical_operator.hpp:95`). By default it calls `chunk.Initialize(allocator, types)`. An operator can override it to set up its output vectors in a special way; projections and joins use this to skip buffer allocation for output columns that only reference or slice an input vector. `DataChunk::GetCapacity()` (`src/include/duckdb/common/types/data_chunk.hpp:64`) returns how many rows the chunk's vectors were set up to hold.

### 5b. Enum: `PhysicalOperatorType`

`src/include/duckdb/common/enums/physical_operator_type.hpp:18` — **no `PHYSICAL_` prefix** on values (unlike `LogicalOperatorType`'s `LOGICAL_` prefix):

```cpp
enum class PhysicalOperatorType : uint8_t {
    INVALID,
    ORDER_BY, LIMIT, STREAMING_LIMIT, LIMIT_PERCENT, TOP_N, WINDOW, UNNEST,
    UNGROUPED_AGGREGATE, HASH_GROUP_BY, PERFECT_HASH_GROUP_BY, PARTITIONED_AGGREGATE,
    LIMITED_DISTINCT, FILTER, PROJECTION, COPY_TO_FILE, BATCH_COPY_TO_FILE,
    RESERVOIR_SAMPLE, STREAMING_SAMPLE, STREAMING_WINDOW, PIVOT, COPY_DATABASE,

    // Scans
    TABLE_SCAN, DUMMY_SCAN, COLUMN_DATA_SCAN, CHUNK_SCAN, RECURSIVE_CTE_SCAN,
    RECURSIVE_RECURRING_CTE_SCAN, CTE_SCAN, DELIM_SCAN, EXPRESSION_SCAN, POSITIONAL_SCAN,

    // Joins
    BLOCKWISE_NL_JOIN, NESTED_LOOP_JOIN, HASH_JOIN, CROSS_PRODUCT, PIECEWISE_MERGE_JOIN,
    IE_JOIN, LEFT_DELIM_JOIN, RIGHT_DELIM_JOIN, POSITIONAL_JOIN, ASOF_JOIN,

    // SetOps
    UNION, RECURSIVE_CTE, RECURSIVE_KEY_CTE, CTE,

    // Updates
    INSERT, BATCH_INSERT, DELETE_OPERATOR, UPDATE, MERGE_INTO,

    // Schema
    CREATE_TABLE, CREATE_TABLE_AS, BATCH_CREATE_TABLE_AS, CREATE_INDEX, ALTER,
    CREATE_SEQUENCE, CREATE_VIEW, CREATE_SCHEMA, CREATE_MACRO, DROP, PRAGMA,
    TRANSACTION, CREATE_TYPE, CREATE_TRIGGER, ATTACH, DETACH,

    // Helpers
    EXPLAIN, EXPLAIN_ANALYZE, EMPTY_RESULT, EXECUTE, PREPARE, VACUUM, EXPORT, SET,
    SET_VARIABLE, LOAD, INOUT_FUNCTION, RESULT_COLLECTOR, RESET, EXTENSION,
    VERIFY_VECTOR, UPDATE_EXTENSIONS, CONNECT, DISCONNECT, EXTERNAL_RESOURCE,

    // Secret
    CREATE_SECRET,

    // Physical-only recursive CTE specialization. Keep appended to preserve existing enum values.
    RECURSIVE_KEY_JOIN, SECURE_VIEW,
    // Source of a MERGE INTO action pipeline. Keep appended to preserve existing enum values.
    MERGE_ACTION_SOURCE,
};
```

### 5c. Concrete C++ classes (by directory, `src/include/duckdb/execution/operator/`)

- **`aggregate/`**: `PhysicalHashAggregate` (`physical_hash_aggregate.hpp`), `PhysicalPerfectHashAggregate` (`physical_perfecthash_aggregate.hpp`), `PhysicalPartitionedAggregate` (`physical_partitioned_aggregate.hpp`), `PhysicalUngroupedAggregate` (`physical_ungrouped_aggregate.hpp`), `PhysicalLimitedDistinct` (`physical_limited_distinct.hpp`), `PhysicalWindow` (`physical_window.hpp`), `PhysicalStreamingWindow` (`physical_streaming_window.hpp`).
- **`join/`**: `PhysicalJoin : public CachingPhysicalOperator` (`physical_join.hpp` — common base), `PhysicalComparisonJoin : public PhysicalJoin` (`physical_comparison_join.hpp`), `PhysicalHashJoin : public PhysicalComparisonJoin` (`physical_hash_join.hpp`), `PhysicalNestedLoopJoin : public PhysicalComparisonJoin` (`physical_nested_loop_join.hpp`), `PhysicalRangeJoin : public PhysicalComparisonJoin` (`physical_range_join.hpp` — base for merge/IE joins), `PhysicalPiecewiseMergeJoin : public PhysicalRangeJoin` (`physical_piecewise_merge_join.hpp`), `PhysicalIEJoin : public PhysicalRangeJoin` (`physical_iejoin.hpp` — Inequality-join algorithm for `<`/`>`/`BETWEEN`-style joins), `PhysicalAsOfJoin : public PhysicalComparisonJoin` (`physical_asof_join.hpp`), `PhysicalBlockwiseNLJoin : public PhysicalJoin` (`physical_blockwise_nl_join.hpp`), `PhysicalCrossProduct : public CachingPhysicalOperator` (`physical_cross_product.hpp`), `PhysicalPositionalJoin` (`physical_positional_join.hpp`), `PhysicalDelimJoin` / `PhysicalLeftDelimJoin` / `PhysicalRightDelimJoin` (`physical_delim_join.hpp`, `physical_left_delim_join.hpp`, `physical_right_delim_join.hpp` — the physical counterpart of decorrelated dependent joins), `PhysicalRecursiveCTEKeyJoin : public CachingPhysicalOperator` (`physical_recursive_cte_key_join.hpp`), `JoinFilterPushdown` (`join_filter_pushdown.hpp` — dynamic filter support object shared with `PhysicalComparisonJoin`).
- **`scan/`**: `PhysicalTableScan` (`physical_table_scan.hpp`), `PhysicalDummyScan` (`physical_dummy_scan.hpp`), `PhysicalColumnDataScan` (`physical_column_data_scan.hpp`), `PhysicalExpressionScan` (`physical_expression_scan.hpp`), `PhysicalPositionalScan` (`physical_positional_scan.hpp`), `PhysicalEmptyResult` (`physical_empty_result.hpp`).
- **`order/`**: `PhysicalOrder` (`physical_order.hpp`), `PhysicalTopN` (`physical_top_n.hpp`).
- **`filter/`**: `PhysicalFilter : public CachingPhysicalOperator` (`physical_filter.hpp` — note it batches small chunks before pushing them downstream, via the `CachingPhysicalOperator` base).
- **`projection/`**: `PhysicalProjection` (`physical_projection.hpp`), `PhysicalUnnest` (`physical_unnest.hpp`), `PhysicalPivot` (`physical_pivot.hpp`), `PhysicalTableInOutFunction` (`physical_tableinout_function.hpp`).
- **`persistent/`** (DML): `PhysicalInsert` (`physical_insert.hpp`), `PhysicalBatchInsert` (`physical_batch_insert.hpp`), `PhysicalUpdate` (`physical_update.hpp`), `PhysicalDelete` (`physical_delete.hpp`), `PhysicalMergeInto` / `PhysicalMergeActionSource` (`physical_merge_into.hpp`), `PhysicalCopyToFile` / `PhysicalBatchCopyToFile` (`physical_copy_to_file.hpp`, `physical_batch_copy_to_file.hpp`), `PhysicalCopyDatabase` (`physical_copy_database.hpp`), `PhysicalExport` (`physical_export.hpp`), and a helper header, `copy_batch_slicer.hpp`.
- **`schema/`** (DDL): `PhysicalCreateTable` (`physical_create_table.hpp`), `PhysicalCreateIndex` (`physical_create_index.hpp`), `PhysicalCreateView` (`physical_create_view.hpp`), `PhysicalCreateSchema`, `PhysicalCreateSequence`, `PhysicalCreateType`, `PhysicalCreateTrigger`, `PhysicalCreateFunction`, `PhysicalAlter`, `PhysicalDrop`, `PhysicalAttach`, `PhysicalDetach`.
- **`set/`**: `PhysicalUnion` (`physical_union.hpp`), `PhysicalRecursiveCTE` / `PhysicalRecursiveCTEStateScan : public PhysicalColumnDataScan` (`physical_recursive_cte.hpp`), `PhysicalCTE` / `PhysicalCTEConsumerSource` (`physical_cte.hpp`).
- **`helper/`**: `PhysicalResultCollector` (`physical_result_collector.hpp` — the terminal sink gathering final query results), `PhysicalResultSink : public PhysicalResultCollector` (`physical_result_sink.hpp` — its header comment calls it "the root operator of every chunk-producing plan"), `PhysicalLimit` / `PhysicalStreamingLimit` / `PhysicalLimitPercent` (`physical_limit.hpp`, `physical_streaming_limit.hpp`, `physical_limit_percent.hpp`), `PhysicalReservoirSample` (`physical_reservoir_sample.hpp`), `PhysicalExplainAnalyze` (`physical_explain_analyze.hpp`), `PhysicalVacuum`, `PhysicalPragma`, `PhysicalSet` / `PhysicalSetVariable`, `PhysicalReset`, `PhysicalPrepare`, `PhysicalExecute`, `PhysicalTransaction`, `PhysicalConnect` / `PhysicalDisconnect`, `PhysicalCreateSecret`, `PhysicalSecureView`, `PhysicalExternalResource`, `PhysicalVerifyVector` (`physical_verify_vector.hpp` — debug-only vector-format fuzzing operator, see §5d).

**Async results.** `PhysicalResultSink` is the root of every plan that produces rows. Results are produced by worker threads into a buffer that the client reads from, with back-pressure. A `ResultFormat` is the single interface for "how a result is delivered": it turns the rows into *result units* on the worker threads. Chunks are the default format; Arrow is just another format, `ArrowFormat` (`src/common/arrow/arrow_format.cpp`), which `duckdb_query_arrow` and ADBC also use. There are no separate Arrow or batch collector operators, and `QueryResult` has no subclasses (PRs #25727, #25801). Rules that follow from this design:
- The planner does not force a statement to materialize its result; the consumer decides (`result_ordering.hpp`). For example, `INSERT ... RETURNING` can be streamed (PR #26880).
- Closing a result before it completes rolls back its statement in autocommit mode (PR #26314). In the C API, destroying an unfinished result aborts it.
- A connection is busy until its statement completes. Submitting a new statement while the connection still holds an unfinished result fails with `DUCKDB_ERROR_RESOURCE_IN_USE` (value 44 in `src/include/duckdb.h`) (PR #26747).
- Execution code for this lives in `src/main/client_context_execution.cpp`, split out of `client_context.cpp`.

### 5d. Logical → physical translation

`PhysicalPlanGenerator` — `src/include/duckdb/execution/physical_plan_generator.hpp`, impl `src/execution/physical_plan_generator.cpp` (constructor at line 18, `Plan` at 24, `ResolveAndPlan` at 30, `PlanInternal` at 56, `CreatePlan` at 75, `CreatePlanInternal` at 84). Full control flow:

```cpp
PhysicalPlanGenerator::PhysicalPlanGenerator(ClientContext &context) : context(context) {}

unique_ptr<PhysicalPlan> PhysicalPlanGenerator::Plan(unique_ptr<LogicalOperator> op) {
    auto &plan = ResolveAndPlan(std::move(op));
    plan.Verify();
    return std::move(physical_plan);
}

PhysicalOperator &PhysicalPlanGenerator::ResolveAndPlan(unique_ptr<LogicalOperator> op) {
    auto &profiler = QueryProfiler::Get(context);

    // Resolve the types of each operator.
    { auto timer = profiler.StartTimer<MetricPhysicalPlannerResolveTypes>();
      op->ResolveOperatorTypes(); }

    // Resolve the column references.
    { auto timer = profiler.StartTimer<MetricPhysicalPlannerColumnBinding>();
      ColumnBindingResolver resolver;
      resolver.VisitOperator(*op); }

    // Create the main physical plan.
    { auto timer = profiler.StartTimer<MetricPhysicalPlannerCreatePlan>();
      physical_plan = PlanInternal(*op); }

    return physical_plan->Root();
}

unique_ptr<PhysicalPlan> PhysicalPlanGenerator::PlanInternal(LogicalOperator &op) {
    if (!physical_plan) {
        physical_plan = make_uniq<PhysicalPlan>(Allocator::Get(context));
    }
    op.estimated_cardinality = op.EstimateCardinality(context);
    physical_plan->SetRoot(CreatePlan(op));
    physical_plan->Root().estimated_cardinality = op.estimated_cardinality;

    auto debug_verify_vector = Settings::Get<DebugVerifyVectorSetting>(context);
    if (debug_verify_vector != DebugVectorVerification::NONE) {
        if (debug_verify_vector != DebugVectorVerification::DICTIONARY_EXPRESSION &&
            debug_verify_vector != DebugVectorVerification::VARIANT_VECTOR &&
            debug_verify_vector != DebugVectorVerification::SHREDDED_VECTOR) {
            physical_plan->SetRoot(Make<PhysicalVerifyVector>(physical_plan->Root(), debug_verify_vector));
        }
    }
    return std::move(physical_plan);
}

PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalOperator &op) {
    const auto repeatable = ClassifyLogicalOperatorRepeatability(op) == LogicalOperatorRepeatability::REPEATABLE;
    auto &result = CreatePlanInternal(op);
    if (!repeatable) {
        non_repeatable_operators.insert(result);
    }
    return result;
}

PhysicalOperator &PhysicalPlanGenerator::CreatePlanInternal(LogicalOperator &op) {
    switch (op.type) {
    case LogicalOperatorType::LOGICAL_GET:
        return CreatePlan(op.Cast<LogicalGet>());
    case LogicalOperatorType::LOGICAL_PROJECTION:
        return CreatePlan(op.Cast<LogicalProjection>());
    case LogicalOperatorType::LOGICAL_EMPTY_RESULT:
        return CreatePlan(op.Cast<LogicalEmptyResult>());
    case LogicalOperatorType::LOGICAL_FILTER:
        return CreatePlan(op.Cast<LogicalFilter>());
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
        return CreatePlan(op.Cast<LogicalAggregate>());
    case LogicalOperatorType::LOGICAL_WINDOW:
        return CreatePlan(op.Cast<LogicalWindow>());
    case LogicalOperatorType::LOGICAL_UNNEST:
        return CreatePlan(op.Cast<LogicalUnnest>());
    case LogicalOperatorType::LOGICAL_LIMIT:
        return CreatePlan(op.Cast<LogicalLimit>());
    case LogicalOperatorType::LOGICAL_SAMPLE:
        return CreatePlan(op.Cast<LogicalSample>());
    case LogicalOperatorType::LOGICAL_ORDER_BY:
        return CreatePlan(op.Cast<LogicalOrder>());
    case LogicalOperatorType::LOGICAL_TOP_N:
        return CreatePlan(op.Cast<LogicalTopN>());
    case LogicalOperatorType::LOGICAL_COPY_TO_FILE:
        return CreatePlan(op.Cast<LogicalCopyToFile>());
    case LogicalOperatorType::LOGICAL_DUMMY_SCAN:
        return CreatePlan(op.Cast<LogicalDummyScan>());
    case LogicalOperatorType::LOGICAL_ANY_JOIN:
        return CreatePlan(op.Cast<LogicalAnyJoin>());
    case LogicalOperatorType::LOGICAL_ASOF_JOIN:
    case LogicalOperatorType::LOGICAL_DELIM_JOIN:
    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
        return CreatePlan(op.Cast<LogicalComparisonJoin>());
    case LogicalOperatorType::LOGICAL_CROSS_PRODUCT:
        return CreatePlan(op.Cast<LogicalCrossProduct>());
    // ... (continues for every LogicalOperatorType)
    }
}
```

Notable stages, in order: (1) `ResolveOperatorTypes()` on the whole tree, (2) `ColumnBindingResolver` (`src/execution/column_binding_resolver.cpp`) converts logical `ColumnBinding{table_index, column_index}` pairs into flat physical-chunk column offsets (physical operators don't carry table indices), (3) `CreatePlanInternal`'s big switch dispatches to overloaded `CreatePlan(LogicalGet&)` / `CreatePlan(LogicalProjection&)` / etc., each implemented in its own file under `src/execution/physical_plan/plan_*.cpp` — 55 files, one per logical operator type: `plan_get.cpp`, `plan_projection.cpp`, `plan_filter.cpp`, `plan_aggregate.cpp`, `plan_window.cpp`, `plan_unnest.cpp`, `plan_limit.cpp`, `plan_order.cpp`, `plan_top_n.cpp`, `plan_comparison_join.cpp` (**this is where join-algorithm selection happens** — hash join vs. merge join vs. IE-join vs. nested-loop, based on predicate types and the earlier `BUILD_SIDE_PROBE_SIDE` optimizer decision), `plan_any_join.cpp`, `plan_asof_join.cpp`, `plan_cross_product.cpp`, `plan_positional_join.cpp`, `plan_delim_join.cpp`, `plan_distinct.cpp`, `plan_set_operation.cpp`, `plan_recursive_cte.cpp`, `plan_cte.cpp`, `plan_insert.cpp`, `plan_update.cpp`, `plan_delete.cpp`, `plan_merge_into.cpp`, `plan_create.cpp`, `plan_create_table.cpp`, `plan_create_index.cpp`, `plan_alter.cpp`, `plan_drop.cpp`, `plan_copy_to_file.cpp`, `plan_copy_database.cpp`, `plan_pivot.cpp`, `plan_sample.cpp`, `plan_dummy_scan.cpp`, `plan_empty_result.cpp`, `plan_expression_get.cpp`, `plan_column_data_get.cpp`, `plan_delim_get.cpp`, and the DDL/helper-statement plumbing files (`plan_explain.cpp`, `plan_export.cpp`, `plan_pragma.cpp`, `plan_prepare.cpp`, `plan_execute.cpp`, `plan_set.cpp`, `plan_reset.cpp`, `plan_load.cpp`, `plan_vacuum.cpp`, `plan_transaction.cpp`, `plan_attach.cpp`, `plan_detach.cpp`, `plan_connect.cpp`, `plan_disconnect.cpp`, `plan_create_secret.cpp`, `plan_secure_view.cpp`, `plan_external_resource.cpp`, `plan_update_extensions.cpp`).
- (4) The physical tree is optionally wrapped in a debug `PhysicalVerifyVector` operator when the `debug_verify_vector` setting requests vector-format fuzzing (i.e., re-representing every intermediate vector as dictionary/constant/etc. to shake out format-specific bugs — excluded for `DICTIONARY_EXPRESSION`/`VARIANT_VECTOR`/`SHREDDED_VECTOR` modes).

### 5e. Handoff object

`PhysicalPlan` — owns the arena of `PhysicalOperator`s and exposes `Root()` — is what's handed to the `Executor` for execution (§6).

### 5f. Example query for this stage

`SELECT o.custkey, SUM(l.extendedprice) FROM orders o JOIN lineitem l ON o.orderkey = l.orderkey GROUP BY o.custkey ORDER BY 2 DESC LIMIT 10` — exercises `PhysicalTableScan` ×2, `PhysicalHashJoin`, `PhysicalHashAggregate` (or `PhysicalPartitionedAggregate` if run in parallel), `PhysicalTopN` (`ORDER BY` + `LIMIT` fused by the `TOP_N` optimizer pass, §4b step 33), and `PhysicalResultCollector` at the root.

---

## 6. Execution engine (vectorized batch model, pipelines, and the JIT question)

### 6a. `DataChunk` / `Vector` — the vectorized batch representation

- `DataChunk` — `src/include/duckdb/common/types/data_chunk.hpp` — a horizontal batch of `Vector`s. This is the object every operator's `Execute`/`Sink`/`GetData` call receives and produces; default/max batch size is `STANDARD_VECTOR_SIZE` (2048 rows in mainline DuckDB; `#define STANDARD_VECTOR_SIZE DEFAULT_STANDARD_VECTOR_SIZE` at `src/include/duckdb/common/vector_size.hpp:20`).
- `Vector` — `src/include/duckdb/common/types/vector.hpp` — a single typed column-batch. Its physical representation is tagged by `VectorType` (`src/include/duckdb/common/enums/vector_type.hpp:15`):

  ```cpp
  enum class VectorType : uint8_t {
      FLAT_VECTOR,       // Flat vectors represent a standard uncompressed vector
      FSST_VECTOR,       // Contains string data compressed with FSST
      CONSTANT_VECTOR,   // Constant vector represents a single constant
      DICTIONARY_VECTOR, // Dictionary vector represents a selection vector on top of another vector
      SEQUENCE_VECTOR,   // Sequence vector represents a sequence with a start point and an increment
      SHREDDED_VECTOR    // Shredded variant vector
  };
  ```

  This confirms the classic flat/constant/dictionary/sequence formats. Two additions beyond the classic set: `FSST_VECTOR` (operate directly on FSST-compressed string data without decompressing — a well-known DuckDB performance trick) and `SHREDDED_VECTOR` (part of v2.0's VARIANT/shredded-column support; see commit `e5bc7c2c76`, "Add support for shredded vector serialization, and clean-up vector serialization by moving to vector buffers").
- Supporting types: `SelectionVector` (`src/include/duckdb/common/types/selection_vector.hpp` — the indirection layer underlying `DICTIONARY_VECTOR`), `VectorBuffer`/`VectorCache` (`vector_buffer.hpp`, `vector_cache.hpp` — allocation/reuse of the underlying data buffers across chunks in a pipeline, avoiding per-chunk allocation).

### 6b. Pipeline / Sink-Source-Operator model

`src/include/duckdb/parallel/` holds the push-based scheduling apparatus:
- `pipeline.hpp` — `Pipeline` (a linear chain of operators from one source to one sink) and `PipelineTask : public ExecutorTask` (the schedulable unit that drives one thread through one pipeline instance):
  ```cpp
  // src/include/duckdb/parallel/pipeline.hpp:29-56
  enum class PipelineInputMode : uint8_t { SCHEDULED_SOURCE, EXTERNAL_INPUT };
  enum class PipelineDependencyType : uint8_t { REQUIRED, OPTIONAL_DEPENDENCY };

  //! A dependency of a Pipeline on another Pipeline within the same MetaPipeline
  struct PipelineDependency {
      PipelineDependency(Pipeline &pipeline_p, PipelineDependencyType type_p);
      weak_ptr<Pipeline> pipeline;
      PipelineDependencyType type;
  };

  class PipelineTask : public ExecutorTask {
      static constexpr const idx_t PARTIAL_CHUNK_COUNT = 50;
  public:
      explicit PipelineTask(Pipeline &pipeline_p, shared_ptr<Event> event_p);
      Pipeline &pipeline;
      unique_ptr<PipelineExecutor> pipeline_executor;
      optional_idx reserved_batch_index;
      // ...
  };
  ```
- `meta_pipeline.hpp` — `MetaPipeline`: a DAG of `Pipeline`s sharing a sink (e.g. the build-side pipeline and probe-side pipeline of a hash join share the join's `GlobalSinkState`).
- `pipeline_executor.hpp` — `PipelineExecutor`: drives one thread through one `Pipeline` instance, chunk by chunk.
- `event.hpp` + `pipeline_*_event.hpp` — the dependency-graph "when is this pipeline stage done" bookkeeping: `PipelineInitializeEvent`, `PipelineFinishEvent`, `PipelineCompleteEvent`, `PipelinePrepareFinishEvent`, `PipelineBroadcastExchange` (`pipeline_broadcast_exchange.hpp`).
- `task_scheduler.hpp` + `task_scheduler_pool.hpp` + `task_scheduler_queue.hpp` + `task_scheduler_token.hpp` — the `TaskScheduler`, backed by a lock-free MPMC queue (`concurrentqueue.hpp`, the vendored moodycamel `concurrentqueue`, also present under `third_party/concurrentqueue`).
- Other headers in the directory: `async_result.hpp`, `interrupt.hpp`, `callback_async_task.hpp`, `sleep_async_task.hpp` (support for operators that wait on I/O and resume later), `scan_read_ahead.hpp` (read-ahead for scans), `pipeline_schedule.hpp`, `parallel_destroy_task.hpp`, `progress_verifier.hpp`, `task_counter.hpp`, `task_executor.hpp`, `task_notifier.hpp`, `base_pipeline_event.hpp`, `pipeline_event.hpp`.
- `executor_task.hpp` / `task.hpp` — the base schedulable unit of work.
- `thread_context.hpp` — per-thread execution context.

`PhysicalOperator` (`src/include/duckdb/execution/physical_operator.hpp`) itself defines the Sink/Source/Operator interface (Operator methods at lines 110-112, Source methods at 154-178, Sink methods at 208-223):

- **Operator (mid-pipeline) interface:**
  ```cpp
  virtual OperatorResultType Execute(ExecutionContext &context, DataChunk &input, DataChunk &chunk,
                                     GlobalOperatorState &gstate, OperatorState &state) const;
  virtual OperatorFinalizeResultType FinalExecute(ExecutionContext &context, DataChunk &chunk,
                                                  GlobalOperatorState &gstate, OperatorState &state) const;
  ```
  Pull-input/push-output row-batch transform — e.g. `PhysicalFilter`, `PhysicalProjection`.
- **Source interface:**
  ```cpp
  virtual unique_ptr<LocalSourceState> GetLocalSourceState(ExecutionContext&, GlobalSourceState&) const;
  virtual unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext&) const;
  protected: virtual SourceResultType GetDataInternal(ExecutionContext&, DataChunk&, OperatorSourceInput&) const;
  public:  SourceResultType GetData(ExecutionContext&, DataChunk&, OperatorSourceInput&) const;  // wraps GetDataInternal
  virtual OperatorPartitionData GetPartitionData(ExecutionContext&, DataChunk&, GlobalSourceState&,
                                                 LocalSourceState&, const OperatorPartitionInfo&) const;
  virtual bool IsSource() const { return false; }
  virtual bool ParallelSource() const { return false; }
  virtual bool HasSourceTasks() const { return false; }  // creates partitioned work not bounded by input chunks
  virtual TableFunctionParallelism SourceParallelism() const;
  ```
  This is where scan/source parallelism is decided per operator (e.g. `PhysicalTableScan::SourceParallelism()` reports whether its underlying table function supports parallel scanning).
- **Sink interface:**
  ```cpp
  //! The sink method is called constantly with new input, as long as new input is available. Note that this
  //! method CAN be called in parallel, proper locking is needed when accessing data inside the GlobalSinkState.
  virtual SinkResultType Sink(ExecutionContext&, DataChunk &chunk, OperatorSinkInput&) const;
  //! The combine is called when a single thread has completed execution of its part of the pipeline, it is
  //! the final time that a specific LocalSinkState is accessible. This method can be called in parallel while
  //! other Sink()/Combine() calls are active on the same GlobalSinkState.
  virtual SinkCombineResultType Combine(ExecutionContext&, OperatorSinkCombineInput&) const;
  //! The finalize is called when ALL threads are finished execution. It is called only once per pipeline, and
  //! is entirely single threaded. If Finalize returns SinkResultType::Finished, the sink is marked as finished
  virtual SinkFinalizeType Finalize(Pipeline&, Event&, ClientContext&, OperatorSinkFinalizeInput&) const;
  virtual SinkNextBatchType NextBatch(ExecutionContext&, OperatorSinkNextBatchInput&) const;
  ```
  This 3-phase Sink → Combine → Finalize protocol is exactly how e.g. `PhysicalHashJoin`'s build side and `PhysicalHashAggregate` accumulate parallel, thread-local partial state (`LocalSinkState`) before a single-threaded finalize step merges everything (`GlobalSinkState`).

**On "morsel-driven parallelism"** [31]**:** the literal word "morsel" does **not** appear anywhere in `src/execution`, `src/parallel` or their headers (confirmed by grep — zero hits). DuckDB's own source does not use that HyPer-paper-derived term internally, even though its parallelism model — dynamic, per-chunk work distribution via `TaskScheduler` plus the source-parallelism hooks above (`GetPartitionData`, `SourceParallelism`, `HasSourceTasks`) — is functionally what the outside world (blog posts, papers) calls "morsel-driven." Worth noting in the doc as a terminology callout: it's an external label applied to DuckDB's design, not a name DuckDB's own code uses.

### 6c. `Executor` / `TaskScheduler`

- `Executor` — `src/include/duckdb/parallel/executor.hpp` — owns the set of `MetaPipeline`s for one query, drives `Event` scheduling, exposes `Executor::Execute`.
- `TaskScheduler` — `src/include/duckdb/parallel/task_scheduler.hpp` — the thread pool; `task_scheduler_pool.hpp` / `task_scheduler_queue.hpp` implement the worker threads and the lock-free work queue.
- **Fair scheduling.** Each query puts its tasks into its own *producer* sub-queue of the moodycamel `ConcurrentQueue`. Worker threads use a `ConsumerToken` and rotate across producers, so one query cannot starve the others (PR #26809; `task_scheduler_token.hpp`). An idle thread yields briefly and then waits; an event wakes only the threads it can unblock.
- **Scan read-ahead** (`scan_read_ahead.hpp`). The async backlog budget is capped at `memory_limit/8`, with a minimum reservation of `memory_limit/16`. Files kept open by read-ahead count against an open-file budget shared with the HTTP client pool. Read-ahead skips row groups that a late Top-N boundary excludes.
- Out-of-memory in a worker thread is reported as a query error, not a process abort (PR #27965).

### 6d. `ExpressionExecutor` — scalar expression evaluation per vector

`src/include/duckdb/execution/expression_executor.hpp` (class at line 23) + `src/execution/expression_executor.cpp` + one file per expression kind under `src/execution/expression_executor/`:

```cpp
//! ExpressionExecutor is responsible for executing a set of expressions and storing the result in a data chunk
class ExpressionExecutor {
    friend class BoundIndex;
public:
    DUCKDB_API explicit ExpressionExecutor(ClientContext &context);
    DUCKDB_API ExpressionExecutor(ClientContext &context, const Expression *expression);
    DUCKDB_API ExpressionExecutor(ClientContext &context, const Expression &expression);
    DUCKDB_API ExpressionExecutor(ClientContext &context, const vector<unique_ptr<Expression>> &expressions);

    //! The expressions of the executor
    vector<const Expression *> expressions;
    //! The data chunk of the current physical operator, used to resolve column references and
    //! determines the output cardinality
    DataChunk *chunk = nullptr;
public:
    // ...
    //! Execute the set of expressions with the given input chunk and store the result in the output chunk
    DUCKDB_API void Execute(DataChunk *input, DataChunk &result);
};
```

Per-expression-kind implementation files: `execute_reference.cpp` (`BOUND_REF`), `execute_constant.cpp` (`BOUND_CONSTANT`), `execute_conjunction.cpp` (`BOUND_CONJUNCTION`), `execute_case.cpp` (`BOUND_CASE`), `execute_comparison.cpp`, `execute_operator.cpp` (`BOUND_OPERATOR`), `execute_function.cpp` (`BOUND_FUNCTION`), `execute_lambda.cpp` (`BOUND_LAMBDA`), `execute_parameter.cpp` (`BOUND_PARAMETER`).

**Confirmed dispatch mechanism** (read directly from `expression_executor.cpp`): a plain **`switch (expr.GetExpressionClass())`** appearing three times in the file (at lines 234, 279 and 322 — for `Execute`, a `Verify`-like pass, and a select/where-optimized pass respectively):

```cpp
switch (expr.GetExpressionClass()) {
case ExpressionClass::BOUND_REF:      ...
case ExpressionClass::BOUND_CASE:     ...
case ExpressionClass::BOUND_CONJUNCTION: ...
case ExpressionClass::BOUND_CONSTANT: ...
case ExpressionClass::BOUND_FUNCTION: ...
case ExpressionClass::BOUND_LAMBDA:   ...
case ExpressionClass::BOUND_OPERATOR: ...
case ExpressionClass::BOUND_PARAMETER: ...
// ...
}
```

Each branch calls a dedicated `ExecuteExpression(BoundXExpr&, ...)` overload that operates on a **whole `Vector`/`DataChunk` at a time** — e.g. `execute_comparison.cpp` runs a templated comparison kernel over the input vectors' formats and produces one output vector per call — rather than compiling the expression tree into a single fused function body per query. This is a textbook **tree-walking interpreter with per-vector (batch-at-a-time) evaluation, not per-row evaluation and not compiled/codegen evaluation.**

The `Select` switch (line 322) has a `BOUND_OPERATOR` case. Operator expressions used as filters (for example `IS NULL` or `IN` lists) produce a selection vector directly, instead of going through `DefaultSelect`, which evaluates the expression into a boolean vector first and then scans it. `AND`/`OR` validity on flat vectors is computed 64 rows at a time (PR #28046).

### 6e. The JIT question — definitively answered: NO, DuckDB does not JIT-compile queries to machine code

Exhaustive case-insensitive greps across the entirety of `src/` (parser, planner, optimizer, execution, parallel all included):

```
$ grep -rniI "\bjit\b" --include=*.cpp --include=*.hpp src        → 0 hits
$ grep -rniI "asmjit" --include=*.cpp --include=*.hpp src         → 0 hits
$ grep -rniI "codegen|code.gen|code generation" --include=*.cpp --include=*.hpp src → 0 hits (one unrelated
                                                                     hit in a *comment* about "amount of code
                                                                     generated for sorting" in sort_key.hpp,
                                                                     meaning compiler-generated C++ template
                                                                     instantiations, not runtime codegen)
$ grep -rniI "computed goto|&&label|goto \*" --include=*.cpp --include=*.hpp src → 0 hits
$ grep -rniI "llvm" --include=*.cpp --include=*.hpp src           → 3 hits, all incidental:
    src/include/duckdb/common/thread_annotation.hpp:11  (a link to Clang's docs)
    src/include/duckdb/common/compatible_with_ipp.hpp:12-13  (a comment noting a shared_ptr
                                                               implementation was borrowed from
                                                               llvm-project's libcxx, i.e. STL-adjacent
                                                               code, unrelated to query execution)
```

Combined with:
- The `ExpressionExecutor`'s plain `switch`-on-`ExpressionClass` dispatch (§6d) — a classic interpreter, not a compile-to-native-code step.
- The absence of any codegen-related source directory (no `src/execution/jit/`, no `src/execution/codegen/`).
- `third_party/` contents (§1d) containing no compiler-construction library whatsoever (no LLVM, no asmjit, no libgccjit) — only `fmt` (string formatting), `re2` (regex), `utf8proc` (Unicode handling), `zstd`/`lz4`/`brotli`/`snappy`/`miniz` (compression), `mbedtls` (crypto/TLS), `jemalloc` (allocator), `parquet`/`yyjson`/`thrift` (data formats), `hyperloglog`/`tdigest` (approximate-aggregate sketches), `fastpforlib`/`fsst` (integer/string compression schemes used by the storage layer), `concurrentqueue` (lock-free MPMC queue for the task scheduler), `pdqsort`/`vergesort`/`ska_sort` (sort algorithms), `catch` (test framework).
- No "experimental JIT" or "adaptive execution" flag exists either. A grep for "adaptive" in header file names and code finds `src/include/duckdb/execution/adaptive_filter.hpp` and its users (the CSV sniffer, the multi-file reader's `multi_file_adaptive_filter_cache.hpp`, scan state, logging). All of them are **unrelated to codegen**:
  ```cpp
  // src/include/duckdb/execution/adaptive_filter.hpp:22-30
  struct AdaptiveFilterState {
      TimePoint monotonic_start;
  };

  enum class AdaptiveFilterSource : uint8_t {
      INITIAL,
      SEEDED,
  };
  ```
  This is a runtime-reordering heuristic for evaluating multiple `WHERE`-clause conjuncts in cheapest-first order based on observed per-conjunct selectivity — still fully interpreted, just adaptively re-ordered.

**Conclusion:** DuckDB is a **vectorized, tree-walking interpreter** — architecturally in the MonetDB/X100 (Vectorwise) lineage: batch-at-a-time (`DataChunk`/`Vector`) processing with virtual-dispatch physical operators and a switch-dispatched expression evaluator. It is explicitly **not** in the HyPer/Umbra lineage, which compiles each query to native machine code (HyPer via LLVM [13], Umbra via a custom compiling backend [14]). The X100 design is described by Boncz, Zukowski and Nes [27]. This can be stated with full confidence and should be one of the doc's clearest, most citable claims.

### 6f. Example queries for this stage

1. `SELECT a, b, a+b AS c FROM t WHERE a > 10 AND b < 100` — good minimal trace target for `ExpressionExecutor` dispatch: the projection list (`a`, `b`, `a+b`) and the `WHERE` conjunction (`a>10 AND b<100`, exercising `AdaptiveFilterState`'s conjunct reordering) both flow through `ExpressionExecutor::Execute` with different `ExpressionClass` branches per column.
2. `SELECT dept, SUM(salary) OVER (PARTITION BY dept ORDER BY hire_date) FROM employees` — exercises `PhysicalWindow`/`PhysicalStreamingWindow`, a `Pipeline` with a materializing sink phase before the window function can be evaluated, and vector formats transitioning between `FLAT_VECTOR` and (post-sort) partitioned slices.

---

## 7. Storage / Transactions / Indexes

### 7a. Single-file storage / block manager

`src/storage/`:
- `single_file_block_manager.cpp` — the on-disk block manager for DuckDB's default single-file format.
- `block_manager.hpp` (`src/include/duckdb/storage/block_manager.hpp:31`) — abstract base class:
  ```cpp
  class BlockManager {
  public:
      BlockManager() = delete;
      BlockManager(BufferManager &buffer_manager, const optional_idx block_alloc_size_p,
                   const optional_idx block_header_size_p);
      virtual ~BlockManager() = default;
      //! The buffer manager
      BufferManager &buffer_manager;
  public:
      BufferManager &GetBufferManager() const { return buffer_manager; }
      //! Creates a new block inside the block manager
      virtual unique_ptr<Block> ConvertBlock(block_id_t block_id, FileBuffer &source_buffer) = 0;
      // ...
  };
  ```
- `block.cpp`, `block_allocator.cpp`, `partial_block_manager.cpp` — block lifecycle and sub-block-granularity allocation (packing multiple small writes into one on-disk block).
- `magic_bytes.cpp` — file header/format-version detection.
- `storage_manager.cpp` — top-level per-attached-database storage orchestration.
- `buffer_manager.cpp` / `standard_buffer_manager.cpp` — the page/buffer cache and eviction policy.
- `checkpoint_manager.cpp` + `src/storage/checkpoint/` — rewrites live data into fresh blocks on `CHECKPOINT`.
- `temporary_file_manager.cpp` / `temporary_memory_manager.cpp` — spill-to-disk support for out-of-core joins/aggregates/sorts.
- `data_table.cpp`, `local_storage.cpp`, `optimistic_data_writer.cpp` — the table-storage abstraction and transaction-local (uncommitted) row storage.
- `src/storage/table/` — row-group / column-segment storage internals (not exhaustively enumerated here).
- `src/storage/statistics/` — per-segment/per-column statistics storage (feeds `statistics_propagator.cpp` in §4g and zone-map-style filter pushdown).
- `src/storage/metadata/` — metadata-block management.
- `src/storage/external_file_cache/` — caching layer for externally-read files (e.g. remote Parquet). It stores the exact byte ranges a reader asked for, not aligned fixed-size blocks (PR #26096; tunable through the `external_file_cache_*` settings). When an external file changes, all its cached blocks are dropped.
- `src/storage/serialization/` — including `serialize_wal.cpp` (WAL entry (de)serialization).
- `object_cache.cpp` — the `ObjectCache` implementation. It evicts cached objects in batches under memory pressure and stops once enough memory is reclaimed.
- `storage_info.cpp` + `version_map.json` — the table of storage-format and serialization versions per release.

**Storage version.** The latest storage version is `StorageVersion::V2_0_0 = 69`. `V2_0_1 = 69` is an alias for it (with `SerializationVersionDeprecated::V2_0_1 = 8`), and `"latest"` points at it (`src/include/duckdb/storage/storage_info.hpp`, `src/storage/storage_info.cpp`, `src/storage/version_map.json`; PR #28020). So v2.0.0 and v2.0.1 read and write the same format.

**Files from older versions.**
- Before storage version 2.0, `NULL` statistics lived only in validity segments. The zonemap check therefore also looks at the validity segment, so a filter that accepts `NULL` (for example `x IS NULL OR x > 2`) does not skip vectors that hold matching `NULL` rows in such files (PR #28355; issue #27566).
- A checkpoint of a file with a storage version below v1.5.0 keeps it readable by those older releases.
- Attaching an encrypted database without naming a storage version keeps its stored version.
- Plans and samples serialized before v2.0.0 keep their meaning (for example the ASOF predicate and `REPEATABLE` samples, PRs #28599, #28605). Deserialized expressions, query nodes and table references are checked for a valid shape (PR #26735). An optional *deserialization policy* can forbid some logical types and literal kinds; `json_deserialize_sql` and `json_execute_serialized_sql` use it to reject `POINTER` (PR #28416).

**Memory limit.** Large allocations go through the buffer manager, so they count against `memory_limit` and can be spilled. Examples: the state arenas of the perfect-hash aggregate and of limited `DISTINCT`, the flattened values of `quantile`, WAL entry buffers, FSST and dictionary analyze buffers, window partition masks and the sort's run-merge buffer (PR #26919 and related PRs). An in-memory database can load more than `memory_limit` when a temp directory is set, because small transient validity and nested-column segments are offloaded to disk too (PR #28146). `max_temp_directory_size` is enforced atomically when temp space is reserved. Join and aggregate hash tables ask the allocator for huge pages (best effort, PR #25869).

### 7b. WAL (Write-Ahead Log)

`WriteAheadLog` class — `src/include/duckdb/storage/write_ahead_log.hpp:44` (`WALInitState` at line 38), impl `src/storage/write_ahead_log.cpp`:

```cpp
enum class WALInitState { NO_WAL, UNINITIALIZED, UNINITIALIZED_REQUIRES_TRUNCATE, INITIALIZED };

//! The WriteAheadLog (WAL) is a log that is used to provide durability. Prior
//! to committing a transaction it writes the changes the transaction made to
//! the database to the log, which can then be replayed upon startup in case the
//! server crashes or is shut down.
class WriteAheadLog {
public:
    //! Initialize the WAL in the specified directory
    explicit WriteAheadLog(StorageManager &storage_manager, const string &wal_path, idx_t wal_size = 0ULL,
                           WALInitState state = WALInitState::NO_WAL,
                           optional_idx checkpoint_iteration = optional_idx());
    // ...
};
```

Replay logic: `src/storage/wal_replay.cpp`. Entry serialization: `src/storage/serialization/serialize_wal.cpp`. Write-time state tracked per-transaction in `src/transaction/wal_write_state.cpp`.

WAL and checkpoint rules:
- A checkpoint **always** starts a new WAL file, even when the current WAL is empty (`StorageManager::WALStartCheckpoint` in `src/storage/storage_manager.cpp`). This guards against a bug (issue #27465, fixed by `8845ee0bdfe`, PR #28274) where a forced checkpoint on an empty WAL kept the old file: a transaction that committed during the checkpoint went to a WAL with the old checkpoint number, and the database would not open again.
- After a restart, indexes stay "unbound" until first use, and WAL replay buffers index changes for them. A checkpoint binds every index that has buffered changes **before** it starts. If binding fails, an automatic checkpoint is skipped and the WAL is kept; an explicit `CHECKPOINT` fails with an error (commit `4dc57471583`, PR #26814).
- WAL replay truncates a broken last entry instead of replaying uncommitted data (PR #25931). It validates the `ParseInfo` type of `ALTER` entries and index allocation sizes (PR #28494).
- WAL entry buffers are allocated through the buffer allocator, so they count against the memory limit.

### 7c. MVCC / row versioning / undo buffers

`src/transaction/`:
- `DuckTransaction` — `src/include/duckdb/transaction/duck_transaction.hpp:36` (impl `duck_transaction.cpp`):
  ```cpp
  class DuckTransaction : public Transaction {
  public:
      DuckTransaction(DuckTransactionManager &manager, ClientContext &context, transaction_t start_time,
                      SnapshotView view, idx_t catalog_version);
      ~DuckTransaction() override;
      //! The start timestamp of this transaction
      transaction_t start_time;
      //! What this transaction sees: its own writes, and everything before its visibility bound
      // ...
  };
  ```
- `DuckTransactionManager` — `src/include/duckdb/transaction/duck_transaction_manager.hpp:34` (`DuckCleanupInfo` at line 23):
  ```cpp
  //! CleanupInfo collects transactions awaiting cleanup.
  //! This ensures we can clean up after releasing the transaction lock.
  struct DuckCleanupInfo {
      //! All transactions in a cleanup info share the same lowest_visibility_bound.
      VisibilityBound lowest_visibility_bound;
      vector<unique_ptr<DuckTransaction>> transactions;
  };

  class DuckTransactionManager : public TransactionManager {
  public:
      explicit DuckTransactionManager(AttachedDatabase &db);
      ~DuckTransactionManager() override;
      static DuckTransactionManager &Get(AttachedDatabase &db);
      //! Start a new transaction
      // ...
  };
  ```
  Starts/commits/rolls back transactions and tracks a `VisibilityBound` per generation, used to garbage-collect old row versions once no active transaction can still see them.
- `UndoBuffer` — `src/include/duckdb/transaction/undo_buffer.hpp:38` (`UndoBufferProperties` at line 26; impl `undo_buffer.cpp` + `undo_buffer_allocator.cpp`) — **the actual MVCC mechanism**:
  ```cpp
  struct UndoBufferProperties {
      idx_t estimated_size = 0;
      bool has_updates = false;
      bool has_deletes = false;
      bool has_index_deletes = false;
      bool has_catalog_changes = false;
      bool has_dropped_entries = false;
  };

  //! The undo buffer of a transaction is used to hold previous versions of tuples
  //! that might be required in the future (because of rollbacks or previous
  //! transactions accessing them)
  class UndoBuffer {
  public:
      struct IteratorState { /* ... */ };
      // ...
  };
  ```
- `commit_state.cpp` / `rollback_state.cpp` / `cleanup_state.cpp` — walk the undo chain to apply, undo, or garbage-collect entries respectively.
- `meta_transaction.cpp` — coordinates multi-database transactions (relevant when multiple databases are `ATTACH`ed and a single transaction spans them). A transaction may `DETACH` a database it has already used; its `COMMIT` then still commits the work done there (commit `684c42a5870`). Reusing a database name inside one transaction raises a transaction error (commit `fd5e7bd0d60`).

**Failures during commit and rollback.** At commit, `LocalStorage::Flush` appends to the shared indexes, then to the table, then records the append in the undo buffer. If a table append fails, its rows and index entries are removed (commit `70e791032db`). If anything **throws** in that window, the database is marked invalid while the commit lock is held, and `WALStartCheckpoint` re-checks invalidation after taking the lock, so a waiting checkpoint cannot save the half-written state (commit `2b210149b3b`, PR #27334). For this reason an index should report a failed append as an error value, as `ART::Append` does; an exception from an extension index (HNSW, RTree) at commit invalidates the database. An update chain stays intact when an update or its cleanup throws (PR #26227). On rollback, every client-context state is told about it even if one of them throws, and a failing rollback cannot escape `ClientContext`'s destructor (PR #26869).
- `transaction_context.cpp`, `transaction_manager.cpp`, `transaction.cpp` — the abstract `Transaction`/`TransactionManager` base classes `DuckTransaction`/`DuckTransactionManager` implement (an indirection that lets non-DuckDB storage engines, e.g. attached SQLite/Postgres via extensions, plug in their own transaction semantics).

### 7d. Compression schemes

`src/storage/compression/`:
- `bitpacking.cpp` / `bitpacking_hugeint.cpp` — frame-of-reference + bit-packing for integers.
- `dictionary/` (directory: `analyze.cpp`, `common.cpp`, `compression.cpp`, `decompression.cpp`) — dictionary encoding for low-cardinality columns.
- `dict_fsst.cpp` + `dict_fsst/` (directory: `analyze.cpp`, `compression.cpp`, `decompression.cpp`) + `fsst.cpp` — FSST (Fast Static Symbol Table [28]), a string-compression scheme designed so strings can be operated on (compared, hashed) without full decompression. `DICT_FSST` combines dictionary encoding with FSST.
- `dictionary_compression.cpp` — the registration glue for the dictionary scheme; `string_uncompressed.cpp` — the uncompressed string layout.
- `rle.cpp` — run-length encoding.
- `patas.cpp` and `chimp/` — floating-point compression schemes drawn from published research (Chimp [35], Patas).
- `alp/` + `alprd.cpp` — ALP (Adaptive Lossless floating-Point compression [36]), DuckDB's newer default scheme for floating-point columns.
- `roaring/` (directory: `analyze.cpp`, `common.cpp`, `compress.cpp`, `metadata.cpp`, `scan.cpp`) — Roaring-bitmap-based compression, used for validity/boolean-heavy columns.
- `zstd.cpp` — general-purpose block compression (wraps `third_party/zstd`).
- `uncompressed.cpp` / `fixed_size_uncompressed.cpp` / `validity_uncompressed.cpp` — fallback/no-compression paths.
- `numeric_constant.cpp`, `empty_validity.cpp` — degenerate-case optimizations (e.g. an entire segment being one constant value or all-valid).
- `standard_compression_state.cpp` / `compression_segment_reader.cpp` — the shared per-segment analyze/compress/scan state machine that every concrete scheme implements against. `CompressionSegmentReader` (`src/include/duckdb/storage/compression/compression_segment_reader.hpp:24`) is "a non-owning, bounds-checked view over [0, size) of a block or page, with a byte position".

The readers of the main schemes are built on `CompressionSegmentReader`, so a corrupt or malicious file raises a `DataCorruptionException` instead of reading past a buffer: dictionary (PR #26228), legacy FSST (PR #26290), Zstd (layout, vector data and overflow pages; cycles in overflow blocks are rejected), Roaring (array positions, run bounds, ordering) and `DICT_FSST` (a "bounded segment layout" with checks on `dict_count`, dictionary indices and string lengths, PR #27965). Bitpacked `BOOL` segments must hold only 0 and 1. Inconsistent persisted column data also raises `DataCorruptionException`. A checkpoint keeps `-0.0` distinct from `0.0` in `FLOAT`/`DOUBLE` columns (PR #25592), and codec scans can fill only selected rows at an output offset ("partial codec selections").

### 7e. ART index

`src/execution/index/art/`. Base class:

```cpp
// src/include/duckdb/execution/index/art/art.hpp:37
class ART : public BoundIndex {
public:
    friend class Leaf;
public:
    //! Index type name for the ART.
    // ...
};
```

Node/implementation files: `node.cpp` / `node_handle.cpp` (generic node dispatch), size-specialized node classes `node48.cpp`, `node256.cpp`, `node256_leaf.cpp` (the ART's characteristic small→large fanout specialization scheme — Node4/Node16/Node48/Node256 in the classic ART paper [33]). `Node4` and `Node16` are not in their own files: they are instantiations of the template `BaseNode<CAPACITY, TYPE>`, declared in `src/include/duckdb/execution/index/art/base_node.hpp:161` and `:178` and implemented in `base_node.cpp`. Small leaves (`Node7Leaf`, `Node15Leaf`) are likewise `BaseLeaf` instantiations in `base_leaf.cpp`, `leaf.cpp` / `base_leaf.cpp` (leaf entries — row-id storage, including multi-row-id leaves for non-unique indexes), `prefix.cpp` / `const_prefix_handle.cpp` / `prefix_handle.cpp` (path compression — the "radix" part of "adaptive radix tree"), `art_key.cpp` (encodes arbitrary DuckDB `Value`s into byte-comparable ART keys — this is where e.g. signed-integer sign-bit flipping and floating-point total-order encoding happen so lexicographic byte comparison matches SQL ordering), `iterator.cpp` (in-order range-scan support for `>`/`<`/`BETWEEN` predicates), `art_builder.cpp` / `art_merger.cpp` (bulk index construction — e.g. on `CREATE INDEX` over existing data, or merging in a large batch of new keys), `art_index.cpp` (top-level per-index bookkeeping), and the header-only helpers `art_operator.hpp` and `art_scanner.hpp`. There are 17 `.cpp` files. Before a fixed-size index buffer is loaded from disk, its offset and allocation size are checked, and index pointers are checked to point at an allocated segment before they are followed. `ART::Append` returns expression errors instead of throwing, so a failed commit can be retried (PR #27334). `UnboundIndex` is not `final`, so extensions can subclass it.

Shared index infrastructure (`src/execution/index/`, one level up from `art/`): `bound_index.cpp` (`BoundIndex` — the base class `ART` derives from) / `unbound_index.cpp` (a lazy, deserialize-on-first-use wrapper so indexes aren't fully loaded until needed), `fixed_size_allocator.cpp` / `fixed_size_buffer.cpp` (the node allocator backing the ART, persisted directly to on-disk blocks — i.e. the index's nodes live in the same block-manager-backed storage as table data), `index_type_set.cpp`.

### 7f. Representative test files (all sections)

- **PEG parser:** `test/sql/peg_parser/enable_peg_parser.test`, `test/sql/peg_parser/peg_syntax_error.test`, `test/sql/peg_parser/transformer/peg_transformer.test`.
- **Correlated subqueries / decorrelation:** `test/sql/subquery/scalar/test_unnest_subquery.test`, `test/sql/subquery/lateral/lateral_unnest.test_slow`, `test/sql/subquery/lateral/lateral_unnest_large.test_slow`, `test/sql/subquery/complex/nested_unnest_subquery.test`, `test/sql/subquery/scalar/test_correlated_aggregate_subquery.test_slow`, `test/sql/subquery/complex/correlated_list_aggregate.test_slow`, `test/sql/types/struct/unnest_struct_subquery.test`.
- **Join order / optimizer:** `test/sql/optimizer/plan/test_recursive_join_order_guard.test`, `test/sql/aggregate/distinct/rewrite/distinct_aggregate_rewrite_join_order.test`, `test/sql/optimizer/expression/test_cse.test`, `test/optimizer/partition_stats_prepared.test`, `test/optimizer/partial_aggregate_pushdown.test`, `test/optimizer/partial_aggregate_precomputation.test`, `test/optimizer/partitioned_execution_coverage.test`, `test/optimizer/partition_stats_exactness.test`.
- **Window functions:** `test/sql/window/test_window_string_agg.test`, `test/sql/window/test_window_dbplyr.test`, `test/sql/window/test_mode_window_varlen.test`, `test/sql/window/test_rank_orderby.test`, `test/sql/window/test_window_interval.test`.
- **ART index:** `test/sql/index/art/test_index_key_rebind.test`, `test/sql/index/art/test_art_empty_close_range.test`, `test/sql/index/art/test_art_tx_update_key.test` (index + MVCC interaction), `test/sql/index/art/insert_update_delete/test_art_update_with_dict_fsst.test`, `test/sql/index/art/insert_update_delete/test_art_simple_update.test`, `test/sql/index/art/insert_update_delete/test_art_sel_vector.test`, `test/sql/index/art/insert_update_delete/test_art_update_with_dict_fsst_and_replay.test`, plus whole subdirectories `test/sql/index/art/{multi_column,constraints,types,issues,scan,nodes,memory,create_drop,vacuum,storage}/`. Fuzzer-found ART edge cases: `test/fuzzer/pedro/art_prefix_error.test`, `test/fuzzer/pedro/art_concatenate_prefix.test`.
- **Compression:** `test/sql/delete/delete_compression_after_restart.test`, `test/sql/delete/delete_compression_blocked_by_old_snapshot.test`, `test/optimizer/compressed_materialization_variant.test`, `test/optimizer/compressed_materialization_grouping_sets.test`, `test/optimizer/compressed_materialization_geometry.test`, `test/fuzzer/duckfuzz/compressed_materialization_hugeint.test`, `test/fuzzer/duckfuzz/compressed_materialization_json.test`, `test/sql/optimizer/secure_view_pushdown_compression.test`.
- **WAL:** `test/sql/storage/test_read_only_wal_replay.test`, `test/sql/attach/attach_wal_alter.test`, `test/sql/attach/attach_replay_with_no_wal_writes.test`, `test/sql/attach/attach_no_wal_writes_mode.test`, `test/sql/catalog/comment_on_wal.test`, `test/sql/trigger/create_trigger_wal.test`, `test/sql/trigger/trigger_after_update_old_new_wal.test`, `test/sql/update/update_delete_wal.test`, `test/fuzzer/pedro/wal_rename_table.test`.

---

## Summary of load-bearing findings for the doc

1. **This checkout (`v2.0-cyanoptera`) has replaced `libpg_query`/Postgres-derived parsing with a native, hand-written PEG parser** (`src/parser/peg/`, grammar in `.gram` files under `src/parser/peg/grammar/statements/`, moved from the `autocomplete` extension into core on 2026-09-24). v2.0.0 was tagged on 2026-10-20 with this parser, so it is the parser of a stable release. If the doc targets DuckDB v1.x, describe the `libpg_query` pipeline instead (and note that v2.0 replaced it); if it targets v2.0, use this PEG architecture. Decide explicitly which version the doc is for and say so in the introduction.
2. **No JIT/codegen anywhere** — zero hits for jit/LLVM/asmjit/codegen/computed-goto across the entire `src/` tree (§6e). Confirmed via `ExpressionExecutor`'s plain `switch`-dispatch tree-walking interpreter operating on whole `Vector`/`DataChunk` batches (§6d), and via `third_party/`'s contents containing no compiler-construction library. DuckDB is firmly in the Vectorwise/X100 vectorized-interpreter lineage [27], not the HyPer/Umbra compiling-engine lineage [13]. This can be stated as a definitive, well-evidenced claim in the doc.
3. **Join ordering is DPhyp-based** — dynamic programming over a query hypergraph, with an explicit paper citation to Moerkotte and Neumann [20] at `src/optimizer/join_order/plan_enumerator.cpp:600`, falling back to a greedy algorithm when the DP search space is too large (`plan_enumerator.cpp:244`). The greedy fallback also keeps the input join order as a candidate (PR #28362).
4. **Subquery decorrelation (Neumann and Kemper's "unnesting arbitrary queries" [12]) happens at the planner level**, not in `src/optimizer/`: `FlattenDependentJoins` in `src/planner/subquery/flatten_dependent_join.cpp`, with `src/optimizer/deliminator.cpp` and `src/optimizer/unnest_rewriter.cpp` doing later cleanup of the resulting delim-join structure in the optimizer pipeline proper.
5. The `BoundStatement{ plan: LogicalOperator, types, names, extra_info }` struct (`src/include/duckdb/planner/bound_statement.hpp`) is the actual "Bind object" handoff out of the binder; binding and initial logical-plan construction are fused in DuckDB (binder methods in `binder/query_node/plan_*.cpp` and `binder/tableref/plan_joinref.cpp` directly build `LogicalGet`/`LogicalProjection`/etc. nodes as a byproduct of binding — there's no separate unoptimized IR distinct from `LogicalOperator`).
6. The optimizer pipeline is a **long, fixed, hardcoded sequence of ~40 named passes** in `Optimizer::RunBuiltInOptimizers()` (`src/optimizer/optimizer.cpp`, called from `Optimizer::Optimize()`), each independently togglable via `disabled_optimizers`, full ordered list given in §4b — plus a 37-rule expression-simplification rule list run as the second pass (§4c).
7. The "morsel-driven parallelism" label from the HyPer paper [31] and blog-post world does not appear verbatim in DuckDB's own source (§6b) — the mechanism it describes (dynamic per-chunk parallel work distribution via `TaskScheduler` + per-operator source-parallelism hooks) is present, just not under that name.

8. Query results are produced asynchronously. A `ResultFormat` (chunks or Arrow) builds output on the worker threads, and `PhysicalResultSink` is the root of every row-producing plan (§5c).

All paths in this document are relative to the root of https://github.com/duckdb/duckdb at commit `cb5673685153`. This file is intended as raw source material for the narrative chapters.
