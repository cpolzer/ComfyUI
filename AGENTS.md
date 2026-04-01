# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Run the server:**
```bash
task dev
# Common flags: --port 8188, --cpu, --listen 0.0.0.0, --disable-api-nodes, --disable-all-custom-nodes
# Or directly: python main.py --port 8188
```

**Install dependencies:**
```bash
pip install .                    # runtime only
pip install . --group dev        # with dev tools (ruff, pylint, taskipy)
pip install . --group test       # with test tools (pytest, opencv, etc.)
pip install . --group dev --group test  # all at once
```

**Unit tests:**
```bash
task unit
# Or: pytest tests-unit
# Single file: pytest tests-unit/comfy_test/folder_path_test.py
```

**Execution/integration tests** (no GPU needed, uses CPU torch):
```bash
task integration
# Or: pytest tests/execution -v --skip-timing-checks
```

**Lint:**
```bash
task lint        # ruff check .
task format      # ruff format .
task pylint      # pylint comfy_api_nodes
task all         # lint + format-check + pylint + test
```

## Architecture

### Request lifecycle

The frontend (served as static assets from the `comfyui-frontend-package`) sends a workflow graph as a JSON "prompt" to `POST /prompt`. The server (`server.py`) validates it via `execution.py::validate_prompt`, then enqueues it in `PromptQueue`. A background thread (`prompt_worker` in `main.py`) dequeues items and runs `PromptExecutor.execute_async`. The executor uses `ExecutionList` (a topological sort in `comfy_execution/graph.py`) to determine node execution order, then calls each node's function with its resolved inputs. Progress and previews stream back over WebSocket.

### Key files

| File | Purpose |
|------|---------|
| `main.py` | Entry point: loads custom nodes, starts prompt worker thread, runs aiohttp server |
| `server.py` | `PromptServer` — aiohttp app with all REST endpoints + WebSocket handler |
| `execution.py` | `PromptExecutor`, `PromptQueue`, `CacheSet`, node-level `execute()` |
| `comfy_execution/graph.py` | `DynamicPrompt` (prompt graph), `ExecutionList` (topological sort) |
| `nodes.py` | All built-in node classes + `NODE_CLASS_MAPPINGS` registry + custom node loader (`init_extra_nodes`) |
| `folder_paths.py` | Global registry of model/input/output/temp directories; most-imported file in the codebase |
| `node_helpers.py` | Shared utility functions used across many node files |

### Directory layout

- `comfy/` — Core ML code: model architectures, samplers, VAE, CLIP, LoRA, memory management, dtype handling
- `comfy_extras/` — Additional built-in node packs (image, video, audio, 3D, ControlNet, etc.)
- `comfy_api_nodes/` — Nodes that call external paid APIs (Comfy Cloud); pylint-enforced
- `comfy_api/` — Node API type definitions (V3 schema system)
- `comfy_execution/` — Graph traversal, caching infrastructure
- `app/` — User management, settings, asset database (SQLAlchemy/Alembic)
- `api_server/` — Internal HTTP routes (terminal, file ops)
- `alembic_db/` — DB migrations
- `tests-unit/` — Fast unit tests (no server required)
- `tests/execution/` — Integration tests that spin up a real ComfyUI server

### Node system (two versions)

**V1 (classic):** A node is a Python class with:
- `INPUT_TYPES(s)` classmethod returning `{"required": {...}, "optional": {...}}`
- `RETURN_TYPES`, `RETURN_NAMES`, `FUNCTION` (name of the method to call), `CATEGORY`
- Optional: `OUTPUT_NODE = True`, `IS_CHANGED`, `VALIDATE_INPUTS`

Custom nodes export `NODE_CLASS_MAPPINGS = {"NodeName": NodeClass}` and optionally `NODE_DISPLAY_NAME_MAPPINGS`.

**V3 (new schema-based):** Nodes subclass from `comfy_api/latest/_io.py` and implement `GET_SCHEMA()`. The schema drives both the UI and execution. V3 nodes auto-generate V1-compatible metadata via `GET_NODE_INFO_V1()`.

### Custom node loading

`nodes.init_extra_nodes()` scans `custom_nodes/` (and other registered paths from `folder_paths`) for Python packages or `.py` files, running any `prestartup_script.py` first. It imports each module and registers its `NODE_CLASS_MAPPINGS` into the global registry.

### Caching

`CacheSet` in `execution.py` manages output caching between runs. Three strategies: classic (last-run only), LRU (fixed count), RAM-aware (evict by available memory). The `IsChangedCache` checks node `IS_CHANGED` methods to decide whether a cached output is still valid.
