# briefbook — Your bot's prompt, in layers you can edit live

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/briefbook**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

briefbook assembles a bot's system prompt from layers in one fixed order, lets you edit it live with a backup under every change, and resets any layer to what you shipped. It calls no model.

## Get it

Buy briefbook at https://dominares.org/tools/briefbook: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python package and the briefbook command, an example folder for a made-up shop, and the full test suite.

Requirements: Python 3.10 or newer, standard library only. tzdata is optional, for named time zones on systems without a time zone database.

This page documents version 1.0.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

A support bot's prompt is not one string. It is a safety floor nobody should edit by accident, a persona, a role, today's date, the links and phone numbers it must get exactly right, the instructions staff pinned in their notes, the events coming up, one thing you want offered this week, and a language rule. Kept in code they cannot be changed without a deploy; kept in a text box they drift, contradict each other, and one bad save blanks the safety layer. briefbook keeps each in its own place and puts them together in the same fixed order every time (top of the prompt first):

```
language directive          style, for the whole turn
standing directives         supreme: lines in your notes that start ALWAYS: NEVER: PIN: DIRECTIVE:
current facts               links, emails, phones; each can have an active-from / active-until date
upcoming events             dated, past ones dropped, a weekly online series collapsed to one line
priority ask                the one need to offer when it fits; never overrides the three above
TODAY                       the weekday, date, time, tomorrow and the day after, from your clock
safety core + shared + role the saved-or-shipped prompt layers
response mode               a style the user picked, appended last and told it cannot override anything above
```

The order is fixed. A surface can leave blocks out, but cannot reorder them. What holds: standing directives above the facts, the priority ask below both, and the response mode last.

- **Live edits you can undo.** `save` writes a layer atomically after snapshotting the old text; `reset` goes back to the shipped text after a snapshot; `restore` brings any backup back (and snapshots what it replaces). The newest 40 backups per layer are kept. A running process picks the edit up on its next turn without a restart (`PromptReloader`), and keeps serving the last good prompt if a reload fails.
- **The safety layer is guarded, not locked.** Saving blank text is refused, and a saved file that is empty falls back to the shipped text. `briefbook pin safety_core` records a fingerprint of the shipped text; after that `briefbook check` exits 1 if the shipped text changes without someone re-pinning it on purpose, and exits 1 if a saved text replaces the pinned layer. `save` itself still writes any non-blank text: the guard is `check`, so run it in CI or before you deploy, and `reset` undoes an override.
- **One place for each fact.** The usual problem: the support email lived in five places and they disagreed. Here a fact lives once, and a labelled line in a note (`MAIN EMAIL: help@example.org`) sets it. **Notes win**: a note's value overrides whatever was saved, even a newer edit, and re-asserts on each sync; an unchanged value writes nothing.
- **Facts with dates.** A fact, and a priority ask, can carry an inclusive window. Outside it the fact simply is not in the prompt; the value stays in the registry. Dates are read on your organization's clock, not the server's.
- **Per-surface control.** Each surface (web chat, phone, an email drafter) names its role layer, an optional shared layer, which blocks it receives, and whether it is a voice channel. A phone call gets its own language rule: phone audio is transcribed by a model, one misheard word once came back as Hindi, the chat rule ("answer in the language of the latest message") applied, and the agent answered a caller in Hindi. On a voice surface `auto` stays in English unless the caller clearly speaks, or asks for, another language.
- **Nothing is cut silently.** Directives and events have a character cap. Over it, the block ends with a counted `(… N more …)` line, and `check` and `assemble` warn. If the cap is too small for even one directive (or the events header), the whole block is left out and both say so: for the standing notes `check` exits 1, because that is the block meant to win.

### The folder

A book is a folder of plain files, so it can live in version control:

```
book.json          surfaces, fact catalog, time zone, pins
ship/*.txt         the shipped layer texts (the baseline you commit)
live/prompts/      saved edits                live/backups/   a snapshot under every change
live/facts.json    fact values and windows    live/asks.json  priority asks
notes/*.md|txt     notes: directives and labelled fact lines
events.json        the events feed: a list, or {"events": [...]}
```

