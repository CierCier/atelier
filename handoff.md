# Atelier handoff

Silver-native TUI component library (`/home/cier/Projects/atelier`) + Silver
compiler hardening (`/home/cier/Projects/silver`). Goal: take over TUI
libraries — 1:1 ratatui experience, then beyond.

## Current state

- **31 test suites pass**, including under `--leak-check`. Demo builds clean,
  all 32 showcase pages (0–31) verified live in a PTY, including page
  navigation and theme cycling.
- Atelier HEAD: `0966aa9` (Calendar). Staged, uncommitted: Heatmap,
  ColumnChart, Sparkline RTL (+ test split for one-file-per-module:
  `column_test.ag` new, RTL cases moved into `gauge_test.ag`). Untracked:
  `handoff.md`, `markdown.ag` +
  `markdown_test.ag` (done, verified — wire into the commit set).
- **Smoothness phase committed `b1312ef`:**
  - `app.ag`: idle loop now blocks in `poll(2)` (POLLIN on stdin, timeout =
    min(next tick, ESC window, 250ms cap)) instead of a 5ms sleep spin —
    zero wakeups and zero output when idle; single draw per pass (the old
    double-draw after every key is gone); ticks fire off a monotonic
    deadline with drift correction, and `on_tick(dt)` receives true
    elapsed ms; input drain caps at 256 bytes per pass so pastes cannot
    starve ticks.
  - `terminal.ag`: `flush()` emits row runs — one move per run, cells
    written contiguously. Contiguous text lands in one write chunk → no
    mid-row tearing.
  - `demo.ag`: meter/bar animations (Gauge, LineGauge, BarChart,
    ColumnChart) interpolate from ms (`pulse_ms` + `demo_ms`) instead of
    stepping whole interval units.
- **Perf phase committed `e08bf65`** (demo benchmark, PTY, 24x100):
  - Gap fill: short same-style gaps between changed cells are rewritten
    instead of repositioning — merges row fragments into one chunk.
  - Differential SGR: `term_write_style_diff` emits one combined sequence
    with only changed attributes (mods off-codes 22/23/24/25/27/28/29);
    bold/dim survive recolors, no reset flicker.
  - Numbers vs pre-perf: bytes −41%, SGR sequences −77%, moves −14% over
    a 32-page walk; animation 471→315 B/s; child CPU ~0.1% either way.
  - Resize stress PASS both modes (grow/shrink/40ms churn/1x40/2x10,
    probes alive after each, clean exits, inline: 0 absolute moves).
  - Idle measured at exactly 0 bytes.
- `a.out` (stale) deleted.

## How to validate

```bash
# Atelier suite (from /home/cier/Projects/atelier)
SILVER_CACHE_DIR=/tmp/opencode/check \
  /home/cier/Projects/silver/target/debug/agc test --no-cache
# Same with leak-check: append --leak-check (expect 31/31)

# Demo (always --no-cache; cached builds are UNSOUND, see gotchas)
SILVER_CACHE_DIR=/tmp/opencode/demo \
  /home/cier/Projects/silver/target/debug/agc run --no-cache examples/demo.ag
# Inline scrollback mode:
#   ... run --no-cache examples/demo.ag -- --inline

# Silver compiler: cargo test -p agc --lib (538 tests)
# Silver integration: python3 tests/run_tests.py <filter> --no-tui
```

## Architecture (do not break)

- Direct/immediate rendering. No widget tree, no canvas framework, no modal
  framework. `Widget` renders state; `StatefulWidget` reads/writes external
  state owned by the app. No closures anywhere.
- `&`/`&mut` refs at all boundaries. Raw pointers only for: nullable
  lookups (`cell_mut`), `u8*` byte buffers/FFI, `Vec` storage + `get_ptr`,
  trait-dictated signatures.
- Every widget with time-varying display carries `UpdateInterval` +
  `update_interval` / `update_interval_ms` / `advance`. (Only exception:
  `Toasts`, which is TTL-driven via `advance(dt)` directly.)
- Owned collections live INSIDE widgets (`Vec<String>` in items, `Log.lines`);
  selection/scroll/offset state stays EXTERNAL.
- Comments are Why-only: never restate how, always why-not + edge cases.

## Gotchas (each paid for in debugging hours)

1. **`--no-cache` is mandatory.** Cached multi-unit codegen drops struct
   field metadata → heap corruption + delayed SIGSEGV (reproduced: cached
   demo dies on Kitt in ~10s, `--no-cache` flat for 7+ min). Never trust a
   `a.out`-sized (~9–10 MB) binary; good builds are ~1.3–1.8 MB.
