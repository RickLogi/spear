# Updates for uv Support

This document summarizes the changes made to support setting up and building SPEAR with [`uv`](https://docs.astral.sh/uv/) instead of Anaconda, and records where things end up on disk as a result.

## Where the virtual environment lives

- Created at the repo root: `.venv` (via `uv venv --python 3.11 .venv`).
- Activated per-shell via `source .venv/bin/activate`, which sets `VIRTUAL_ENV` and puts `.venv/bin` on `PATH`.
- Activation does **not** persist across separate shell/subprocess invocations (e.g., separate terminal tabs, or separate tool invocations that spawn their own shell) — `source .venv/bin/activate` must be re-run in each new shell before running SPEAR's Python tools.
- `uv`-created virtual environments don't ship a `pip` executable, so `uv pip ...` is used instead wherever the tooling previously shelled out to `pip`.

## Doc changes

### `docs/getting_started.md`
- Added a `uv` alternative alongside the existing Anaconda instructions for installing Git/CMake: on Linux, install both via the system package manager (`sudo apt-get install git cmake`) instead of `conda install`/`pip install cmake`.
- Added a `uv` alternative alongside the existing `conda create`/`conda activate`/`pip install -e python` instructions:
  ```console
  uv venv --python 3.11 .venv
  source .venv/bin/activate
  sudo apt-get install gcc   # Linux only
  uv pip install -e python
  ```
- Noted that SPEAR's command-line tools (e.g., `tools/install_python_extension.py`) detect an active virtual environment via `VIRTUAL_ENV`, which `uv venv`/`source .venv/bin/activate` sets automatically — no extra flags needed as long as the venv is activated before running a tool.

### `docs/controlling_with_natural_language.md`
- Reworded "activate our `spear-env` Anaconda environment" to "activate our `spear-env` Python environment (Anaconda or `uv`)".

## Code changes

### `tools/install_python_extension.py`
Previously this script unconditionally required a conda installation (`assert conda_script is not None`), which hard-fails on a machine with only `uv` installed. Now, on macOS and Linux:
- If `VIRTUAL_ENV` is set (i.e., a `uv`/`venv` environment is already active), conda discovery is skipped entirely — `cmd_prefix` no longer activates conda, and `pip_cmd` becomes `"uv pip"` when `uv` is available on `PATH`.
- Otherwise, the original conda-based behavior (searching `~/anaconda3`, `~/miniconda3`, etc.) is unchanged.
- Windows is untouched (not part of this user's workflow, and conda is embedded in the documented PowerShell flow there).

### `tools/install_python_package_in_editor_env.py`
On Linux, the Python interpreter bundled under `Engine/Binaries` doesn't ship its own `Python.h`. Building C-extension dependencies (e.g., `psutil`) against it previously failed. Fixed by pointing the compiler at the headers that do ship, under `Engine/Source`:
```python
env["CPATH"] = <unreal_engine_dir>/Engine/Source/ThirdParty/Python3/Linux/include + os.pathsep + env.get("CPATH", "")
```
This `env` is passed explicitly to the `pip install` subprocess call.

### `python/spear/utils/tool_utils.py`
`RunUAT.sh`/`Build.sh` invoke their child process via `env VAR=val cmd args...` (the standard coreutils calling convention for setting environment variables before exec). On this machine, `~/.local/bin/env` — a shell script installed by `uv`'s installer that's only meant to be `source`d (like `~/.cargo/env`) to add `~/.local/bin` to `PATH` — shadows the real `/usr/bin/env` because `~/.local/bin` is prepended to `PATH`. That shim doesn't implement the `env VAR=val cmd args...` convention at all, so invoking it as a command silently no-ops instead of running the intended command, with no error output.

This caused two distinct failures during the Unreal build pipeline:
- `env -- chmod +x .../SpearSim.sh` during staging silently did nothing, so the staged executable never got its executable bit set (originally surfaced as a `Win32Exception (13): Permission denied` once .NET tried to invoke the non-executable file).
- `env uebp_LogFolder=... dotnet AutomationTool.dll BuildCookRun ...` silently no-op'd and returned exit code 0 in ~10ms, without doing any build/cook/stage/package/archive/pak work.

Fixed by prepending `/usr/bin` to `PATH` for the subprocess `RunUAT.sh`/`Build.sh` are run in, scoped to just that subprocess (no changes to the user's shell dotfiles or global `PATH`):
```python
env = os.environ.copy()
if sys.platform in ["darwin", "linux"]:
    env["PATH"] = "/usr/bin:" + env.get("PATH", "")
process = subprocess.Popen(cmd, ..., env=env)
```

## Verification performed

- `python tools/install_python_extension.py --unreal-engine-dir ... ` built and installed the `spear_ext` extension module using `uv pip install -e ...` under an active `.venv`.
- `python tools/install_python_package_in_editor_env.py` installed the `spear` package into the Unreal Editor's bundled Python environment, including C-extension dependencies (`psutil`), after the `CPATH` fix.
- `python tools/run_uat.py --unreal-engine-dir ... -build -cook -stage -package -archive -pak -UbtArgs="-UbaVisualizer"` completed the full pipeline (`BUILD SUCCESSFUL`, exit code 0) and produced `cpp/unreal_projects/SpearSim/Standalone-Development/Linux/SpearSim.sh`, executable.
- `examples/render_image_multi_view/run.py`, configured via a `user_config.yaml` pointing `GAME_EXECUTABLE` at that `SpearSim.sh`, ran end-to-end: launched the standalone executable, rendered 300 images, and encoded them into `video.mp4`.
