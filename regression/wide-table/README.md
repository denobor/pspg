# "internal error" On Tables 63 Characters Wide Or More

A reproduction for a regression introduced in pspg 5.9.0 and still
present on master. Kept on a branch of a fork so that the finding, the
fixtures and the bisection survive; it is not a fix.

## TOPLINE

pspg 5.9.0 aborts with `internal error` on any psql-format table whose
total width is 63 characters or more, at every terminal size and
whatever the cell contents. 5.8.16 renders the same input correctly.
Six `leave("internal error")` assertions were added to `src/infra.c`
in 5.9.0; one of them fires.

## REPRODUCING IT

```bash
regression/wide-table/check            # the pspg on PATH
regression/wide-table/check ./pspg     # a build under test
```

Three fixtures sit either side of the threshold:

| Fixture | Border width | 5.8.16  | 5.9.0 |
|-------------|------|---------|----------------|
| `w62.psql`  | 62   | renders | renders |
| `w63.psql`  | 63   | renders | `internal error` |
| `w80.psql`  | 80   | renders | `internal error` |

A pass on `w62` with a failure on `w63` is this bug. Failures on all
three are something else, which is why `w62` is in the set.

The fixtures are plain psql output, five lines each, with no multibyte
characters. The content does not matter; only the width does.

## WHAT IS ESTABLISHED

- 5.8.16 contains **no** `leave("internal error")` in `src/infra.c`.
  5.9.0 contains **six**, added in the same change that introduced the
  `charlen_limit` / `dsplen_limit` bounds checks.
- The one that can fire on ASCII input is `charlen_limit`, which calls
  `leave` when its `limit` argument is below 1:

      inline int
      charlen_limit(const char *str, int limit)
      {
          if (limit < 1)
              leave("internal error");

- The regression reproduces at every terminal size, so it is not a
  question of the table being wider than the window.
- Master still reproduced it when last checked (d30073e, 2026-10-22),
  despite the multibyte fixes that landed after 5.9.0.

## WHAT IS NOT ESTABLISHED

**Which call site passes the non-positive limit.** The candidates are
in `src/print.c`, `src/table.c`, `src/pretty-csv.c` and `src/export.c`.
The `src/print.c` path in `print_row` is the most likely, because it
walks a decoration row byte by byte while rewriting ASCII borders as
ACS line characters, and it is reached for exactly this kind of input:

    while (bytes > 0)
    {
        int column_format = desc->headline_transl[offsetx];
        ...
        int len = charlen_limit(rowstr, bytes);

Its own loop condition keeps `bytes` above zero, so either `bytes`
arrives negative or the firing site is elsewhere. Settling it needs a
debug build and a breakpoint on `leave`, which has not been done.

Anyone picking this up should start there rather than from the
symptom.

## WHY 63

Not known. 63 is one less than 64, which invites a theory about a
buffer or a bit width, but nothing here confirms one. The threshold
was found by bisecting fixture widths, not by reading the code.

## MEANWHILE

Downstream, the remedy is to stay on 5.8.16. The pinned Homebrew
formula and the same fixtures are kept at `config/brew/` in the
denviro repository, whose README says when the pin may be dropped:
when `check` passes against a newer build.

## IF THIS GOES UPSTREAM

Not yet filed. A report wants the three fixtures, the version
comparison above, and the honest note that the call site is not
pinned down — a maintainer who knows the width arithmetic will find it
faster than a bisect will.
