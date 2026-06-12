# mcppin

**Trust-On-First-Use pinning and drift detection for MCP tool definitions.**

An MCP client trusts whatever tool list a server advertises: each tool's `name`,
its `description` (which the model reads as instructions), and its `inputSchema`.
A malicious or compromised server can silently change a tool's description — a
prompt-injection vector — or widen its schema *after* you have approved it. This
is the **"tool poisoning" / "rug pull"** attack, and a client that re-reads the
tool list on every connection has no way to notice.

`mcppin` fingerprints a server's tool manifest the first time you approve it
(trust on first use), then verifies every later connection against that pin and
tells you exactly what drifted — and for changed tools, whether the
**description** or the **inputSchema** moved.

## Install

```bash
pip install "git+https://github.com/cognis-digital/mcppin.git"
```

Pure standard library, no runtime dependencies. Requires Python 3.10+.

## Use it as a CI gate

Export your server's `tools/list` result to JSON, then:

```bash
# First approval: record the trusted manifest
mcppin pin acme-mcp tools.json

# In CI / before each session: fail (exit 1) if anything drifted
mcppin verify acme-mcp tools.json
```

`verify` exits `0` when the live manifest matches the pin, `1` on drift, and `2`
if the server was never pinned (use `--tofu` to pin automatically on first use).

```text
$ mcppin verify acme-mcp tools_today.json
DRIFT detected for acme-mcp:
  ~ changed: read_file (description)
```

The pin store defaults to `~/.mcppin/pins.json` (resolved from your home
directory, so it works from any directory); override with `--store`.

## Use it as a library

```python
from mcppin.core import PinStore

store = PinStore("pins.json")
store.pin("acme-mcp", tools)            # tools = list of MCP tool dicts

report = store.verify("acme-mcp", live_tools)
if report.is_drift:
    raise RuntimeError(f"MCP tool drift: {report.changed or report.added or report.removed}")
```

`tools` is a list of tool objects exactly as returned by the MCP `tools/list`
call. Reordering tools is **not** drift; only changes to a tool's name,
description, or input schema are.

## What is and isn't covered

- Fingerprints cover `name`, `description`, and `inputSchema` — the parts that
  define a tool's behavioural contract. Cosmetic fields (e.g. `annotations`) are
  intentionally ignored so they don't cause false drift.
- `mcppin` detects *change*; it does not decide whether a change is benign. A
  legitimate update will also show as drift — re-pin to accept it.

## Development

```bash
pip install "git+https://github.com/cognis-digital/mcppin.git"
git clone https://github.com/cognis-digital/mcppin.git && cd mcppin
python -m pytest tests/ -q
```

## License

MIT — see [LICENSE](LICENSE).
