# Build System Evolution: From Makefile to Multi-Platform CI/CD

## Overview

The build system is the backbone of any complex software project, and for a cross-platform browser like Camoufox, it represents one of the most critical technical challenges. Building a Firefox-based browser requires compiling millions of lines of C++, Rust, and JavaScript code, managing complex dependencies, cross-compiling for multiple platforms, and packaging everything into distributable archives.

This document chronicles the evolution of Camoufox's build infrastructure across **39 commits**, spanning from July 26, 2024 to January 2025. What began as a simple Makefile grew into a sophisticated multi-platform build system with Docker containerization, GitHub Actions CI/CD, automated packaging, and comprehensive developer tooling.

### Why Build Systems Matter for Browser Development

Building a Firefox-based browser is orders of magnitude more complex than typical software development:

1. **Scale**: Firefox contains ~20 million lines of code across C++, Rust, JavaScript, and Python
2. **Compilation Time**: A full build can take 45-90 minutes even on modern hardware
3. **Dependencies**: Requires LLVM/Clang, Rust toolchain, Python, Node.js, and platform-specific SDKs
4. **Cross-Platform**: Must support Linux, Windows, and macOS with different architectures (x86_64, ARM64, i686)
5. **Toolchain Complexity**: Requires specific versions of compilers and libraries to avoid breaking builds
6. **Disk Space**: Build artifacts can consume 30-50GB per target platform
7. **Memory Requirements**: Peak memory usage during linking can exceed 16GB

### Build System Goals

The Camoufox build system was designed with these objectives:

- **Reproducibility**: Same source code produces identical binaries across environments
- **Developer-Friendly**: Easy setup for new contributors with minimal manual steps
- **Automation**: CI/CD pipeline for building releases automatically
- **Multi-Platform**: Support building all targets from a single Linux host
- **Efficiency**: Leverage caching and incremental builds where possible
- **Debugging**: Tools for developers to iterate quickly on patches
- **Packaging**: Automated creation of distributable archives with all assets

## Commit Timeline Visualization

The 39 commits can be visualized in phases showing the evolution:

```
Timeline: July 2024 ────────────► January 2025

Phase 1: Foundation & Docker (July-Aug 2024)
├─ 5b6de88  Add Dockerfile & cleanup
├─ a48c414  Fix rustup not found in Docker
├─ dc63ec8  Clone & patch in same local repo
├─ b48e994  Diff crate fix with -U10
├─ 75bf5f3  Add _READY flag to avoid building unpatched src
├─ e899a38  Allow use of local ~/.mozbuild
└─ c7d634c  Makefile & dev script additions

Phase 2: Multi-Platform Build System (Aug 2024)
├─ 717aa9d  Fix JSON format failures
├─ bf24500  Allow multiple build targets (MAJOR)
├─ d191db2  Fix windows packaging vcredist
├─ 44573c1  Fix target font path on non-linux
├─ a96fed2  Add "Create new patch" to dev UI
├─ a58428b  Hotfix Windows launcher packaging
└─ 0448ea1  Launcher fixes for Windows

Phase 3: CI/CD Implementation (Aug 2024)
├─ ab42c65  Add appveyor pipeline
├─ c142266  Set default Python version in CI/CD
├─ a891914  Remove revert from dir command
├─ 40bcdc6  Migrate to GitHub Actions (MAJOR)
├─ 80a6572  Packaging & macos exec fixes
├─ 726fcb6  Fix .mozbuild caching
├─ 9a91b9f  Fix package "aria2c" -> "aria2"
├─ 33137af  Add workflow dispatch trigger
├─ 1d31bad  Merge gh-actions branch
├─ 73386d6  Add target arch matrix
├─ a766940  Fix build resetting workspace
└─ f78c5a2  Add rustc & fix launcher errors

Phase 4: Optimization & Refinement (Sep-Dec 2024)
├─ 9bb17f5  Bump upload/download artifact to v4
├─ bc2f591  Fix bootstrap
├─ 598a856  Fix outdated libstdc++ on i686
├─ b371928  Remove .mozbuild caching
├─ 2d26310  Fix rust issues
├─ bbe1cbe  Memory benchmark scripts via podman
├─ 3e524aa  Include jvv validator file
├─ 3b235c5  Pass secret to fetch command
├─ 1478854  Downgrade LLVM to 18
├─ 6ed2a63  LLVM & rust version mismatch bugfix
├─ 02729d6  Remove cross link time optimization
├─ 2f2af93  Update packager for tar.xz
├─ 9f6d55f  Extract inner tar in packager
├─ aeae79f  Add "unbusy" command
├─ 17b5284  Use shutil for cross-env moves
└─ ef6a545  Final Dockerfile fixes
```

## Commit Analysis: 39 Commits

### Phase 1: Foundation & Docker (July-August 2024)

#### Commit 1: `5b6de88` - Add Dockerfile & cleanup (July 26, 2024)

**Impact**: Established containerized build environment

This first infrastructure commit added Docker support to enable reproducible builds across different host systems. The Dockerfile created a standardized Ubuntu-based environment with all necessary build dependencies.

**Files Changed**: 8 files (+86, -18 lines)
- Added `Dockerfile` with Ubuntu base image
- Updated `Makefile` to support Docker builds
- Removed pre-built launcher binary (3.4MB)
- Renamed `window-hijacker.patch` to `viewport-hijacker.patch`

**The Initial Dockerfile**:
```dockerfile
FROM ubuntu:latest
WORKDIR /app
COPY . /app

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential make msitools wget unzip \
    python3 python3-dev python3-pip \
    git p7zip-full golang-go aria2 curl rsync \
    ca-certificates && update-ca-certificates

# Fetch Firefox & apply initial patches
RUN make setup-minimal && \
    make mozbootstrap && \
    mkdir -p /app/dist

VOLUME /root/.mozbuild
VOLUME /app/dist

ENTRYPOINT ["python3", "./multibuild.py"]
```

This Dockerfile included:
- All Mozilla build prerequisites (build-essential, msitools)
- Python tooling for build scripts
- Camoufox-specific tools (aria2 for downloads, 7z for archives)
- CA certificates for secure downloads
- Volume mounts for `.mozbuild` cache and output artifacts

**Why Docker?**

Docker solved several critical problems:
1. **Dependency Hell**: Different Linux distributions have different versions of LLVM, Rust, etc.
2. **Reproducibility**: Builds in the container are identical regardless of host OS
3. **CI/CD Ready**: Same Dockerfile can be used locally and in GitHub Actions
4. **Clean Environment**: No contamination from host system packages

**Technical Deep Dive - Build Volumes**:

The Docker setup uses two critical volumes:

1. `/root/.mozbuild` - Mozilla's build state directory (~5GB)
   - Contains downloaded toolchains (Clang, Rust, Node.js)
   - Caches compiled dependencies
   - Stores cross-compilation SDKs

2. `/app/dist` - Output directory for built packages
   - Populated after successful builds
   - Mounted to host for easy artifact retrieval

#### Commit 2: `a48c414` - Fix rustup not found in Docker (July 27, 2024)

**Impact**: Fixed Rust toolchain installation in containerized builds

**Problem**: The initial Docker image installed `rustc` via apt, but Firefox's build system requires `rustup` to manage multiple Rust versions and add compilation targets for cross-platform builds.

**Solution**: Added rustup installation via official installer:
```dockerfile
RUN curl https://sh.rustup.rs -sSf | bash -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"
```

**Why This Matters**:
- Firefox uses specific Rust versions that may not match system packages
- Cross-compilation requires adding targets like `aarch64-unknown-linux-gnu`
- Rustup manages toolchain updates automatically

**Files Changed**: `Dockerfile`

#### Commit 3: `dc63ec8` - Clone & patch in same local repo (July 28, 2024)

**Impact**: Unified build workflow to use a single source directory

**Before**: Build process would extract Firefox source to a temporary directory, apply patches, and copy to build location.

**After**: Extract once, initialize git repository, and track all changes in-place.

**New Workflow**:
```bash
# Extract Firefox source
tar -xJf firefox-128.0.source.tar.xz -C camoufox-128.0-1

# Initialize git for patch tracking
cd camoufox-128.0-1
git init -b main
git add -f -A
git commit -m "Initial commit"
git tag -a unpatched -m "Initial commit"

# Apply patches (all tracked in git)
patch -p1 < ../patches/fingerprint-injection.patch
git commit -m "Applied fingerprint injection"
```

**Benefits**:
1. **Patch Development**: Developers can use `git diff` to create new patches
2. **Revert Support**: Easy rollback with `git reset --hard unpatched`
3. **Interactive Development**: Make changes, test, commit, repeat
4. **Workspace Management**: Track which patches are applied

**Files Changed**: Makefile, patch.py

#### Commit 4: `b48e994` - Diff crate fix with -U10 (July 29, 2024)

**Impact**: Improved patch readability with more context lines

Changed `git diff` output format to include 10 lines of context (default is 3):
```bash
git diff -U10 > patch.diff
```

**Why More Context?**
- Helps understand where in the file changes occur
- Reduces patch application failures when line numbers shift
- Makes code review easier

**Files Changed**: Developer scripts

#### Commit 5: `75bf5f3` - Makefile: Add _READY flag to avoid building unpatched src (July 30, 2024)

**Impact**: Prevented accidental builds of unpatched Firefox

**Problem**: Running `make build` before `make dir` would attempt to build vanilla Firefox, wasting hours of compilation time.

**Solution**: Added a safety check:
```makefile
build: unbusy
	@if [ ! -f $(cf_source_dir)/_READY ]; then \
		make dir; \
	fi
	cd $(cf_source_dir) && ./mach build $(_ARGS)
```

The `_READY` file is created by `make dir` after all patches are successfully applied.

**Files Changed**: Makefile (+2, -1)

#### Commit 6: `e899a38` - Dockerfile: Allow use of local ~/.mozbuild, etc. (July 31, 2024)

**Impact**: Major improvement to Docker build caching

**Problem**: Every Docker build would re-download Mozilla toolchains (~5GB), taking 15-20 minutes even before compilation started.

**Solution**: Mount host's `.mozbuild` directory into container:
```bash
docker run -v ~/.mozbuild:/root/.mozbuild \
           -v $(pwd)/dist:/app/dist \
           camoufox-builder
```

**Build Time Improvement**:
- **First build**: ~90 minutes (download + compile)
- **Second build**: ~50 minutes (compile only, toolchains cached)

**Technical Details**:

The `.mozbuild` directory structure:
```
~/.mozbuild/
├── clang/           # LLVM/Clang toolchain (~2GB)
├── rust/            # Rust compiler and stdlib (~1GB)
├── node/            # Node.js for build scripts (~200MB)
├── nasm/            # Assembler for media codecs (~50MB)
├── dump_syms/       # Symbol dumper for crash reports (~100MB)
└── vs/              # Visual Studio libraries for Windows builds (~1.5GB)
```

**Files Changed**: Dockerfile (+23, -8), README.md, multibuild.py

#### Commit 7: `c7d634c` - Makefile & dev script additions (August 1, 2024)

**Impact**: Enhanced developer workflow with new commands

Added several new Makefile targets:
```makefile
edit-cfg:   # Edit camoufox.cfg in-place
	$(EDITOR) $(cf_source_dir)/obj-x86_64-pc-linux-gnu/dist/bin/camoufox.cfg

workspace:  # Set up workspace for editing a specific patch
	@make check-arg $(_ARGS);
	# Check if patch is applied, reverse if needed
	make checkpoint || true
	make patch $(_ARGS)

checkpoint: # Save current state
	cd $(cf_source_dir) && git commit -m "Checkpoint" -a -uno
```

Enhanced `scripts/developer.py` with:
- Patch status indicators (APPLIED, NOT APPLIED, BROKEN)
- Better error reporting when patches fail
- Workspace management for editing patches

**Files Changed**: Makefile (+8, -1), scripts/developer.py (+39, -12)

### Phase 2: Multi-Platform Build System (August 2024)

#### Commit 8: `717aa9d` - Fix JSON format failures when packaging (August 1, 2024)

**Impact**: Resolved packaging errors caused by malformed JSON

**Problem**: Search engine manifest had invalid JSON that passed Firefox's lenient parser but failed strict validation during packaging.

**Fixed File**: `additions/browser/components/search/extensions/none/manifest.json`

Before:
```json
{
  "name": "None",
  "description": "No search",
  "manifest_version": 2,
  "hidden": true,  // Invalid: trailing comma
}
```

After:
```json
{
  "name": "None",
  "description": "No search",
  "manifest_version": 2,
  "hidden": true
}
```

**Files Changed**: 1 file (+4, -2)

#### Commit 9: `bf24500` - multibuild: Allow multiple build targets, Makefile changes, more (August 1, 2024)

**Impact**: Revolutionary change enabling matrix builds across platforms and architectures

**Major Feature**: Multi-target build script

**New Architecture**:
```python
# Before: Build one target at a time
$ make build os=linux arch=x86_64

# After: Build multiple targets in one command
$ python3 multibuild.py \
    --target linux windows macos \
    --arch x86_64 arm64
```

The new `multibuild.py` implements a clean build management system:

```python
@dataclass
class BSYS:
    target: str
    arch: str

    def build(self):
        """Build the Camoufox source code"""
        os.environ['BUILD_TARGET'] = f'{self.target},{self.arch}'
        run('make build')

    def package(self):
        """Package the Camoufox source code"""
        run(f'make package-{self.target} arch={self.arch}')

    def update_target(self):
        """Change the build target"""
        os.environ['BUILD_TARGET'] = f'{self.target},{self.arch}'
        run('make set-target')
```

**Build Matrix**:
```
linux    × x86_64 ✓
linux    × arm64  ✓
linux    × i686   ✓
windows  × x86_64 ✓
windows  × arm64  ✗ (clang++-cl missing)
windows  × i686   ✓
macos    × x86_64 ✓
macos    × arm64  ✓
macos    × i686   ✗ (unsupported)
```

**New Makefile Target**:
```makefile
set-target:
	python3 scripts/patch.py $(version) $(release) --mozconfig-only
```

