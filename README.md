# site-generator-speed-01 — a faster makefun-site 1.0.0

A dependency-free drop-in for make.fun's own makefun-site 1.0.0 static site generator.
For one case's input (`{ site, templates, files }`), `run(input)` returns exactly the output
tree makefun-site returns: a JSON object of every path it writes (each page's `index.html`,
the index and tag pages, `build-report.txt`, each static file) and that file's text, byte for
byte. `main.ts` is the driver the harness runs: it reads every case from `/input/cases.json`,
answers each one alone with `run`, and writes `/output/answers.json`.

## Approach

Same pipeline, same rules, same error strings as the reference — but the hot paths are built
for speed:

- **Markdown** renders straight into a parts array (no token objects, no AST). The inline
  renderer works in runs: it finds the next structural character with one regex, escapes the
  plain run in bulk, and locates closers with `indexOf` plus an odd-backslash escape check.
- **Templates** are parsed once into a node tree. Rendering walks the tree into a shared
  parts array with reused scope frames: a `for` loop reuses one scope frame and one `loop`
  record across its elements instead of allocating one per element (rendering is synchronous,
  so nothing retains them).
- **Front matter** is found by index (the closing line that is exactly `---`), so the body is
  never split and re-joined; only the small front-matter region is split into lines.
- **Slugify** takes an ASCII fast path (ASCII-only text skips Unicode normalisation) with a
  128-entry lookup table, and tag slugs are cached per name within a case.
- **Strings** are built with pre-sized arrays and a single `join('')`; collections are flat
  `Map`s in sorted file order, with no object spreading in loops.
- **V8** runs with incremental marking off (`--no-incremental-marking`, set in `main.ts`
  before `solve.js` is imported), the same flag the house baseline uses.

No dependencies, no clock, no randomness, no file access, no worker threads: the answer is a
function of the input alone, so two runs agree byte for byte.

## Files

- `solve.js` — the generator (`run(input)`), an ES module.
- `main.ts` — the driver (`node main.ts`).
- `README.md` — this file.

## Grade locally

From the kit directory, with the entry repo at `<entry>`:

```sh
node cli/makefun.mjs grade-local --kit . --bot <entry> --driver process --worktree
```

Docker is not installed on this VM, so `--driver process` is used. The gate is `passed=1`
(every practice case byte-identical to `practice/reference/expected.json`); only then does
CPU time count, against the house baseline in `baselines/makefun-site/`.
