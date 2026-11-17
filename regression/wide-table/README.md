# "internal error" From translate_headline — Resolved In 5.9.1

A record of a regression that cost a downstream user a day, kept so
the next person who meets the symptom can skip the investigation.

## TOPLINE

pspg 5.9.0 could abort with `internal error` before drawing anything.
The cause was `translate_headline` measuring one buffer against
another buffer's end pointer; a bounds assertion added in the same
release turned a latent out-of-bounds read into an abort. Fixed
upstream in `61847ae` and released in **5.9.1**. Nothing to do here
but upgrade.

## THE CAUSE

`src/table.c`, in `translate_headline`:

```c
-  nextchar = namesline + charlen_range(namesline, ptrend);
-  display_width = dsplen_range(namesline, ptrend);
+  nextchar = namesline + charlen_range(namesline, namesline_end);
+  display_width = dsplen_range(namesline, namesline_end);
```

`ptrend` points into the **headline**. The string being measured is
the **namesline**. The two are separate allocations, so `ptrend` is
not a valid end pointer for `namesline` and the read could run past
its end.

5.9.0 also added six `leave("internal error")` assertions to
`src/infra.c`, among them:

```c
dsplen_range(const char *str, const char *end)
{
    if (end <= str)
        leave("internal error");
```

When the headline happened to be allocated **below** the namesline,
`end <= str` held and pspg aborted. The same code in 5.8.16 read out
of bounds silently and usually got away with it. The 5.9.0 assertion
did not create the bug; it made an old one visible.

## WHY IT LOOKED LIKE A WIDTH PROBLEM

It is heap-layout dependent, not width dependent. Allocation sizes
move the two buffers relative to each other, so a bigger result set
changes whether `ptrend` lands below `namesline`. A downstream bisect
on table width found a clean threshold of 63 characters on one data
set and concluded the width was the trigger. It was not.

That conclusion was tested and does not hold. Against a real 5.9.0
build, **117 synthetic tables** — varying column count, column width,
row count, column-name length and trailing-space handling, including
widths either side of 63 — produced no failure at all. Only the
original query reproduced it.

The lesson for anyone chasing something similar: a synthetic fixture
will not reproduce a layout-dependent fault, and a fixture that passes
on the broken version is worse than none, because it reads as evidence.

## VERSIONS

| Version | Behaviour |
|---------|-----------------------------------------------|
| 5.8.16  | Reads out of bounds, renders anyway |
| 5.9.0   | Asserts and aborts with `internal error` |
| 5.9.1   | Fixed |

Upstream issue: okbob/pspg#273, opened 2026-10-30, closed 2026-11-06.
Fix commit: `61847ae`, 2026-11-02.

## WHY THIS BRANCH EXISTS

A pinned Homebrew formula held 5.8.16 downstream for five weeks while
the cause was unknown. The pin is retired and the formula is kept as a
worked example in the denviro repository, at `config/brew/`.

The fork carries no patch and proposes none. It is somewhere to put
one next time.