`briefbook init my-book` creates a small one. `book.json` (see `examples/shop/book.json`):

| key | meaning |
| --- | --- |
| `timezone` | IANA name, default `UTC`. Today, "tomorrow" and every window use it. A name the database does not know is a `check` error, and the clock runs on UTC until it is fixed. |
| `preferred_language` | `auto` (default), `en`, `es`, `vi`, `tl`, or a code you add under `languages`. |
| `languages` | `{"fr": "LANGUAGE: Réponds toujours en français."}` adds or overrides a directive. |
| `facts` | the catalog: `key`, `label`, `kind` (`url`, `email`, `phone`, `statement`), `aliases` (the note labels that set it), `in_prompt`, `default`, `one_of`, `groups`. |
| `surfaces` | `{name: {role, core, shared, blocks, voice, language}}`. `core` defaults to `safety_core`; `blocks` is any of `language directives facts events ask today`. |
| `exclude_markers` | notes whose title or first line contains one are skipped whole. Default `DO NOT TOUCH`, `INTERNAL`. |
| `caps` | `{"directives": 4000, "events": 14000}` characters. |
| `fact_group` | show only facts with no `groups` or with this group. |
| `pin` | `{layer: sha256}`, written by `briefbook pin`. |

`one_of` is for a family such as phone numbers: of the facts that share it, only the first one that has an active value is given, so the bot gives one number, not three.

### Commands

```bash
briefbook show examples/shop --surface phone --now 2026-10-03T13:56        # the assembled prompt
briefbook show examples/shop --surface chat --now 2026-10-03T13:56 --report --mode email_reply
briefbook init my-book                               # a small starter book
briefbook surfaces examples/shop                     # the surfaces and their layers
briefbook layers examples/shop                       # which layers are shipped, saved, or drifted
briefbook save my-book chat --file chat.txt          # or --text "..." or --file -   (old text backed up)
briefbook history my-book chat                       # backups, newest first
briefbook restore my-book chat BACKUP_ID
briefbook reset my-book chat                         # back to the shipped text (backed up first)
briefbook facts my-book --set main_email=help@example.org
briefbook facts my-book --window sale_url --from 2026-10-01 --until 2026-10-05
briefbook sync-notes my-book                         # labelled note lines set facts; notes win
briefbook asks my-book add --title "A second helper" --until 2026-10-31
briefbook check my-book --strict                     # exit 1 on errors (and, with --strict, warnings)
briefbook check my-book --now 2026-10-03             # read dates as of then (default: now)
briefbook pin my-book safety_core
```

`--report` prints one line per part with its size and where each layer came from:

```
language      364 chars
directives    414 chars
facts         510 chars
events        659 chars
ask           491 chars
today         628 chars
layers        650 chars
mode          798 chars
layer core: safety_core (shipped)
layer shared: persona (shipped)
layer role: chat (shipped)
```

Exit codes: `0` ok, `1` `check` found errors, `2` a usage problem (a message on stderr says what).

### From Python

```python
from datetime import datetime
from briefbook import Book

book = Book("examples/shop")
a = book.assemble("phone", now=datetime(2026, 10, 3, 13, 56))
print(a.text.splitlines()[0][:37])
print([p.name for p in a.parts])
```

```
LANGUAGE (phone call): Speak English.
['language', 'directives', 'facts', 'today', 'layers']
```

