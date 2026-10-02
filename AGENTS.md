# AGENTS.md

MCP (Model Context Protocol) server for [openWB](https://openwb.de) EV charging hardware.

## Layout

- `src/openwb_mcp/` — the package. `src/` layout: the package is **installed** into the venv
  by `uv sync`, and pytest resolves it via `pythonpath = ["src"]`.
- `pyproject.toml` — single source of truth for deps **and** all ruff/mypy/pytest config.
  There are no separate `ruff.toml` / `mypy.ini` / `pytest.ini` / `setup.cfg` files; don't add
  them.
- `uv.lock` — **committed** (not gitignored). Deps must be added with `uv add <pkg>` /
  `uv add --dev <pkg>` so the lockfile stays in sync. Do not hand-edit `dependencies`.

## Canonical commands

```
uv sync                          # install deps from uv.lock
uv run ruff check .              # lint
uv run ruff format .             # format
uv run mypy .                    # typecheck
uv run pytest                    # unit tests
uv run pytest tests/test_x.py -k test_name   # single test
```

Order when it matters: `ruff check` -> `ruff format` -> `mypy` -> `pytest`. Commit only when
all four are clean.

**Always go through `uv run`.** `ruff` and `mypy` are not on `PATH` globally, so bare
`ruff`/`mypy` fail with "command not found" — that is not a broken venv.

`ruff format` rewrites files; run it before `mypy`/`pytest` or CI-style checks will disagree
with your diff.

## Gotchas

- `uv run pytest` currently exits **5** ("no tests ran") — the suite is empty, not broken.
  `tests/.gitkeep` keeps the tracked dir alive; `testpaths = ["tests"]` errors on a fresh
  clone if that dir is missing.
- mypy runs `strict = true` and targets `python_version = "3.11"`, while the venv itself is
  Python **3.14.7**. Don't "fix" this by pinning mypy to the interpreter version — 3.11 is
  the deliberate floor declared in `requires-python`.
- Venv tool versions (verified): ruff 0.16.10, mypy 2.4.0, pytest 9.1.1, fastmcp 4.0.10.
  The **global** `fastmcp` (3.3.1) and `pytest` (9.0.2) are older, separate installs in
  system Python — never rely on them for the project.
- No CI and no pre-commit hooks exist yet, so nothing catches failures automatically.

## Undecided: how to reach openWB

**This is the key open design question — do not guess it.** openWB can be driven several
ways and the choice is not yet made:

- local HTTP API on the wallbox itself (differs between openWB 1.9 and 2.x)
- MQTT topics published by openWB
- the openWB cloud API

Ask before implementing any openWB client. Keep the transport behind an interface/port so
it can be swapped, and put openWB endpoints in config/env rather than hardcoding a host.