2. **Generic-to-generic calls now work (committed in Silver `bfb3432`).**
   `run_app` → `run_app_view` wrapper depends on it. If List text ever shows
   garbage pointers again, suspect reborrow first and check with gdb.
3. **`agc check` passes what codegen rejects.** Inline `*vec.get_ptr(i)`
   derefs and generic-to-generic calls (pre-fix) both pass check but fail
   build. Always verify with a real build, never check alone.
4. **Match arms must unify types** (`_ : { half = false; }`, not `{}`).
5. **`.drop()` only resolves on types with a `Drop` impl.** Trim paths bind
   to a named local and rely on scope-exit field drops (leak-verified).
6. **Match on enum values, not literals** for temporaries; test names are
   `snake_case` `#[test]` fns (harness wraps each in `test_start`).
7. **Silver language limits (found writing Calendar/Heatmap):** no struct
   literals in expression position (use an `impl` `new()`); `&mut` methods
   cannot be called through an immutable `&Buffer` binding in tests — take
   `Buffer*` and call through the pointer; a method returning the struct
   type cannot use a bare `return;` (make it `void` — `set()`); by-value-
   receiver methods (`Style.patch`) cannot be called through a `Style*`
   local — bind a value copy via `.get()` first.
8. **Test assertions on untouched buffer cells expect ch=0, not space.**
   Only cells a widget actually writes hold 0x20 (a zero-value glyph);
   blanks-by-omission are 0 (see Sparkline RTL tests vs Table spacing).
9. **PTY key injection:** keys sent before the app's first frame are
   discarded by the raw-mode switch (TCSAFLUSH). Wait ~3s after build
   output appears, then send keys with delays; drain the PTY continuously
   or the TUI blocks on a full buffer (the harness times out, not the app).
10. **`term_pty_test` / `term_loop_test` need `/bin/echo`.** Both hardcode
    that path, so they fail on NixOS (no FHS `/bin`; child exits 127 with
    no output). Verified 19/19 + 20/20 green inside a kitty child with a
    resolvable echo — mechanism sound, failure purely environmental.
10. **`str` has no `.equals`.** Compare span/text borrows with a byte-loop
    helper (`streq` in `markdown_test.ag`); only `String` has `.equals`.
11. **`Line.from` pre-populates a default span.** Builders that fill spans
    manually must start from an empty `Line` (see `Markdown.blank_line`),
    or every line renders a ghost unstyled segment.
12. **Reassignment after `.drop()` trips the move checker.** Build the
    replacement in a helper fn returning `move` instead of drop-and-reuse
    (see `markdown_bullet`).

## Phase 3 — COMPLETE (uncommitted)

All four Phase 3 items are implemented, tested, and showcased:

- ✅ **Calendar** (committed `0966aa9`): CalendarState owns month/year/
  selected; `move_month` wraps years and clamps selections; day-stepping
  rolls across boundaries. Sunday/Monday-first, adjacent-month filler,
  day marks. Local proleptic Gregorian date math (no std date type).
- ✅ **Heatmap** (`atelier/widgets/heatmap.ag`): fixed cols×rows grid,
  2-wide cells + 1-col gap (stride 3; rows flush). Values clamp to
  [lo, hi] and map through a ramp (`Vec<Style>`; default 5 color steps
  over full blocks). `set()` grows storage lazily and ignores out-of-grid
  writes. Legend row (lo/hi) is reserved unless `show_legend(false)`.
- ✅ **ColumnChart** (`atelier/widgets/column.ag`): vertical bars, even
  slots (1–2 cell bars centered, last slot absorbs remainder). Height > 2
  reserves a value-label row above full bars; height 1 shows bare tips;
  zero bars render as blank ticks (visible baseline, not gaps). Label row
  on the bottom; `max_value` fallback computes at render (field stays 0).
- ✅ **Sparkline RTL** (`gauge.ag`): `rtl(true)` mirrors the whole flow —
  newest value at the LEFT edge, history growing rightward (LTR: newest
  at the right edge, partial windows hugging it). Extracted `spark_glyph`.

## Roadmap (in order)

1. **Commit the set** (suite is 31/31 leak-clean: staged Phase-3 set +
   untracked Markdown + smoothness phase in app.ag/terminal.ag/demo.ag).
2. Markdown widget done (chat subset: headings, fences, lists, quotes,
   rules, bold/italic/code/strike/links/escapes → owned `Text`). Deliberate
   omissions: tables, setext headings, indented code blocks, `_` toggling.
3. Upstream or track: generic-to-generic codegen gap is FIXED and committed
   in Silver; `agc check`-vs-build divergence (gotcha 3) is still open.
4. Long-lived-process growth (`SOURCE_REGISTRY`, profiler) only matters for
   LSP embedding, not CLI.
