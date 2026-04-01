# ComfyUI Dependency & Task Runner Modernization

**Date:** 2026-04-01
**Author:** opencode
**Status:** Draft

## Problem

ComfyUI manages dependencies across 3 scattered `requirements.txt` files plus undocumented deps in README text:

- `requirements.txt` — 37 runtime deps
- `manager_requirements.txt` — 1 optional dep
- `tests-unit/requirements.txt` — 4 test deps
- `tests/README.md` — mentions `opencv-python` and `scikit-image` in prose

The `pyproject.toml` exists but only contains tool config (`ruff`, `pylint`). No `[project.dependencies]`, no installability via `pip install .`. No task runner — commands are scattered across README, AGENTS.md, and READMEs in subdirectories.

## Solution

Migrate all dependency declarations into `pyproject.toml` using PEP 621 and PEP 735, add `taskipy` as a task runner, and remove all `requirements.txt` files.

## Architecture

### Dependency Groups

All dependencies live in `pyproject.toml`:

```toml
[project]
name = "ComfyUI"
version = "0.18.1"
description = "The most powerful and modular visual AI engine"
readme = "README.md"
license = { file = "LICENSE" }
requires-python = ">=3.10"
dependencies = [
    # Runtime dependencies (from requirements.txt)
    "comfyui-frontend-package==1.42.8",
    "comfyui-workflow-templates==0.9.39",
    "comfyui-embedded-docs==0.4.3",
    "torch",
    "torchsde",
    "torchvision",
    "torchaudio",
    "numpy>=1.25.0",
    "einops",
    "transformers>=4.50.3",
    "tokenizers>=0.13.3",
    "sentencepiece",
    "safetensors>=0.4.2",
    "aiohttp>=3.11.8",
    "yarl>=1.18.0",
    "pyyaml",
    "Pillow",
    "scipy",
    "tqdm",
    "psutil",
    "alembic",
    "SQLAlchemy",
    "filelock",
    "av>=14.2.0",
    "comfy-kitchen>=0.2.8",
    "comfy-aimdo>=0.2.12",
    "requests",
    "simpleeval>=1.0.0",
    "blake3",
    "kornia>=0.7.1",
    "spandrel",
    "pydantic~=2.0",
    "pydantic-settings~=2.0",
    "PyOpenGL",
    "glfw",
]

[dependency-groups]
dev = [
    "ruff",
    "pylint",
    "taskipy",
]
test = [
    "pytest>=7.8.0",
    "pytest-aiohttp",
    "pytest-asyncio",
    "websocket-client",
    "opencv-python",
    "scikit-image",
]
manager = [
    "comfyui_manager==4.1",
]
```

### Taskipy Configuration

```toml
[tool.taskipy.tasks]
dev = { cmd = "python main.py", help = "Start ComfyUI server" }
dev-cpu = { cmd = "python main.py --cpu", help = "Start ComfyUI server (CPU only)" }
lint = { cmd = "ruff check .", help = "Run ruff linter" }
format = { cmd = "ruff format .", help = "Format code with ruff" }
format-check = { cmd = "ruff format --check .", help = "Check code formatting" }
pylint = { cmd = "pylint comfy_api_nodes", help = "Run pylint on API nodes" }
unit = { cmd = "pytest tests-unit", help = "Run unit tests" }
integration = { cmd = "pytest tests/execution -v --skip-timing-checks", help = "Run integration tests" }
inference = { cmd = "pytest tests/inference", help = "Run inference tests" }
quality = { cmd = "pytest tests/inference tests/execution -v --skip-timing-checks", help = "Quality regression tests" }
test = { cmd = "pytest tests-unit tests/execution -v --skip-timing-checks", help = "Run unit + integration tests" }
all = { cmd = "ruff check . && ruff format --check . && pylint comfy_api_nodes && pytest tests-unit tests/execution -v --skip-timing-checks", help = "Run all checks" }
manager = { cmd = "pip install -e . --group manager", help = "Install manager dependencies" }
```

### Files Changed

| File | Action | Reason |
|------|--------|--------|
| `pyproject.toml` | Rewrite | Add `[project]`, `[dependency-groups]`, `[tool.taskipy]` |
| `requirements.txt` | Delete | Superseded by `[project.dependencies]` |
| `manager_requirements.txt` | Delete | Superseded by `[dependency-groups.manager]` |
| `tests-unit/requirements.txt` | Delete | Superseded by `[dependency-groups.test]` |
| `tests/README.md` | Update | Replace manual pip install instructions with taskipy commands |
| `tests-unit/README.md` | Update | Replace manual pip install instructions with taskipy commands |
| `README.md` | Update | Replace install/run instructions with taskipy commands |
| `AGENTS.md` | Update | Update Commands section with taskipy commands |

## Installation Commands

### Runtime only
```bash
pip install .
```

### With dev tools
```bash
pip install --group dev
```

### With test tools
```bash
pip install --group test
```

### With manager
```bash
pip install --group manager
```

### All at once (pip 24.0+)
```bash
pip install . --group dev --group test
```

## Task Commands

All tasks run via `task <name>` after installing dev dependencies:

| Task | Description |
|------|-------------|
| `task dev` | Start ComfyUI server on default port |
| `task dev-cpu` | Start ComfyUI server (CPU only) |
| `task lint` | Run ruff linter |
| `task format` | Format code with ruff |
| `task format-check` | Check code formatting without modifying |
| `task pylint` | Run pylint on comfy_api_nodes |
| `task unit` | Run unit tests |
| `task integration` | Run integration/execution tests |
| `task inference` | Run inference tests |
| `task quality` | Quality regression tests |
| `task test` | Run unit + integration tests |
| `task all` | Run all checks (lint + format-check + pylint + test) |
| `task manager` | Install manager dependencies |

## GPU-Specific PyTorch

The README GPU-specific install instructions remain unchanged — users install the appropriate PyTorch variant first, then `pip install .`. The `torch` in `[project.dependencies]` is a fallback for CPU-only installs.

Example:
```bash
# NVIDIA
pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu130
pip install .

# AMD (Linux)
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm7.2
pip install .
```

## Migration Notes

- PEP 735 requires pip ≥ 24.0. Users on older pip should upgrade: `pip install --upgrade pip`
- The `--group` flag is the modern way to install dependency groups. For older pip, `pip install .[dev]` won't work since we're using PEP 735, not optional-dependencies. This is an acceptable trade-off.
- Custom node loading is unaffected — it reads from `custom_nodes/` directory, not from dependency config.
