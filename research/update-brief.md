# Brief: updating the book to the new DuckDB checkout (November 2026)

You are updating part of "How DuckDB Works", a from-source internals book in
`/home/nick.othieno/Investigations/how_duckdb_works/`. Read `README.md` and
`chapters/00-front-matter.md` (only skim them if they are not yours) to learn the
book's conventions.

## What changed

- The book was written against `/home/nick.othieno/Github/duckdb` at commit
  `d8a1bd4f4f` (2026-09-25), branch `v2.0-cyanoptera`.
- The checkout is now at `cb5673685153` (2026-11-12, `git describe`:
  `v2.0.0-1041-gcb567368515`), same branch. That is 3,846 commits later.
  **v2.0.0 has been tagged since the first edition.** The book's README described
  the old commit as "v1.5.5 + ~14,600 commits". Use `git describe` and
  `git log` if you need exact facts about tags.
- Useful commands (run in `/home/nick.othieno/Github/duckdb`):
  - `git diff d8a1bd4f4f HEAD --stat -- <path>` and `git diff d8a1bd4f4f HEAD -- <path>`
  - `git log --oneline d8a1bd4f4f..HEAD -- <path>`
  - `git show HEAD:<path>` / `grep -n` on the working tree (the working tree is HEAD)
  - `git log -S'<symbol>' --oneline d8a1bd4f4f..HEAD` to find where a symbol went
- Do not modify, build, or check out anything in the DuckDB repo. There is no
  built binary of this version; work from source and the `test/` suite.
  Do not run `make`.

## What to do with each file you own

1. Read the whole file.
2. Check every source claim against HEAD:
   - every `file:line` citation: fix the path and line numbers;
   - every code excerpt: it must be verbatim from HEAD (re-flowed for width only);
   - every description of behavior, class, function, enum, setting, pass order,
     or test file: fix it if HEAD differs. Test files cited by path must exist at
     HEAD; if a test moved or changed, update the path and what it shows.
3. Look for meaningful new work in your topic area:
   `git log --oneline d8a1bd4f4f..HEAD -- <the source dirs your chapters cover>`.
   Add coverage of real changes (new or removed passes, rewrites, operators,
   storage formats, settings, bug fixes that change documented behavior) at the
   same depth the chapter already uses: verbatim excerpts with `file:line`,
   ASCII diagrams where they help, and worked SQL from `test/**/*.test` files
   where possible. Skip trivial changes (CI, formatting, benchmarks).
4. Where documented behavior changed (not just moved lines), say so in the text
   with a short callout so a returning reader can find it:
   `> **Changed since the first edition (d8a1bd4f4f → cb5673685153):** ...`
   Do not add callouts for line-number-only moves.
5. Keep all existing detail that is still true. Remove or rewrite only what is
   now wrong. Do not shorten the chapter.

## Writing style

- All new and changed text must be in **simple English**: short sentences, common
  words, one idea per sentence. Explain a term the first time you use it. Do not
  invent jargon. Prefer a small concrete example over an abstract sentence.
- When you edit a paragraph, simplify that paragraph too. Leave untouched
  paragraphs as they are (the owner chose not to rewrite the whole book).
- Keep the existing Markdown structure, heading numbers and ASCII diagram style.

## Citations (IEEE)

- The book is moving to IEEE numbered references. You do not assign numbers.
  Cite external works with keys from `research/references-registry.md`, written
  `[@key]`, with any locator after a comma: `[@didi06, slide 9]`,
  `[@neumann2015unnesting, Sec. 3]`, `[@didi08, pp. 26–34]`.
- Convert every existing citation of an external work in your files to this
  form. Examples: "DiDi-03, slide 12" → `[@didi03, slide 12]`;
  "Neumann & Kemper's 2015 paper" → "Neumann and Kemper's paper
  [@neumann2015unnesting]". Authors' names may stay in the sentence when that
  reads naturally. Bare URLs to external works become a citation key.
- Source files and test files are NOT cited with keys. They keep inline
  `path:line` form.
- If you need a source that is not in the registry, use a new key and write its
  details to `research/refs-pending/<your-agent-id>.md` (authors, title,
  venue/publisher, year, volume/issue/pages, URL, DOI — only what you actually
  know or saw). Do not invent details. Do not browse the web; a later pass
  verifies all references online.

## Boundaries

- Edit only the files you own, plus:
  - `research/refs-pending/<your-agent-id>.md` (only if needed), and
  - `research/update-log/<your-agent-id>.md` (required): a short list per file
    of what changed in DuckDB and what you changed in the book, plus any doubts.
    This feeds the README, front matter and change summary.
- Do not edit `README.md`, `chapters/00-front-matter.md`, the bibliography
  appendix, or other agents' files, unless they are listed as yours.
  If a cross-reference in your file points at a section number in another
  chapter, leave it, and note it in your update log if it may be wrong.
- Do not run git commands that change the book's repo (no commit, add, stash,
  checkout). Reading with `git diff` in the book directory is fine.
- Use the Read tool for big files in parts if needed. Use Edit for changes.

## When done

Reply with a short report (under 300 words): files changed, the main DuckDB
changes you documented, how many citations you converted, any pending
references you added, and anything you were unsure about.
