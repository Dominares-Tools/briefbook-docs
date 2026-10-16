# Changelog

## 1.0.1

Fixes from a review of 1.0.0.

- The standing notes block can no longer vanish without a word. `assemble` now compares the directives found with the lines the block carries and warns whenever any are left out, including when the cap leaves none (the block is then empty, not a header over nothing). `check` runs the same comparison: a partial cut is a warning, a dropped block is an error. New `directives_status`.
- `check` fails when a saved text replaces a pinned layer, not only when the pinned shipped text changes. The README no longer says the safety layer "cannot be blanked": blank saves are refused and the pin guards the shipped text; `save` of any non-blank text still writes, and `check` is the guard.
- A misspelled time zone is a `check` error (a missing zone database stays a warning), `assemble` warns, and the TODAY reference line names the zone actually used (UTC), not the misspelled one. New `tz_status`, `effective_tz_name`.
- Ask `active_from` / `active_until` are validated on add, update and replace: a value that is not a date, or a window that ends before it starts, raises `ValueError` (the CLI exits 2) instead of leaving the ask live forever. A bad date already in a file is still read as no bound, and `check` warns about it.
- The events cap is strict at any size: a cap too small for the header plus the marker line gives no block and a warning (`events_rendered` no longer counts the marker line).
- `check` warns about a surface with no core (safety) layer and takes `--now` / `now=` to read dates as of a given moment.
- `LayerStore.save`, `clear` and `restore` take the store's lock, so two saves at once each leave a backup of what they replaced.
- Block order is recorded as deliberate: language rule first, then standing directives, facts, events, the priority ask, TODAY, the layers, and the response mode last. It is briefbook's own fixed order and differs from the order of the app it was extracted from (directives, facts, ask, language, events, TODAY). What matters is the same: directives above the facts, the ask below both, the mode last. The description now says "fixed order", not "order of authority".
- The example book's dates were moved so `briefbook check examples/shop` stays clean for years (the sale window ends 2036-10-05) and its timestamps are fixture values.
- Tests ran on Python 3.10, 3.11, 3.12, 3.13 and 3.14.

## 1.0.0

First release.

- Layered prompt assembly in a fixed order: language directive, standing directives, current facts, upcoming events, priority ask, TODAY, safety core + shared + role layers, response mode.
- Per-surface control: role, core and shared layers, which blocks, and a voice flag that swaps the `auto` language rule for one that stays in English on a phone call.
- Layer store with shipped text and live edits: atomic save, snapshot before every save and reset, restore (itself undoable), 40 backups per layer, layer names validated as file stems, empty saved text falls back to shipped, `PromptReloader` that re-reads only on a file change and keeps the last good prompt on error.
- Fact registry: typed facts, active-from / active-until windows, labelled note lines that set facts (notes win; unchanged values write nothing; bold glyphs, non-breaking spaces and curly apostrophes folded; bracketed annotation stripped), `one_of` families, audience groups, `verify`.
- Standing directives from notes, with excluded notes, a strict character cap and a counted overflow line.
- Events block: past dropped, online series collapsed with a cadence, served areas carried, strict cap, health numbers.
- Priority asks: ranked, windowed, region and language targeting, locked atomic store with backups; bare dates read on the organization's clock.
- Response modes, language directives (`auto`, `en`, `es`, `vi`, `tl`, plus your own), TODAY block.
- `briefbook` command line: `show`, `surfaces`, `layers`, `save`, `reset`, `history`, `restore`, `facts`, `sync-notes`, `asks`, `check`, `pin`, `init`.
