# Update log — cross-chapter consistency pass (after u01–u11)

DuckDB HEAD `cb5673685153`. All edits are in `chapters/` (not README, not appendix-c).

## 1. Optimizer pass numbers

- Checked Chapter 7 §7.4 against `src/optimizer/optimizer.cpp`: 48 `RunOptimizer` calls =
  REMOTE_PUSHDOWN (line 223, in `OptimizeStatement`, not counted) + 45 numbered passes
  (EXPRESSION_REWRITER #1 at :253 … SCALAR_FN_PUSHDOWN #45 at :561) + 2 EXTENSION hooks.
  SELF_JOIN_AGGREGATE #3, REDUNDANT_ORDER_KEYS #21. The overview list matches.
- All part files (u06–u10), Ch. 8 (JOIN_ORDER #17, BUILD_SIDE_PROBE_SIDE #27) and parts 15–20 (u09's CSE #25,
  STATISTICS_PROPAGATION #34) already agreed with §7.4; no conflicting numbers found.
- Ch. 6 §6.7/§6.10: "positions 11 and 23" (REMOTE_PUSHDOWN counted) → #10/#22, §7.4 numbering; callout.
- Ch. 19: dropped the "first edition #N; #M at HEAD" dual form in running text (HEAD numbers only);
  rewrote both pass-numbering asides (Ch. 6/Ch. 8 first-edition discrepancies now explained as
  history); first-edition numbers kept once in the existing callout.
- Ch. 1 callout: "~45 passes still holds" → 48 call sites / 45 numbered passes, #3 and #21.
- "~40"/"~43" pass counts → "~45" in Ch. 1, 5, 8, 9, 19, glossary, overview, parts 16, 23, 24, 25 (~20 places).
- Distance words fixed: part 03 ("two passes later" → five), part 11 (×2, "three" → four), part 25
  ("sixteen" → fifteen).
- Overview: `FilterStatisticsOptimizer` note corrected (its inner steps run under
  STATISTICS_PROPAGATION / FILTER_PUSHDOWN timers and toggles).

## 2. Delim joins become CTEs by default

Verified: `delim_join_as_cte` default `true` at both commits (flipped by `a7b37725088`, 2026-06-16;
deprecated by `9b532ae44bc`); `DecorrelateIndependent` (flatten_dependent_join.cpp:281-292) calls
`DelimJoinCTERewriter::Rewrite`; `VerifyNoDelim` throws if any DELIM_JOIN/DELIM_GET remains;
`Planner::CreatePlan` (planner.cpp:258) calls it. Ch. 6 §6.4.1 is correct.
Corrected (with "already wrong at d8a1bd4f4f" callouts): Ch. 1 (also "inside the binder" → planner),
Ch. 9 §9.3.2 (physical planner never calls PlanDelimJoin by default), Ch. 19 §19.3.3–§19.3.7 (Query B
trace now labelled as the `delim_join_as_cte = false` form; default CTE form described), glossary
(DelimJoin, Decorrelation), overview #10 DELIMINATOR, part 07 (filter pushdown DELIM branch,
CheckMarkToSemi), part 09 (fuzzer test), part 13 (§13.1, §13.3, §13.3.6, §13.4, §13.5, §13.6:
deliminator.test never sets it false, so it runs the CTE form), part 20, part 26 §7.26.6.
Not verified: which joins the CTE eliminators remove for Query B (no binary).

## 3. Stale counts

- Rule count 35 → 37: parts 04 (×3), 05, 06 (×4), 08, 16. (Overview's "35 more rules" comment is
  correct: 37 minus the 2 shown.)
- Logical operator counts: already correct (57/61/67) everywhere.
- 64-relation ceiling: Ch. 8 §8.4 and §8.11 reworded — only the exact search's neighbour-set cache
  key assumes relation numbers < 64; no limit on relation sets.
- PEG "day before": part 08 method note fixed. "pre-release" checkout: part 16 → "v2.0.0 plus 1,041
  commits"; part 10 "past v2.0.0" → same.
- `LogicalDelimJoin` as class name: Ch. 9, parts 09 (×2), 14, 20 fixed.

## 4. Cross-references

- part 07:104 §1.7 → §1.3; part 16:233 §7.16.2.1 → named subsection in §7.16.2;
  part 21 §21.1.4.1 (×3; internal, not a paper) → §21.1.7.
- Ch. 9: §9.5.1/.3/.4/.5 → §9.4.x (8 refs); §9.4.2 → §9.3.2 for `delim_scans`.
- Ch. 14 §14.11 → "Chapter 9 (§9.3.1)". Ch. 2 "Chapter 7, Part 24" is correct.
- Glossary: PhysicalResultSink → §9.3.10 only; Read-ahead (no chapter covers it); UnboundIndex → Ch. 18.
- A script confirmed every remaining `§N.x` reference matches a heading.

## Not resolved / for others

- `chapters/appendix-c-bibliography.md:244-247` still says "pre-release … ~14,600 commits" and
  "one day before" (off-limits). A new untracked `appendix-c-references.md` appeared during this pass.
- Part 09 uses 7.9/7.10 headings that clash with part 10 (pre-existing).
