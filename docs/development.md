# Development Guide

Development environment setup and workflows for ECOS Chip Compiler.

## Installation

### Option 1: Bazel Dev Setup (Recommended)

If not using Nix, set up the full development environment with a single command:

```bash
bazel run //:prepare_dev
```

This runs three steps:
1. Creates/syncs the Python venv with editable workspace packages:
   ```bash
   uv sync --frozen --all-groups --python 3.11 \
     --no-build-isolation-package ecc-dreamplace \
     --no-binary-package ecc-tools-bin \
     --verbose
   ```
   Keep the `ecc-dreamplace` and `ecc-tools-bin` flags; a plain `uv sync` can leave the native workspace packages non-editable.
2. Builds and extracts the ECC runtime bundle → `chipcompiler/tools/ecc/bin/`
3. Builds and installs DreamPlace operators (source build, dev only) → `chipcompiler/thirdparty/ecc-dreamplace/dreamplace/ops/`

ECC-Tools and DreamPlace are built in parallel by Bazel. On memory-constrained machines, limit parallelism:
```bash
bazel run //:prepare_dev --jobs=1
```

### Option 2: Nix Development Shell

```bash
nix develop  # Provides Python 3.11+, uv, Yosys, and native build deps

# Inside the shell:
uv sync --frozen --all-groups --python 3.11 \
  --no-build-isolation-package ecc-dreamplace \
  --no-binary-package ecc-tools-bin \
  --verbose
source .venv/bin/activate
```

Auto-load with direnv:
```bash
echo "use flake" > .envrc
direnv allow
```

