# Running a Compilation

Tactus exposes a dedicated `compile` subcommand that builds a configuration aimed at compiling IAL and starts (optionally) the compilation suite.

## Quick start

```
tactus compile --ial-tag develop
```

This will:

1. Build a config from `config.toml` + host-specific overrides + `compile_suite.toml`
2. Set `ial.ial_version` to `develop`
3. Generate a case named `IAL_develop_compile`
4. Start the compilation suite (`CompilationSuiteDefinition`) because `-d` (equivalent to `--dry-run`) is not passed as an argument

## What the command does

The `compile` subcommand is a specialization of `tactus case`. It:

* Sets `ial.ial_version` from `--ial-tag` (default: `develop`)
* Sets `ial.compile.ial_git_repo` from `--ial-repo`, when provided
* Always merges the following modification files on top of the user-supplied config:
  * `tactus/data/config_files/modifications/@HOST@.toml`
  * `tactus/data/config_files/modifications/compile_suite.toml`
* Forwards everything else (output path, start-suite flag, keep-def-file, expand-config) to `tactus case`

`--ial-tag` and `--ial-repo` are applied as two separate config updates, so both can be supplied together safely.

## Required configuration

For the compilation suite to run end-to-end, the following keys are read by the task classes documented below:

| Key | Used by | Notes |
| --- | --- | --- |
| `ial.ial_version` | `IALClone`, `TactusBundleBuild` (when `ial.compile.install` is enabled), macro `@IAL_VERSION@` | Set via `--ial-tag`; also the branch/tag checked out after cloning |
ld` | Bundle working directory |
| `ial.arch` | `TactusBundleBuild`, macro `@ARCH@` | Build architecture |
| `ial.compiler` | `TactusBundleBuild`, macro `@COMPILER@` |
| `ial.compile.ial_git_repo` | `IALClone` | Required if cloning IAL |
| `ial.compile.git_token` | `IALClone`, `TactusBundleCreate` | Optional; enables HTTPS token auth |
| `ial.compile.ial_dir` | `IALClone`, `TactusBundleCreate` | Local IAL checkout path |
| `ial.compile.arch_dir` | `TactusBundleCreate` | Passed to `ecbundle create --arch-dir` |
| `ial.compile.bundle_file` | `TactusBundleCreate` | ECBundle YAML |
| `ial.compile.dir` | `TactusBundleCreate`, `TactusBundleBui Compiler used to build IAL (`intel` / `gnu` / `""`) |
| `ial.compile.install` | `TactusBundleBuild` | Whether to install into the shared `@INSTALL_DIR@` tree |

See the sections below for the full configuration surface of each task.

---

# Bundle Compilation Tasks

This module provides three compilation-related task classes:

* `IALClone` — clones the IAL (IFS/Arpege Library) Git repository
* `TactusBundleCreate` — creates or updates an ECBundle source bundle
* `TactusBundleBuild` — builds the bundle and optionally installs it into a shared, versioned location

These tasks are designed to work within the Tactus framework and use `ecbundle` for source management and compilation orchestration.

---

# Overview

The workflow is typically:

1. **Clone IAL repository** (optional)

   * Fetch IAL sources from a Git repository onto a local directory, then check out the configured version/branch
2. **Create/update the bundle**

   * Clone/update repositories defined in a bundle YAML
   * Optionally merge in local bundle overrides
3. **Build the bundle**

   * Configure and compile all sources
   * Install binaries either locally (per-experiment) or into a shared, versioned install tree

---

# Tasks

## `IALClone`

Clones the IAL Git repository into a local directory and checks out the configured version.

### Purpose

* Clones the IAL source repository
* Checks out the configured version/branch
* Skips cloning if the target directory already exists

### Configuration Keys

| Key                       | Description                                            |
| ------------------------- | ------------------------------------------------------ |
| `ial.ial_version`         | Version/branch checked out after cloning               |
| `ial.compile.ial_git_repo`| URL of the IAL Git repository (supports `[TOKEN]` placeholder) |
| `ial.compile.git_token`   | Git token substituted into `[TOKEN]` placeholder       |
| `ial.compile.ial_dir`     | Local destination directory for the IAL clone          |

