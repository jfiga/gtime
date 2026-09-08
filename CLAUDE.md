# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

gtime (PyPI package `gtime`) is a small, dependency-light Python CLI for looking up, comparing, and managing city times across time zones, with fuzzy city/country search, a favorites list, meeting-time conversion, and a live watch mode. Console entry point: `gtime = "gtime.cli:main"` (also runnable via `python -m gtime`).

## Development commands

Install in editable mode with dev deps:
```bash
pip install -e .
pip install -r requirements.txt   # or: pip install pytest
```

Run the full test suite:
```bash
pytest tests/
```

Run a single test:
```bash
pytest tests/test_gtime.py::test_add_and_list_favorite
pytest tests/test_comprehensive_city_search.py -k test_help
```

Run the CLI locally during development:
```bash
python -m gtime <city>
# or, once installed:
gtime <city>
```

`tests/test_comprehensive_city_search.py` has several `test_*`-named functions (`test_city_searches`, `test_country_searches`, `test_fuzzy_searches`, `test_case_sensitivity`, `test_suggestions`) that take fixture-shaped arguments (`cities`, `countries`, `capitals`) never defined as pytest fixtures — these error out under `pytest` (pre-existing, not a regression signal). The rest of that file's tests, and all of `test_gtime.py`, are genuine and pass.

## Architecture

The package has no internal subpackages — three flat modules with a strict dependency direction: `data.py` → `core.py` → `cli.py`.

- **`gtime/data.py`** — pure static data, no logic: `CITY_DB` (list of `(city, country, tz_name, emoji)` tuples) and `COUNTRY_CAPITALS` (country → capital city name, used to resolve a country-name query to its capital's entry in `CITY_DB`).
- **`gtime/core.py`** — all non-rendering logic:
  - Favorites persistence: `load_favorites`/`save_favorites`, reading/writing `~/.gtime_favorites.json` as a flat JSON array of canonical city names.
  - City resolution: `get_city_by_name` (exact match, falling back to `fuzzy_search_city`) and `suggest_cities` (typo suggestions). `fuzzy_search_city` is a *tiered cascade*, tried in this order: exact city match → city starts-with → exact country match (via `COUNTRY_CAPITALS`) → country starts-with/contains → city substring (only for queries ≥5 chars, to avoid junk matches like "usa"→"Busan") → `thefuzz` fuzzy match (score > 60). Both lookup functions are `@lru_cache`d since `CITY_DB` is static per process.
  - Cosmetic helpers: `get_time_emoji`/`get_greeting`/`get_funny_footer` (time-of-day flavor text), plus a terminal-emoji-compatibility shim (`_is_terminal_compatible`/`_get_safe_emoji`) that swaps emoji with variation selectors (e.g. `☀️`) for plain fallbacks on terminals known to mis-render them (kitty, ghostty, Terminal.app).
- **`gtime/cli.py`** — everything user-facing: hand-rolled `sys.argv` parsing in `main()` (no argparse/click) and all Rich-based rendering (`print_city_time`, `print_favorites`, `print_compare`, `watch_mode`, `print_help`). `parse_meeting_time` parses `meeting at/on <time> [TZ]` input (12h/24h formats, hardcoded UTC/GMT/EST/EDT/CST/CDT/MST/MDT/PST/PDT/CET/CEST/JST/IST abbreviation table) and returns a naive local-time `datetime` plus a human-readable timezone label.

Commands are dispatched via string comparisons on `args[0]` in `main()` — `add`, `remove`, `list`, `compare`, `meeting`, `watch`, `-h`/`--help`, and a bare `<city name>` fallback. `watch_mode` wraps any of the render functions in an `os.system('clear')` + 60s countdown loop.

## Data notes

`CITY_DB` and `COUNTRY_CAPITALS` are maintained by hand as literal Python data — adding a city means appending a tuple to `CITY_DB`; adding/fixing a country's capital means editing `COUNTRY_CAPITALS`. Keep both in sync: a capital name in `COUNTRY_CAPITALS` must exactly match a `(city, country)` pair in `CITY_DB`, or `fuzzy_search_city`'s country lookup silently falls back to the first city listed for that country.