The Nix shell provides toolchains and `uv`; it does not replace the project sync. Run the same `uv sync` command inside the shell. Binary cache at [serve.eminrepo.cc](https://serve.eminrepo.cc).

## Bazel Build System

Bazel is used for reproducible release builds and ECC-Tools C++ compilation. Requires Bazel 8+ and `uv` on PATH.

```bash
bazel build //chipcompiler/thirdparty:ecc_py_cmake       # ECC-Tools C++ build
bazel run //bazel/scripts:install_dreamplace              # Build + install DreamPlace .so (via @ecc-dreamplace)
bazel run //bazel/scripts:clean_dreamplace                # Remove installed DreamPlace artifacts
bazel run //bazel/scripts:prepare_dev                     # Full dev environment setup (ECC + DreamPlace)
```

Use `--config=ghproxy` behind restricted networks. For `git_override` deps (e.g. `ecos-bazel`), configure git mirror directly:

```bash
git config --global url."https://ghfast.top/https://github.com/".insteadOf "https://github.com/"
bazel build --config=ghproxy //...
```

Python deps are managed via `uv.lock`; development sync uses the workspace sources in `[tool.uv.sources]`. To update: edit `pyproject.toml`, run `uv lock`, then rerun the dev `uv sync` command from [Installation](#installation).

## Release Builds

### PyInstaller CLI Release Artifact

Tagged ECC releases publish a Linux x86_64 PyInstaller onedir CLI bundle:

```text
ecc-cli-linux-x86_64.tar.gz
```

Build the release artifact locally with:

```bash
bazel build //:build_ecc_cli_bundle
mkdir -p dist/release
gzip -n -9 -c bazel-bin/build_ecc_cli_bundle/ecc.tar \
  > dist/release/ecc-cli-linux-x86_64.tar.gz
```

The archive extracts to an `ecc` executable and `_internal/` runtime directory.
After extraction:

```bash
./ecc --version
```

Release notes for tagged ECC releases are generated with `git-cliff` using
[`ecc/.github/cliff.toml`](../.github/cliff.toml). The release workflow appends
the generated changelog before the CLI artifact checksum block that is published
to GitHub Releases.

### Manual Python Package
```bash
uv build
```

### Manual Bazel Wheel Build (ECC Runtime + auditwheel)

For developer validation, build a portable wheel with Bazel-managed ECC runtime
and hermetic uv/Python:

```bash
bazel build //:raw_wheel   # Sandboxed, cacheable — produces raw .whl
bazel run //:build_wheel   # auditwheel repair + smoke test
```

Artifacts:
- Raw wheels: `dist/wheel/raw/`
- Repaired wheels: `dist/wheel/repaired/`
- auditwheel report: `dist/wheel/reports/show.txt`

Requirements:
- Linux x86_64
- `auditwheel` (installed via dev deps)

Common failures:
- `auditwheel` missing: run the dev `uv sync` command from [Installation](#installation)
- `ecc_py*.so` missing after bundle extraction: build/install runtime (`bazel run //:prepare_dev`)
- auditwheel policy mismatch (e.g. glibc symbols too new): rebuild on older compatible base or adjust target policy
- missing runtime libraries: inspect `dist/wheel/reports/show.txt`

### DreamPlace Wheel Build

DreamPlace has its own standalone build, CI/CD, and release pipeline.

- **Default dev mode**: run the root sync command from [Installation](#installation). It installs `ecc`, `ecc-dreamplace`, and `ecc-tools-bin` from the uv workspace as editable packages, so source edits in `chipcompiler/thirdparty/ecc-dreamplace` and `chipcompiler/thirdparty/ecc-tools` are reflected without reinstalling. Keep `--no-build-isolation-package ecc-dreamplace` and `--no-binary-package ecc-tools-bin`; without them the native packages can fall back to non-editable builds.
  `bazel run //:prepare_dev` still builds the source-tree operators so source debug mode can use the compiled artifacts when needed.

- **CI/release mode**: `.github/actions/setup-python-deps` downloads pre-built GitHub Release wheels for `ecc-dreamplace` and `ecc-tools-bin`, installs them with `uv pip install --no-deps`, then runs:

  ```bash
  uv sync --frozen --all-groups --python 3.11 --inexact \
    --no-install-package ecc-dreamplace \
    --no-install-package ecc-tools-bin
  ```

  This keeps CI/release environments on the pre-installed wheels instead of replacing them with editable workspace builds.

### DreamPlace Debug Modes

ECC currently uses two different DreamPlace modes during development:

- **Default mode**: `ecc-dreamplace` is installed into `.venv` as an editable workspace package pointing at
  `chipcompiler/thirdparty/ecc-dreamplace`.
- **Forced source debug mode**: debugger startup prepends
  `chipcompiler/thirdparty/ecc-dreamplace` to `PYTHONPATH`, so `import dreamplace`
  resolves to the submodule source tree even if another install is present.

Use the default mode for normal development and release-like source validation. Use forced source
debug mode only when you need to make the import override explicit for debugger sessions.

#### Default Mode

- Use the dev `uv sync` command from [Installation](#installation); it keeps the runtime pointed at editable workspace packages.
- LSP configuration such as `python.analysis.extraPaths` can improve source navigation, but it does not change runtime imports.
- Python breakpoints land in `chipcompiler/thirdparty/ecc-dreamplace/dreamplace/*.py` because the editable install points at the workspace source.

#### Forced Source Debug Mode

This mode adds a `PYTHONPATH` override for the debug session. It is only needed
when you want to force source-tree imports explicitly; it does not replace the editable `uv sync`.

1. Build and install DreamPlace operators into the source tree:

   ```bash
   bazel run //bazel/scripts:install_dreamplace
   ```

2. Launch the debugger with `PYTHONPATH` pointing at the submodule root:

   ```json
   {
     "name": "ECC Debug (dreamplace source)",
     "type": "debugpy",
     "request": "launch",
     "program": "${workspaceFolder}/chipcompiler/cli/main.py",
     "cwd": "${workspaceFolder}",
     "python": "${workspaceFolder}/.venv/bin/python",
     "justMyCode": false,
     "subProcess": true,
     "console": "integratedTerminal",
     "env": {
       "PYTHONPATH": "${workspaceFolder}/chipcompiler/thirdparty/ecc-dreamplace:${env:PYTHONPATH}"
     }
   }
   ```

This works because `chipcompiler/thirdparty/ecc-dreamplace` contains the `dreamplace/`
package root. When it appears before `site-packages` on `PYTHONPATH`, Python imports
the submodule source tree first.

`subProcess: true` is recommended because ECC flow steps run in `multiprocessing.Process`.

#### Behavior And Limits

- Breakpoints in `chipcompiler/thirdparty/ecc-dreamplace/dreamplace/*.py` will hit after the default editable sync; forced source debug mode keeps that source path first on `PYTHONPATH`.
- Python source edits take effect with the editable sync. Rebuild/reinstall DreamPlace operators when compiled `.so` artifacts change.
- This override is aimed at Python code. It does not provide C++ or `.so` source-level debugging.
- If source debug mode fails with missing DreamPlace operators, rerun `bazel run //bazel/scripts:install_dreamplace`.
- If you only need editor navigation and not runtime overrides, prefer LSP-only configuration such as `python.analysis.extraPaths`.

#### Optional LSP-Only Navigation

If you want jump-to-definition into the submodule source while keeping runtime behavior
unchanged, add the source roots to your IDE configuration instead of `PYTHONPATH`:

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.analysis.extraPaths": [
    "${workspaceFolder}/chipcompiler/thirdparty/ecc-dreamplace",
    "${workspaceFolder}/chipcompiler/thirdparty/ecc-tools"
  ]
}
```

This improves indexing and navigation only. Debugger and runtime imports still follow
the installed interpreter environment.


## Code Quality

```bash
# Format & lint (recommended)
uv run ruff format chipcompiler/ test/
uv run ruff check chipcompiler/ test/

# Type check
uv run ty check              # Recommended (configured in pyproject.toml)
uv run pyright chipcompiler/
uv run mypy chipcompiler/

# Legacy
uv run black chipcompiler/ test/
uv run isort chipcompiler/ test/
```

## Testing

```bash
uv run pytest test/                                    # All tests
uv run pytest test/test_tools_yosys_utility.py -v     # Specific file
uv run pytest test/ --cov=chipcompiler --cov-report=term-missing  # Coverage
uv run pytest test/formal/ -v                          # Formal verification tests only
```

### Formal Verification

z3-based formal verification. See [test/formal/README.md](../test/formal/README.md) for details on the approach, test inventory, and known bugs found.

## Add a New EDA Tool

### 1. Create Structure
```bash
mkdir -p chipcompiler/tools/<tool_name>/{configs,scripts}
touch chipcompiler/tools/<tool_name>/{__init__.py,builder.py,runner.py,utility.py}
```

### 2. Implement Interface

**builder.py:**
```python
from chipcompiler.data import Workspace, WorkspaceStep, StepEnum

def build_step(workspace: Workspace, step: StepEnum) -> WorkspaceStep:
    return WorkspaceStep(workspace=workspace, step=step, tool="<tool_name>")

def build_step_space(workspace_step: WorkspaceStep) -> None:
    workspace_step.create_directories()

def build_step_config(workspace_step: WorkspaceStep) -> None:
    config = {"input": workspace_step.input_path, "output": workspace_step.output_path}
    workspace_step.write_config(config)
```

**runner.py:**
```python
import subprocess
from chipcompiler.data import WorkspaceStep, StateEnum

def is_eda_exist() -> bool:
    try:
        subprocess.run(["<tool_name>", "--version"], capture_output=True, check=True)
        return True
    except (subprocess.CalledProcessError, FileNotFoundError):
        return False

def run_step(workspace_step: WorkspaceStep) -> StateEnum:
    try:
        result = subprocess.run(
            ["<tool_name>", "-c", workspace_step.config_file],
            cwd=workspace_step.path, capture_output=True, timeout=workspace_step.timeout
        )
        return StateEnum.Success if result.returncode == 0 else StateEnum.Incomplete
    except Exception as e:
        workspace_step.log_error(str(e))
        return StateEnum.Incomplete
```

**__init__.py:**
```python
from .builder import build_step, build_step_space, build_step_config
from .runner import is_eda_exist, run_step

__all__ = ["build_step", "build_step_space", "build_step_config", "is_eda_exist", "run_step"]
```

### 3. Add Configs & Scripts
- JSON templates in `configs/`
- TCL/Python/Shell scripts in `scripts/`

### 4. Integrate into Flow
Update `EngineFlow.build_default_steps()` or use `add_step()`.

### 5. Write Tests
```python
# test/test_tools_<tool_name>.py
import pytest
from chipcompiler.tools.<tool_name> import is_eda_exist, run_step

@pytest.mark.skipif(not is_eda_exist(), reason="<tool_name> not installed")
def test_run_step():
    # Test implementation
    pass
```

## Integrating a Thirdparty Tool into the Build System

ECC uses a dual-build strategy: **uv-managed Python deps** for dev, **Bazel** (+ Nix) for release. Reference `ecc-dreamplace` as a working example.

**Principle**: Avoid modifying the thirdparty tool's own build system (CMakeLists, setup.py, etc.). Prefer solving build issues from the Bazel side or ECC's build configuration (cache entries, env vars, wrapper scripts).

### 1. Python deps

Add the package to root `pyproject.toml`, choose either a local workspace source or a locked wheel URL under `[tool.uv.sources]`, then `uv lock`.

### 2. Dev build

If the tool has compiled artifacts (`.so`, generated configs):
- Add a Bazel build target in `chipcompiler/thirdparty/BUILD.bazel`
- Create `bazel/scripts/install-<tool>.sh` with manifest-based install/clean (see `install-dreamplace.sh`)
- Register `install_<tool>` and `clean_<tool>` targets in `bazel/scripts/BUILD.bazel`
- Wire into `prepare-dev.sh` with explicit `RUNFILES_DIR="${RF}"`

### 3. Release build

Add runtime artifacts to `//chipcompiler:chipcompiler_runtime_data` (consumed by `raw_wheel`). For Nix, add to flake build inputs.

### 4. Bazel sandbox deps

Add the tool's extra to `uv_export(extras=[...])` in `MODULE.bazel`; reference as `@pypi//<pkg>` in BUILD files.

### 5. EDA tool module

Create `chipcompiler/tools/<tool>/` with `__init__.py`, `builder.py`, `runner.py`. Each tool must implement `is_eda_exist`, `build_step`, `run_step`. Integrate into flow via `EngineFlow.build_default_steps()` or `add_step()`. See [Add a New EDA Tool](#add-a-new-eda-tool) above for the full interface and code examples.

### Pitfalls

- **Runfiles**: child scripts called from `prepare-dev.sh` cannot resolve `RUNFILES_DIR` from `BASH_SOURCE[0]` — pass it explicitly
- **File ownership**: use `cp --no-preserve=ownership` and `tar --no-same-owner` when extracting Bazel outputs to avoid root-owned files in devcontainer builds

## CLI Usage

For command-line automation and scripting, run CLI via Nix:

```bash
# Create a project skeleton with ecc.toml, rtl/, constraints/, and runs/
nix run .#cli -- init gcd

# After editing gcd/ecc.toml and adding RTL files
nix run .#cli -- check --project gcd
nix run .#cli -- run --project gcd
nix run .#cli -- status --project gcd
nix run .#cli -- metrics --project gcd
nix run .#cli -- log --project gcd
```

The project config is the CLI input surface:

```toml
[design]
name = "gcd"
top = "gcd"
rtl = ["rtl/gcd.v"]
clock_port = "clk"
frequency_mhz = 100.0

[pdk]
name = "ics55"
root = "/path/to/ics55"

[flow]
preset = "rtl2gds"
run = "default"
```

For filelist mode, set `design.rtl` to a single filelist path, for example
`rtl = ["rtl/filelist.f"]`. Multiple RTL sources should be listed in the
filelist rather than as multiple `design.rtl` entries.

If you need an interactive environment for development, use `nix develop`.

REST API reference: Examples: **[examples/gcd](examples/gcd/README.md)**

### Yosys Runtime Resolution

Resolution priority in `chipcompiler/tools/yosys/utility.py`:
1. Bundled runtime (`CHIPCOMPILER_OSS_CAD_DIR`)
2. System PATH (`yosys`)

Runtime handling:
- `get_yosys_command()` - Side-effect-free detection
- `get_yosys_runtime()` - Returns `(command, env)` for subprocess (no global `os.environ` mutation)
- `check_slang_plugin()` - Preflight check: `yosys -p "plugin -i slang"`

#### Troubleshooting: Yosys executable not found

If you see this error:

```
RuntimeError: Yosys executable not found in system PATH, and CHIPCOMPILER_OSS_CAD_DIR is not set.
Please install yosys or set CHIPCOMPILER_OSS_CAD_DIR to the OSS CAD Suite root directory.
```

Download the [OSS CAD Suite](https://github.com/YosysHQ/oss-cad-suite-build/releases) pre-built package for your platform, extract it, and set the environment variable:

```bash
# Download and extract (example for Linux x86_64)
wget https://github.com/YosysHQ/oss-cad-suite-build/releases/download/<version>/oss-cad-suite-linux-x64-<date>.tgz
tar -xzf oss-cad-suite-linux-x64-<date>.tgz

# Set the environment variable
export CHIPCOMPILER_OSS_CAD_DIR=/path/to/oss-cad-suite

# Or add yosys to PATH directly
source /path/to/oss-cad-suite/environment
```

### PDK Runtime Resolution

Resolution priority for `get_pdk("ics55")` in `chipcompiler/data/pdk.py`:
1. Explicit `pdk_root` argument
2. `CHIPCOMPILER_ICS55_PDK_ROOT` env var
3. Legacy `ICS55_PDK_ROOT` env var
4. In-repo default: `chipcompiler/thirdparty/icsprout55-pdk`

Backend supports `POST /api/workspace/set_pdk_root` to set runtime path. Workspace creation persists resolved root in `parameters.json` as `PDK Root`.

Example: `CHIPCOMPILER_ICS55_PDK_ROOT=/path/to/pdk chipcompiler`

## Common Workflows

### Debug Flow Step
1. Check `workspace_step.logs/` for tool output
2. Inspect `workspace_step.config/` for configs
3. Verify `workspace_step.input/` files
4. Run individual step: `EngineFlow.run_step()`

### Modify Flow Sequence
1. Edit `EngineFlow.build_default_steps()` or use `add_step()`
2. Persist: `flow.save()` → `workspace.flow.json`
3. Run: `flow.run_steps()` (skips successful steps)
4. Reset: `clear_states()` to re-run

## Related Documentation

- [Architecture](architecture.md) - System design and patterns