### Token Substitution

If the repository URL contains the placeholder `[TOKEN]`, it is replaced with the value of `ial.compile.git_token` before cloning. This allows the token to be embedded into HTTPS Git URLs, e.g.:

```text
https://[TOKEN]@github.com/ecmwf/ial.git
```

becomes:

```text
https://<actual-token>@github.com/ecmwf/ial.git
```

### Behavior

* If `ial.compile.ial_dir` already exists: the clone step is skipped and an info message is logged.
* Otherwise the task clones the repository:

```bash
git clone <ial_git_repo> <ial_dir>
```

* In **either case**, the task then always runs a checkout of the configured version:

```bash
cd <ial_dir>; git checkout <ial_version>
```

> Unlike a "clone straight onto a branch" approach, the checkout step runs unconditionally — including on runs where the directory already existed from a previous task — so it also serves to move an existing checkout onto a newly requested `ial.ial_version`.

---

## `TactusBundleCreate`

Creates or updates an ECBundle source tree.

### Purpose

* Reads the configured bundle YAML
* Optionally merges in an update bundle YAML (for local IAL overrides or other customizations)
* Executes:

```bash
ecbundle create
```

---

### Configuration Keys

| Key                          | Description                                                                                  |
| ---------------------------- | -------------------------------------------------------------------------------------------- |
| `ial.compile.dir`            | Directory where bundle sources are created (defaults to `@CASEDIR@/bundle`)                  |
| `ial.compile.arch_dir`       | Directory where arch files can be found; passed to `ecbundle create --arch-dir`               |
| `ial.compile.git_token`      | Optional GitHub token                                                                        |
| `ial.compile.bundle_file`    | ECBundle YAML file (defaults to `@TACTUS_HOME@/data/compilation/@CYCLE@/bundle.yml`)         |
| `ial.compile.bundle_update`  | If `True`, merges an additional update YAML on top of the base bundle file                   |
| `ial.compile.update_bundle_file` | YAML file used to override/extend the base bundle when `ial.compile.bundle_update` is enabled |
| `ial.compile.ial_dir`        | Optional local IAL source override (exported as `IAL_DIR` environment variable)              |

---

### Bundle Update Mechanism

When `ial.compile.bundle_update` is enabled, the task:

1. Loads the original bundle YAML
2. Loads the update bundle YAML
3. Merges them via `merge_dicts(..., overwrite=True, remove_none=True)`:

   * `overwrite=True` — values in the update file override values in the original
   * `remove_none=True` — keys explicitly set to `None` in the update file are removed from the merged result
4. Writes the merged result to:

```text
@CASEDIR@/bundle-local-ial.yaml
```

YAML formatting is preserved using `ruamel.yaml` with:

* `preserve_quotes = True`
* indentation: mapping=4, sequence=4, offset=2
* line width: 4096

Example transformation:

#### Before (original)

```yaml
ial-source:
  git: github.com/ecmwf/ial.git
  version: 1.2.0
```

#### Update YAML

```yaml
ial-source:
  git: ~
  version: ~
  dir: /path/to/local/ial
```

#### After (merged)

```yaml
ial-source:
  dir: /path/to/local/ial
```

---

### IAL_DIR Environment Variable

Before invoking `ecbundle`, the task exports:

```bash
IAL_DIR=<substituted ial.compile.ial_dir>
```

This allows the bundle YAML to reference `${IAL_DIR}` for local IAL source overrides.

---

### Git Authentication

#### SSH Mode (default)

If no Git token is provided:

```python
os.environ["GITHUB"] = "git@github.com:"
```

Repositories are cloned using SSH access.

#### Token Mode

If `ial.compile.git_token` is set:

```bash
--github-token <TOKEN>
```

is passed to `ecbundle`.

