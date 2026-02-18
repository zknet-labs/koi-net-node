# AGENTS.md — koi-net Node Development Guide

This file is for AI coding agents (and humans) working in this repository.
This is a **template** for building koi-net nodes. The canonical upstream template
lives at [BlockScience/koi-net-node-template](https://github.com/BlockScience/koi-net-node-template).

---

## Project Overview

A **koi-net node** is a Python service that participates in the koi-net distributed
knowledge network. Nodes process `KnowledgeObject`s through a handler pipeline and
communicate with peers via HTTP webhooks or polling.

**Key dependency**: `koi-net~=1.2.0` (brings pydantic, structlog, fastapi, uvicorn,
httpx, cryptography, networkx, python-dotenv, rid-lib).

**Python**: ≥3.10 (developed with 3.12).

---

## Setup & Install

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -e .
```

Create `.env` (required for private key encryption):

```
PRIV_KEY_PASSWORD=<your_password>
```

On first run, `config.yaml` and `priv_key.pem` are auto-generated from Python defaults.

---

## Running the Node

```bash
python -m koi_net_<your_name>_node
```

Replace `koi_net_<your_name>_node` with the actual package name in `src/`.

---

## Build / Lint / Test Commands

This template ships with **no preconfigured test runner or linter**. Add them to
`pyproject.toml` as needed:

```toml
[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy"]

[tool.pytest.ini_options]
testpaths = ["tests"]

[tool.ruff]
line-length = 88
```

Once added:

```bash
# Run all tests
pytest

# Run a single test file
pytest tests/test_handlers.py

# Run a single test by name
pytest tests/test_handlers.py::test_my_handler

# Lint
ruff check src/

# Format
ruff format src/

# Type check
mypy src/
```

---

## Project Structure

```
src/
  koi_net_<name>_node/
    __init__.py      # empty
    __main__.py      # entry point: MyNode().run()
    config.py        # MyNodeConfig(FullNodeConfig)
    core.py          # MyNode(FullNode) — wires config + handlers
    handlers.py      # @KnowledgeHandler.create decorated functions
pyproject.toml
.env                 # PRIV_KEY_PASSWORD — never commit
config.yaml          # auto-generated on first run — may commit
priv_key.pem         # auto-generated — never commit
.rid_cache/          # local knowledge cache — gitignored
log.ndjson           # rotating NDJSON log — gitignored
```

When creating a new node from this template, rename:

1. `src/koi_net_CHANGE_NAME_node/` → `src/koi_net_<name>_node/`
2. `node_name` in `config.py`
3. `name` in `pyproject.toml`

---

## Core Patterns

### Node class (`core.py`)

```python
from koi_net.core import FullNode
from .config import MyNodeConfig
from .handlers import my_handler

class MyNode(FullNode):
    config_schema = MyNodeConfig
    knowledge_handlers = FullNode.knowledge_handlers + [my_handler]
```

`FullNode()` returns a `NodeContainer` (dependency injection via `NodeAssembler`).
`knowledge_handlers` is a list — **order matters**, the pipeline runs handlers in
list order. Append your handlers after the built-in ones unless you need to override.

### Config class (`config.py`)

```python
from rid_lib.types import KoiNetNode
from koi_net.config.full_node import (
    FullNodeConfig, KoiNetConfig, ServerConfig, NodeProfile, NodeProvides
)

class MyNodeConfig(FullNodeConfig):
    koi_net: KoiNetConfig = KoiNetConfig(
        node_name="my-node-name",       # human-readable identifier
        node_profile=NodeProfile(
            provides=NodeProvides(
                event=[],               # RID types this node broadcasts
                state=[]                # RID types this node serves on demand
            )
        ),
        rid_types_of_interest=[KoiNetNode],  # what this node subscribes to
        first_contact=NodeContact()     # bootstrap peer {rid, url}
    )
    server: ServerConfig = ServerConfig(host="127.0.0.1", port=8000)
```

Config is Pydantic `BaseModel`. The node RID and public key are auto-generated from
the private key on first run. Do not hardcode `node_rid` — let it auto-generate.

### Handlers (`handlers.py`)

```python
import structlog
from koi_net.processor.handler import (
    KnowledgeHandler, HandlerType, HandlerContext, KnowledgeObject
)

log = structlog.stdlib.get_logger()

@KnowledgeHandler.create(
    handler_type=HandlerType.Bundle,
    rid_types=[MyRIDType],              # empty = all types
    event_types=[EventType.NEW])        # empty = all event types
def my_handler(ctx: HandlerContext, kobj: KnowledgeObject):
    # Return options:
    # - kobj (possibly mutated) → continue pipeline
    # - None                    → continue pipeline unchanged
    # - STOP_CHAIN              → halt pipeline immediately
    log.info(f"Processing {kobj.rid!r}")
    return kobj
```

**Five handler types** (pipeline executes in this order):

| `HandlerType` | `kobj` fields available | Typical purpose                                    |
| ------------- | ----------------------- | -------------------------------------------------- |
| `RID`         | `rid`, `event_type`     | Filter/validate before fetching manifest           |
| `Manifest`    | + `manifest`            | Hash/timestamp checks, set `normalized_event_type` |
| `Bundle`      | + `contents`            | Validate contents, write to cache                  |
| `Network`     | + full bundle           | Populate `kobj.network_targets`                    |
| `Final`       | + full bundle           | Side effects after broadcast                       |

**`HandlerContext` API** (available inside any handler):

```python
ctx.identity.rid           # this node's KoiNetNode RID
ctx.identity.profile       # this node's NodeProfile
ctx.config                 # full config object
ctx.cache.read(rid)        # → Bundle | None
ctx.cache.exists(rid)      # → bool
ctx.event_queue.push(event=..., target=peer_rid)
ctx.kobj_queue.push(rid=some_rid)
ctx.kobj_queue.push(bundle=some_bundle, event_type=EventType.UPDATE)
ctx.graph                  # NetworkGraph of edges/peers
ctx.effector.deref(rid)    # resolve RID → Bundle (cache → network fallback)
```

---

## Code Style

### Imports

- Order: stdlib → third-party → local (relative)
- Multi-line imports use parentheses (no backslash continuation)
- Relative imports for intra-package: `from .config import MyNodeConfig`

### Types

- Type-annotate all function parameters and return types
- Use `|` union syntax (Python 3.10+), not `Optional[X]`
- Configuration classes extend Pydantic `BaseModel` (via koi-net base classes)
- No `as any`, no `# type: ignore`

### Naming

- Classes: `PascalCase` (`MyNodeConfig`, `MyNode`)
- Functions, variables, modules: `snake_case`
- Package names: `snake_case` with underscores (`koi_net_<name>_node`)
- Constants: `UPPER_SNAKE_CASE`

### Logging

Use `structlog` — the library configures it automatically at import time:

```python
import structlog
log = structlog.stdlib.get_logger()   # module-level, not inside functions

log.debug("Cache miss")
log.info(f"Proposing edge with {peer_rid!r}")
log.warning(f"Hash mismatch for {node_rid!r}, dropping")
```

- Use `!r` for RIDs in log messages
- Use plain f-strings (not structlog key-value args — not the convention here)
- `debug` → internal transitions; `info` → lifecycle events; `warning` → anomalies

### Error Handling

- Rely on koi-net's `ErrorHandler` component for unhandled pipeline exceptions
- Return `STOP_CHAIN` (import from `koi_net.processor.handler`) to abort a pipeline
  run without raising — prefer this over raising in handlers
- Log warnings before returning `STOP_CHAIN` so failures are traceable

### Docstrings

No docstrings in the template — follow the upstream koi-net style (minimal, only
where non-obvious). If adding them, use plain one-line summaries.

---

## Files to Never Commit

`.env`, `priv_key.pem`, `.rid_cache/`, `log.ndjson`, `__pycache__/`, `config.yaml`
(optional — committing a sanitized `config.yaml` is fine to share defaults).

These are already in `.gitignore`.

---

## Key Reference Links

- koi-net library source: <https://github.com/BlockScience/koi-net>
- Node template upstream: <https://github.com/BlockScience/koi-net-node-template>
- rid-lib: <https://github.com/BlockScience/rid-lib>
