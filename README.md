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
  terminal.ag        double-buffered Terminal + SGR backend over std.term
  widgets/
    block.ag         borders, titles
    paragraph.ag     word wrap, scroll, alignment
    list.ag          selectable items, highlight, scroll window
tests/               `agc test` suites (one file per module)
examples/
  demo.ag            interactive showcase (list navigation + detail pane)
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
- **Full redraw per frame**: after `flush`, the back buffer keeps stale
  content, so the application re-renders everything; untouched cells diff
  clean. `clear()` resets both buffers and the screen together.
- **Temporaries cannot be borrowed**: compare enums against bound locals,
  never literals (`Color red = Color.Red; ... != red`).
- **`match` is an expression**: all arms must yield one type; solver logic
  dispatches on tag functions with if/else chains instead.