> [!WARNING]
> Updating the remote repository while keeping the same branch/version may fail if the local branch is already tracking a different remote.
>
> Example error:
>
> ```text
> + git remote add eeec494cba20d4c7ae560cc38b7a8b14 git@github.com:/uandrae/IAL
> ERROR: Branch feature/toolchain-flags was already tracking origin/feature/toolchain-flags. Manual intervention needed.
> ERROR: Could not download or update ial-source ...
> ```
>
> This happens because Git refuses to change the upstream tracking configuration automatically when the branch already tracks another remote.
>
> In this case, remove the existing source directory or manually reconfigure the branch tracking before rerunning the bundle creation step.

---

### Generated Command

```bash
cd <compile_dir>; ecbundle create [--github-token <TOKEN>] --bundle <bundle_file> --update --arch-dir <arch_dir>
```

---

## `TactusBundleBuild`

Builds an ECBundle source tree, either as a local per-experiment install or into a shared, versioned install tree.

### Purpose

* Builds source repositories produced by `TactusBundleCreate`
* Supports multiple architectures and compilers
* Supports precision selection (`prec` / `R32`)
* Supports Ninja builds
* Supports clean rebuilds
* Optionally installs into a shared `@INSTALL_DIR@` tree with a `latest` pointer, instead of a purely local per-experiment install
* Backs up the resolved `bundle.yml` next to the build output

---

### Configuration Keys

| Key                     | Description                                                                                  |
| ----------------------- | ---------------------------------------------------------------------------------------------- |
| `ial.arch`              | Build architecture configuration (defaults to `@COMPILER@/default`)                            |
| `ial.compile.dir`       | Bundle source directory (output of `TactusBundleCreate`)                                       |
| `ial.compiler`          | Compiler used to build IAL: `intel`, `gnu`, or `""` (defaults to `intel`)                       |
| `ial.ial_version`       | Version/branch used to name the shared install location when `ial.compile.install` is enabled  |
| `ial.compile.ninja`     | Enable Ninja builds (defaults to `false`)                                                      |
| `ial.compile.skip_build`| Skip build if install already exists (defaults to `false`)                                     |
| `ial.compile.clean_build`| Clean build directory before compiling (defaults to `false`)                                  |
| `ial.compile.install`   | Install into the shared `@INSTALL_DIR@` tree instead of only locally (defaults to `false`)      |
| `task.args.prec`        | Precision selector: `prec` (double) or `R32` (single). Defaults to `prec`.                     |

---

### Precision Modes

| Precision | Effect                                            |
| --------- | ------------------------------------------------- |
| `prec`    | Default double-precision build                    |
| `R32`     | Adds `--without-double-precision` to ecbundle    |

---

### Install Modes

`TactusBundleBuild` supports two install layouts, controlled by `ial.compile.install`:

* **Local install** (`ial.compile.install = false`, the default): binaries are built straight into `@CASEDIR@/install/<precision>`. Nothing outside the case directory is touched.
* **Shared install** (`ial.compile.install = true`): binaries are installed into a shared, version-named tree under `@INSTALL_DIR@`, and the local `@CASEDIR@/install/<precision>` becomes a symlink pointing at that shared location. This lets multiple experiments/cases reuse the same compiled binaries for a given `ial.ial_version`.

#### Shared install layout

When `ial.compile.install` is enabled, the shared install root is:

```text
@INSTALL_DIR@/<ial_version>/<precision>/<compiler>
```

and the actual install directory adds an architecture-derived subpath on top (see **Install Subpath** below):

```text
@INSTALL_DIR@/<ial_version>/<precision>/<compiler>/<install_subpath>
```

A `latest` pointer is also maintained:

```text
@INSTALL_DIR@/latest -> @INSTALL_DIR@/<ial_version>
```

---

### Install Subpath

`get_install_subpath()` computes an additional path segment appended to the shared install root, so that different architecture/compiler combinations under the same bundle don't collide:

1. It resolves `<ial.compile.dir>/source/arch/<ial.arch>` — following it as a symlink if it is one.
2. It looks for `ial.compiler` among the components of the resolved path.
3. If found, everything **after** that component is returned as the subpath.
4. If `ial.compiler` does not appear anywhere in the resolved arch path, the method returns `None` — which will produce an invalid install path, so `ial.compiler` must correspond to an actual segment of the resolved `ial.arch` path.

---

### Build Directories

The builder creates:

```text
build/<precision>
```