This updates only the `mozconfig` file without reapplying all patches, enabling quick target switches during development.

**Code Refactoring**:

Created `scripts/_mixin.py` with shared utilities:
```python
def find_src_dir(base_path, version, release):
    """Locate Camoufox source directory"""
    return f'camoufox-{version}-{release}'

def get_moz_target(target, arch):
    """Convert target/arch to Mozilla triplet"""
    moz_targets = {
        ('linux', 'x86_64'): 'x86_64-pc-linux-gnu',
        ('linux', 'arm64'):  'aarch64-unknown-linux-gnu',
        ('linux', 'i686'):   'i686-pc-linux-gnu',
        ('windows', 'x86_64'): 'x86_64-pc-windows-msvc',
        ('windows', 'i686'):   'i686-pc-windows-msvc',
        ('macos', 'x86_64'): 'x86_64-apple-darwin',
        ('macos', 'arm64'):  'aarch64-apple-darwin',
    }
    return moz_targets.get((target, arch))

def list_patches():
    """Get all patch files in order"""
    patches = sorted(glob.glob('../patches/*.patch'))
    return patches

def patch(patch_file, reverse=False, silent=False):
    """Apply or reverse a patch"""
    flag = '-R' if reverse else ''
    cmd = f'patch -p1 {flag} -i "{patch_file}"'
    if silent:
        cmd += ' > /dev/null 2>&1'
    os.system(cmd)
```

**Files Changed**: 8 files (+406, -278)

**Impact Summary**:
- Enabled building all 7 supported targets with one command
- Reduced code duplication across build scripts
- Improved error handling and progress reporting
- Made it possible to implement CI/CD matrix builds

#### Commit 10: `d191db2` - Fix windows packaging not finding vcredist (August 5, 2024)

**Impact**: Fixed Windows packaging by correctly locating Visual C++ redistributables

**Problem**: Windows builds require distributing MSVC runtime DLLs (`msvcp140.dll`, `vcruntime140.dll`) but the path was hardcoded.

**Solution**: Dynamic path resolution based on architecture:
```makefile
vcredist_arch := $(shell echo $(arch) | sed 's/x86_64/x64/' | sed 's/i686/x86/')

package-windows:
	python3 scripts/package.py windows \
		--includes \
			~/.mozbuild/vs/VC/Redist/MSVC/14.38.33135/$(vcredist_arch)/Microsoft.VC143.CRT/*.dll \
		--version $(version) --release $(release) --arch $(arch)
```

**Architecture Mapping**:
- `x86_64` → `x64` (MSVC uses x64, not x86_64)
- `i686` → `x86` (32-bit)
- `arm64` → `arm64` (same)

**Files Changed**: Makefile (+4, -2)

#### Commit 11: `44573c1` - Fix target font path on non-linux systems (August 6, 2024)

**Impact**: Fixed font bundling for Windows and macOS

**Problem**: Font bundling assumed Linux-style subdirectory structure, but Windows and macOS require flat font directories.

**Linux Font Structure**:
```
fonts/
├── windows/
│   ├── arial.ttf
│   ├── times.ttf
│   └── courier.ttf
├── macos/
│   ├── helvetica.ttf
│   └── menlo.ttf
└── linux/
    ├── dejavu.ttf
    └── liberation.ttf
```

**Windows/macOS Requirement** (flat structure):
```
fonts/
├── arial.ttf
├── times.ttf
├── courier.ttf
├── helvetica.ttf
└── menlo.ttf
```

**Fix in `scripts/package.py`**:
```python
# Linux: Copy folders as-is
if target == 'linux':
    for font in fonts or []:
        shutil.copytree(
            os.path.join('bundle', 'fonts', font),
            os.path.join(fonts_dir, font),
            dirs_exist_ok=True,
        )
# Non-Linux: Flatten folder structure
else:
    os.makedirs(fonts_dir, exist_ok=True)
    for font in fonts or []:
        for file in list_files(root_dir=os.path.join('bundle', 'fonts', font), suffix='*'):
            shutil.copy2(file, os.path.join(fonts_dir, os.path.basename(file)))
```

**Why Different?**

Windows and macOS font APIs expect fonts in a single directory for performance. Linux's fontconfig can efficiently handle nested directories.

**Files Changed**: scripts/package.py (+16, -8)

#### Commit 12: `a96fed2` - Add "Create new patch" to dev UI (August 5, 2024)

**Impact**: Streamlined patch creation workflow

Added workflow to `scripts/developer.py`:
```python
case "Create new patch":
    # Reset camoufox, apply all patches, then create a checkpoint
    reset_camoufox()
    with temp_cd('..'):
        run('make dir')
        run('make checkpoint')
    easygui.msgbox(
        "Created new patch workspace. You can test Camoufox with 'make run'.\n\n"
        "When you are finished, write your workspace back to a new patch.",
        "New Patch Workspace",
    )
```

**Workflow**:
1. Developer selects "Create new patch" in UI
2. System resets to clean state and applies all patches
3. Git checkpoint created
4. Developer makes changes, tests with `make run`
5. Developer selects "Write workspace to patch" to save

**Files Changed**: scripts/developer.py (+19, -1)

#### Commit 13: `a58428b` - Hotfix Windows launcher packaging (August 6, 2024)

**Impact**: Fixed Windows executable inclusion in packages

Corrected launcher filename for Windows in packaging script:
```python
# Windows launcher is "launch.exe", not "launch"
if target == 'windows':
    launcher_name = 'launch.exe'
else:
    launcher_name = 'launch'
```

**Files Changed**: scripts/package.py (+3, -2)

#### Commit 14: `0448ea1` - Launcher fixes for Windows (August 7, 2024)

**Impact**: Fixed Windows-specific launcher issues

**Changes**:
1. **Go Module Updates**: Added Windows-specific dependencies
   ```go
   require (
       github.com/shirou/gopsutil/v3 v3.21.11
       golang.org/x/sys v0.0.0-20211216021012-1d35b9e2eb4e
   )
   ```

2. **MaskConfig Path Handling**: Fixed Windows path parsing in C++
   ```cpp
   // Handle Windows paths with backslashes
   std::replace(path.begin(), path.end(), '\\', '/');
   ```

3. **Launcher Process Management**: Improved Windows process group handling
   ```go
   // Windows requires different process creation flags
   cmd.SysProcAttr = &syscall.SysProcAttr{
       CreationFlags: syscall.CREATE_NEW_PROCESS_GROUP,
   }
   ```

**Files Changed**: 5 files (+85, -9)

### Phase 3: CI/CD Implementation (August 2024)

#### Commit 15: `ab42c65` - Add appveyor pipeline (August 13, 2024)

**Impact**: First CI/CD implementation using AppVeyor

**Initial CI Configuration** (`appveyor.yml`):
```yaml
version: 1.0.{build}
image: Ubuntu2004

install:
  - sudo apt-get update
  - sudo apt-get install -y python3 aria2 p7zip-full

build_script:
  - make fetch
  - make setup-minimal
  - make mozbootstrap
  - python3 multibuild.py --target linux --arch x86_64

artifacts:
  - path: dist/*
    name: CamoufoxBuilds

deploy:
  provider: GitHub
  auth_token:
    secure: $(GITHUB_TOKEN)
  on:
    APPVEYOR_REPO_TAG: true
```

**Why AppVeyor First?**
- Free for open-source projects
- Native Linux support
- Simple configuration
- Integrated artifact hosting

**Limitations Discovered**:
- 60-minute time limit per build
- Limited disk space (20GB)
- No cache persistence between builds
- Slow VM provisioning

**Files Changed**: 1 file (+38)

#### Commit 16: `c142266` - Set default Python version in CI/CD (August 14, 2024)

**Impact**: Fixed Python version conflicts

AppVeyor's Ubuntu 20.04 image had multiple Python versions. Forced Python 3.8+:
```yaml
install:
  - sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.8 1
```

**Files Changed**: appveyor.yml

#### Commit 17: `a891914` - Makefile: Remove revert from dir command (August 14, 2024)

**Impact**: Fixed workflow for editing patches

**Problem**: `make dir` would run `make revert`, destroying uncommitted work.

**Solution**: Removed revert from `dir` target. Developers must explicitly run `make revert` when needed.

```makefile
# Before:
dir:
	make revert
	python3 scripts/patch.py $(version) $(release)

# After:
dir:
	@if [ ! -d $(cf_source_dir) ]; then \
		make setup; \
	fi
	python3 scripts/patch.py $(version) $(release)
	touch $(cf_source_dir)/_READY
```

**Files Changed**: Makefile

#### Commit 18: `40bcdc6` - CI/CD: Migrate from Appveyor to Github Actions (August 14, 2024)

**Impact**: Major CI/CD upgrade with better performance and flexibility

**Why Migrate?**
1. **Time Limits**: AppVeyor's 60-minute limit was insufficient for full builds (~75 minutes)
2. **Storage**: GitHub Actions provides 100GB disk space vs AppVeyor's 20GB
3. **Caching**: Better cache management with actions/cache
4. **Matrix Builds**: Native support for building multiple targets in parallel
5. **Integration**: Better integration with GitHub releases

**New Workflow** (`.github/workflows/build.yml`):
```yaml
name: Build and Release

on:
  workflow_dispatch:
  push:
    tags:
      - "*"

jobs:
  build:
    runs-on: ubuntu-24.04
    strategy:
      matrix:
        target: [linux, windows, macos]
        arch: [x86_64, arm64, i686]
        exclude:
          - target: windows
            arch: arm64
          - target: macos
            arch: i686

    steps:
      - name: Maximize build space
        uses: AdityaGarg8/remove-unwanted-software@v4.1
        with:
          remove-dotnet: "true"
          remove-android: "true"
          remove-haskell: "true"
          remove-codeql: "true"
          remove-docker-images: "true"

      - uses: actions/checkout@v2

      - name: Set up Go
        uses: actions/setup-go@v2
        with:
          go-version: "1.23"

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: "3.11"

      - name: Set up LLVM
        run: |
          wget https://apt.llvm.org/llvm.sh
          chmod +x llvm.sh
          sudo ./llvm.sh 18

      - name: Fetch source
        env:
          CAMOUFOX_PASSWD: ${{ secrets.CAMOUFOX_PASSWD }}
        run: make fetch

      - name: Setup and bootstrap
        run: |
          make setup-minimal
          make mozbootstrap

      - name: Create swap space
        run: |
          sudo fallocate -l 16G /swapfile
          sudo chmod 600 /swapfile
          sudo mkswap /swapfile
          sudo swapon /swapfile

      - name: Build
        run: python3 ./multibuild.py --target ${{ matrix.target }} --arch ${{ matrix.arch }}

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: CamoufoxBuilds-${{ matrix.target }}-${{ matrix.arch }}
          path: dist/*

  release:
    needs: build
    permissions:
      contents: write
    runs-on: ubuntu-latest

    steps:
      - name: Download all artifacts
        uses: actions/download-artifact@v4
        with:
          path: artifacts

      - name: Create Release
        uses: softprops/action-gh-release@v1
        if: startsWith(github.ref, 'refs/tags/')
        with:
          files: artifacts/**/*
          generate_release_notes: true
          draft: true
```

**Key Features**:

1. **Matrix Builds**: Parallel builds for 7 targets (Linux x86_64/arm64/i686, Windows x86_64/i686, macOS x86_64/arm64)

2. **Disk Space Management**: Removes ~40GB of unused software before build

3. **Swap Space**: Adds 16GB swap to handle linker memory requirements

4. **Modular Setup**: Separate steps for toolchain installation, source fetch, and build

5. **Automated Releases**: Creates GitHub releases with all artifacts on tag push

**Performance Comparison**:

| Platform | AppVeyor | GitHub Actions | Improvement |
|----------|----------|----------------|-------------|
| Build Time | 75 min (timeout) | 60 min | 20% faster |
| Disk Space | 20GB | 100GB | 5x more |
| Parallel Jobs | 1 | 7 | 7x parallelism |
| Cache Size | None | 10GB/repo | ∞ better |

**Files Changed**: 4 files (+80, -41)

#### Commit 19: `80a6572` - Packaging & macos exec fixes (August 14, 2024)

**Impact**: Fixed macOS app bundle structure and executable permissions

**macOS Application Bundle Structure**:
```
Camoufox.app/
└── Contents/
    ├── Info.plist
    ├── MacOS/
    │   └── camoufox-bin    (executable)
    └── Resources/
        ├── browser/
        ├── fonts/
        ├── chrome.css
        └── camoufox.cfg
```

**Packaging Fix**:
```python
if target == 'macos':
    # Move Camoufox/Camoufox.app -> Camoufox.app
    nightly_dir = os.path.join(temp_dir, 'Camoufox')
    shutil.move(
        os.path.join(nightly_dir, 'Camoufox.app'),
        os.path.join(temp_dir, 'Camoufox.app')
    )
    # Remove old app dir
    shutil.rmtree(nightly_dir)

    # Set Resources as target directory
    target_dir = os.path.join(temp_dir, 'Camoufox.app', 'Contents', 'Resources')
```

**Executable Permissions**:
```bash
# Ensure launcher has execute permissions
chmod +x Camoufox.app/Contents/MacOS/camoufox-bin
```

**Files Changed**: scripts/package.py

#### Commit 20: `726fcb6` - CI/CD: Fix .mozbuild caching (August 14, 2024)

**Impact**: Attempted to add caching for Mozilla build tools

**Attempted Configuration**:
```yaml
- name: Cache .mozbuild
  uses: actions/cache@v3
  with:
    path: ~/.mozbuild
    key: mozbuild-${{ runner.os }}-${{ hashFiles('upstream.sh') }}
```

**Problem**: Cache became too large (>10GB) and wasn't saving properly

**Later Reverted**: See commit `b371928`

**Files Changed**: .github/workflows/build.yml (+8, -2)

#### Commit 21: `9a91b9f` - Fix package "aria2c" -> "aria2" (August 14, 2024)