`assemble(surface, now=, mode=, mode_text=, region=, lang=)` returns an `Assembly` with `.text`, `.parts`, `.layers` (each layer's source and whether it differs from ship), `.warnings` and `.report()`. Nothing in it calls a model.

Live edits, served to a running bot:

```python
import tempfile
from briefbook import LayerStore, PromptReloader

store = LayerStore(tempfile.mkdtemp() + "/live", {"safety_core": "Never invent facts.", "chat": "Be brief."})
bot = PromptReloader(store, "chat")
print(bot.prompt.replace("\n\n", " | "))
store.save("chat", "Be brief and kind.")
print(bot.prompt.replace("\n\n", " | "))
store.clear("chat")
print(bot.prompt.replace("\n\n", " | "), len(store.list_backups("chat")))
```

```
Never invent facts. | Be brief.
Never invent facts. | Be brief and kind.
Never invent facts. | Be brief. 1
```

Facts with a window, set from a note:

```python
import tempfile
from datetime import date
from briefbook import FactSpec, FactStore

facts = FactStore(tempfile.mkdtemp() + "/facts.json", [FactSpec("sale_url", "Sale page", "url", aliases=("SALE LINK",))])
facts.apply_notes([{"body": "SALE LINK: https://shop.example/sale"}])
facts.set_window("sale_url", "2026-10-01", "2026-10-05")
print(facts.build_block(date(2026, 10, 3)).splitlines()[1])
print(facts.build_block(date(2026, 10, 6)) == "")
```

```
- Sale page: https://shop.example/sale
True
```

Every block builder is also usable alone:

```python
from briefbook import build_directives_block, build_response_mode_block, language_directive

notes = [{"title": "Rules", "body": "NEVER: promise a repair date"}]
print(build_directives_block(notes).splitlines()[1])
print(language_directive("auto", voice=True)[:37])
print(build_response_mode_block("quick_answer").strip()[:23])
```

```
- [Rules] NEVER: promise a repair date
LANGUAGE (phone call): Speak English.
SESSION RESPONSE MODE —
```

The rest of the surface: `build_events_block(events, today, cap)`, `events_health`, `directives_status(notes, cap)` (what the standing notes block carries and what it left out), `tz_status(name)`, `Book.check(now=)` returning `(errors, warnings)`, `AskStore` (`add`, `update`, `replace`, `delete`) with `select_active_ask` and `build_ask_block`, `FactStore.verify` / `load_active` / `build_reference_block`, `LayerStore.compose` / `list_backups` / `restore` / `drift`, `RESPONSE_MODES`, `build_today_block`.

### Response modes

Four ship: `normal` (nothing injected), `email_reply` (a paste-ready first-person reply, three sentences at most, ending in one bounded question), `long_form`, `quick_answer` (internal Q&A in one or two lines), plus `custom`. The user can edit the text; the server prefers the text it is sent and uses the preset's own only when the text is empty. Control characters are stripped and the text is capped at 1500 characters. The block is appended after everything else and says it controls tone, length and format only.

### Limits

- It builds the prompt. It does not call a model, count tokens, or judge whether a prompt is any good.
- No business-hours logic. The TODAY block states the date, the weekday, the time and the next two days from your clock; if you want "open now" in it, work that out yourself and pass the lines in (`build_today_block(..., extra_lines=[...])`; `assemble` does not take extra lines).
- Files are read on every `assemble`, with no cache. That is fine for a book of a few dozen small files; it is not built for thousands of events per turn.
- Notes are plain `.txt` or `.md` files, one note per file. It does not read a notes app's database; export to files first. A note is excluded as a whole, by its title or first line.
- Directive markers are `DIRECTIVE:`, `ALWAYS:`, `NEVER:`, `PIN:` at the start of a line. Prose that merely says "always" is not pinned.
- Built-in language directives: `auto`, `en`, `es`, `vi`, `tl`. Others are yours to add under `languages`.
- Locking is an advisory file lock (`flock`; a byte-range lock on Windows) so two editors saving at once do not lose each other. It does not protect a shared network drive that lacks locks, where it degrades to no lock rather than failing the save. The Windows path is written but the tests ran on POSIX only.
- Time zones by name need the IANA database; if it is missing the clock falls back to UTC and `check` warns (`pip install ".[tz]"`). A misspelled name also falls back to UTC, and `check` fails on it.
- Ask dates (`--from`, `--until`) must parse as a date or ISO datetime; a typo is refused when the ask is added. An ask that already has a bad date in its file is read as having no bound there, and `check` warns.
- A window with a bare date (`2026-10-05`) runs through the end of that day on the organization's clock.
- There is no web panel. The library has the calls one needs (`save`, `restore`, `FactStore.load_full`, `AskStore.replace`); drawing the form is yours.
- Python 3.10 or newer. The runtime is the standard library only.

---

Docs mirror of the private repository Dominares-Tools/briefbook; updated with each release.
