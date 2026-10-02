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
- Runtime deps are only `fastmcp` and `httpx` (added explicitly rather than relied on as a
  fastmcp transitive). Dev deps also include `pytest-asyncio`.
- `mcp.get_client()`/`set_client()` in `server.py` are the injection seam: the tools build
  their client from the environment on first use, so importing `server.py` never requires
  `OPENWB_BASE_URL`.

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

- The suite is real now (70 tests). Async tests use **pytest-anyio** via `pytestmark =
  pytest.mark.anyio` plus an `anyio_backend` fixture, with `asyncio_mode = "strict"`.
- `tests/` is a package (`tests/__init__.py`) and imports use absolute paths
  (`from tests.mock_openwb import …`). Without the `__init__.py`, `uv run mypy .` fails with
  "Source file found twice under different module names".
- `tests/mock_openwb.py` is the stand-in openWB: a stdlib `ThreadingHTTPServer` on loopback
  speaking the SimpleAPI dialect. Use it instead of a real openWB, which needs explicit
  user permission.
- mypy runs `strict = true` and targets `python_version = "3.11"`, while the venv itself is
  Python **3.14.7**. Don't "fix" this by pinning mypy to the interpreter version — 3.11 is
  the deliberate floor declared in `requires-python`.
- Venv tool versions (verified): ruff 0.16.10, mypy 2.4.0, pytest 9.1.1, fastmcp 4.0.10.
  The **global** `fastmcp` (3.3.1) and `pytest` (9.0.2) are older, separate installs in
  system Python — never rely on them for the project.
- No CI and no pre-commit hooks exist yet, so nothing catches failures automatically.

## Decided: how to reach openWB

**HTTP SimpleAPI, read-only.** The transport is decided, so don't re-litigate it:

- Endpoint: `/openWB/simpleAPI/simpleapi.php` on the openWB itself (openWB 2.1.9+), per
  https://wiki.openwb.de/doku.php?id=openwb:vc:2.1.9:simpleapi
- That wiki page documents **two** transports: this HTTP endpoint *and* an MQTT topic tree
  (`openWB/simpleAPI/*`, writes to `openWB/simpleAPI/set/*`). Only HTTP is implemented.
- **Write parameters are deliberately not wired up yet** (`set_chargemode`, `chargecurrent`,
  `chargepoint_lock`, `bat_mode`, `instant_charging_*`, `vehicle`, `manual_soc`). Adding
  them means adding MCP tools that can change charging behaviour — ask first.
- `ports.OpenWBClient` is the seam; an MQTT adapter can be added beside the HTTP one
  without touching `server.py` or `models.py`. Keep it that way.
- The SimpleAPI names the component in the query parameter (`?get_chargepoint_all=3`), so
  one request yields one component. `SimpleApiClient.get_status` fans out concurrently
  instead of trying to batch, which the API cannot express.

If you add an adapter, keep every host in config/env (`config.py`) — nothing hardcoded.

## Rules

* Don't directly contact real openWB without explicit permission of the user.
* Generate a Mockserver for testing — `tests/mock_openwb.py` is the one to extend.
* Read-only by default. Anything that can change charging state needs the user's go-ahead.
* ruff selects `ANN` but exempts `tests/**`; mypy `strict` covers tests too, so annotate
  test functions' parameters and return types.