under `@CASEDIR@`, regardless of install mode. The install directory (`exp_bindir`) depends on `ial.compile.install`:

```text
@CASEDIR@/install/<precision>                                    # local install
@INSTALL_DIR@/<ial_version>/<precision>/<compiler>/<install_subpath>   # shared install
```

---

### Bundle Backup

Before building, the task attempts to copy:

```text
<bundle_dir>/source/bundle.yml
```

to `@CASEDIR@/bundle.yml`. This preserves a snapshot of what was actually built. If the source file is missing, the failure is logged and execution continues.

---

### Pre-build Cleanup

If the build is not being skipped, any existing content at `@CASEDIR@/install/<precision>` is removed before building: it is unlinked if it's a symlink, or fully removed with `shutil.rmtree` if it's a real directory. This prevents a fresh build from silently mixing with a stale local install or a stale symlink from a previous run.

---

### Shared Install Symlinks (`make_install_arch_symlink`)

When `ial.compile.install` is enabled, after building the task:

1. Calls `make_install_arch_symlink()`: if the source bundle's arch directory (`<bundle_dir>/source/arch/<arch>`) has a `default` symlink, that same symlink (its target, not the resolved file) is copied into the shared install root as `<install_dir_root>/default`, replacing any existing `default` link there.
2. Refreshes the `@INSTALL_DIR@/latest` symlink to point at `@INSTALL_DIR@/<ial_version>`, removing a stale symlink first if one exists.
3. Symlinks the local per-experiment install path `@CASEDIR@/install/<precision>` to the shared `<install_dir>` computed above.

When `ial.compile.install` is disabled, none of this runs — the local install path *is* the real build output, with no shared-tree bookkeeping.

---

### Build Command

The build command assembled by the task is:

```bash
cd <bundle_dir>; ecbundle build \
  --arch <arch> \
  [--ninja] \
  --forecast-only \
  [--clean] \
  [--without-double-precision] \
  --install-dir=<install_dir> \
  --install \
  --build-dir=<build_dir>
```

Optional flags:

| Option                       | Trigger                        |
| ---------------------------- | ------------------------------- |
| `--ninja`                    | `ial.compile.ninja=True`        |
| `--clean`                    | `ial.compile.clean_build=True`  |
| `--without-double-precision` | `precision == "R32"`            |

> **Current status:** in the present code, the `ecbundle build` invocation itself is commented out inside `execute()` — the command string is still assembled but `batch_job.run(...)` is not called for it. Only the pre-build cleanup, and (when `ial.compile.install` is enabled) the shared-install symlink management, actually execute. This should be re-enabled before relying on this task to produce fresh binaries end-to-end.

---

### Skip Build Logic

If:

```python
ial.compile.skip_build == True
```

and:

```text
<install_dir>/bin/MASTERODB
```

exists, the build step is skipped. The shared-install symlink bookkeeping (when `ial.compile.install` is enabled) still runs regardless of `skip_build`, so the local install path is kept pointing at the right shared location.

---

# Notes

* This build path targets `ecbundle==2.5.0` (pinned in `pyproject.toml`)
* All compilation-related configuration lives under the nested `[ial]` / `[ial.compile]` sections — a flat `[compile]` schema with bundle-hash-based caching existed at one point but has been removed; update any config files or macro references using `compile.*` keys back to `ial.*` / `ial.compile.*`
* The `@COMPILER@` macro is sourced from `ial.compiler`, and `@ARCH@` from `ial.arch` (see `include/macros.toml`)
* Case names for the compilation suite use the `@IAL_VERSION@` macro (e.g. `IAL_develop_compile`), not a separate tag macro
* Host-specific overrides (e.g. `modifications/compile_atos_bologna.toml`) can set `submission.bindir`, a task-specific `MASTERODB` bindir override, and `ial.compile.arch_dir` for a given platform
* The `ecbundle` binary is resolved as `<python-bin-dir>/ecbundle`, i.e. it must be installed in the same environment as Tactus
* `IAL_DIR` is always exported from `ial.compile.ial_dir`, regardless of whether `bundle_update` is enabled, so bundle YAMLs can rely on it being set