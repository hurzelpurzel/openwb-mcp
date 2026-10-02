# openwb-mcp

An [MCP](https://modelcontextprotocol.io) server for [openWB](https://openwb.de) EV charging
hardware. It talks to openWB **2.1.9+** over the
[SimpleAPI HTTP endpoint](https://wiki.openwb.de/doku.php?id=openwb:vc:2.1.9:simpleapi)
(`/openWB/simpleAPI/simpleapi.php`).

**Read-only for now.** The SimpleAPI's write parameters (`set_chargemode`, `chargecurrent`,
`chargepoint_lock`, …) are not wired up, so no tool can start or stop a charging session.

## Setup

```bash
uv sync
OPENWB_BASE_URL=http://openwb.fritz.box uv run openwb-mcp   # stdio transport
```

`OPENWB_BASE_URL` is the only required variable. A bare host, a base URL or the full
`simpleapi.php` URL are all accepted; the path is filled in for you.

| Variable | Default | Meaning |
| --- | --- | --- |
| `OPENWB_BASE_URL` | — | Host or URL of the openWB. Required. |
| `OPENWB_TOKEN` | — | Sent as `Authorization: Bearer …`. |
| `OPENWB_USERNAME` / `OPENWB_PASSWORD` | — | Basic auth. Both or neither. |
| `OPENWB_TIMEOUT_S` | `10` | HTTP timeout in seconds. |
| `OPENWB_VERIFY_TLS` | `true` | TLS certificate verification. |
| `OPENWB_CHARGEPOINT_IDS` | lowest id | Comma separated chargepoint ids. |
| `OPENWB_COUNTER_IDS` | lowest id | Comma separated counter ids. |
| `OPENWB_BATTERY_IDS` | lowest id | Comma separated battery ids. |
| `OPENWB_PV_IDS` | lowest id | Comma separated PV inverter ids. |

An empty id list uses the openWB's *auto-id* feature, which resolves to the lowest id of
that component — and to nothing at all when the openWB has no such component.

## Tools

| Tool | Returns |
| --- | --- |
| `get_openwb_status` | Everything the configured ids ask for, in one snapshot. |
| `get_chargepoint` | State, charge mode, plug state, per-phase power/current/voltage, energy counters, SoC. |
| `get_counter` | Grid power, per-phase values, imported/exported energy, grid frequency. |
| `get_battery` | Power, state of charge, charged/discharged energy. |
| `get_pv` | Current power and daily, monthly, yearly yield. |

Every id argument is optional and defaults to the openWB's lowest id.

## Design

```
MCP tools (server.py)
      │  depends on the Protocol only
      ▼
OpenWBClient (ports.py)
      ▲
      │  implemented by
SimpleApiClient (simpleapi.py) ──httpx──▶ openWB
```

`ports.OpenWBClient` is the seam: the openWB's MQTT topics (`openWB/simpleAPI/*`, per the
same wiki page) could be added as a second implementation without touching the tools or
the models. `config.py` keeps every host out of the code and reads it from the environment.

## Development

```bash
uv run ruff check .
uv run ruff format .
uv run mypy .
uv run pytest
```

Tests never touch a real openWB. `tests/mock_openwb.py` is a stdlib HTTP server that speaks
the SimpleAPI dialect — `<kind>_<id>` JSON keys, auto-id resolution, `raw=true`, and
Bearer/basic auth — so the client is exercised over a real socket:

```python
from tests.mock_openwb import OpenWBState, chargepoint, serve

with serve(OpenWBState(chargepoints={0: chargepoint()})) as url:
    ...  # url is http://127.0.0.1:<port>
```
