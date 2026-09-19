# Atelier

A Silver-native terminal UI library, designed after ratatui: retained cell
buffers, constraint layouts, composable styles, and diffed rendering. No
closures anywhere — the application loop renders widgets directly, then
flushes.

## Layout

```
silver.toml          package manifest ([lib.atelier], [bin.demo])
atelier.ag           root re-export
atelier/
  style.ag           Color, Modifier flags, composable Style
  text.ag            Span / Line / Text, UTF-8 decode, wcwidth table
  layout.ag          Rect, Constraint, Flex, deterministic Layout solver
  buffer.ag          retained Cell grid, ranges, frame diffing
  widget.ag          Widget / StatefulWidget protocols
  update.ag          elapsed-time update intervals
  terminal.ag        double-buffered Terminal + SGR backend over std.term
  widgets/
    block.ag         borders, titles, padding
    paragraph.ag     word wrap, scroll, alignment
    list.ag          selectable items, highlight, scroll window
    tabs.ag          horizontal tab navigation
    gauge.ag         gauges, line gauges, sparklines, bar charts, spinners
    chart.ag         multi-series step-line charts with axes
    chrome.ag        status key-hint bars, labeled dividers
    overlay.ag       region clear + centered popup rects
    table.ag         columns, headers, selection, scrolling
    scrollbar.ag     standalone position gutter
    tree.ag          expandable flat preorder trees
    input.ag         single-line editor state and cursor
    diff.ag          semantic review/diff lines
    kitt.ag          scanning Text overlay with cadence
tests/               `agc test` suites (one file per module)
examples/
  demo.ag            interactive showcase for every widget (list navigation,
                       realistic app previews, IBM Carbon / Tokyo Night /
                       Dracula themes; 100ms poll drives animations, `p`
                       play/pauses, `u` steps one frame, `t` cycles theme,
                       `b` flashes changed regions)
```

## Use

Atelier is a Silver package. Inside your own package, depend on it and
import modules by path:

```silver
import atelier.widgets.list;
import atelier.terminal;
```

```silver
Terminal t = Terminal.full_screen(&STDOUT);
t.hide_cursor();
Frame f = t.frame();
my_list.render(&area, f.buf, &state);
t.flush();
```

## Develop

```bash
agc test --no-cache     # run all suites (from the package root)
agc run --no-cache examples/demo.ag   # try it (real terminal required)
```

`--no-cache` is currently required: cached builds partition modules into
separate codegen units whose `.agm` signatures drop struct field metadata
and trait-impl membership, which a multi-file widget library cannot use.
Single-unit builds are also faster for a codebase this size.

Development needs a Silver toolchain whose `std.term` exists (Silver
0.2.6+ with the terminal module), either installed or via
`SILVER_SYSROOT`. The `std` symlink in this repo (gitignored) points at a
local checkout for that purpose.

## Design notes

- **Cells** hold one codepoint plus resolved style; `0` is empty. Combining
  marks are skipped, wide chars advance two cells.
- **Text borrows, `Text` owns**: spans are non-owning views; `Text` keeps
  backing `String`s whose buffers never move under `Vec` growth.
- **Full redraw per frame**: `frame()` clears the back buffer before widgets
  render, so shrinking text or changing panes cannot leak stale cells.
  `refresh()`/`clear()` reset both buffers and the screen together.
- **Dirty-region flashes**: the demo's optional update flash targets only the
  list, preview, header, or footer rectangles changed by the event.
- **Interactive previews**: a 100ms poll advances gauges, charts, tabs,
  spinners, table state, scroll position, tree expansion, input cursor, and
  diff content so stateful widgets are shown changing rather than as static
  samples. `p` play/pauses, `u` steps a single frame, `t` cycles the theme.
- **State stays external**: selection, scrolling, and editor cursor state use
  `StatefulWidget`, so persistence and navigation policies remain application
  owned rather than hidden inside a renderer.
- **Reference-driven components**: the widget set covers the repeated
  surfaces found in btop, Pi, Grok Build, OpenCode, and Hermes—chrome,
  selectors, metrics, tables, trees, editors, scrollbars, and diffs—without
  importing their app-specific event or agent layers.
- **Borrowed rendering**: widget and app entry points take `&`/`&mut`
  refs. Raw pointers remain only where they mean something: nullable
  lookups (`cell_mut`), `u8*` byte buffers and FFI, `Vec` storage and
  element access, and signatures dictated by `std` traits.
- **Fill the box**: backgrounds paint the whole inner area; meters fill
  every row, single-row chrome (spinner, tabs, rules) centers vertically,
  and bars resolve 1/8-cell fractional tips (`▏`–`▉`).
- **Temporaries cannot be borrowed**: compare enums against bound locals,
  never literals (`Color red = Color.Red; ... != red`).
- **`match` is an expression**: all arms must yield one type; solver logic
  dispatches on tag functions with if/else chains instead.