**Impact**: Fixed package name typo in dependencies

Ubuntu package name is `aria2`, not `aria2c` (the executable is `aria2c` but package is `aria2`):

```makefile
debs := python3 python3-dev python3-pip p7zip-full golang-go msitools wget aria2
```

**Files Changed**: Makefile

#### Commit 22: `33137af` - CI/CD: Add workflow dispatch trigger (August 14, 2024)

**Impact**: Enabled manual CI/CD runs

Added `workflow_dispatch` to allow triggering builds manually from GitHub UI:
```yaml
on:
  workflow_dispatch:  # Add manual trigger
  push:
    tags:
      - "*"
```

**Usage**:
1. Go to Actions tab on GitHub
2. Select "Build and Release" workflow
3. Click "Run workflow"
4. Select branch
5. Click "Run workflow" button

**Files Changed**: .github/workflows/build.yml

#### Commit 23: `1d31bad` - Merge gh-actions branch into main (August 15, 2024)

**Impact**: Finalized GitHub Actions migration

Merged experimental `gh-actions` branch into main after successful testing. Removed AppVeyor completely.

**Files Changed**: Multiple (merge commit)

#### Commit 24: `73386d6` - CI/CD: Add target arch matrix (August 16, 2024)

**Impact**: Properly configured matrix to handle target/arch combinations

**Improved Matrix Configuration**:
```yaml
strategy:
  matrix:
    target: [linux, windows, macos]
    arch: [x86_64, arm64, i686]
    exclude:
      # Windows ARM64 builds fail (.mozbuild does not include clang++-cl)
      - target: windows
        arch: arm64
      # macOS i686 is unsupported
      - target: macos
        arch: i686
```

This creates 7 parallel jobs:
1. linux-x86_64
2. linux-arm64
3. linux-i686
4. windows-x86_64
5. windows-i686
6. macos-x86_64
7. macos-arm64

**Files Changed**: .github/workflows/build.yml (+13, -8), multibuild.py (+3)

#### Commit 25: `a766940` - Makefile: Fix `build` resetting workspace when editing patch (August 17, 2024)

**Impact**: Fixed accidental workspace resets during development

**Problem**: The `build` target would call `make dir`, which would reapply all patches, destroying uncommitted changes.

**Solution**: Only run `make dir` if `_READY` flag is missing:
```makefile
build: unbusy
	@if [ ! -f $(cf_source_dir)/_READY ]; then \
		make dir; \
	fi
	cd $(cf_source_dir) && ./mach build $(_ARGS)
```

**Files Changed**: Makefile

#### Commit 26: `f78c5a2` - Dockerfile: Add rustc & fix launcher errors (August 18, 2024)

**Impact**: Improved Docker image with better Rust support

Added both system `rustc` (for quick scripts) and `rustup` (for Firefox builds):
```dockerfile
RUN apt-get update && apt-get install -y \
    build-essential make msitools wget unzip rustc \
    ...

RUN curl https://sh.rustup.rs -sSf | bash -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"
```

**Why Both?**
- System `rustc`: Fast installation, good for launcher builds
- Rustup: Required for Firefox compilation and cross-compilation targets

**Files Changed**: Dockerfile

### Phase 4: Optimization & Refinement (September-December 2024)

#### Commit 27: `9bb17f5` - CI/CD: Bump upload/download artifact to v4 (September 11, 2024)

**Impact**: Updated to faster artifact handling

GitHub Actions v4 artifacts use new API with better performance:
```yaml
# Before (v3):
- uses: actions/upload-artifact@v3

# After (v4):
- uses: actions/upload-artifact@v4
```

**Improvements in v4**:
- Faster uploads (parallel chunk uploading)
- Better compression
- Reduced storage costs
- Automatic cleanup of old artifacts

**Files Changed**: .github/workflows/build.yml (+2, -2)

#### Commit 28: `bc2f591` - LibreWolf: Fix bootstrap (September 12, 2024)

**Impact**: Fixed bootstrap script compatibility

Updated `scripts/bootstrap.py` to work with LibreWolf-specific modifications. This script is from Firefox upstream and needed adjustments for Camoufox's build process.

**Files Changed**: scripts/bootstrap.py

#### Commit 29: `598a856` - CI/CD: Fix outdated libstdc++ on i686 (September 11, 2024)

**Impact**: Fixed 32-bit build failures

**Problem**: i686 cross-compilation requires 32-bit versions of C++ standard library.

**Solution**:
```yaml
- name: Set up LLVM
  run: |
    sudo ./llvm.sh 18
    if [ "${{ matrix.arch }}" != "x86_64" ]; then
      sudo apt-get install -y libc6-i386 lib32gcc-s1 lib32stdc++6 gcc-multilib g++-multilib
    fi
```

**Required Packages**:
- `libc6-i386`: 32-bit C standard library
- `lib32gcc-s1`: 32-bit GCC runtime
- `lib32stdc++6`: 32-bit C++ standard library
- `gcc-multilib`, `g++-multilib`: Multi-architecture compiler support

**Files Changed**: .github/workflows/build.yml (+1, -1)

#### Commit 30: `b371928` - CI/CD: Remove .mozbuild caching (September 30, 2024)

**Impact**: Removed problematic cache

**Problem**:
- Cache grew too large (>10GB)
- Cache restore was slower than re-downloading
- Cache invalidation caused issues

**Solution**: Just re-download toolchains every time. Download takes ~10 minutes but is reliable.

**Trade-off Analysis**:
- **With Cache**: 65 min build (5 min restore + 60 min compile)
- **Without Cache**: 70 min build (10 min download + 60 min compile)
- **Reliability**: Without cache is 100% reliable

**Files Changed**: .github/workflows/build.yml (-10 lines)

#### Commit 31: `2d26310` - Dockerfile: Fix rust issues (October 3, 2024)

**Impact**: Fixed Rust toolchain conflicts in Docker

**Problem**: System rustc and rustup versions conflicted

**Solution**: Use only rustup, remove system rustc:
```dockerfile
# Remove system rustc installation
RUN apt-get update && apt-get install -y \
    build-essential make msitools wget unzip \
    # Removed: rustc
    python3 python3-dev python3-pip \
    ...

# Use rustup exclusively
RUN curl https://sh.rustup.rs -sSf | bash -s -- -y
```

**Files Changed**: Dockerfile

#### Commit 32: `bbe1cbe` - Memory benchmark scripts via podman (October 15, 2024)

**Impact**: Added containerized benchmarking system

Created comprehensive benchmark suite for memory profiling:

**Directory Structure**:
```
scripts/benchmark/
├── Dockerfile
├── Makefile
└── benchmark.py
```

**Benchmark Dockerfile**:
```dockerfile
FROM debian:latest
WORKDIR /app

# Install Python and essential tools
RUN apt-get update && apt-get install -y \
    python3 python3-pip curl wget xvfb \
    && apt-get clean

# Install Playwright and Camoufox
RUN pip3 install playwright tabulate camoufox[geoip] --break-system-packages && \
    playwright install-deps && \
    playwright install firefox && \
    python3 -m camoufox fetch

COPY benchmark.py /app/
ENTRYPOINT ["python3", "/app/benchmark.py"]
```

**Benchmark Script** (`benchmark.py`):
```python
def get_firefox_memory(name):
    """Get total memory usage of all processes"""
    result = subprocess.run(["ps", "-C", name, "-o", "rss="],
                           capture_output=True, text=True)
    memory_kb = sum(int(line.strip()) for line in result.stdout.splitlines())
    return memory_kb / 1024  # Convert to MB

def run_playwright(mode, browser_name):
    headless = mode == "headless"
    memory_usage = []

    # Set up virtual display for headless testing
    virt = VirtualDisplay()

    for url in urls:
        page = browser.new_page()
        page.goto(url)
        time.sleep(5)

        # Measure memory over 10 seconds
        memory = get_average_memory(
            name="camoufox-bin",
            duration=10
        )
        memory_usage.append((url, memory))

    return memory_usage
```

**Benchmark Makefile**:
```makefile
.PHONY: build run clean

build:
	podman build -t camoufox-bench .

run:
	podman run --rm camoufox-bench \
		--mode headless \
		--browser camoufox

clean:
	podman rmi camoufox-bench
```

**Usage**:
```bash
cd scripts/benchmark
make build
make run MODE=headless BROWSER=camoufox
```

**Output**:
```
=== MEMORY RESULTS FOR CAMOUFOX ===
+-------------------+-------------------+
| URL               | Memory Usage (MB) |
+===================+===================+
| about:blank       |           285.3   |
+-------------------+-------------------+
| https://google.com|           412.7   |
+-------------------+-------------------+
| https://yahoo.com |           523.1   |
+-------------------+-------------------+
```

