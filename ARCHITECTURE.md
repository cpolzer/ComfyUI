# ComfyUI Architecture

## Overview

ComfyUI is a graph-based execution engine for running machine learning workflows, primarily focused on Stable Diffusion and related generative AI models. Users compose workflows as node graphs which are submitted as JSON and executed via topological sort with intelligent caching and memory management.

## Key Directories

| Directory | Purpose |
|-----------|---------|
| `comfy/` | Core ML infrastructure: model architectures, samplers, VAE, CLIP, LoRA, memory management, dtype handling |
| `comfy_extras/` | Additional built-in node packs (image, video, audio, 3D, ControlNet, Flux, etc.) |
| `comfy_api/` | V3 node API type definitions and schema system |
| `comfy_api_nodes/` | Nodes that call external paid APIs (Comfy Cloud) |
| `comfy_execution/` | Graph traversal, caching infrastructure, execution utilities |
| `app/` | User management, settings, asset database (SQLAlchemy/Alembic) |
| `api_server/` | Internal HTTP routes (terminal, file ops) |
| `alembic_db/` | Database migrations |
| `blueprints/` | REST API endpoint blueprints |
| `middleware/` | HTTP middleware (cache control, CORS, etc.) |
| `script_examples/` | Example scripts for programmatic workflow execution |
| `utils/` | Shared utilities (config loading, MIME types) |

## Request Lifecycle

1. **Entry** (`main.py`): CLI args parsed, paths configured, custom nodes loaded, `PromptServer` created
2. **HTTP** (`server.py`): Client submits workflow graph via `POST /prompt`
3. **Validation** (`execution.py`): `validate_prompt()` checks node types, input types, required inputs
4. **Queue**: Valid prompts enqueued in `PromptQueue` (thread-safe priority queue)
5. **Execution**: Background `prompt_worker` thread dequeues, creates `PromptExecutor`, runs `execute_async()`
6. **Graph**: `ExecutionList` performs topological sort, executes nodes in dependency order
7. **Response**: Progress and previews stream back over WebSocket; results stored in history

## Node System

### V1 (Classic)

A Python class with:
- `INPUT_TYPES(s)` returning `{"required": {...}, "optional": {...}}`
- `RETURN_TYPES`, `RETURN_NAMES`, `FUNCTION` (method name), `CATEGORY`
- Optional: `OUTPUT_NODE = True`, `IS_CHANGED`, `VALIDATE_INPUTS`

Exports via `NODE_CLASS_MAPPINGS = {"NodeName": NodeClass}`.

### V3 (Schema-based)

Nodes subclass from `comfy_api/latest/_io.py` and implement `GET_SCHEMA()`. The schema drives both the UI and execution. V3 nodes auto-generate V1-compatible metadata via `GET_NODE_INFO_V1()`.

### Custom Node Loading

`nodes.init_extra_nodes()` scans `custom_nodes/` for Python packages, runs any `prestartup_script.py` first, then imports and registers `NODE_CLASS_MAPPINGS` (V1) or `comfy_entrypoint()` (V3).

## Caching

`CacheSet` in `execution.py` manages output caching with four strategies:

| Strategy | Class | Behavior |
|----------|-------|----------|
| CLASSIC (default) | `HierarchicalCache` | Last-run only |
| LRU | `LRUCache` | Fixed count, evicts least recently used |
| RAM_PRESSURE | `RAMPressureCache` | Evicts by available system RAM |
| NONE | `NullCache` | No caching |

The `IsChangedCache` checks node `IS_CHANGED` methods to determine whether cached outputs are still valid.

## Memory Management

- **Smart offloading**: Models stay loaded between prompts; unloaded only when needed
- **VRAM states**: DISABLED, NO_VRAM, LOW_VRAM, NORMAL_VRAM, HIGH_VRAM, SHARED
- **Pinned memory**: CUDA host-pinned memory for faster CPU→GPU transfers
- **Async offload**: Multiple CUDA/XPU streams for overlapping weight transfer with computation
- **OOM handling**: All models unloaded automatically on out-of-memory

## Graph Execution

`ExecutionList` (in `comfy_execution/graph.py`) extends topological sort with:
- Cache-aware scheduling (skips cached nodes)
- UX-friendly node picking (prioritizes output nodes for immediate preview)
- Lazy input support
- Subgraph expansion (nodes can dynamically add new nodes during execution)
- Cycle detection and error reporting
