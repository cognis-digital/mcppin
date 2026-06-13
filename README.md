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

<!-- cognis:domains:start -->
## Domains

**Primary domain:** AI & ML  ·  **JTF MERIDIAN division:** ATHENA-PRIME · SAGE

**Topics:** `cognis` `ai` `llm` `machine-learning` `mcp` `agent-security` `crypto` `web3`

Part of the **Cognis Neural Suite** — 300+ source-available tools organized across 12 domains under the JTF MERIDIAN command structure. See the [suite on GitHub](https://github.com/cognis-digital) and [jtf-meridian](https://github.com/cognis-digital/jtf-meridian) for how the pieces fit together.
<!-- cognis:domains:end -->

## Usage — step by step

`mcppin` fingerprints an MCP server's tool manifest (trust-on-first-use) and verifies later connections against that pin.

1. **Install** (pure stdlib, Python 3.10+):
   ```bash
   pip install "git+https://github.com/cognis-digital/mcppin.git"
   ```
2. **Capture the manifest** you want to trust — export the server's `tools/list` result to `tools.json` (a bare array, or `{"tools": [...]}`).
3. **Pin it on first approval** (writes to `~/.mcppin/pins.json`, override with `--store`):
   ```bash
   mcppin pin acme-mcp tools.json
   ```
4. **Verify a live manifest** on every later session; exit `1` on drift, `2` if not yet pinned (use `--tofu` to pin automatically the first time):
   ```bash
   mcppin verify acme-mcp tools.json
   mcppin verify acme-mcp tools.json --tofu
   ```
5. **Inspect what's pinned** when you need to audit:
   ```bash
   mcppin list                 # pinned servers + tool counts
   mcppin show acme-mcp        # per-tool fingerprints
   ```
6. **Gate CI** — `verify` returns non-zero on any added/removed/changed tool, so a drifted description or widened schema fails the pipeline:
   ```bash
   mcppin verify acme-mcp tools.json || { echo "MCP tool drift!"; exit 1; }
   ```

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

## Interoperability

`{}` composes with the 300+ tool Cognis suite — JSON in/out and a shared
OpenAI-compatible `/v1` backbone. See **[INTEROP.md](INTEROP.md)** for the
suite map, composition patterns, and reference stacks.

## License

Cognis Open Collaboration License (COCL) 1.0 — a source-available license
(free for non-commercial use; commercial use requires a separate license). See
[LICENSE](LICENSE).