**Files Changed**: 3 files added (scripts/benchmark/*)

#### Commit 33: `3e524aa` - Include jvv validator file in packaging (October 20, 2024)

**Impact**: Added JSONvv validator to packages

Ensured `settings/camoucfg.jvv` (JSON validation schema) is included in all packages:
```makefile
package-linux:
	python3 scripts/package.py linux \
		--includes \
			settings/chrome.css \
			settings/camoucfg.jvv \
			settings/properties.json \
		...
```

**Why Needed**: The Python library uses this schema file to validate configuration JSON before passing to browser.

**Files Changed**: Makefile

#### Commit 34: `3b235c5` - CI/CD: Pass secret to fetch command (October 25, 2024)

**Impact**: Enabled private patch fetching in CI/CD

Added secret for downloading proprietary patches:
```yaml
- name: Fetch source
  env:
    CAMOUFOX_PASSWD: ${{ secrets.CAMOUFOX_PASSWD }}
  run: make fetch
```

**How It Works**:
```makefile
fetch:
	# Fetching private patches...
	@if [ -z "$$CAMOUFOX_PASSWD" ]; then \
		echo "Skipping private patches..."; \
	else \
		aria2c -o rev-$(closedsrc_rev).7z \
			"https://camoufox.com/pipeline/rev-$(closedsrc_rev).7z" && \
		7z x -p"$$CAMOUFOX_PASSWD" rev-$(closedsrc_rev).7z && \
		rm rev-$(closedsrc_rev).7z; \
	fi
```

**Files Changed**: .github/workflows/build.yml, Makefile

#### Commit 35: `1478854` - CI/CD: Downgrade LLVM to 18 (November 15, 2024)

**Impact**: Fixed compiler version incompatibility

**Problem**: Firefox 133+ has issues with LLVM 19. Requires LLVM 18.

**Solution**:
```yaml
- name: Set up LLVM
  run: |
    wget https://apt.llvm.org/llvm.sh
    chmod +x llvm.sh
    sudo ./llvm.sh 18  # Force version 18
    sudo apt-get install -y lld-18 clang-18
    sudo update-alternatives --install /usr/bin/ld.lld ld.lld /usr/bin/ld.lld-18 100
```

**Files Changed**: .github/workflows/build.yml

#### Commit 36: `6ed2a63` - CI/CD: LLVM & rust version mismatch bugfix (November 16, 2024)

**Impact**: Fixed Rust/LLVM version synchronization

Ensured Rust and LLVM versions are compatible. Firefox requires specific version combinations.

**Files Changed**: .github/workflows/build.yml

#### Commit 37: `02729d6` - Remove cross link time optimization (November 18, 2024)

**Impact**: Disabled LTO to reduce build time

**Change in `assets/base.mozconfig`**:
```diff
-ac_add_options --enable-lto=cross
+# ac_add_options --enable-lto=cross
```

**Trade-off**:
- **With LTO**: ~90 min build, 5-10% smaller binaries, slightly faster runtime
- **Without LTO**: ~60 min build, larger binaries, acceptable performance

**Decision**: Faster builds more valuable than marginal optimizations.

**Files Changed**: assets/base.mozconfig

#### Commit 38: `2f2af93` - Update packager to search for tar.xz (December 2024)

**Impact**: Fixed packaging for Firefox 135+

Firefox changed package format from `.tar.bz2` to `.tar.xz`:
```python
# Updated search pattern
search_path = os.path.abspath(
    f'obj-{moz_target}/dist/camoufox-{args.version}-{args.release}.*.tar.xz'
)
```

**Files Changed**: scripts/package.py

#### Commit 39: `9f6d55f` - Extract inner tar in packager (December 2024)

**Impact**: Fixed nested tar.xz extraction

`.tar.xz` files are actually `.tar` files compressed with xz. Need double extraction:
```python
if package_file.endswith('.tar.xz'):
    # Rerun on the tar file
    package_tar = package_file[:-3]  # Remove ".xz"
    return add_includes_to_package(
        package_file=os.path.join(temp_dir, package_tar),
        includes=includes,
        fonts=fonts,
        new_file=new_file,
        target=target,
    )
```

**Files Changed**: scripts/package.py

#### Commit 40: `aeae79f` - Makefile: Add "unbusy" command (January 2025)

**Impact**: Added command to unlock locked files

**Problem**: On build failures, Firefox leaves lock files that prevent rebuilds.

**Solution**:
```makefile
unbusy:
	rm -rf $(cf_source_dir)/obj-x86_64-pc-linux-gnu/dist/bin/camoufox-bin \
		$(cf_source_dir)/obj-x86_64-pc-linux-gnu/dist/bin/camoufox \
		$(cf_source_dir)/obj-x86_64-pc-linux-gnu/dist/bin/launch
```

**Usage**:
```bash
# Build fails with "file is busy" errors
make unbusy
make build
```

**Files Changed**: Makefile

#### Commit 41: `17b5284` - Use shutil to move files (supports cross-env moves) (January 2025)

**Impact**: Fixed cross-device file moves

**Problem**: `os.rename()` fails when source and destination are on different filesystems/partitions.

**Solution**: Use `shutil.move()` which copies+deletes when needed:
```python
# Before:
os.rename(source, dest)

# After:
shutil.move(source, dest)
```

**Files Changed**: multibuild.py, scripts/package.py

#### Commit 42: `ef6a545` - Dockerfile fixes (January 2025)

**Impact**: Final Dockerfile refinements

Latest improvements to Docker build environment:
- Better layer caching
- Reduced image size
- Improved build reliability

**Files Changed**: Dockerfile

## Build System Architecture Deep Dive

### The Three-Layer Build Architecture

The Camoufox build system uses a three-layer architecture:

```
┌─────────────────────────────────────────────────────────┐
│ Layer 3: User Interface                                 │
│ - make targets (Makefile)                               │
│ - multibuild.py CLI                                     │
│ - developer.py GUI                                      │
│ - GitHub Actions workflows                              │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Layer 2: Build Orchestration                            │
│ - scripts/patch.py (applies patches)                    │
│ - scripts/package.py (creates packages)                 │
│ - scripts/_mixin.py (shared utilities)                  │
│ - upstream.sh (version configuration)                   │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Layer 1: Firefox Build System                           │
│ - mach (Mozilla's build tool)                           │
│ - mozconfig (build configuration)                       │
│ - configure.py (platform detection)                     │
│ - make/ninja (actual compilation)                       │
└─────────────────────────────────────────────────────────┘
```

#### Layer 1: Firefox Build System (Mozilla's Tools)

Mozilla provides a complete build system with `mach`:

**What mach does**:
1. **Environment Detection**: Detects OS, architecture, available tools
2. **Dependency Management**: Downloads toolchains to `.mozbuild`
3. **Configuration**: Generates build files from mozconfig
4. **Compilation**: Orchestrates C++/Rust/JS compilation
5. **Linking**: Links everything into `libxul.so` (the main Firefox library)
6. **Packaging**: Creates distributable archives

**Key mach commands used by Camoufox**:
```bash
./mach bootstrap          # Set up build environment
./mach build             # Compile everything
./mach package           # Create archive
./mach run               # Run the browser
./mach clobber           # Clean build artifacts
```

#### Layer 2: Build Orchestration (Camoufox Scripts)

Camoufox's Python scripts wrap Mozilla's build system:

**scripts/patch.py**:
- Reads `upstream.sh` for version info
- Applies all patches in order
- Configures mozconfig for target platform
- Adds Rust cross-compilation targets

**scripts/package.py**:
- Runs `./mach package`
- Extracts the archive
- Adds fonts, assets, launcher
- Repackages with 7zip

**scripts/_mixin.py**:
- Shared utilities for all scripts
- Target/architecture translation
- File finding and path manipulation
- Subprocess execution helpers

#### Layer 3: User Interface (High-Level Commands)

**Makefile**: Provides memorable commands for developers
```makefile
make build    # Actually runs: cd $(cf_source_dir) && ./mach build
make package  # Actually runs: python3 scripts/package.py ...
make run      # Actually runs: cd $(cf_source_dir) && ./mach run
```

**multibuild.py**: Enables matrix builds
```python
# Build 7 targets with one command
python3 multibuild.py --target linux windows macos --arch x86_64 arm64
```

**GitHub Actions**: Automates releases
```yaml
# Triggered by git tag, builds all targets, creates release
```

### Build Flow: From Source to Package

Let's trace a complete build from start to finish:

```
┌─────────────────────────────────────────────────────────┐
│ 1. User runs: make build arch=x86_64 os=linux           │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Makefile sets BUILD_TARGET=linux,x86_64               │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Checks for _READY flag, runs 'make dir' if missing   │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 4. scripts/patch.py reads BUILD_TARGET                   │
│    - Generates mozconfig for x86_64-pc-linux-gnu         │
│    - Adds rustup target aarch64-unknown-linux-gnu        │
│    - Applies all 20+ patches to source                   │
│    - Creates _READY flag file                            │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 5. cd into camoufox-135.0.1-beta.24/                     │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 6. ./mach build                                          │
│    ├─ Read mozconfig                                     │
│    ├─ Run configure.py (detect platform)                 │
│    ├─ Generate build.ninja                               │
│    ├─ ninja compile (parallel compilation)               │
│    │   ├─ Compile 500+ C++ files (~40 min)               │
│    │   ├─ Compile 200+ Rust crates (~10 min)             │
│    │   ├─ Process 100+ JS files (~2 min)                 │
│    │   └─ Link libxul.so (~5 min, high memory)           │
│    └─ Create omnijar (pack JS into single file)          │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 7. Build complete! Binary at:                            │
│    obj-x86_64-pc-linux-gnu/dist/bin/camoufox-bin         │
└─────────────────────────────────────────────────────────┘
```

**Now packaging**:

```
┌─────────────────────────────────────────────────────────┐
│ 1. User runs: make package-linux arch=x86_64             │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 2. python3 scripts/package.py linux --arch x86_64        │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 3. cd camoufox-135.0.1-beta.24/ && ./mach package        │
│    Creates: obj-.../dist/camoufox-135.0.1-beta.24.tar.xz │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 4. Extract tar.xz to temporary directory                 │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 5. Add Camoufox-specific files:                          │
│    ├─ Copy settings/chrome.css                           │
│    ├─ Copy settings/camoucfg.jvv                         │
│    ├─ Copy settings/properties.json                      │
│    ├─ Copy bundle/fonts/windows/ to fonts/windows/       │
│    ├─ Copy bundle/fonts/macos/ to fonts/macos/           │
│    ├─ Copy bundle/fonts/linux/ to fonts/linux/           │
│    └─ Copy bundle/fontconfigs/                           │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 6. Remove unneeded files:                                │
│    ├─ uninstall/                                         │
│    ├─ pingsender*                                        │
│    └─ vaapitest, glxtest                                 │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 7. Repackage with 7zip (max compression):                │
│    7z u camoufox-135.0.1-beta.24-lin.x86_64.zip -r -mx=9 │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 8. Final package ready! (~100MB)                         │
│    camoufox-135.0.1-beta.24-lin.x86_64.zip               │
└─────────────────────────────────────────────────────────┘
```

### Dependency Resolution and Version Pinning

**Version Constraints**:

| Dependency | Version | Reason |
|------------|---------|--------|
| LLVM/Clang | 18.x | Firefox 133+ requires LLVM 18 (not 19) |
| Rust | 1.75+ | Mozilla specifies minimum Rust version |
| Python | 3.8-3.11 | Mach requires Python 3.8+, CI uses 3.11 |
| Go | 1.23+ | Launcher uses modern Go features |
| Node.js | 18.x | Downloaded by Mozilla automatically |

**Why Version Pinning Matters**:

Firefox is extremely sensitive to toolchain versions. Examples of breakage:

1. **LLVM 19**: Introduced code generation changes that break Firefox's JIT compiler
2. **Rust 1.74**: Missing APIs that Firefox depends on
3. **Python 3.12**: Changed behavior of `distutils` causing mach failures
4. **Node.js 20**: Module resolution changes break build scripts

The build system explicitly installs correct versions:

```yaml
# .github/workflows/build.yml
- name: Set up Python
  uses: actions/setup-python@v2
  with:
    python-version: "3.11"  # Pinned!

- name: Set up LLVM
  run: |
    sudo ./llvm.sh 18  # Pinned!
```

## Build System Components

### 1. Makefile Architecture

The Makefile is the central orchestrator of the build system. It provides ~40 targets for different build operations.

#### Design Philosophy

The Makefile follows these principles:

1. **Idempotent Operations**: Running the same command twice is safe
2. **Smart Defaults**: `make build` does the right thing without arguments
3. **Fail-Fast**: Errors stop execution immediately
4. **Self-Documenting**: `make help` lists all targets
5. **Composable**: Targets can be chained (e.g., `make clean build package`)

#### Core Build Targets

**`fetch`** - Download Firefox source and private patches:
```makefile
fetch:
	# Fetching private patches...
	@if [ -d "patches/private" ]; then \
		echo "Found patches/private. Skipping..."; \
	else \
		if [ -z "$$CAMOUFOX_PASSWD" ]; then \
			echo "Skipping private patches..."; \
		else \
			aria2c -o rev-$(closedsrc_rev).7z \
				"https://camoufox.com/pipeline/rev-$(closedsrc_rev).7z" && \
			7z x -p"$$CAMOUFOX_PASSWD" rev-$(closedsrc_rev).7z && \
			rm rev-$(closedsrc_rev).7z; \
		fi; \
	fi
	# Fetching Firefox source...
	aria2c -x16 -s16 -k1M -o $(ff_source_tarball) \
		"https://archive.mozilla.org/pub/firefox/releases/$(version)/source/firefox-$(version).source.tar.xz"
```

**`setup`** - Extract Firefox source and initialize git repo:
```makefile
setup: setup-minimal
	# Initialize local git repo for development
	cd $(cf_source_dir) && \
		git init -b main && \
		git add -f -A && \
		git commit -m "Initial commit" && \
		git tag -a unpatched -m "Initial commit"
```

**`dir`** - Apply all patches:
```makefile
dir:
	@if [ ! -d $(cf_source_dir) ]; then \
		make setup; \
	fi
	python3 scripts/patch.py $(version) $(release)
	touch $(cf_source_dir)/_READY
```

**`build`** - Compile Camoufox:
```makefile
build: unbusy
	@if [ ! -f $(cf_source_dir)/_READY ]; then \
		make dir; \
	fi
	cd $(cf_source_dir) && ./mach build $(_ARGS)
```

**`package-linux`** - Create Linux package:
```makefile
package-linux:
	python3 scripts/package.py linux \
		--includes \
			settings/chrome.css \
			settings/camoucfg.jvv \
			settings/properties.json \
			bundle/fontconfigs \
		--version $(version) \
		--release $(release) \
		--arch $(arch) \
		--fonts windows macos linux
```

#### Developer Targets

**`edits`** - Open developer GUI:
```makefile
edits:
	python3 ./scripts/developer.py $(version) $(release)
```

**`run`** - Run Camoufox for testing:
```makefile
run:
	cd $(cf_source_dir) \
	&& rm -rf ~/.camoufox obj-x86_64-pc-linux-gnu/tmp/profile-default \
	&& CAMOU_CONFIG=$${CAMOU_CONFIG:-'{}'} \
	&& CAMOU_CONFIG="$${CAMOU_CONFIG%?}, \"debug\": true}" ./mach run $(args)
```

**`workspace`** - Set up workspace for editing a patch:
```makefile
workspace:
	@make check-arg $(_ARGS);
	@if (cd $(cf_source_dir) && patch -p1 -R --dry-run --force -i ../$(_ARGS)) > /dev/null 2>&1; then \
		echo "Patch is already applied. Unapplying..."; \
		make unpatch $(_ARGS); \
	else \
		echo "Patch is not applied. Proceeding..."; \
	fi
	make checkpoint || true
	make patch $(_ARGS)
```

**`tests`** - Run Playwright tests:
```makefile
tests:
	cd ./tests && \
	bash run-tests.sh \
		--executable-path ../$(cf_source_dir)/obj-x86_64-pc-linux-gnu/dist/bin/camoufox-bin \
		$(if $(filter true,$(headful)),--headful,)
```

#### Advanced Targets

**`bootstrap`** - Set up complete build environment:
```makefile
bootstrap: dir
	(sudo apt-get -y install $(debs) || sudo dnf -y install $(rpms) || sudo pacman -Sy $(pacman))
	make mozbootstrap

mozbootstrap:
	cd $(cf_source_dir) && MOZBUILD_STATE_PATH=$$HOME/.mozbuild ./mach --no-interactive bootstrap --application-choice=browser
```

**`clean`** - Remove build artifacts:
```makefile
clean:
	cd $(cf_source_dir) && git clean -fdx && ./mach clobber
	make revert
```

**`distclean`** - Remove everything:
```makefile
distclean:
	rm -rf $(cf_source_dir) $(ff_source_tarball)
```

#### Variable System

```makefile
include upstream.sh
export

cf_source_dir := camoufox-$(version)-$(release)
ff_source_tarball := firefox-$(version).source.tar.xz

debs := python3 python3-dev python3-pip p7zip-full golang-go msitools wget aria2
rpms := python3 python3-devel p7zip golang msitools wget aria2
pacman := python python-pip p7zip go msitools wget aria2

vcredist_arch := $(shell echo $(arch) | sed 's/x86_64/x64/' | sed 's/i686/x86/')
```

**`upstream.sh`** contains version information:
```bash
version=135.0.1
release=beta.24
closedsrc_rev=1.0.0
```

### 2. Mozconfig Files

Mozilla's build system is configured through `.mozconfig` files. Camoufox uses a layered approach:

#### Base Configuration (`assets/base.mozconfig`)

```makefile
ac_add_options --enable-application=browser

ac_add_options --allow-addon-sideload
ac_add_options --disable-crashreporter
ac_add_options --disable-backgroundtasks
ac_add_options --disable-debug
ac_add_options --disable-default-browser-agent
ac_add_options --disable-tests
ac_add_options --disable-updater
ac_add_options --enable-release

ac_add_options --disable-system-policies

ac_add_options --with-app-name=camoufox
ac_add_options --with-branding=browser/branding/camoufox

ac_add_options --with-unsigned-addon-scopes=app,system

ac_add_options --enable-bootstrap

export MOZ_REQUIRE_SIGNING=

mk_add_options MOZ_CRASHREPORTER=0
mk_add_options MOZ_DATA_REPORTING=0
mk_add_options MOZ_SERVICES_HEALTHREPORT=0
mk_add_options MOZ_TELEMETRY_REPORTING=0
mk_add_options MOZ_INSTALLER=0
mk_add_options MOZ_AUTOMATION_INSTALLER=0
```

**Key Settings**:
- `--with-app-name=camoufox`: Sets binary name
- `--disable-updater`: No auto-updates
- `--disable-tests`: Skip test suite compilation (saves 30+ minutes)
- `--enable-release`: Optimization flags
- `--with-unsigned-addon-scopes`: Allow unsigned addons

#### Platform-Specific Configs

**Linux** (`assets/linux.mozconfig`):
```makefile
# ac_add_options --enable-alsa
```

Currently minimal—Linux is the primary development platform.

**Windows** (`assets/windows.mozconfig`):
```makefile
ac_add_options --disable-maintenance-service
ac_add_options --disable-update-agent
```

Disables Windows-specific update services.

**macOS** (`assets/macos.mozconfig`):
```makefile
ac_add_options --disable-update-agent
# ac_add_options --disable-alsa

# Packaging related
# export DSYMUTIL="$MOZBUILD/clang/bin/dsymutil"
# export DMG_TOOL="$MOZBUILD/dmg/dmg"
# export HFS_TOOL="$MOZBUILD/dmg/hfsplus"
```

macOS cross-compilation currently requires manual SDK setup (commented out).

#### Build Target Configuration

The `scripts/patch.py` script dynamically generates the final mozconfig:

```python
def _update_mozconfig(self):
    """Helper for adding additional mozconfig code"""
    mozconfig = "mozconfig"

    # Read base config
    with open(mozconfig_backup, 'r') as f:
        content = f.read()

    # Add target option
    content += f"\nac_add_options --target={self.moz_target}\n"

    # Add target-specific mozconfig
    target_mozconfig = os.path.join("..", "assets", f"{self.target}.mozconfig")
    if os.path.exists(target_mozconfig):
        with open(target_mozconfig, 'r') as f:
            content += f.read()

    # Write final mozconfig
    with open(mozconfig, 'w') as f:
        f.write(content)
```

**Example Final Mozconfig** (Linux x86_64):
```makefile
# Base configuration...
ac_add_options --target=x86_64-pc-linux-gnu
# Linux-specific options...
```

### 3. Docker Containerization Strategy

#### The Production Dockerfile

```dockerfile
FROM ubuntu:latest

WORKDIR /app

# Copy project files
COPY . /app

# Install necessary packages
RUN apt-get update && apt-get install -y \
    # Mach build tools
    build-essential make msitools wget unzip rustc \
    # Python
    python3 python3-dev python3-pip \
    # Camoufox build system tools
    git p7zip-full golang-go aria2 curl rsync \
    # CA certificates
    ca-certificates \
    && update-ca-certificates

# Install Rust via rustup
RUN curl https://sh.rustup.rs -sSf | bash -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

# Fetch Firefox & apply initial patches
RUN make setup-minimal && \
    make mozbootstrap && \
    mkdir -p /app/dist

# Mount .mozbuild directory and dist folder
VOLUME /root/.mozbuild
VOLUME /app/dist

ENTRYPOINT ["python3", "./multibuild.py"]
```

#### Building the Docker Image

```bash
docker build -t camoufox-builder .
```

This creates an image (~2GB) with all dependencies pre-installed.

#### Running Builds in Docker

**Basic Build**:
```bash
docker run --rm \
  -v ~/.mozbuild:/root/.mozbuild \
  -v $(pwd)/dist:/app/dist \
  camoufox-builder \
  --target linux --arch x86_64
```

**Multi-Platform Build**:
```bash
docker run --rm \
  -v ~/.mozbuild:/root/.mozbuild \
  -v $(pwd)/dist:/app/dist \
  camoufox-builder \
  --target linux windows macos \
  --arch x86_64 arm64
```

**Interactive Development**:
```bash
docker run -it --rm \
  -v ~/.mozbuild:/root/.mozbuild \
  -v $(pwd):/app \
  --entrypoint /bin/bash \
  camoufox-builder

# Inside container:
make dir
make build
make run
```

#### Docker Performance Considerations

**Volume Mounting**:
- `.mozbuild`: MUST be mounted to avoid re-downloading toolchains
- `dist/`: Output directory for artifacts
- Project root (optional): For live development

**Resource Limits**:
```bash
docker run --rm \
  --memory=16g \          # 16GB RAM minimum
  --memory-swap=32g \     # 32GB total (RAM + swap)
  --cpus=8 \             # Use 8 CPU cores
  -v ~/.mozbuild:/root/.mozbuild \
  camoufox-builder
```

**Storage Requirements**:
- Docker image: ~2GB
- `.mozbuild`: ~5GB
- Source + build: ~30GB per target
- Total: ~40GB minimum

### 4. CI/CD Pipeline with GitHub Actions

#### Workflow File (`.github/workflows/build.yml`)

The complete CI/CD pipeline:

```yaml
name: Build and Release

on:
  workflow_dispatch:  # Manual trigger
  push:
    tags:
      - "*"  # Trigger on any tag

jobs:
  build:
    runs-on: ubuntu-24.04
    strategy:
      matrix:
        target: [linux, windows, macos]
        arch: [x86_64, arm64, i686]
        exclude:
          # Windows ARM64: clang++-cl missing in .mozbuild
          - target: windows
            arch: arm64
          # macOS i686: Unsupported by Apple
          - target: macos
            arch: i686

    steps:
      # Step 1: Free up disk space (~40GB)
      - name: Maximize build space
        uses: AdityaGarg8/remove-unwanted-software@v4.1
        with:
          remove-dotnet: "true"
          remove-android: "true"
          remove-haskell: "true"
          remove-codeql: "true"
          remove-docker-images: "true"
          remove-cached-tools: "true"
          remove-swapfile: "true"

      # Step 2: Additional cleanup
      - name: Remove unwanted tools
        run: |
          sudo apt-get remove -y '^aspnetcore-.*' > /dev/null
          sudo apt-get remove -y '^dotnet-.*' > /dev/null
          sudo apt-get remove -y '^llvm-.*' > /dev/null
          sudo apt-get remove -y 'php.*' > /dev/null
          sudo apt-get remove -y '^mongodb-.*' > /dev/null
          sudo apt-get remove -y '^mysql-.*' > /dev/null
          sudo apt-get autoremove -y > /dev/null
          sudo apt-get clean > /dev/null

      # Step 3: Checkout code
      - uses: actions/checkout@v2
        with:
          fetch-depth: 1  # Shallow clone

      # Step 4: Install Go for launcher
      - name: Set up Go
        uses: actions/setup-go@v2
        with:
          go-version: "1.23"

      # Step 5: Install Python
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: "3.11"

      # Step 6: Install LLVM/Clang
      - name: Set up LLVM
        run: |
          wget https://apt.llvm.org/llvm.sh
          chmod +x llvm.sh
          sudo ./llvm.sh 18
          sudo apt-get install -y lld-18 clang-18

          # Install 32-bit libraries for i686 builds
          if [ "${{ matrix.arch }}" != "x86_64" ]; then
            sudo apt-get install -y libc6-i386 lib32gcc-s1 lib32stdc++6 gcc-multilib g++-multilib
          fi

          sudo update-alternatives --install /usr/bin/ld.lld ld.lld /usr/bin/ld.lld-18 100

      # Step 7: Check available space
      - name: Check disk space
        run: df -h

      # Step 8: Install build dependencies
      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y msitools p7zip-full aria2

      # Step 9: Fetch Firefox source
      - name: Fetch source
        env:
          CAMOUFOX_PASSWD: ${{ secrets.CAMOUFOX_PASSWD }}
        run: make fetch

      # Step 10: Setup build environment
      - name: Setup and bootstrap
        run: |
          make setup-minimal
          make mozbootstrap
          mkdir -p dist

      # Step 11: Create swap space (critical for linking)
      - name: Create swap space
        run: |
          sudo fallocate -l 16G /swapfile
          sudo chmod 600 /swapfile
          sudo mkswap /swapfile
          sudo swapon /swapfile
          free -h

      # Step 12: Build!
      - name: Build
        run: python3 ./multibuild.py --target ${{ matrix.target }} --arch ${{ matrix.arch }}

      # Step 13: Upload artifacts
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: CamoufoxBuilds-${{ matrix.target }}-${{ matrix.arch }}
          path: dist/*

  # Job 2: Create GitHub Release
  release:
    needs: build
    permissions:
      contents: write
    runs-on: ubuntu-latest

    steps:
      # Download all build artifacts
      - name: Download all artifacts
        uses: actions/download-artifact@v4
        with:
          path: artifacts

      # Create release with all artifacts
      - name: Create Release
        uses: softprops/action-gh-release@v1
        if: startsWith(github.ref, 'refs/tags/')
        with:
          files: artifacts/**/*
          generate_release_notes: true
          draft: true
          prerelease: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### CI/CD Workflow Explained

**Trigger Conditions**:
1. **workflow_dispatch**: Manual trigger from GitHub UI
2. **push.tags**: Automatic trigger when pushing a git tag

**Matrix Strategy**:

GitHub Actions creates 7 parallel jobs:
```
Job 1: linux-x86_64
Job 2: linux-arm64
Job 3: linux-i686
Job 4: windows-x86_64
Job 5: windows-i686
Job 6: macos-x86_64
Job 7: macos-arm64
```

Each job runs independently on its own runner.

**Disk Space Management**:

GitHub runners start with ~14GB free space. Build requires ~50GB. Solution:
1. Remove .NET (~3GB)
2. Remove Android SDK (~12GB)
3. Remove Haskell (~8GB)
4. Remove CodeQL (~5GB)
5. Remove Docker images (~8GB)
6. Remove cached tools (~4GB)

Result: ~60GB free space.

**Swap Space**:

Firefox linker (lld) can use >16GB RAM during link stage. Runners have 7GB RAM. Solution:
```bash
sudo fallocate -l 16G /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

This creates 16GB swap, giving 23GB total memory (7GB RAM + 16GB swap).

**Artifact Management**:

Actions v4 uploads artifacts in parallel chunks, ~3x faster than v3:
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: CamoufoxBuilds-${{ matrix.target }}-${{ matrix.arch }}
    path: dist/*
```

Each artifact contains one `.zip` file (~100MB compressed, ~300MB uncompressed).

**Release Creation**:

After all 7 build jobs complete, the release job:
1. Downloads all 7 artifacts
2. Creates draft GitHub release
3. Uploads all 7 packages
4. Generates release notes from commits

**Secrets Management**:

```yaml
env:
  CAMOUFOX_PASSWD: ${{ secrets.CAMOUFOX_PASSWD }}
```

This secret is set in repository settings and used to decrypt private patches.

### 5. Multi-Architecture Support

Camoufox supports three architectures on three platforms:

#### Architecture Matrix

| Platform | x86_64 | ARM64 | i686 | Total |
|----------|--------|-------|------|-------|
| Linux    | ✅ | ✅ | ✅ | 3 |
| Windows  | ✅ | ❌ | ✅ | 2 |
| macOS    | ✅ | ✅ | ❌ | 2 |
| **Total**| **3** | **2** | **2** | **7** |

#### Mozilla Build Triplets

Mozilla uses target triplets to identify platforms:

```python
def get_moz_target(target, arch):
    """Convert target/arch to Mozilla triplet"""
    moz_targets = {
        ('linux', 'x86_64'): 'x86_64-pc-linux-gnu',
        ('linux', 'arm64'):  'aarch64-unknown-linux-gnu',
        ('linux', 'i686'):   'i686-pc-linux-gnu',
        ('windows', 'x86_64'): 'x86_64-pc-windows-msvc',
        ('windows', 'i686'):   'i686-pc-windows-msvc',
        ('macos', 'x86_64'): 'x86_64-apple-darwin',
        ('macos', 'arm64'):  'aarch64-apple-darwin',
    }
    return moz_targets.get((target, arch))
```

**Triplet Format**: `<arch>-<vendor>-<os>-<environment>`
- **arch**: x86_64, aarch64, i686
- **vendor**: pc (generic), apple (Apple hardware), unknown (no vendor)
- **os**: linux, windows, darwin (macOS)
- **environment**: gnu (glibc), msvc (Microsoft), (none for macOS)

#### Cross-Compilation from Linux

All builds are performed on Linux hosts, even for Windows and macOS:

**Linux Builds**: Native compilation
```bash
./mach build --target=x86_64-pc-linux-gnu
```

**Windows Builds**: Cross-compilation using MinGW
```bash
./mach build --target=x86_64-pc-windows-msvc
```

Mozilla's build system downloads Windows SDK (~1.5GB) to `~/.mozbuild/vs/`

**macOS Builds**: Cross-compilation using Clang
```bash
./mach build --target=x86_64-apple-darwin
```

Requires macOS SDK (not included by default due to licensing).

#### Rust Target Management

Rust needs targets installed for cross-compilation:

```python
def add_rustup(*targets):
    """Add rust targets"""
    for rust_target in targets:
        run(f'~/.cargo/bin/rustup target add "{rust_target}"')

def _update_rustup(target):
    """Add rust targets for the given target"""
    if target == "linux":
        add_rustup("aarch64-unknown-linux-gnu", "i686-unknown-linux-gnu")
    elif target == "windows":
        add_rustup("x86_64-pc-windows-msvc", "i686-pc-windows-msvc")
    elif target == "macos":
        add_rustup("x86_64-apple-darwin", "aarch64-apple-darwin")
```

This is automatically called by `scripts/patch.py` before building.

#### Build Output Directories

Each target has its own build directory:

```
camoufox-135.0.1-beta.24/
├── obj-x86_64-pc-linux-gnu/          # Linux x86_64
├── obj-aarch64-unknown-linux-gnu/     # Linux ARM64
├── obj-i686-pc-linux-gnu/             # Linux i686
├── obj-x86_64-pc-windows-msvc/        # Windows x86_64
├── obj-i686-pc-windows-msvc/          # Windows i686
├── obj-x86_64-apple-darwin/           # macOS x86_64
└── obj-aarch64-apple-darwin/          # macOS ARM64
```

Each directory contains ~10GB of build artifacts.

## Cross-Platform Compilation

### Linux Build Process

Linux is the primary development platform and the host for all cross-compilation.

#### Native Linux Build

**Step 1: Setup**
```bash
make fetch          # Download Firefox source
make setup          # Extract and initialize git
make bootstrap      # Install dependencies
```

**Step 2: Apply Patches**
```bash
make dir           # Apply all Camoufox patches
```

**Step 3: Build**
```bash
BUILD_TARGET=linux,x86_64 make build
```

This invokes:
```bash
cd camoufox-135.0.1-beta.24
./mach build
```

Mach (Mozilla's build tool) does:
1. Configure build with mozconfig
2. Run `./configure` to detect system
3. Generate Makefiles with `make`
4. Compile C++ with Clang (~45 min)
5. Compile Rust with rustc (~10 min)
6. Link with lld (~5 min)
7. Process JavaScript (~2 min)
8. Generate omnijar (~1 min)

**Total Time**: ~60 minutes on 8-core machine

**Step 4: Package**
```bash
make package-linux arch=x86_64
```

Creates: `camoufox-135.0.1-beta.24-lin.x86_64.zip`

#### ARM64 Linux Build

Same as x86_64 but with cross-compilation:

```bash
BUILD_TARGET=linux,arm64 make set-target
make build-launcher arch=arm64 os=linux
make build
make package-linux arch=arm64
```

Requires:
- `aarch64-unknown-linux-gnu` Rust target
- ARM64 cross-compilation toolchain (auto-installed by Mozilla)

#### i686 Linux Build

32-bit Linux build:

```bash
BUILD_TARGET=linux,i686 make set-target
make build-launcher arch=i686 os=linux
make build
make package-linux arch=i686
```

Requires:
- `i686-unknown-linux-gnu` Rust target
- 32-bit development libraries: `libc6-i386 lib32gcc-s1 lib32stdc++6`
- Multi-lib GCC: `gcc-multilib g++-multilib`

### Windows Cross-Compilation (MinGW)

Building Windows binaries from Linux using Microsoft's MSVC toolchain.

#### How It Works

Mozilla's build system downloads a complete Windows development environment:

```
~/.mozbuild/vs/
├── VC/
│   ├── Tools/
│   │   └── MSVC/
│   │       └── 14.38.33135/
│   │           ├── bin/
│   │           │   ├── Hostx64/
│   │           │   │   ├── x64/
│   │           │   │   │   ├── cl.exe       # MSVC compiler
│   │           │   │   │   ├── link.exe     # MSVC linker
│   │           │   │   │   └── lib.exe      # MSVC librarian
│   │           │   │   └── x86/
│   │           │   │       └── (32-bit tools)
│   │           └── lib/
│   │               ├── x64/
│   │               └── x86/
│   └── Redist/
│       └── MSVC/
│           └── 14.38.33135/
│               ├── x64/
│               │   └── Microsoft.VC143.CRT/
│               │       ├── msvcp140.dll
│               │       └── vcruntime140.dll
│               └── x86/
│                   └── Microsoft.VC143.CRT/
└── Windows Kits/
    └── 10/
        ├── Include/
        └── Lib/
```

**Total Size**: ~1.5GB

#### Windows x86_64 Build

```bash
BUILD_TARGET=windows,x86_64 make set-target
make build-launcher arch=x86_64 os=windows
make build
make package-windows arch=x86_64
```

**Mozconfig Additions**:
```makefile
ac_add_options --target=x86_64-pc-windows-msvc
ac_add_options --disable-maintenance-service
ac_add_options --disable-update-agent
```

**Packaging Includes**:
- All Firefox binaries (`.exe`, `.dll`)
- MSVC redistributables (`msvcp140.dll`, `vcruntime140.dll`, `vcruntime140_1.dll`)
- Windows fonts (Arial, Times, Courier)
- Linux fonts (for fingerprinting)
- Launcher (`launch.exe`)

**Output**: `camoufox-135.0.1-beta.24-win.x86_64.zip` (~100MB)

#### Windows i686 Build

32-bit Windows:

```bash
BUILD_TARGET=windows,i686 make set-target
make build-launcher arch=i686 os=windows
make build
make package-windows arch=i686
```

Uses `x86` version of MSVC redistributables.

**Output**: `camoufox-135.0.1-beta.24-win.i686.zip`

#### Windows ARM64 Status

**Not Currently Supported** ❌

**Reason**: Mozilla's `.mozbuild` doesn't include ARM64 MSVC toolchain (`clang++-cl`).

**Blocked By**: Mozilla bug tracking ARM64 Windows support.

### macOS Cross-Compilation Challenges

Building macOS binaries from Linux is the most challenging target.

#### Requirements

1. **macOS SDK**: Contains headers and libraries for macOS APIs
2. **Clang**: LLVM compiler with macOS support
3. **cctools**: Apple's toolchain (ld, as, dsymutil)
4. **libxar**: For .pkg creation
5. **dmg tool**: For .dmg creation

#### SDK Licensing Issues

Apple's macOS SDK has restrictive licensing:
- Cannot be redistributed
- Must be extracted from Xcode
- Legal only on Apple hardware

**Solution**: Users must provide their own SDK.

#### Current Implementation

The `assets/macos.mozconfig` has experimental cross-compilation setup (commented out):

```makefile
# Packaging related
# export DSYMUTIL="$MOZBUILD/clang/bin/dsymutil"
# export DMG_TOOL="$MOZBUILD/dmg/dmg"
# export HFS_TOOL="$MOZBUILD/dmg/hfsplus"

# Build related
# CROSS=$MOZBUILD
# CCTOOLS=$CROSS/cctools
# mk_add_options "export PATH=$MOZBUILD/clang/bin:$CCTOOLS/bin:$PATH"
# mk_add_options "export LD_LIBRARY_PATH=$MOZBUILD/clang/lib:$CCTOOLS/lib"
# export CC="$CROSS/clang/bin/clang"
# export CXX="$CROSS/clang/bin/clang++"

# SDK path (user must provide)
# ac_add_options --with-macos-sdk="$MOZBUILD/MacOSX14.4.sdk"
```

#### macOS x86_64 Build

**Prerequisites**:
1. Extract macOS SDK from Xcode
2. Place in `~/.mozbuild/MacOSX14.4.sdk`
3. Uncomment macOS mozconfig options

**Build**:
```bash
BUILD_TARGET=macos,x86_64 make set-target
make build-launcher arch=x86_64 os=macos
make build
make package-macos arch=x86_64
```

**Packaging Creates**:
```
camoufox-135.0.1-beta.24-mac.x86_64.zip
└── Camoufox.app/
    └── Contents/
        ├── Info.plist
        ├── MacOS/
        │   ├── camoufox-bin
        │   └── launch
        └── Resources/
            ├── browser/
            ├── fonts/
            └── camoufox.cfg
```

#### macOS ARM64 (Apple Silicon) Build

Same process but with `aarch64-apple-darwin` target:

```bash
BUILD_TARGET=macos,arm64 make set-target
make build-launcher arch=arm64 os=macos
make build
make package-macos arch=arm64
```

**Note**: Can be built on Intel Mac and will run on Apple Silicon via Rosetta 2.

#### Universal Binaries

Future enhancement could create Universal binaries (x86_64 + ARM64):

```bash
# Build both architectures
make build TARGET=macos ARCH=x86_64
make build TARGET=macos ARCH=arm64

# Combine with lipo
lipo -create \
  camoufox-x86_64/camoufox-bin \
  camoufox-arm64/camoufox-bin \
  -output camoufox-universal/camoufox-bin
```

Not currently implemented due to complexity.

### Dependency Management

#### System Dependencies

**Debian/Ubuntu**:
```bash
sudo apt-get install -y \
    python3 python3-dev python3-pip \
    p7zip-full golang-go msitools wget aria2 \
    build-essential
```

**Fedora/RHEL**:
```bash
sudo dnf install -y \
    python3 python3-devel \
    p7zip golang msitools wget aria2
```

**Arch Linux**:
```bash
sudo pacman -Sy \
    python python-pip \
    p7zip go msitools wget aria2
```

#### Mozilla-Managed Dependencies

These are automatically downloaded to `~/.mozbuild/`:

**Toolchains**:
- **Clang/LLVM 18** (~2GB): C/C++ compiler
- **rustc 1.75** (~1GB): Rust compiler
- **Node.js 18** (~200MB): Build scripts
- **NASM 2.15** (~50MB): Assembler for media codecs
- **dump_syms** (~100MB): Debug symbol dumper

**Cross-Compilation Toolchains**:
- **Windows SDK** (~1.5GB): MSVC compiler and libraries
- **macOS SDK** (user-provided): Apple headers and libraries

**Build Tools**:
- **cbindgen** (~50MB): Rust→C bindings
- **sccache** (~30MB): Compiler cache
- **wasi-sdk** (~200MB): WebAssembly compiler

#### Python Dependencies

Build scripts use standard library only (no pip install needed).

Python library dependencies (for end users):
```bash
pip install camoufox
```

This installs:
- `playwright`: Browser automation
- `browserforge`: Fingerprint generation
- `requests`: HTTP client
- `pydantic`: Config validation

## Developer Tools

### Developer UI (`scripts/developer.py`)

GUI tool for managing patches and build workspace.

#### Launch

```bash
make edits
```

Opens EasyGUI interface with options:

#### Main Menu

```
┌─────────────────────────────────────────┐
│   Camoufox Dev Tools                     │
├─────────────────────────────────────────┤
│ • Reset workspace                        │
│ • Edit a patch                           │
│ • Create new patch                       │
│ ──────────────────────────────────────── │
│ • List patches currently applied         │
│ • Select patches                         │
│ • Reverse patches                        │
│ • Find broken patches (resets workspace) │
│ ──────────────────────────────────────── │
│ • See current workspace                  │
│ • Write workspace to patch               │
│ • Set checkpoint                         │
└─────────────────────────────────────────┘
```

#### Key Features

**1. Patch Status Indicators**

```python
def check_patch(patch_file):
    """Returns (can_apply, can_reverse, is_broken)"""
    can_apply = not bool(
        os.system(f'patch -p1 --dry-run --force -i "{patch_file}" > /dev/null 2>&1')
    )
    can_reverse = not bool(
        os.system(f'patch -p1 -R --dry-run --force -i "{patch_file}" > /dev/null 2>&1')
    )
    return can_apply, can_reverse, not (can_apply or can_reverse)
```

Status labels:
- `[APPLIED]` - Currently applied
- `[NOT APPLIED]` - Can be applied
- `[BROKEN]` - Cannot apply (conflicts)
- `[BOOTSTRAP]` - Bootstrap patch (always breaks after patching)

**2. Edit a Patch**

Workflow:
1. Select patch from list
2. System resets workspace
3. Applies all patches up to (but not including) selected patch
4. Creates git checkpoint
5. Applies selected patch (showing any conflicts)
6. Developer makes changes
7. Developer saves workspace to patch file

**3. Create New Patch**

Workflow:
1. Select "Create new patch"
2. System resets and applies all patches
3. Git checkpoint created
4. Developer makes changes to source
5. Developer tests with `make run`
6. Developer selects "Write workspace to patch"
7. Saves `git diff` to new `.patch` file

**4. Find Broken Patches**

Iterates through all patches, testing if each applies cleanly:

```python
for patch_file in list_patches():
    if reject_files := get_rejects(patch_file):
        broken_patches.append((patch_file, reject_files))
```

Shows:
- Which patches are broken
- Number of rejects per file
- Contents of `.rej` files

**5. Write Workspace to Patch**

Saves current uncommitted changes to a patch:

```python
run(f'git diff > {file_path}')
```

### Patch Management System

#### Patch Directory Structure

```
patches/
├── 0-base-changes.patch         # Bootstrap patches (run first)
├── 0-disable-telemetry.patch
├── browser-init.patch
├── chromeutil.patch
├── config.patch
├── fingerprint-injection.patch  # Core patches
├── fonts.patch
├── juggler.patch
├── navigator-webdriver.patch
├── permissions.patch
├── remap-keys.patch
├── viewport-hijacker.patch
└── private/                      # Private patches (encrypted)
    ├── private-feature-1.patch
    └── private-feature-2.patch
```

#### Patch Application Order

Patches are applied in lexicographical order:
1. Patches starting with `0-` are bootstrap patches
2. Other patches are feature patches
3. Private patches are applied last

#### Creating Patches

**Manual Method**:
```bash
cd camoufox-135.0.1-beta.24
# Make changes
git diff > ../patches/my-new-feature.patch
```

**Dev UI Method**:
```bash
make edits
# Select "Create new patch"
# Make changes
# Test with "make run"
# Select "Write workspace to patch"
```

#### Applying Patches

**All Patches**:
```bash
make dir
```

**Single Patch**:
```bash
make patch ./patches/fingerprint-injection.patch
```

**Reverse Patch**:
```bash
make unpatch ./patches/fingerprint-injection.patch
```

#### Editing Patches

**Using Workspace**:
```bash
make workspace ./patches/fingerprint-injection.patch
# Make changes
git diff > ./patches/fingerprint-injection.patch
```

**Using Dev UI**:
```bash
make edits
# Select "Edit a patch"
# Select patch to edit
# Make changes
# Save workspace
```

### Testing and Debugging Tools

#### Running Tests

**Playwright Tests**:
```bash
make tests
```

Runs test suite in `tests/` directory.

**Headful Mode** (see browser):
```bash
make tests headful=true
```

**Specific Test**:
```bash
cd tests
pytest test_fingerprinting.py -v
```

#### Manual Testing

**Run Camoufox**:
```bash
make run
```

**With Configuration**:
```bash
CAMOU_CONFIG='{"navigator.userAgent": "Custom UA"}' make run
```

**With Arguments**:
```bash
make run args="--devtools"
```

**With Launcher**:
```bash
make run-launcher
```

#### Debugging

**View Logs**:
```bash
cd camoufox-135.0.1-beta.24
./mach run 2>&1 | tee debug.log
```

**Firefox DevTools**:
```bash
make run args="--devtools"
```

**Browser Console**:
```bash
make run args="--jsconsole"
```

**GDB Debugging**:
```bash
cd camoufox-135.0.1-beta.24
gdb --args ./obj-x86_64-pc-linux-gnu/dist/bin/camoufox-bin
```

#### Performance Profiling

**Gecko Profiler**:
```bash
make run args="--profile"
```

**Memory Benchmarks**:
```bash
cd scripts/benchmark
make build
make run MODE=headless BROWSER=camoufox
```

## Packaging System

### Binary Packaging for Each Platform

The `scripts/package.py` script handles all packaging operations.

#### Core Packaging Logic

```python
def main():
    """Main packaging function"""
    args = get_args()

    # Determine file extension
    file_extensions = {
        'linux': 'tar.xz',
        'macos': 'dmg',
        'windows': 'zip'
    }
    file_ext = file_extensions[args.os]

    # Build the package
    src_dir = find_src_dir('.', args.version, args.release)
    moz_target = get_moz_target(target=args.os, arch=args.arch)

    with temp_cd(src_dir):
        # Create package files
        run('./mach package')

        # Find package files
        search_path = os.path.abspath(
            f'obj-{moz_target}/dist/camoufox-{args.version}-{args.release}.*.{file_ext}'
        )

    # Copy package file to root
    for file in glob.glob(search_path):
        shutil.copy2(file, '.')
        break

    # Add includes to the package
    new_name = f'camoufox-{args.version}-{args.release}-{args.os[:3]}.{args.arch}.zip'
    add_includes_to_package(
        package_file=package_file,
        includes=args.includes,
        fonts=args.fonts,
        new_file=new_name,
        target=args.os,
    )
```

#### Package Modification

After Mozilla creates the base package, we add custom files:

```python
def add_includes_to_package(package_file, includes, fonts, new_file, target):
    with tempfile.TemporaryDirectory() as temp_dir:
        # Extract package
        run(f'7z x {package_file} -o{temp_dir}')

        # Determine target directory
        if target == 'macos':
            target_dir = os.path.join(temp_dir, 'Camoufox.app', 'Contents', 'Resources')
        else:
            target_dir = temp_dir

        # Add includes
        for include in includes or []:
            if os.path.isdir(include):
                shutil.copytree(include, os.path.join(target_dir, os.path.basename(include)))
            else:
                shutil.copy2(include, target_dir)

        # Add fonts
        fonts_dir = os.path.join(target_dir, 'fonts')
        for font in fonts or []:
            # Platform-specific font handling
            ...

        # Remove unneeded paths
        for path in UNNEEDED_PATHS:
            shutil.rmtree(os.path.join(target_dir, path), ignore_errors=True)

        # Repackage
        run(f'7z u {new_file} {temp_dir}/* -r -mx=9')
```

### Font Bundling

Fonts are critical for fingerprinting consistency.

#### Font Structure

```
bundle/fonts/
├── linux/
│   ├── DejaVuSans.ttf
│   ├── DejaVuSans-Bold.ttf
│   ├── LiberationSans-Regular.ttf
│   └── ...
├── macos/
│   ├── SFNSDisplay.ttf
│   ├── Helvetica.ttc
│   ├── Times.ttc
│   └── ...
└── windows/
    ├── arial.ttf
    ├── times.ttf
    ├── courier.ttf
    └── ...
```

Each platform includes fonts from other platforms for fingerprinting spoofing.

#### Platform-Specific Font Packaging

**Linux** (keep folder structure):
```python
if target == 'linux':
    for font in fonts or []:
        shutil.copytree(
            os.path.join('bundle', 'fonts', font),
            os.path.join(fonts_dir, font),
            dirs_exist_ok=True,
        )
```

Result:
```
fonts/
├── linux/
│   └── (linux fonts)
├── windows/
│   └── (windows fonts)
└── macos/
    └── (macos fonts)
```

**Windows/macOS** (flatten structure):
```python
else:
    os.makedirs(fonts_dir, exist_ok=True)
    for font in fonts or []:
        for file in list_files(root_dir=os.path.join('bundle', 'fonts', font), suffix='*'):
            shutil.copy2(file, os.path.join(fonts_dir, os.path.basename(file)))
```

Result:
```
fonts/
├── arial.ttf
├── times.ttf
├── helvetica.ttc
├── dejavu.ttf
└── ...
```

#### Font Configuration

Linux includes fontconfig files:

```
bundle/fontconfigs/
└── fonts.conf
```

Tells fontconfig where to find bundled fonts:
```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <dir>/path/to/camoufox/fonts</dir>
</fontconfig>
```

### Asset Inclusion

Each package includes platform-specific assets:

#### Common Assets (All Platforms)

```makefile
--includes \
    settings/chrome.css \
    settings/camoucfg.jvv \
    settings/properties.json
```

**chrome.css**: Custom browser chrome styling
**camoucfg.jvv**: JSON validation schema
**properties.json**: Default configuration values

#### Linux-Specific

```makefile
--includes \
    bundle/fontconfigs
```

Fontconfig files for loading bundled fonts.

#### Windows-Specific

```makefile
--includes \
    ~/.mozbuild/vs/VC/Redist/MSVC/14.38.33135/$(vcredist_arch)/Microsoft.VC143.CRT/*.dll
```

MSVC runtime DLLs required to run on Windows without Visual Studio installed:
- `msvcp140.dll` - C++ standard library
- `vcruntime140.dll` - C runtime
- `vcruntime140_1.dll` - C runtime continuation

#### macOS-Specific

No extra includes needed (app bundle is self-contained).

### Version Management

Version information is stored in `upstream.sh`:

```bash
version=135.0.1
release=beta.24
closedsrc_rev=1.0.0
```

**version**: Firefox version
**release**: Camoufox release identifier
**closedsrc_rev**: Private patch version

#### Updating Versions

```bash
# Edit upstream.sh
vim upstream.sh

# Source it in Makefile
include upstream.sh
export

# Use in commands
make fetch   # Downloads Firefox $(version)
make dir     # Creates camoufox-$(version)-$(release)
```

## Hands-On: Setting Up a Build Environment

Let's walk through setting up a complete Camoufox build environment from scratch.

### Prerequisites

**Hardware Requirements**:
- CPU: 8+ cores recommended (minimum 4)
- RAM: 16GB+ recommended (minimum 8GB)
- Disk: 60GB free space minimum
- OS: Linux (Ubuntu 22.04+ / Fedora 38+ / Arch Linux)

**Time Requirements**:
- Initial setup: ~30 minutes
- First build: ~90 minutes
- Subsequent builds: ~45 minutes

### Step 1: Clone Repository

```bash
git clone https://github.com/daijro/camoufox.git
cd camoufox
```

### Step 2: Install System Dependencies

**Ubuntu/Debian**:
```bash
sudo apt-get update
sudo apt-get install -y \
    python3 python3-dev python3-pip \
    p7zip-full golang-go msitools wget aria2 \
    build-essential git curl
```

**Fedora/RHEL**:
```bash
sudo dnf install -y \
    python3 python3-devel \
    p7zip golang msitools wget aria2 \
    make gcc-c++ git curl
```

**Arch Linux**:
```bash
sudo pacman -Sy \
    python python-pip \
    p7zip go msitools wget aria2 \
    base-devel git curl
```

### Step 3: Fetch Firefox Source

```bash
make fetch
```

This downloads:
- Firefox source tarball (~450MB)
- Private patches (if `CAMOUFOX_PASSWD` is set)

**Expected Output**:
```
Fetching private patches...
Skipping private patches...
Fetching the Firefox source tarball...
[###############################] 100%
Download complete: firefox-135.0.1.source.tar.xz
```

### Step 4: Extract and Setup

```bash
make setup
```

This:
1. Extracts Firefox source (~3GB uncompressed)
2. Copies Camoufox additions
3. Initializes git repository
4. Tags initial state as `unpatched`

**Expected Output**:
```
Extracting Firefox source...
Copying Camoufox additions...
Initializing git repository...
[main (root-commit) abc1234] Initial commit
 485 files changed, 123456 insertions(+)
```

### Step 5: Bootstrap Build Environment

```bash
make bootstrap
```

This:
1. Installs remaining system packages
2. Runs Mozilla's bootstrap script
3. Downloads toolchains to `~/.mozbuild/` (~5GB)

**Expected Output**:
```
Running mozbootstrap...
Downloading Clang... [###########] 2.1GB
Downloading Rust... [###########] 1.0GB
Downloading Node... [###########] 200MB
Bootstrap complete!
```

**This step only needs to be done once.**

### Step 6: Apply Patches

```bash
make dir
```

Applies all Camoufox patches.

**Expected Output**:
```
Applying patches...
-> chromeutil.patch
-> browser-init.patch
-> fingerprint-injection.patch
-> juggler.patch
...
Complete! Ready to build.
```

### Step 7: Build Camoufox

```bash
make build
```

First build takes ~60-90 minutes.

**Expected Output**:
```
0:00.50 Running configure
0:01.20 Generating build files...
0:02.00 Building...
10:30.00 Compiling dom/base/Navigator.cpp
20:45.00 Compiling dom/base/nsGlobalWindowInner.cpp
...
55:00.00 Linking libxul.so
58:30.00 Processing omnijar
60:00.00 Build complete!
```

**Build Progress Phases**:
1. **Configure** (0-2 min): Detect system, generate Makefiles
2. **Early Compile** (2-20 min): Compile independent translation units
3. **Parallel Compile** (20-55 min): Compile all C++/Rust in parallel
4. **Linking** (55-58 min): Link everything into `libxul.so` (uses lots of RAM)
5. **Finalization** (58-60 min): Create omnijar, copy files

### Step 8: Test Build

```bash
make run
```

Should open Camoufox browser window.

**Expected Output**:
```
Starting Camoufox...
[GFX1-]: Initialized X11 window
[GFX1-]: Created GL context
```

**Test Configuration**:
```bash
CAMOU_CONFIG='{"navigator.userAgent": "Test UA"}' make run
```

### Step 9: Build Launcher

```bash
make build-launcher arch=x86_64 os=linux
```

Compiles the Go launcher.

**Expected Output**:
```
Building launcher...
go build -o dist/launch
Complete: legacy/launcher/dist/launch
```

### Step 10: Package

```bash
make package-linux arch=x86_64
```

Creates distributable package.

**Expected Output**:
```
Running mach package...
Creating archive...
Adding fonts...
Adding assets...
Complete: camoufox-135.0.1-beta.24-lin.x86_64.zip
```

### Common Issues and Solutions

#### Issue: "Failed to download toolchain"

**Solution**: Check internet connection, try again:
```bash
rm -rf ~/.mozbuild/*
make mozbootstrap
```

#### Issue: "Out of disk space"

**Solution**: Each target needs ~30GB. Clean old builds:
```bash
make clean
make distclean
```

#### Issue: "Build fails during linking"

**Solution**: Insufficient RAM. Add swap:
```bash
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
make build
```

#### Issue: "Patch fails to apply"

**Solution**: Reset and try again:
```bash
make revert
make dir
```

### Building for Multiple Platforms

#### Build All Targets

```bash
python3 multibuild.py \
    --target linux windows macos \
    --arch x86_64 arm64
```

This builds 6 targets in sequence.

#### Build Specific Targets

```bash
# Linux x86_64 and ARM64
python3 multibuild.py --target linux --arch x86_64 arm64

# Windows only
python3 multibuild.py --target windows --arch x86_64 i686

# macOS only
python3 multibuild.py --target macos --arch x86_64 arm64
```

### Using Docker

#### Build Docker Image

```bash
docker build -t camoufox-builder .
```

#### Run Build in Docker

```bash
docker run --rm \
    -v ~/.mozbuild:/root/.mozbuild \
    -v $(pwd)/dist:/app/dist \
    camoufox-builder \
    --target linux --arch x86_64
```

#### Interactive Docker Session

```bash
docker run -it --rm \
    -v ~/.mozbuild:/root/.mozbuild \
    -v $(pwd):/app \
    --entrypoint /bin/bash \
    camoufox-builder
```

Inside container:
```bash
make dir
make build
make package-linux arch=x86_64
```

### Development Workflow

#### Making Changes

1. **Edit a Patch**:
```bash
make edits
# Select "Edit a patch"
# Select patch to edit
# Make changes to source code
# Save workspace to patch
```

2. **Test Changes**:
```bash
make build
make run
```

3. **Create Patch**:
```bash
make edits
# Select "Write workspace to patch"
# Choose filename
```

#### Iterating Quickly

**Incremental Builds**:
```bash
# Make small change to one file
make build

# Mach automatically detects what needs rebuilding
# Takes ~2-5 minutes instead of full 60 minutes
```

**Build Specific Files**:
```bash
cd camoufox-135.0.1-beta.24
./mach build dom/base
```

**Skip Launcher Rebuild**:
```bash
# Launcher rarely changes
make build  # Skips launcher by default
```

### CI/CD Usage

#### Trigger Manual Build

1. Go to repository on GitHub
2. Click "Actions" tab
3. Select "Build and Release"
4. Click "Run workflow"
5. Select branch
6. Click "Run workflow" button

#### Create Release Build

```bash
# Tag version
git tag v135.0.1-beta.24
git push origin v135.0.1-beta.24

# GitHub Actions automatically:
# - Builds all 7 targets
# - Creates draft release
# - Uploads all packages
```

#### Monitor Build Progress

GitHub Actions shows real-time logs for each matrix job:
- linux-x86_64: 60 min
- linux-arm64: 62 min
- linux-i686: 65 min
- windows-x86_64: 58 min
- windows-i686: 63 min
- macos-x86_64: 70 min
- macos-arm64: 72 min

Total parallel time: ~72 min (longest job)

## External References

### Firefox Build Documentation

1. **Firefox Build System**
   - Main docs: https://firefox-source-docs.mozilla.org/
   - Build prerequisites: https://firefox-source-docs.mozilla.org/setup/
   - mozconfig options: https://firefox-source-docs.mozilla.org/build/buildsystem/mozconfigs.html

2. **mach (Mozilla Build Tool)**
   - mach overview: https://firefox-source-docs.mozilla.org/mach/
   - mach commands: https://firefox-source-docs.mozilla.org/mach/commands.html

3. **Cross-Compilation**
   - Cross-compiling Firefox: https://firefox-source-docs.mozilla.org/contributing/build/cross-compile.html
   - Windows cross-compile: https://firefox-source-docs.mozilla.org/build/buildsystem/rust.html

4. **Build Performance**
   - Build optimization: https://firefox-source-docs.mozilla.org/contributing/build/artifact_builds.html
   - ccache/sccache: https://github.com/mozilla/sccache

### Docker & Containerization

1. **Docker Documentation**
   - Dockerfile reference: https://docs.docker.com/engine/reference/builder/
   - Multi-stage builds: https://docs.docker.com/build/building/multi-stage/
   - Docker volumes: https://docs.docker.com/storage/volumes/

2. **Podman (Docker Alternative)**
   - Podman basics: https://podman.io/getting-started/
   - Docker to Podman: https://podman.io/whatis.html

### GitHub Actions

1. **GitHub Actions Documentation**
   - Workflow syntax: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions
   - Matrix builds: https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs
   - Artifacts: https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts

2. **GitHub Actions Marketplace**
   - actions/checkout: https://github.com/actions/checkout
   - actions/upload-artifact: https://github.com/actions/upload-artifact
   - actions/download-artifact: https://github.com/actions/download-artifact
   - softprops/action-gh-release: https://github.com/softprops/action-gh-release

### Build Tools & Toolchains

1. **LLVM/Clang**
   - LLVM website: https://llvm.org/
   - Clang compiler: https://clang.llvm.org/
   - lld linker: https://lld.llvm.org/

2. **Rust**
   - Rust website: https://www.rust-lang.org/
   - rustup: https://rustup.rs/
   - Cross-compilation: https://rust-lang.github.io/rustup/cross-compilation.html

3. **Go (for Launcher)**
   - Go website: https://go.dev/
   - Cross-compilation: https://go.dev/doc/install/source#environment

### Compression & Archiving

1. **7-Zip**
   - 7-Zip website: https://www.7-zip.org/
   - p7zip (Linux): https://github.com/jinfeihan57/p7zip

2. **aria2 (Download Manager)**
   - aria2 website: https://aria2.github.io/
   - aria2 manual: https://aria2.github.io/manual/en/html/

### Linux Build Tools

1. **Make**
   - GNU Make manual: https://www.gnu.org/software/make/manual/
   - Make tutorial: https://makefiletutorial.com/

2. **Git**
   - Git documentation: https://git-scm.com/doc
   - Git patches: https://git-scm.com/docs/git-format-patch

### Related Projects

1. **LibreWolf** (Privacy-focused Firefox fork)
   - Repository: https://gitlab.com/librewolf-community/browser/source
   - Build scripts: https://gitlab.com/librewolf-community/browser/source/-/tree/main/scripts

2. **Playwright**
   - Documentation: https://playwright.dev/
   - Firefox patches: https://github.com/microsoft/playwright/tree/main/browser_patches/firefox

3. **BrowserForge** (Fingerprint Generation)
   - Repository: https://github.com/daijro/browserforge
   - Documentation: https://github.com/daijro/browserforge#readme

### Advanced Build Techniques

#### Incremental Builds

Firefox's build system supports incremental builds—only recompiling changed files.

**How Incremental Builds Work**:

1. **Dependency Tracking**: Build system tracks which headers each source file includes
2. **Modification Detection**: Uses timestamps to detect changed files
3. **Smart Recompilation**: Only recompiles files affected by changes
4. **Linking Optimization**: Only relinks libraries if object files changed

**Example**:
```bash
# Full build: 60 minutes
make build

# Change Navigator.cpp
vim camoufox-135.0.1-beta.24/dom/base/Navigator.cpp

# Incremental build: 2-5 minutes
make build
```

**What Gets Rebuilt**:
- `Navigator.cpp` → recompile
- `Navigator.o` → relink into `libxul.so`
- Other files → skip

**What Forces Full Rebuild**:
- Changing mozconfig (build configuration changed)
- Updating Rust version (toolchain changed)
- Modifying widely-included headers (`mozilla-config.h`)
- Running `make clean`

#### Parallel Compilation

The build system uses all available CPU cores:

**Compilation Parallelism**:
```bash
# Automatic: Uses all cores
./mach build

# Manual control: Use 8 cores
./mach build -j8

# Serial compilation (debugging)
./mach build -j1
```

**Parallel Strategies**:

1. **C++ Compilation**: Each `.cpp` file compiles independently (500+ parallel jobs)
2. **Rust Compilation**: Crates compile in dependency order (200+ parallel jobs)
3. **JavaScript Processing**: Bundles process in parallel
4. **Linking**: Serial (only one linker runs at a time, uses lots of RAM)

**Performance Scaling**:

| CPU Cores | Build Time | Efficiency |
|-----------|------------|------------|
| 1 core | 240 min | 100% |
| 2 cores | 125 min | 96% |
| 4 cores | 70 min | 85% |
| 8 cores | 45 min | 66% |
| 16 cores | 30 min | 50% |
| 32 cores | 25 min | 30% |

Efficiency drops with more cores due to linking bottleneck and dependency chains.

#### Caching Strategies

**sccache (Shared Compilation Cache)**:

Mozilla uses sccache to cache compiled object files:

```bash
# Enable sccache
export SCCACHE_DIR=~/.cache/sccache
./mach build
```

**How sccache Works**:
1. Hash source file + compiler flags + headers
2. Check cache for matching hash
3. If found, use cached object file (instant)
4. If not found, compile and cache result

**Benefits**:
- Switching git branches: ~10 min instead of 60 min
- Clean builds with same code: ~5 min instead of 60 min
- Shared cache across projects

**Why Camoufox Doesn't Use It**:
- CI builds are always clean (no benefit)
- Local development benefits from incremental builds instead
- Cache storage is expensive (10GB+)

#### Build Artifacts

**What Gets Generated**:

```
obj-x86_64-pc-linux-gnu/
├── dist/
│   ├── bin/
│   │   ├── camoufox-bin              # Main browser binary (200MB)
│   │   ├── libxul.so                 # Core Firefox library (180MB)
│   │   ├── libmozsandbox.so          # Sandbox library
│   │   ├── libmozgtk.so              # GTK integration
│   │   ├── libmozavcodec.so          # Media codecs
│   │   ├── browser/                  # Browser chrome
│   │   ├── fonts/                    # System fonts
│   │   └── omni.ja                   # JavaScript bundle (50MB)
│   └── camoufox-135.0.1-beta.24.tar.xz  # Package archive
├── toolkit/
│   └── (compiled toolkit objects)
├── dom/
│   └── (compiled DOM objects)
├── js/
│   └── (compiled SpiderMonkey objects)
└── (20GB of other build artifacts)
```

**Artifact Sizes**:
- Object files (`.o`): ~10GB
- Libraries (`.so`): ~500MB
- Final binary: ~200MB
- Compressed package: ~100MB

#### Build Optimization Flags

**Debug vs Release Builds**:

**Debug Build** (`--disable-optimize`):
- No optimizations: Fast compilation (30 min)
- Large binary: ~500MB
- Slow runtime: 50% slower than release
- Includes debug symbols: Good for debugging
- Use case: Development iteration

**Release Build** (`--enable-release`):
- Optimizations enabled: Slow compilation (60 min)
- Small binary: ~200MB
- Fast runtime: Maximum performance
- No debug symbols: Crashes are hard to debug
- Use case: Distribution

**Camoufox Uses**: Release builds for distribution, debug builds for development.

**LTO (Link-Time Optimization)**:

```makefile
# Enable LTO
ac_add_options --enable-lto=cross
```

**LTO Benefits**:
- 5-10% smaller binaries
- 2-5% faster runtime
- Cross-function optimizations

**LTO Costs**:
- 30% longer build time (90 min instead of 60 min)
- 50% more RAM during linking (24GB peak)
- Harder to debug crashes

**Decision**: Camoufox disabled LTO (commit `02729d6`) to prioritize build speed.

#### Troubleshooting Build Failures

**Common Build Errors**:

**1. "Out of Memory" During Linking**

```
c++: fatal error: Killed signal terminated program cc1plus
```

**Solution**: Add swap space
```bash
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

**2. "Patch fails to apply"**

```
patch: **** malformed patch at line 42: @@ -1,5 +1,6 @@
```

**Solution**: Patch is broken or for wrong Firefox version
```bash
make revert
git pull  # Get latest patches
make dir
```

**3. "rustup: command not found"**

```
error: command 'rustup' not found
```

**Solution**: Install rustup
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

**4. "Python: No module named 'mozbuild'"**

```
ModuleNotFoundError: No module named 'mozbuild'
```

**Solution**: Run bootstrap
```bash
make mozbootstrap
```

**5. "ninja: error: loading 'build.ninja': No such file or directory"**

```
ninja: error: loading 'build.ninja': No such file or directory
```

**Solution**: Reconfigure
```bash
cd camoufox-135.0.1-beta.24
./mach configure
./mach build
```

**6. "Text file busy"**

```
cannot execute binary file: Text file busy
```

**Solution**: Browser is running, close it
```bash
killall camoufox-bin
make unbusy
make build
```

#### Build Performance Analysis

**Where Time Is Spent** (60-minute build):

| Phase | Time | % of Total | Parallelizable |
|-------|------|------------|----------------|
| Configure | 2 min | 3% | No |
| C++ Compilation | 35 min | 58% | Yes (500+ jobs) |
| Rust Compilation | 10 min | 17% | Yes (200+ jobs) |
| JavaScript Processing | 2 min | 3% | Yes |
| Linking | 8 min | 13% | No |
| Packaging | 3 min | 5% | No |

**Bottlenecks**:
1. **Linking**: Single-threaded, memory-intensive
2. **Rust Dependencies**: Must build in order
3. **Configuration**: Serial operations

**Optimization Opportunities**:
- Distributed compilation (distcc)
- Better linker (mold is 2x faster than lld)
- Precompiled headers
- Module builds (C++20 modules)

### Build System Comparison: Camoufox vs Others

How does Camoufox's build system compare to similar projects?

#### Camoufox vs LibreWolf

**LibreWolf** (another Firefox fork):

| Aspect | Camoufox | LibreWolf |
|--------|----------|-----------|
| Build Tool | Makefile + Python | GitLab CI YAML |
| CI/CD | GitHub Actions | GitLab CI |
| Patch Format | Unified diffs | Unified diffs |
| Developer UI | Python GUI (EasyGUI) | None |
| Multi-platform | 7 targets | 3 targets |
| Docker | Yes | Yes |
| Build Time | ~60 min | ~70 min |

**Similarities**:
- Both use Mozilla's mach
- Both apply patches to Firefox source
- Both use Docker for CI/CD

**Differences**:
- Camoufox has more sophisticated developer tooling
- LibreWolf focuses on x86_64 only
- Camoufox builds more architectures (ARM64, i686)

#### Camoufox vs Chromium Forks

**Chromium Forks** (Brave, Ungoogled Chromium):

| Aspect | Camoufox (Firefox) | Brave (Chromium) |
|--------|-------------------|------------------|
| Build System | Mach + Make | GN + Ninja |
| Build Time | 60 min | 120 min |
| RAM Required | 16GB | 32GB |
| Disk Space | 50GB | 100GB |
| Compiler | Clang 18 | Clang 17 |
| Language | C++ + Rust | C++ |
| Patch Method | Unified diffs | GN flags |

**Firefox Advantages**:
- Faster builds
- Less RAM required
- Simpler patching

**Chromium Advantages**:
- Better build caching
- More modular architecture
- Better documentation

### Future Build System Improvements

**Planned Enhancements**:

1. **Distributed Compilation**
   - Use distcc/icecc for network compilation
   - Reduce single-machine build time to ~20 min
   - Requires build farm infrastructure

2. **Improved Caching**
   - Use GitHub Actions cache more effectively
   - Cache compiled artifacts between runs
   - Reduce CI build time to ~30 min

3. **Universal Binaries**
   - macOS: Combine x86_64 + ARM64 into one .app
   - Requires building both architectures
   - Better user experience (one download)

4. **Prebuilt Toolchains**
   - Package LLVM, Rust, Node.js into one download
   - Eliminate 10-minute bootstrap time
   - Faster contributor onboarding

5. **Better Error Messages**
   - Detect common errors (out of memory, missing deps)
   - Provide actionable solutions
   - Reduce debugging time

6. **Incremental Packaging**
   - Only repackage changed files
   - Reduce packaging time from 3 min to 30 sec
   - Faster iteration during development

7. **Build Analytics**
   - Track build times per commit
   - Identify performance regressions
   - Optimize slow compilation units

8. **ARM64 Native Builds**
   - Build on ARM64 hardware (Raspberry Pi, AWS Graviton)
   - Faster ARM64 builds (native vs cross-compiled)
   - Better ARM64 support

---

## Conclusion

The evolution of Camoufox's build system represents a journey from a simple Makefile to a sophisticated multi-platform CI/CD pipeline. Through 39 commits spanning 6 months, the build infrastructure grew to support:

- **7 build targets** (3 platforms × multiple architectures)
- **Docker containerization** for reproducible builds
- **GitHub Actions CI/CD** with parallel matrix builds
- **Comprehensive developer tooling** with GUI patch management
- **Automated packaging** with font bundling and asset inclusion
- **Cross-platform compilation** from a single Linux host

The build system's evolution mirrors the project's maturation, transforming from a development tool into a production-grade infrastructure capable of building release-quality binaries automatically. This foundation enables rapid iteration on anti-detection features while maintaining the complex requirements of Firefox-based browser development.

**Key Takeaways**:

1. **Start Simple, Iterate**: The Makefile started with basic targets and grew organically
2. **Docker for Reproducibility**: Containerization solved countless "works on my machine" issues
3. **CI/CD is Essential**: Automated builds catch issues early and enable reliable releases
4. **Developer Experience Matters**: GUI tools and clear workflows encourage contributions
5. **Cross-Platform is Hard**: Windows/macOS cross-compilation required months of refinement
6. **Disk Space is Critical**: Cleared ~40GB from GitHub runners to make builds possible
7. **Memory Management**: 16GB swap space solved linker memory exhaustion
8. **Caching Trade-offs**: Sometimes not caching is more reliable than complex cache logic

The build system continues to evolve, with future improvements planned for Universal macOS binaries, improved caching strategies, and reduced build times through distributed compilation.
