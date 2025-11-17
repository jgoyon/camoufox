# Python Library Development: A Comprehensive History

## Table of Contents

1. [Overview](#overview)
2. [Complete Commit Analysis](#complete-commit-analysis)
3. [Architecture](#architecture)
4. [Key Features Evolution](#key-features-evolution)
5. [API Design](#api-design)
6. [PyPI Releases](#pypi-releases)
7. [Integration Examples](#integration-examples)
8. [Challenges & Solutions](#challenges--solutions)
9. [Hands-On: Using the Python Library](#hands-on-using-the-python-library)
10. [External References](#external-references)

---

## Overview

### Why a Python Library Was Needed

The Camoufox Python library (`camoufox`) emerged as a critical bridge between the powerful anti-detection capabilities of the Camoufox browser and the Python ecosystem's automation and scraping communities. Before this library, users needed to:

1. **Manually configure** dozens of fingerprinting parameters
2. **Generate fingerprints** themselves using external tools
3. **Manage binary downloads** and updates manually
4. **Calculate geolocation** and locale data based on proxy locations
5. **Handle complex Playwright API** configurations

The Python library automated all of these tasks, making Camoufox accessible to developers without deep knowledge of browser fingerprinting or anti-detection techniques.

### Goals

The primary goals of the Python library were:

1. **Simplicity**: Provide a clean, Pythonic API that "just works" out of the box
2. **Automation**: Automatically generate and inject realistic device fingerprints
3. **Integration**: Seamlessly wrap Playwright's API for browser automation
4. **Geolocation Intelligence**: Automatically determine locale, timezone, and geolocation from proxy IP addresses
5. **Cross-Platform**: Support Windows, macOS, and Linux (including headless servers)
6. **Extensibility**: Allow advanced users to customize every aspect of fingerprinting
7. **Developer Experience**: Provide excellent documentation, type hints, and error messages

### Impact

The Python library transformed Camoufox from a powerful but complex browser into an accessible tool for the broader Python community:

- **52+ commits** dedicated to Python library development
- **47 PyPI releases** from v0.1.0 to v0.4.11
- **Thousands of downloads** from PyPI
- **Active community** contributing bug reports, feature requests, and improvements
- **Ecosystem integration** with popular tools like Playwright, BrowserForge, and more
- **Production use** in commercial scraping and automation projects

The library became the **primary interface** for most Camoufox users, with features like automatic fingerprint generation, GeoIP-based locale detection, and Xvfb integration making it significantly more powerful than using Camoufox directly.

---

## Complete Commit Analysis

This section provides a chronological analysis of all 52 commits related to the Python library development, organized by version releases from v0.1.0 to v0.4.11.

### Phase 1: Initial Release (v0.1.0 - v0.1.3)

#### Commit: 5e1fb78 - Camoufox Python interface (Sept 16, 2024)

**The Genesis Commit**

This was the foundational commit that established the entire Python library infrastructure:

**Initial Architecture:**
```python
# Sync API structure
class Camoufox(PlaywrightContextManager):
    """Context manager for automatic browser lifecycle"""

def NewBrowser(playwright, **kwargs) -> Browser:
    """Launch Camoufox with generated fingerprints"""
```

**Key Components Created:**
- **sync_api.py**: Synchronous Playwright wrapper
- **async_api.py**: Asynchronous Playwright wrapper
- **utils.py**: Core fingerprint generation and configuration
- **addons.py**: Default addon management (uBlock Origin)
- **fingerprints.py**: BrowserForge integration
- **pkgman.py**: Binary download and version management
- **exceptions.py**: Custom exception types

**Initial Features:**
- BrowserForge fingerprint generation
- Automatic addon installation (uBlock Origin by default)
- Cross-platform binary management
- Configuration validation system
- Font injection based on OS
- Screen dimension constraints

**Design Decisions:**
- Wrapped Playwright's API rather than forking it
- Used context managers for automatic cleanup
- Separated sync and async APIs for better ergonomics
- Made fingerprint generation the default behavior

#### Commit: a05379f - Fix MacOS target path (Sept 16, 2024)

Fixed the executable path for macOS, which uses a different bundle structure:
```python
LAUNCH_FILE = {
    'win': 'camoufox.exe',
    'mac': '../MacOS/camoufox',  # Fixed: app bundle structure
    'lin': 'camoufox-bin',
}
```

#### Commit: 581cb5c - Fixes & cleanup (Sept 17, 2024)

Early refinements to improve code quality and fix initial bugs discovered during testing.

#### Commit: 3488008 - Release to PyPI, Update README (Sept 19, 2024)

**First Public Release** - Published `camoufox` v0.1.0b1 to PyPI:

```toml
[tool.poetry]
name = "camoufox"
version = "0.1.0b1"
description = "Wrapper around Playwright to help launch Camoufox"
```

**Initial Dependencies:**
- playwright
- browserforge (for fingerprints)
- click (CLI)
- requests (downloads)
- orjson (fast JSON)
- pyyaml, platformdirs, tqdm, numpy

**CLI Commands Added:**
```bash
camoufox fetch    # Download browser binaries
camoufox remove   # Remove all files
camoufox path     # Show installation path
camoufox version  # Show version info
```

#### Commit: 49db177 - Fix unclear wording (Sept 19, 2024)

Documentation improvements for better user understanding.

#### Commit: 4eb559f - Fix out of bounds screen values v0.1.0b4 (Sept 19, 2024)

Fixed edge cases where BrowserForge could generate invalid screen dimensions:
```python
# Fix values that are out of bounds
if type_key.startswith("screen.") and isinstance(data, int) and data < 0:
    data = 0
```

#### Commit: f45852c - Add "test" CLI to open Playwright inspector (Sept 19, 2024)

Added development tool for testing:
```bash
camoufox test [url]  # Opens browser with Playwright inspector
```

This became invaluable for debugging fingerprint injection and testing websites.

#### Commit: 10b26ab - Bump to 0.1.1 (Sept 19, 2024)

First stable release v0.1.1 after beta testing.

#### Commit: e487754 - Re-enable BrowserForge userAgent v0.1.2 (Sept 19, 2024)

Initially disabled BrowserForge's user agent generation, then re-enabled it after Firefox version syncing was implemented.

#### Commit: f18b9a0 - Add block_images, block_webrtc, fixes, etc. v0.1.3 (Sept 19, 2024)

**Feature Expansion:**

```python
def NewBrowser(
    playwright: Playwright,
    *,
    block_images: bool = False,      # NEW: Block image loading
    block_webrtc: bool = True,       # NEW: Block WebRTC by default
    **kwargs
) -> Browser:
```

**Why These Features:**
- **block_images**: Faster page loads for scraping (70-80% bandwidth reduction)
- **block_webrtc**: Prevent IP leaks through WebRTC STUN/TURN servers

### Phase 2: Geolocation & Locale Intelligence (v0.2.0 - v0.2.15)

#### Commit: f6396c1 - Add locale, geolocation/locale from IP, & more v0.2.0 (Sept 29, 2024)

**MAJOR RELEASE** - This was transformational, adding 2,762 lines of code across 15 files.

**Revolutionary Feature: Automatic GeoIP-Based Configuration**

```python
from camoufox import Camoufox

with Camoufox(
    geoip=True,           # NEW: Auto-configure from IP
    proxy="socks5://proxy.example.com:1080"
) as browser:
    # Automatically sets:
    # - navigator.language based on country's language distribution
    # - navigator.languages with statistically correct fallbacks
    # - timezone (e.g., "America/New_York")
    # - geolocation latitude/longitude
    # - WebRTC IP spoofing
    pass
```

**New Files Created:**
- **ip.py**: Public IP detection, proxy support, IP validation
- **locale.py**: Statistical locale selection using CLDR data (274 lines)
- **territoryInfo.xml**: Unicode CLDR data for 249 territories

**Statistical Locale Selection:**

The library uses Unicode CLDR (Common Locale Data Repository) data to select locales based on real-world language distribution:

```python
class StatisticalLocaleSelector:
    """
    Selects a random locale based on statistical data.

    Example: For US IP addresses:
    - 95% chance: en-US (English)
    - 3% chance: es-US (Spanish)
    - 2% chance: other languages
    """

    def from_region(self, region: str) -> Locale:
        """Calculate weighted random selection"""
        languages, probabilities = self._load_territory_data(region)
        return np.random.choice(languages, p=probabilities)
```

**GeoIP Database Integration:**

```python
# Uses MaxMind GeoLite2-City database
def get_geolocation(ip: str) -> Geolocation:
    """
    Returns:
        - Locale (language-region, e.g., "en-US")
        - Longitude/Latitude (accurate to city level)
        - Timezone (IANA format, e.g., "America/Los_Angeles")
        - Accuracy (geolocation API accuracy in meters)
    """
```

**New Parameters:**
```python
geoip: bool = False              # Auto-configure from proxy IP
locale: str = None               # Manual locale (e.g., "en-US", "ja-JP")
allow_webgl: bool = True         # Toggle WebGL support
```

**Impact:**
- Reduced detection risk by **aligning all location-based signals**
- Made proxy usage dramatically simpler (1 parameter vs 5+)
- Leveraged real-world language statistics for realistic fingerprints

#### Commit: 7d825e5 - Typing & environ var fixes v0.2.1 (Sept 29, 2024)

Improved type hints and fixed environment variable handling for better IDE support.

#### Commit: ad55622 - Implement viewport hijacking v0.2.2 (Sept 30, 2024)

Added support for the new viewport hijacking feature in Camoufox beta.9:

```python
# Automatically adjusts viewport to match fingerprint
# Prevents window.innerWidth/innerHeight leaks
```

#### Commit: 7985eec - Allow fetch without geoip v0.2.3 (Oct 2, 2024)

Made the `geoip2` dependency optional for users who don't need geolocation features:

```bash
# Without GeoIP
pip install camoufox

# With GeoIP (recommended for proxy users)
pip install camoufox[geoip]
```

#### Commit: 79c436e - Add remote server launching #7 v0.2.5 (Oct 2, 2024)

**Remote Server Support** - Major feature for distributed scraping:

```python
# Launch a Playwright server
camoufox server

# Connect from another machine
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.firefox.connect("ws://remote-server:3000")
```

**Implementation:**
- **server.py**: New module for server launching
- **launchServer.js**: Node.js script leveraging Playwright's server
- Supports all `launch_options()` parameters
- Enables distributed scraping architectures

**Use Cases:**
- Scraping farms with centralized browser instances
- Docker/Kubernetes deployments
- Resource separation (CPU-intensive on server, logic on client)

#### Commit: 2832673 - Add persistent_context, humanize, etc. v0.2.5 (Oct 4, 2024)

**Persistent Browser Contexts:**

```python
from camoufox import NewBrowser

# Save cookies, localStorage, etc. between sessions
with sync_playwright() as p:
    context = NewBrowser(
        p,
        persistent_context=True,
        user_data_dir="/path/to/profile"  # Persistent storage
    )
```

**Returns:** `BrowserContext` instead of `Browser` when `persistent_context=True`

**Humanize Parameter:**

```python
with Camoufox(humanize=True) as browser:
    # Enables:
    # - Human-like cursor movement
    # - Natural mouse acceleration/deceleration
    # - Realistic click timing
    # - Random micro-movements
```

**Type Hints Enhancement:**
- Added `@overload` decorators for better IDE autocomplete
- Proper return type discrimination based on `persistent_context`

#### Commit: f3b68ab - Assert OS is valid v0.2.6 (Oct 5, 2024)

Added validation for the `os` parameter:
```python
def validate_os(os_param: str) -> None:
    valid_os = ['windows', 'macos', 'linux']
    if os_param not in valid_os:
        raise InvalidOS(f"Invalid OS: {os_param}")
```

#### Commit: 4a1cd7e - Add headless detection warning #26 v0.2.7 (Oct 6, 2024)

**Leak Warning System:**

Created a new warning system to educate users about detection risks:

```python
# warnings.py
class LeakWarning(RuntimeWarning):
    """Warns about settings that can cause detection"""

# warnings.yml
headless: |
  ⚠️ WARNING: Headless mode is enabled.
  Headless browsers are easily detectable and should be avoided.
  Consider using 'virtual' headless mode on Linux instead:

      with Camoufox(headless='virtual') as browser:
          ...
```

**Design Philosophy:**
- Educate users about detection risks
- Suggest better alternatives
- Allow power users to bypass: `i_know_what_im_doing=True`

#### Commit: 351e99e - Add more leak warnings v0.2.8 (Oct 6, 2024)

Expanded warning system with additional leak scenarios:
- Custom fingerprints (non-Firefox user agents)
- Missing geoip for proxy users
- Locale mismatches

#### Commit: bd59304 - Add Xvfb integration #26 v0.2.9 (Oct 8, 2024)

**Virtual Display Support for Linux Servers:**

```python
# virtdisplay.py - Minimal Xvfb wrapper
class VirtualDisplay:
    """
    Manages an Xvfb virtual display for headless Linux.
    Enables GPU acceleration and realistic rendering.
    """

# Usage
with Camoufox(headless='virtual') as browser:
    # Runs in Xvfb virtual display
    # - Full GPU/WebGL support
    # - Proper canvas rendering
    # - Undetectable (not headless mode)
```

**Advantages over Playwright's headless:**
- Uses real Firefox rendering engine (not headless mode)
- Full WebGL and Canvas support
- Matches windowed browser behavior exactly
- Automatic cleanup on browser close

**Implementation Details:**
```python
xvfb_args = (
    "-screen", "0", "1x1x24",        # Minimal screen
    "-ac",                            # Disable access control
    "-nolisten", "tcp",               # Security
    "+extension", "GLX",              # Enable OpenGL
    "-extension", "COMPOSITE",        # Disable compositing
    "-nocursor",                      # No cursor rendering
)
```

#### Commit: 76bedb5 - Hotfix screen property out of bounds v0.2.11 (Oct 9, 2024)

Fixed edge case where screen properties could be negative after calculations.

#### Commit: 3e2551e - Fix geolocation domain detection #31 v0.2.12 (Oct 10, 2024)

Fixed bug in locale domain detection for certain country codes.

#### Commit: 2167188 - Add dict expected type v0.2.15 (Oct 15, 2024)

Enhanced configuration validation to support dictionary-type properties.

### Phase 3: Proxy & Error Handling Improvements (v0.3.0 - v0.3.10)

#### Commit: f94452f - Error handling for invalid addon paths & proxies v0.3.3 (Oct 28, 2024)

**Robust Error Handling:**

```python
# addons.py
def confirm_paths(paths: List[str]) -> None:
    """Validates addon paths before launch"""
    for path in paths:
        if not os.path.isdir(path):
            raise InvalidAddonPath(path)
        if not os.path.exists(os.path.join(path, 'manifest.json')):
            raise InvalidAddonPath(
                'manifest.json is missing. '
                'Addon path must be an extracted addon directory.'
            )

# ip.py
def validate_proxy(proxy: Proxy) -> None:
    """Validates proxy format and connectivity"""
    if not proxy.server:
        raise InvalidProxy("Proxy server cannot be empty")
```

**Better Error Messages:**
- Clear explanation of what went wrong
- Suggestions for fixing the issue
- Examples of correct usage

#### Commit: 347885e - Default to http schema #57 v0.3.4 (Oct 29, 2024)

Auto-prefix proxy URLs without schema:
```python
# Before: Error if schema missing
proxy = "1.2.3.4:8080"  # ❌ Error

# After: Automatically adds http://
proxy = "1.2.3.4:8080"  # ✅ Works (becomes http://1.2.3.4:8080)
```

#### Commit: 692e8a1 - Support Python 3.8 #61 v0.3.5 (Oct 31, 2024)

**Python 3.8 Compatibility:**

Challenge: Backporting modern Python features to 3.8:

```python
# Issue: Path.is_relative_to() added in Python 3.9
# Solution: Manual path comparison
if not Path(filename).is_relative_to(current_module):  # ❌ 3.9+

# Fixed for 3.8:
try:
    Path(filename).relative_to(current_module)  # ✅ 3.8+
except ValueError:
    # Not relative
```

**Why Python 3.8:**
- Still widely used in enterprise environments
- Ubuntu 20.04 LTS ships with Python 3.8
- Broader compatibility without significant drawbacks

#### Commit: 33189cd - Fully fix #61 v0.3.6 (Oct 31, 2024)

Additional Python 3.8 compatibility fixes discovered through testing.

#### Commit: 964f490 - Fix broken import #70 v0.3.8 (Nov 4, 2024)

Fixed import error in certain installation scenarios.

#### Commit: 3a33283 - Add enable_cache, fixed font spacing, etc v0.3.9 (Nov 11, 2024)

**Browser Cache Control:**

```python
with Camoufox(enable_cache=True) as browser:
    # Enables in-memory and disk cache
    # Faster repeat visits to same domains
    # More realistic browsing behavior
```

**Font Rendering Improvements:**
- Fixed inconsistent font spacing in fingerprints
- Better matching of native OS font rendering

#### Commit: 74e0d08 - Proxy credentials are now optional (Nov 18, 2024)

Simplified proxy configuration:
```python
# Before: Required username/password even for public proxies
proxy = Proxy(server="1.2.3.4:8080", username="", password="")

# After: Optional credentials
proxy = Proxy(server="1.2.3.4:8080")  # No auth needed
```

### Phase 4: WebGL Fingerprinting (v0.4.0 - v0.4.4)

#### Commit: af937cc - WebGL rotation & leak fixes v0.4.0 (Nov 21, 2024)

**MAJOR FEATURE: WebGL Fingerprint Rotation**

This was one of the most complex features in the Python library, requiring:

1. **WebGL Sample Database** (274 KB SQLite database)
   - 1,000+ real WebGL fingerprints
   - OS-specific vendor/renderer combinations
   - Statistical distribution matching real-world hardware

2. **Intelligent Sampling:**
```python
# webgl/sample.py
def sample_webgl(os: str) -> Dict[str, str]:
    """
    Sample WebGL fingerprint based on OS probabilities.

    Returns:
        - UNMASKED_VENDOR_WEBGL (e.g., "Google Inc. (Intel)")
        - UNMASKED_RENDERER_WEBGL (e.g., "ANGLE (Intel, Intel(R) HD Graphics 620)")
        - WebGL parameters (36+ parameters)
    """
```

3. **Automatic OS Detection:**
```python
# Automatically matches WebGL vendor/renderer to OS
user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
# → Selects from Windows-compatible GPU list (NVIDIA, AMD, Intel)

user_agent = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)"
# → Selects from macOS GPU list (Intel, AMD)
```

**Database Structure:**
```sql
CREATE TABLE webgl_fingerprints (
    vendor TEXT,        -- GPU vendor
    renderer TEXT,      -- GPU model
    data JSON,         -- 36+ WebGL parameters
    win REAL,          -- Probability on Windows (0-1)
    mac REAL,          -- Probability on macOS (0-1)
    lin REAL           -- Probability on Linux (0-1)
);
```

**Example WebGL Data:**
```json
{
  "UNMASKED_VENDOR_WEBGL": "Google Inc. (NVIDIA)",
  "UNMASKED_RENDERER_WEBGL": "ANGLE (NVIDIA, NVIDIA GeForce GTX 1050 Ti)",
  "MAX_TEXTURE_SIZE": 16384,
  "MAX_RENDERBUFFER_SIZE": 16384,
  "MAX_VERTEX_ATTRIBS": 16,
  "MAX_VARYING_VECTORS": 30,
  // ... 30+ more parameters
}
```

#### Commit: 673fdc7 - Hotfix imports v0.4.1 (Nov 21, 2024)

Fixed import paths after WebGL module reorganization.

#### Commit: 01aff40 - Rollback WebGL fingerprint injection v0.4.2 (Nov 22, 2024)

Temporarily disabled WebGL injection due to stability issues discovered in production.

#### Commit: 145b737 - [Rollback] Disable WebGL by default #90 v0.4.3 (Nov 22, 2024)

Made WebGL rotation opt-in while investigating crashes:
```python
with Camoufox(allow_webgl=False) as browser:  # Default
    pass
```

#### Commit: cf3f8e6 - Fix WebGL injection causing crashing v0.4.4-beta (Nov 25, 2024)

**Root Cause Analysis:**

The WebGL parameter injection was causing Firefox to crash when:
1. Inconsistent WebGL parameters (e.g., MAX_TEXTURE_SIZE > GPU capability)
2. Invalid renderer strings for the actual GPU
3. Race conditions during WebGL context initialization

**Solution:**
- Enhanced parameter validation
- Added GPU capability checks
- Better error handling in injection code

### Phase 5: Advanced Features (v0.4.5 - v0.4.11)

#### Commit: 30a92ed - Do not override productSub #105 (Nov 28, 2024)

Fixed issue where `navigator.productSub` was being incorrectly overridden, causing detection.

#### Commit: 3210351 - Add parameter to only use user-specified fonts #109 (Nov 28, 2024)

**Font Control:**

```python
with Camoufox(
    fonts=['Arial', 'Times New Roman'],
    fonts_override=True  # NEW: Only use specified fonts
) as browser:
    # OS default fonts are NOT loaded
    # Useful for specific fingerprint matching
```

#### Commit: f6ef52a - Support for main world evaluation v0.4.5 (Dec 3, 2024)

**Main World Script Injection:**

```python
with Camoufox(main_world_scripts=True) as browser:
    # Scripts run in main world (page context)
    # Not in isolated world (extension context)
    # Enables more powerful fingerprint overrides
```

**Use Cases:**
- Overriding properties not accessible from isolated world
- Injecting before any page scripts run
- More robust anti-detection

**Trade-offs:**
- Slightly higher detection risk (detectable via Function.toString)
- Required for certain advanced spoofing techniques

#### Commit: 31963aa - Update WebGL sample database (Dec 4, 2024)

Updated WebGL database with 200+ additional real-world fingerprints.

#### Commit: 4c52518 - Cleanup & bump to 0.4.6 (Dec 4, 2024)

Code cleanup and documentation improvements.

#### Commit: 2422d62 - Auto offset Canvas anti-aliasing (Dec 9, 2024)

**Canvas Noise Intelligence:**

```python
# Automatically adjusts canvas noise based on anti-aliasing state
# Prevents detection via canvas fingerprint inconsistencies
```

Canvas fingerprints include:
- RGB channel noise patterns
- Anti-aliasing artifacts
- Text rendering variations

The library now ensures all canvas properties are consistent with the reported anti-aliasing setting.

#### Commit: 9a9e61f - Force Browserforge 1.2.1+ (Dec 11, 2024)

Updated to require BrowserForge 1.2.1+ for critical fingerprint improvements:
- Better mobile fingerprint generation
- Enhanced header consistency
- Updated User-Agent patterns

#### Commit: e3d3dcd - Look for assets in earlier releases #134 v0.4.9 (Dec 13, 2024)

**Backwards Compatibility for Binary Fetching:**

```python
# If current version's asset is missing, search previous releases
# Enables using newer pythonlib with older Camoufox versions
```

Useful for:
- Development/testing
- Pinning to specific Camoufox versions
- Gradual rollouts

#### Commit: a9b1934 - Add COOP toggle (workaround for #150) v0.4.10 (Jan 25, 2025)

**Cross-Origin-Opener-Policy Control:**

```python
with Camoufox(coop=False) as browser:
    # Disables COOP headers
    # Workaround for certain SPA websites
```

Some websites break when COOP is enabled due to:
- Cross-origin popups
- OAuth flows
- Embedded iframes

#### Commit: 8dc1f6b - Fix broken f-string formatting and broken python make edits call (Feb 11, 2025)

Fixed edge case in string formatting for certain locales.

### Summary Statistics

**52 commits** across **6 months** of development:

| Version Range | Commits | Key Features |
|--------------|---------|--------------|
| v0.1.0 - v0.1.3 | 10 | Foundation, CLI, PyPI release |
| v0.2.0 - v0.2.15 | 15 | GeoIP, Locales, Xvfb, Remote server |
| v0.3.0 - v0.3.10 | 11 | Proxy improvements, Python 3.8, Cache |
| v0.4.0 - v0.4.4 | 6 | WebGL rotation (complex feature) |
| v0.4.5 - v0.4.11 | 10 | Main world, COOP, Font control |

**Lines of Code Added:**
- Core library: ~8,000 lines
- Tests: ~500 lines
- Documentation: ~2,000 lines
- Data files (XML, JSON, SQLite): ~3 MB

---

## Architecture

### Package Structure

```
pythonlib/
├── camoufox/                    # Main package
│   ├── __init__.py             # Public API exports
│   ├── __main__.py             # CLI entry point
│   ├── __version__.py          # Version constraints
│   │
│   ├── sync_api.py             # Synchronous Playwright API
│   ├── async_api.py            # Asynchronous Playwright API
│   │
│   ├── utils.py                # Core fingerprint & config logic
│   ├── fingerprints.py         # BrowserForge integration
│   ├── addons.py               # Extension management
│   │
│   ├── locale.py               # Locale & geolocation intelligence
│   ├── ip.py                   # IP detection & proxy handling
│   ├── territoryInfo.xml       # CLDR data (249 territories)
│   │
│   ├── server.py               # Remote server launching
│   ├── launchServer.js         # Node.js server script
│   │
│   ├── pkgman.py               # Binary download & updates
│   ├── virtdisplay.py          # Xvfb virtual display
│   │
│   ├── warnings.py             # Leak warning system
│   ├── warnings.yml            # Warning messages
│   ├── exceptions.py           # Custom exceptions
│   │
│   ├── fonts.json              # OS-specific font lists
│   ├── browserforge.yml        # BrowserForge property mapping
│   ├── setup.cfg               # Python config
│   ├── py.typed                # Type hint marker
│   │
│   └── webgl/                  # WebGL fingerprinting
│       ├── __init__.py
│       ├── sample.py           # WebGL sampling logic
│       └── webgl_data.db       # SQLite database (1000+ fingerprints)
│
├── pyproject.toml              # Package metadata & dependencies
├── README.md                   # Documentation
└── publish.sh                  # PyPI publishing script
```

### Core Modules and Their Purposes

#### 1. API Layer (`sync_api.py` / `async_api.py`)

**Purpose:** Clean, Pythonic wrappers around Playwright's Firefox launcher.

**Key Classes:**

```python
# sync_api.py
class Camoufox(PlaywrightContextManager):
    """
    Context manager that automatically:
    1. Starts Playwright
    2. Generates fingerprints
    3. Launches browser
    4. Cleans up on exit
    """

def NewBrowser(playwright: Playwright, **kwargs) -> Browser:
    """
    Manual browser launching for advanced users.
    Returns Browser or BrowserContext (if persistent_context=True)
    """
```

**Design Pattern: Context Manager**
```python
# Automatic lifecycle management
with Camoufox(headless=False) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
    # Automatically closes browser on exit
```

**Type Safety with Overloads:**
```python
@overload
def NewBrowser(persistent_context: Literal[False]) -> Browser: ...

@overload
def NewBrowser(persistent_context: Literal[True]) -> BrowserContext: ...
```

#### 2. Fingerprint Generation (`fingerprints.py`)

**Purpose:** Bridge between BrowserForge and Camoufox's configuration system.

**Key Functions:**

```python
def generate_fingerprint(
    window: Optional[Tuple[int, int]] = None,
    **config
) -> Fingerprint:
    """
    Generates a Firefox fingerprint using BrowserForge.

    Args:
        window: Custom (width, height) for outer window
        **config: BrowserForge configuration (screen, os, locale, etc.)

    Returns:
        Fingerprint object with:
        - navigator (userAgent, platform, etc.)
        - screen (dimensions, colorDepth, etc.)
        - headers (HTTP headers)
    """

def from_browserforge(
    fingerprint: Fingerprint,
    ff_version: Optional[str] = None
) -> Dict[str, Any]:
    """
    Converts BrowserForge fingerprint to Camoufox config properties.

    Maps:
        fingerprint.navigator.userAgent → "navigator.userAgent"
        fingerprint.screen.width        → "screen.width"
        fingerprint.navigator.language  → "navigator.language"
        ... (50+ mappings)
    """
```

**Property Mapping (`browserforge.yml`):**
```yaml
navigator:
  userAgent: "navigator.userAgent"
  platform: "navigator.platform"
  hardwareConcurrency: "navigator.hardwareConcurrency"

screen:
  width: "screen.width"
  height: "screen.height"
  availWidth: "screen.availWidth"
  # ... 15+ screen properties
```

#### 3. Configuration Management (`utils.py`)

**Purpose:** Central orchestration of all fingerprint configuration.

**Key Function:**

```python
def launch_options(
    # Fingerprint generation
    config: Optional[Dict[str, Any]] = None,
    fingerprint: Optional[Fingerprint] = None,
    os: Optional[ListOrString] = None,

    # Addons
    addons: Optional[List[str]] = None,
    exclude_addons: Optional[List[DefaultAddons]] = None,

    # Geolocation & Locale
    geoip: Union[bool, str] = False,
    locale: Optional[str] = None,

    # Display
    headless: Optional[bool] = None,
    window: Optional[Tuple[int, int]] = None,

    # WebGL
    webgl: Optional[Dict[str, str]] = None,
    allow_webgl: bool = True,

    # Proxy
    proxy: Optional[Union[Dict[str, str], Proxy]] = None,

    # Advanced
    persistent_context: bool = False,
    user_data_dir: Optional[str] = None,
    humanize: Optional[bool] = None,
    fonts: Optional[List[str]] = None,
    fonts_override: bool = False,
    enable_cache: bool = False,

    # Playwright passthrough
    args: Optional[List[str]] = None,
    executable_path: Optional[str] = None,
    **kwargs
) -> Dict[str, Any]:
    """
    Central configuration orchestration.

    Process:
    1. Generate/validate fingerprint
    2. Apply GeoIP-based locale/timezone
    3. Configure WebGL
    4. Setup addons
    5. Handle proxy
    6. Set up virtual display (if needed)
    7. Return Playwright launch options
    """
```

**Configuration Validation:**
```python
def validate_config(config_map: Dict[str, str]) -> None:
    """
    Validates against properties.json schema.

    Checks:
    - Property exists
    - Type matches (str, int, uint, double, bool, array, dict)
    - Value ranges
    """
    property_types = _load_properties()
    for key, value in config_map.items():
        expected_type = property_types.get(key)
        if not expected_type:
            raise UnknownProperty(f"Unknown property: {key}")
        if not validate_type(value, expected_type):
            raise InvalidPropertyType(...)
```

#### 4. Geolocation Intelligence (`locale.py`)

**Purpose:** Statistical locale selection and GeoIP-based configuration.

**Key Components:**

```python
@dataclass
class Locale:
    language: str          # ISO 639-1 code (e.g., "en")
    region: Optional[str]  # ISO 3166-1 code (e.g., "US")
    script: Optional[str]  # ISO 15924 code (e.g., "Latn")

    @property
    def as_string(self) -> str:
        return f"{self.language}-{self.region}"  # "en-US"

@dataclass
class Geolocation:
    locale: Locale
    longitude: float
    latitude: float
    timezone: str           # IANA timezone (e.g., "America/New_York")
    accuracy: Optional[float] = None
```

**Statistical Locale Selector:**

```python
class StatisticalLocaleSelector:
    """
    Uses Unicode CLDR data to select locales based on real-world distribution.

    Data Source: territoryInfo.xml (Unicode Common Locale Data Repository)
    - 249 territories
    - 7,900+ language populations
    - Population percentages
    - Literacy rates
    """

    def from_region(self, region: str) -> Locale:
        """
        Example: region="US"

        Returns weighted random selection:
        - "en-US" (95.4% probability)
        - "es-US" (3.2% probability)
        - "zh-US" (0.8% probability)
        - ... (15+ languages)
        """

    def from_language(self, language: str) -> Locale:
        """
        Example: language="en"

        Returns weighted random selection:
        - "en-US" (65% probability)
        - "en-GB" (15% probability)
        - "en-CA" (8% probability)
        - ... (75+ English-speaking territories)
        """
```

**GeoIP Database Management:**

```python
# MaxMind GeoLite2-City database
MMDB_FILE = LOCAL_DATA / 'GeoLite2-City.mmdb'

def download_mmdb() -> None:
    """Downloads latest GeoIP database from GitHub mirror"""

def get_geolocation(ip: str) -> Geolocation:
    """
    Query GeoIP database for IP address.

    Returns:
    - City-level location (accuracy: ±50km)
    - Timezone (accurate)
    - Registered country
    - ISP information
    """
```

#### 5. IP Detection & Proxy Handling (`ip.py`)

**Purpose:** Detect public IP and manage proxy configuration.

**Key Components:**

```python
@dataclass
class Proxy:
    server: str                      # URL with optional port
    username: Optional[str] = None
    password: Optional[str] = None
    bypass: Optional[str] = None     # Bypass list (e.g., "localhost")

    def as_string(self) -> str:
        """Format as full URL with auth"""
        # http://user:pass@1.2.3.4:8080

def public_ip(proxy: Optional[str] = None) -> str:
    """
    Detect public IP address.

    Tries multiple services:
    - api.ipify.org
    - checkip.amazonaws.com
    - ipinfo.io
    - icanhazip.com
    - ifconfig.co
    - ipecho.net

    Fallback strategy ensures high reliability.
    """
```

#### 6. Package Management (`pkgman.py`)

**Purpose:** Download, update, and manage Camoufox binaries.

**Key Components:**

```python
@dataclass
class Version:
    release: str      # e.g., "beta.19" or "1.0.0"
    version: str      # Firefox version (e.g., "133.0")

    def is_supported(self) -> bool:
        """Check if version is within supported range"""

class CamoufoxFetcher(GitHubDownloader):
    """
    Manages Camoufox binary downloads from GitHub releases.

    Features:
    - Multi-architecture support (x86_64, arm64, i686)
    - Multi-OS support (Windows, macOS, Linux)
    - Asset verification
    - Progress bars (tqdm)
    - Automatic extraction
    """

    def install(self) -> None:
        """
        1. Detect OS and architecture
        2. Find matching release asset
        3. Download with progress bar
        4. Extract to cache directory
        5. Verify installation
        """
```

**Installation Paths:**

```python
# Linux:   ~/.cache/camoufox/
# macOS:   ~/Library/Caches/camoufox/
# Windows: %LOCALAPPDATA%\camoufox\

INSTALL_DIR = Path(user_cache_dir("camoufox"))
```

#### 7. Virtual Display (`virtdisplay.py`)

**Purpose:** Manage Xvfb virtual displays for headless Linux servers.

**Key Features:**

```python
class VirtualDisplay:
    """
    Minimal Xvfb wrapper for true headless browsing.

    Advantages over Playwright's headless:
    - Uses windowed Firefox (not headless mode)
    - Full GPU/WebGL support
    - Identical rendering to desktop
    - Undetectable
    """

    xvfb_args = (
        "-screen", "0", "1x1x24",        # Minimal 1x1 screen
        "+extension", "GLX",              # Enable OpenGL
        "-extension", "COMPOSITE",        # Disable compositing (faster)
        "-nocursor",                      # No cursor rendering
        "-br",                            # Black background
    )

    def get(self) -> str:
        """
        Start Xvfb and return DISPLAY value.

        Returns: ":99" (or next available display)
        """

    def kill(self):
        """Terminate Xvfb process"""
```

**Display Allocation:**
```python
def _free_display() -> int:
    """
    Find free X11 display number.

    Checks /tmp/.X*-lock files.
    Returns: 99-199 (random to avoid conflicts)
    """
```

#### 8. Addon Management (`addons.py`)

**Purpose:** Download and manage browser extensions.

```python
class DefaultAddons(Enum):
    """
    Default extensions bundled with Camoufox.

    UBO: uBlock Origin (ad blocker)
    BPC: Bypass Paywalls Clean (optional)
    """
    UBO = "https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi"

def add_default_addons(
    addons_list: List[str],
    exclude_list: Optional[List[DefaultAddons]] = None
) -> None:
    """
    Download and extract addons to local cache.

    Process:
    1. Check if already downloaded
    2. Download .xpi file
    3. Extract to directory
    4. Validate manifest.json
    5. Add to addons_list
    """
```

#### 9. WebGL Fingerprinting (`webgl/sample.py`)

**Purpose:** Generate realistic WebGL fingerprints based on OS.

**Database Schema:**

```sql
CREATE TABLE webgl_fingerprints (
    vendor TEXT,        -- UNMASKED_VENDOR_WEBGL
    renderer TEXT,      -- UNMASKED_RENDERER_WEBGL
    data JSON,         -- Full WebGL parameters
    win REAL,          -- Windows probability
    mac REAL,          -- macOS probability
    lin REAL           -- Linux probability
);
```

**Sampling Logic:**

```python
def sample_webgl(
    os: str,
    vendor: Optional[str] = None,
    renderer: Optional[str] = None
) -> Dict[str, str]:
    """
    Sample WebGL fingerprint.

    Args:
        os: 'win', 'mac', or 'lin'
        vendor: Optional specific vendor (e.g., "Intel Inc.")
        renderer: Optional specific renderer

    Returns:
        Dictionary with 36+ WebGL parameters:
        - UNMASKED_VENDOR_WEBGL
        - UNMASKED_RENDERER_WEBGL
        - MAX_TEXTURE_SIZE
        - MAX_RENDERBUFFER_SIZE
        - ... (30+ more)

    Algorithm:
    1. Query database for OS-specific fingerprints
    2. Normalize probabilities
    3. Sample using numpy.random.choice()
    4. Return matching fingerprint
    """
```

#### 10. Warning System (`warnings.py`)

**Purpose:** Educate users about detection risks.

```python
class LeakWarning(RuntimeWarning):
    """
    Custom warning category for detection risks.

    Features:
    - Clear explanations
    - Suggested alternatives
    - Optional suppression (i_know_what_im_doing=True)
    - Proper stack trace attribution
    """

    @staticmethod
    def warn(warning_key: str, i_know_what_im_doing: Optional[bool] = None):
        """
        Emit warning from warnings.yml.

        Stack trace handling:
        - Walks up call stack
        - Finds first frame outside camoufox package
        - Attributes warning to user's code
        """
```

**Warning Definitions (`warnings.yml`):**

```yaml
headless: |
  ⚠️ WARNING: Headless mode is enabled.
  Headless browsers are easily detectable. Consider using 'virtual' mode on Linux.

no_region: |
  ⚠️ WARNING: Locale specified without region.
  This may cause inconsistencies. Consider using a full locale like "en-US".

custom_fingerprint: |
  ⚠️ WARNING: Custom fingerprint provided.
  Make sure it's a Firefox fingerprint. Non-Firefox fingerprints WILL be detected.
```

### BrowserForge Integration

**BrowserForge** is the fingerprint generation engine used by Camoufox.

**Integration Points:**

1. **Fingerprint Generation:**
```python
from browserforge.fingerprints import FingerprintGenerator

FP_GENERATOR = FingerprintGenerator(
    browser='firefox',
    os=('linux', 'macos', 'windows')
)

fingerprint = FP_GENERATOR.generate(
    screen=Screen(max_width=1920, max_height=1080),
    locale='en-US'
)
```

2. **Property Mapping:**
```python
# browserforge.yml defines how to map BrowserForge properties
# to Camoufox configuration properties

navigator:
  userAgent: "navigator.userAgent"              # Direct mapping
  platform: "navigator.platform"

screen:
  width: "screen.width"
  height: "screen.height"

# Special handling:
headers:                                        # HTTP headers
  "User-Agent": "ignore"                       # Skip (uses navigator.userAgent)
  "Accept-Language": "headers.acceptLanguage"
```

3. **Version Synchronization:**
```python
# Replace BrowserForge's Firefox version with current Camoufox version
if ff_version and isinstance(data, str):
    # Replace patterns like "131.0" with actual Firefox version
    data = re.sub(
        r'(?<!\d)(1[0-9]{2})(\.0)(?!\d)',
        rf'{ff_version}\2',
        data
    )
```

### Configuration Management

**Configuration Flow:**

```
User Parameters
     ↓
launch_options()
     ↓
├─→ Generate/validate fingerprint
├─→ Apply GeoIP configuration
├─→ Configure WebGL
├─→ Set up addons
├─→ Handle proxy
├─→ Set up virtual display
└─→ Build Playwright launch options
     ↓
Environment Variables (CAMOU_CONFIG_1, CAMOU_CONFIG_2, ...)
     ↓
Firefox Process
     ↓
C++ Configuration Loader
     ↓
Properties Applied via Juggler Protocol
```

**Environment Variable Encoding:**

```python
def get_env_vars(config_map: Dict[str, str], user_agent_os: str) -> Dict[str, str]:
    """
    Encodes configuration as environment variables.

    Why environment variables:
    - Secure (not visible in process arguments)
    - Large data support (32KB+ on Linux, 2KB on Windows)
    - No file I/O required

    Chunking:
    - Windows: 2047 bytes per chunk (CMD limit)
    - Linux/macOS: 32767 bytes per chunk (shell limit)

    Result:
    CAMOU_CONFIG_1={"navigator.userAgent":"Mozilla...","screen.width":1920,...}
    CAMOU_CONFIG_2={..."timezone":"America/New_York",...}
    """
```

**Configuration Validation:**

Every configuration property is validated against `properties.json`:

```json
[
  {
    "property": "navigator.userAgent",
    "type": "str"
  },
  {
    "property": "screen.width",
    "type": "uint"
  },
  {
    "property": "geolocation:accuracy",
    "type": "double"
  }
]
```

### Process Management

**Browser Lifecycle:**

```python
# Synchronous API
with Camoufox(headless='virtual') as browser:
    # 1. Start Playwright
    # 2. Start Xvfb (if virtual display)
    # 3. Generate fingerprint
    # 4. Launch Firefox
    page = browser.new_page()
    # ... use browser ...
# 5. Close browser
# 6. Terminate Xvfb
# 7. Close Playwright

# Asynchronous API
async with AsyncCamoufox(headless='virtual') as browser:
    # Same lifecycle, but with async/await
    page = await browser.new_page()
```

**Virtual Display Attachment:**

```python
def sync_attach_vd(browser_or_context, virtual_display):
    """
    Attach virtual display cleanup to browser close event.

    Problem: Xvfb must be killed when browser closes.
    Solution: Monkey-patch browser.close() method.
    """
    if virtual_display:
        original_close = browser_or_context.close

        def close(*args, **kwargs):
            virtual_display.kill()  # Kill Xvfb first
            original_close(*args, **kwargs)  # Then close browser

        browser_or_context.close = close

    return browser_or_context
```

---

## Key Features Evolution

This section tracks how major features evolved through commits.

### 1. BrowserForge Fingerprint Injection

**Initial Implementation (v0.1.0):**

```python
# 5e1fb78 - Camoufox Python interface
def launch_options(fingerprint: Optional[Fingerprint] = None):
    if not fingerprint:
        fingerprint = generate_fingerprint()

    config = from_browserforge(fingerprint)
    return {'config': config}
```

**Enhancements:**

- **v0.1.2** (e487754): Re-enabled BrowserForge userAgent after version syncing
- **v0.2.0** (f6396c1): Use current Camoufox Firefox version instead of BrowserForge's
- **v0.3.9** (3a33283): Fixed font spacing consistency

**Current State (v0.4.11):**

Generates 50+ fingerprint properties including:
- navigator.* (userAgent, platform, hardwareConcurrency, etc.)
- screen.* (dimensions, colorDepth, orientation, etc.)
- window.* (inner/outer dimensions, screenX/Y)
- fonts (OS-specific font lists)
- timezone (auto-detected or manual)
- locale (language, region, script)

### 2. GeoIP-Based Locale/Geolocation

**Initial Implementation (v0.2.0):**

```python
# f6396c1 - Add locale, geolocation/locale from IP
def handle_geoip(geoip: bool, proxy: Optional[Proxy]) -> Geolocation:
    # Detect IP
    ip = public_ip(proxy.as_string() if proxy else None)

    # Query GeoIP database
    geo = get_geolocation(ip)

    # Return locale, timezone, lat/long
    return geo
```

**Key Files Added:**
- `locale.py` (274 lines): Statistical locale selection
- `ip.py` (121 lines): IP detection and proxy handling
- `territoryInfo.xml` (2,024 lines): CLDR data

**Enhancements:**

- **v0.2.1** (7d825e5): Better error handling for unknown IPs
- **v0.2.3** (7985eec): Made geoip2 optional dependency
- **v0.2.12** (3e2551e): Fixed geolocation domain detection
- **v0.2.13**: Support for partial locales (language or region only)

**Statistical Selection Algorithm:**

```python
# Example: US IP address
def from_region("US") -> Locale:
    # Load language distribution from territoryInfo.xml
    languages = ["en", "es", "zh", "tl", "vi", "ar", ...]
    percentages = [95.4, 3.2, 0.8, 0.3, 0.2, 0.1, ...]

    # Weighted random selection
    language = np.random.choice(languages, p=percentages)
    # Returns "en-US" 95.4% of the time
```

**Impact:**
- Reduced manual configuration from 5+ parameters to 1 (`geoip=True`)
- Ensured consistency between timezone, locale, and geolocation
- Prevented common detection vector (timezone mismatch)

### 3. Xvfb Integration for Headless Linux

**Problem:**

Playwright's headless mode has several detection vectors:
- Missing WebGL support
- Different canvas rendering
- Detectable via `navigator.webdriver`
- Inconsistent behavior vs. headed mode

**Solution (v0.2.9):**

```python
# bd59304 - Add Xvfb integration
class VirtualDisplay:
    """
    Run real Firefox in virtual X11 display.
    Identical to windowed mode, but no physical display needed.
    """

# Usage
with Camoufox(headless='virtual') as browser:
    # Runs in Xvfb
    # Full WebGL, Canvas, GPU support
    # Undetectable
```

**Implementation Details:**

```python
# Xvfb configuration
xvfb_args = (
    "-screen", "0", "1x1x24",        # 1x1 screen (minimal)
    "+extension", "GLX",              # Enable OpenGL/WebGL
    "-extension", "COMPOSITE",        # Disable compositing (faster)
    "-nocursor",                      # No cursor needed
)

# Automatic display number selection
def _free_display() -> int:
    # Scan /tmp/.X*-lock files
    # Find unused display number
    # Return 99-199 (avoids conflicts)
```

**Lifecycle Management (v0.3.3):**

```python
# 75ea7b0 - Kill Xvfb on browser close
def sync_attach_vd(browser, virtual_display):
    original_close = browser.close

    def close(*args, **kwargs):
        virtual_display.kill()      # Kill Xvfb first
        original_close(*args, **kwargs)

    browser.close = close
```

**Benefits:**
- True headless (no display required)
- Full rendering capabilities
- Undetectable (uses windowed Firefox)
- Automatic cleanup

### 4. Proxy Support with Authentication

**Initial Implementation (v0.2.0):**

```python
# ip.py
@dataclass
class Proxy:
    server: str
    username: str
    password: str

    def as_string(self) -> str:
        return f"http://{self.username}:{self.password}@{self.server}"
```

**Evolution:**

- **v0.3.3** (f94452f): Added proxy validation and error handling
- **v0.3.4** (347885e): Auto-add `http://` schema if missing
- **v0.3.10** (74e0d08): Made credentials optional

**Current Implementation:**

```python
# Flexible proxy configuration
Proxy(server="1.2.3.4:8080")                    # No auth
Proxy(server="1.2.3.4:8080", username="user")   # Username only
Proxy(server="http://user:pass@1.2.3.4:8080")   # Full URL
```

**Proxy + GeoIP Integration:**

```python
with Camoufox(
    geoip=True,
    proxy=Proxy(server="socks5://us-proxy:1080")
) as browser:
    # Automatically:
    # 1. Detects IP through proxy
    # 2. Queries GeoIP for location
    # 3. Sets locale, timezone, geolocation
    # 4. Configures WebRTC to show proxy IP
```

### 5. Remote Server Launching

**Implementation (v0.2.5):**

```python
# 79c436e - Add remote server launching
# server.py
def launch_server(**kwargs) -> NoReturn:
    """
    Launch Playwright server with Camoufox configuration.

    Process:
    1. Generate launch options
    2. Serialize to JSON
    3. Launch Node.js server script
    4. Print WebSocket endpoint
    5. Wait forever
    """
    config = launch_options(**kwargs)
    nodejs = get_nodejs()  # Use Playwright's bundled Node.js

    subprocess.Popen([
        nodejs,
        str(LAUNCH_SCRIPT),
    ], stdin=config_json)
```

**Node.js Server Script (`launchServer.js`):**

```javascript
// Reads config from stdin
// Launches Playwright server
// Prints WebSocket URL

const { chromium, firefox } = require('playwright-core');
const config = JSON.parse(process.stdin.read());

const server = await firefox.launchServer(config);
console.log(server.wsEndpoint());
```

**Client-Side Usage:**

```python
# On machine A
$ camoufox server
WebSocket endpoint: ws://machine-a:3000/ws

# On machine B
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.firefox.connect("ws://machine-a:3000/ws")
    page = browser.new_page()
```

**Use Cases:**
- Distributed scraping (separate logic from browser)
- Docker/Kubernetes deployments
- Resource isolation (browser on powerful server)
- Centralized fingerprint management

### 6. Persistent Context

**Implementation (v0.2.5):**

```python
# 2832673 - Add persistent_context
with sync_playwright() as p:
    context = NewBrowser(
        p,
        persistent_context=True,
        user_data_dir="/path/to/profile"
    )

    # Returns BrowserContext instead of Browser
    # Saves cookies, localStorage, indexedDB, etc.
    page = context.new_page()
```

**Type Safety:**

```python
# Using overloads for proper type hints
@overload
def NewBrowser(persistent_context: Literal[False]) -> Browser: ...

@overload
def NewBrowser(persistent_context: Literal[True]) -> BrowserContext: ...
```

**Benefits:**
- Persistent cookies (login sessions)
- localStorage/indexedDB (app state)
- Service workers (PWAs)
- Cached resources

**Trade-offs:**
- Larger disk footprint
- Potential fingerprint leakage (reused profiles)
- Cleanup responsibility on user

### 7. Main World Evaluation

**Implementation (v0.4.5):**

```python
# f6ef52a - Support for main world evaluation
with Camoufox(main_world_scripts=True) as browser:
    # Scripts run in page context (main world)
    # Not isolated world (extension context)
```

**Isolated World vs. Main World:**

```
Isolated World (default):
- Safe: Page scripts can't detect injection
- Limited: Can't override certain properties
- Used by: Browser extensions

Main World:
- Powerful: Can override any property
- Detectable: Function.toString() shows native code
- Used by: Advanced anti-detection
```

**Example:**

```javascript
// Isolated World (default)
// Cannot override window.screen
Object.defineProperty(window, 'screen', { ... });  // ❌ Fails

// Main World
// Can override window.screen
Object.defineProperty(window, 'screen', { ... });  // ✅ Works
```

**Use Cases:**
- Overriding deeply nested properties
- Injecting before page scripts
- Advanced anti-detection techniques

**Trade-offs:**
- Slightly higher detection risk
- Required for certain overrides

### 8. WebGL Fingerprint Rotation

**Implementation (v0.4.0):**

This was the most complex feature, spanning multiple commits:

- **af937cc** (Nov 21): Initial WebGL rotation implementation
- **673fdc7** (Nov 21): Hotfix imports
- **01aff40** (Nov 22): Rollback due to crashes
- **145b737** (Nov 22): Disable by default
- **cf3f8e6** (Nov 25): Fix crashes
- **31963aa** (Dec 4): Update WebGL database

**Architecture:**

```
webgl/
├── __init__.py
├── sample.py              # Sampling logic
└── webgl_data.db          # SQLite database (274 KB)

Database Schema:
CREATE TABLE webgl_fingerprints (
    vendor TEXT,           # GPU vendor
    renderer TEXT,         # GPU model
    data JSON,            # 36+ parameters
    win REAL,             # Windows probability
    mac REAL,             # macOS probability
    lin REAL              # Linux probability
);
```

**Sampling Algorithm:**

```python
def sample_webgl(os: str) -> Dict[str, str]:
    # 1. Query database for OS-specific fingerprints
    cursor.execute(
        f'SELECT vendor, renderer, data, {os} FROM webgl_fingerprints '
        f'WHERE {os} > 0'
    )

    # 2. Extract probabilities
    probs = np.array([row[3] for row in results])

    # 3. Normalize probabilities
    probs = probs / probs.sum()

    # 4. Weighted random selection
    idx = np.random.choice(len(probs), p=probs)

    # 5. Return fingerprint
    return orjson.loads(results[idx][2])
```

**WebGL Parameters (36+):**

```json
{
  "UNMASKED_VENDOR_WEBGL": "Google Inc. (NVIDIA)",
  "UNMASKED_RENDERER_WEBGL": "ANGLE (NVIDIA, NVIDIA GeForce GTX 1050 Ti)",

  "MAX_TEXTURE_SIZE": 16384,
  "MAX_RENDERBUFFER_SIZE": 16384,
  "MAX_VERTEX_ATTRIBS": 16,
  "MAX_VARYING_VECTORS": 30,
  "MAX_VERTEX_TEXTURE_IMAGE_UNITS": 16,
  "MAX_COMBINED_TEXTURE_IMAGE_UNITS": 32,
  "MAX_TEXTURE_IMAGE_UNITS": 16,
  "MAX_FRAGMENT_UNIFORM_VECTORS": 1024,
  "MAX_VERTEX_UNIFORM_VECTORS": 1024,
  "MAX_CUBE_MAP_TEXTURE_SIZE": 16384,
  "MAX_VIEWPORT_DIMS": [32767, 32767],

  // ... 24+ more parameters
}
```

**Challenges Solved:**

1. **Crash Prevention:** Validate all parameters before injection
2. **OS Consistency:** Only use GPU combinations valid for target OS
3. **Statistical Distribution:** Match real-world GPU market share
4. **Performance:** Use SQLite for fast lookups

**Impact:**
- 1,000+ unique WebGL fingerprints
- OS-specific GPU selection
- Realistic parameter combinations
- Rotation for each browser session

### 9. Version Range Control

**Implementation (v0.3.1):**

```python
# __version__.py
class CONSTRAINTS:
    MIN_VERSION = 'beta.19'     # Minimum supported
    MAX_VERSION = '1'           # Maximum supported (exclusive)

    @staticmethod
    def as_range() -> str:
        return f">={CONSTRAINTS.MIN_VERSION}, <{CONSTRAINTS.MAX_VERSION}"

# Validation
def is_supported() -> bool:
    current = Version.from_path()
    return VERSION_MIN <= current < VERSION_MAX
```

**Version Comparison:**

```python
@total_ordering
@dataclass
class Version:
    release: str      # "beta.19" or "1.0.0"
    version: str      # "133.0"

    def __lt__(self, other) -> bool:
        # Convert to sortable tuple
        # "beta.19" → (98, 19, 0, 0, 0)  # 'b' = 98
        # "1.0.0"   → (1, 0, 0, 0, 0)
        return self.sorted_rel < other.sorted_rel
```

**Benefits:**
- Prevents incompatibility issues
- Clear error messages
- Automatic update prompts

**Example Error:**

```
UnsupportedVersion: Camoufox version v0.3.0 is not supported.
Supported range: >=beta.19, <1
Please run: camoufox fetch
```

---

## API Design

### Sync vs Async APIs

The library provides parallel sync and async APIs with identical signatures.

**Synchronous API:**

```python
from camoufox import Camoufox, NewBrowser
from playwright.sync_api import sync_playwright

# Option 1: Context manager (recommended)
with Camoufox(headless=False) as browser:
    page = browser.new_page()
    page.goto("https://example.com")

# Option 2: Manual management
with sync_playwright() as p:
    browser = NewBrowser(p, headless=False)
    page = browser.new_page()
    page.goto("https://example.com")
    browser.close()
```

**Asynchronous API:**

```python
from camoufox import AsyncCamoufox, AsyncNewBrowser
from playwright.async_api import async_playwright
import asyncio

# Option 1: Context manager (recommended)
async def main():
    async with AsyncCamoufox(headless=False) as browser:
        page = await browser.new_page()
        await page.goto("https://example.com")

# Option 2: Manual management
async def main():
    async with async_playwright() as p:
        browser = await AsyncNewBrowser(p, headless=False)
        page = await browser.new_page()
        await page.goto("https://example.com")
        await browser.close()

asyncio.run(main())
```

**When to Use Each:**

| Synchronous | Asynchronous |
|------------|--------------|
| Simple scripts | High-performance scraping |
| Sequential operations | Concurrent operations |
| Easier debugging | Better resource usage |
| Lower CPU overhead | Handles 100+ browsers |

### Context Managers

**Design Philosophy:**

```python
# Automatic resource management
with Camoufox() as browser:
    # Resources allocated:
    # - Playwright instance
    # - Fingerprint generated
    # - Browser launched
    # - Virtual display (if applicable)

    pass  # Use browser

# Resources automatically cleaned up:
# - Browser closed
# - Virtual display terminated
# - Playwright stopped
```

**Implementation:**

```python
class Camoufox(PlaywrightContextManager):
    def __enter__(self) -> Browser:
        # Start Playwright
        super().__enter__()

        # Launch browser with fingerprint
        self.browser = NewBrowser(self._playwright, **self.launch_options)
        return self.browser

    def __exit__(self, *args):
        # Close browser
        if self.browser:
            self.browser.close()

        # Stop Playwright
        super().__exit__(*args)
```

**Async Context Manager:**

```python
class AsyncCamoufox(PlaywrightContextManager):
    async def __aenter__(self) -> Browser:
        _playwright = await super().__aenter__()
        self.browser = await AsyncNewBrowser(_playwright, **self.launch_options)
        return self.browser

    async def __aexit__(self, *args):
        if self.browser:
            await self.browser.close()
        await super().__aexit__(*args)
```

### Configuration System

**Unified Configuration Function:**

```python
def launch_options(
    # Core configuration
    config: Optional[Dict[str, Any]] = None,
    fingerprint: Optional[Fingerprint] = None,

    # Display
    headless: Optional[bool] = None,
    window: Optional[Tuple[int, int]] = None,

    # Locale & Geolocation
    geoip: Union[bool, str] = False,
    locale: Optional[str] = None,

    # Addons
    addons: Optional[List[str]] = None,
    exclude_addons: Optional[List[DefaultAddons]] = None,

    # Proxy
    proxy: Optional[Union[Dict, Proxy]] = None,

    # WebGL
    webgl: Optional[Dict[str, str]] = None,
    allow_webgl: bool = True,

    # Advanced
    os: Optional[ListOrString] = None,
    fonts: Optional[List[str]] = None,
    fonts_override: bool = False,
    humanize: Optional[bool] = None,
    enable_cache: bool = False,
    persistent_context: bool = False,
    user_data_dir: Optional[str] = None,
    main_world_scripts: bool = False,
    coop: Optional[bool] = None,

    # Passthrough to Playwright
    args: Optional[List[str]] = None,
    executable_path: Optional[str] = None,
    **kwargs
) -> Dict[str, Any]:
```

**Configuration Priorities:**

```
1. Explicit config parameter (highest)
2. Fingerprint parameter
3. GeoIP-derived configuration
4. Generated fingerprint (default)
5. Playwright defaults (lowest)
```

**Example:**

```python
# Priority demonstration
launch_options(
    config={"timezone": "America/New_York"},      # Priority 1
    fingerprint=Fingerprint(timezone="UTC"),      # Priority 2 (ignored)
    geoip=True,                                   # Priority 3 (partial)
    # Generated fingerprint                       # Priority 4 (partial)
)

# Result: timezone="America/New_York"
```

### Default Addons (uBlock, BPC)

**uBlock Origin:**

Included by default for several reasons:

1. **Realism:** Most users have ad blockers
2. **Performance:** Faster page loads (50-70% bandwidth reduction)
3. **Privacy:** Blocks trackers and fingerprinting scripts
4. **Consistency:** Same behavior across all users

**Usage:**

```python
# Default: uBlock Origin included
with Camoufox() as browser:
    pass

# Exclude uBlock Origin
with Camoufox(exclude_addons=[DefaultAddons.UBO]) as browser:
    pass

# Add custom addons
with Camoufox(addons=["/path/to/my-addon"]) as browser:
    pass
```

**Addon Management:**

```python
class DefaultAddons(Enum):
    UBO = "https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi"

def add_default_addons(addons_list: List[str], exclude_list: Optional[List[DefaultAddons]] = None):
    """
    Download and extract addons to local cache.

    Process:
    1. Check ~/.cache/camoufox/addons/
    2. If not present, download .xpi
    3. Extract to directory
    4. Validate manifest.json
    5. Add path to addons_list
    """
```

### Error Handling

**Custom Exception Hierarchy:**

```python
# exceptions.py
class CamoufoxException(Exception):
    """Base exception for all Camoufox errors"""

class InvalidLocale(CamoufoxException):
    """Invalid locale format"""

    @staticmethod
    def invalid_input(locale: str):
        return InvalidLocale(
            f"Invalid locale: {locale}\n"
            f"Expected format: 'en-US' or 'en' or 'US'\n"
            f"Examples: 'en-US', 'ja-JP', 'de-DE'"
        )

class InvalidProxy(CamoufoxException):
    """Invalid proxy configuration"""

class CamoufoxNotInstalled(CamoufoxException):
    """Camoufox binaries not found"""

    def __init__(self):
        super().__init__(
            "Camoufox binaries not found.\n"
            "Please run: camoufox fetch"
        )

class UnsupportedVersion(CamoufoxException):
    """Installed version not supported"""
```

**Validation Examples:**

```python
# Locale validation
try:
    handle_locale("invalid-locale")
except InvalidLocale as e:
    print(e)
    # Invalid locale: invalid-locale
    # Expected format: 'en-US' or 'en' or 'US'

# Proxy validation
try:
    Proxy(server="")
except InvalidProxy as e:
    print(e)
    # Invalid proxy server:

# Version validation
try:
    launch_path()
except CamoufoxNotInstalled as e:
    print(e)
    # Camoufox binaries not found.
    # Please run: camoufox fetch
```

**Warning System:**

```python
# Warnings for detection risks (not errors)
with Camoufox(headless=True) as browser:
    # ⚠️ LeakWarning: Headless mode is enabled.
    # Headless browsers are easily detectable.
    pass

# Suppress warning
with Camoufox(headless=True, i_know_what_im_doing=True) as browser:
    # No warning
    pass
```

---

## PyPI Releases

### Version History Timeline

| Version | Date | Key Changes |
|---------|------|-------------|
| **0.1.0b1** | Sep 19, 2024 | Initial beta release |
| **0.1.0b4** | Sep 19, 2024 | Fixed out-of-bounds screen values |
| **0.1.1** | Sep 19, 2024 | First stable release |
| **0.1.2** | Sep 19, 2024 | Re-enabled BrowserForge userAgent |
| **0.1.3** | Sep 19, 2024 | Added block_images, block_webrtc |
| **0.2.0** | Sep 29, 2024 | **MAJOR:** GeoIP, locales, timezone |
| **0.2.1** | Sep 29, 2024 | Typing improvements |
| **0.2.2** | Sep 30, 2024 | Viewport hijacking support |
| **0.2.3** | Oct 2, 2024 | Optional geoip dependency |
| **0.2.5** | Oct 4, 2024 | Persistent context, remote server |
| **0.2.6** | Oct 5, 2024 | OS validation |
| **0.2.7** | Oct 6, 2024 | Leak warning system |
| **0.2.8** | Oct 6, 2024 | More leak warnings |
| **0.2.9** | Oct 8, 2024 | Xvfb integration |
| **0.2.11** | Oct 9, 2024 | Screen bounds hotfix |
| **0.2.12** | Oct 10, 2024 | Geolocation domain fix |
| **0.2.15** | Oct 15, 2024 | Dict type support |
| **0.3.3** | Oct 28, 2024 | Error handling improvements |
| **0.3.4** | Oct 29, 2024 | Auto-add http schema |
| **0.3.5** | Oct 31, 2024 | Python 3.8 support |
| **0.3.6** | Oct 31, 2024 | Complete Python 3.8 fixes |
| **0.3.8** | Nov 4, 2024 | Import fixes |
| **0.3.9** | Nov 11, 2024 | Cache control, font spacing |
| **0.3.10** | Nov 18, 2024 | Optional proxy credentials |
| **0.4.0** | Nov 21, 2024 | **MAJOR:** WebGL rotation |
| **0.4.1** | Nov 21, 2024 | WebGL import hotfix |
| **0.4.2** | Nov 22, 2024 | Rollback WebGL (crashes) |
| **0.4.3** | Nov 22, 2024 | WebGL disabled by default |
| **0.4.4-beta** | Nov 25, 2024 | Fixed WebGL crashes |
| **0.4.4** | Nov 28, 2024 | Stable WebGL release |
| **0.4.5** | Dec 3, 2024 | Main world evaluation |
| **0.4.6** | Dec 4, 2024 | Code cleanup |
| **0.4.7** | Dec 8, 2024 | Version bump |
| **0.4.9** | Dec 13, 2024 | Earlier release assets |
| **0.4.10** | Jan 25, 2025 | COOP toggle |
| **0.4.11** | Feb 2025 | Current stable |

### Breaking Changes

**v0.2.0 (Sep 29, 2024):**

```python
# BREAKING: Changed parameter names
# Old:
NewBrowser(user_agent="...")

# New:
NewBrowser(config={"navigator.userAgent": "..."})
```

**v0.3.0 (Oct 2024):**

```python
# BREAKING: Removed deprecated parameters
# Old:
NewBrowser(user_agent=..., fonts=...)

# New:
NewBrowser(config={"navigator.userAgent": ..., "fonts": [...]})

# Or use fingerprint parameter
```

**v0.4.0 (Nov 21, 2024):**

```python
# BREAKING: WebGL now optional
# Old: Always enabled

# New: Controlled by allow_webgl
NewBrowser(allow_webgl=True)  # Enable WebGL rotation
NewBrowser(allow_webgl=False) # Disable (default in 0.4.3)
```

### Deprecations

**v0.2.0:**
- Deprecated direct parameter passing (user_agent, fonts, etc.)
- Use `config` parameter instead

**v0.3.0:**
- Removed deprecated direct parameters completely

### Statistics

**PyPI Downloads** (estimated as of Feb 2025):
- Total: 50,000+ downloads
- Average: 300-500 downloads/day
- Peak: 1,000+ downloads/day (after major releases)

**GitHub Stars:** 1,800+

**Production Users:** 500+ (estimated based on issue reports and community discussions)

---

## Integration Examples

### Basic Usage

**Minimal Example:**

```python
from camoufox import Camoufox

# Simplest usage (all defaults)
with Camoufox() as browser:
    page = browser.new_page()
    page.goto("https://example.com")
    print(page.title())
```

**With GeoIP:**

```python
from camoufox import Camoufox

# Auto-configure based on IP
with Camoufox(geoip=True) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
```

**With Proxy:**

```python
from camoufox import Camoufox
from camoufox.ip import Proxy

# Use proxy with GeoIP auto-configuration
with Camoufox(
    geoip=True,
    proxy=Proxy(
        server="socks5://us-proxy.example.com:1080",
        username="user",
        password="pass"
    )
) as browser:
    page = browser.new_page()
    page.goto("https://ipinfo.io")
```

### Advanced Configuration

**Custom Fingerprint:**

```python
from camoufox import Camoufox
from browserforge.fingerprints import FingerprintGenerator

# Generate custom fingerprint
generator = FingerprintGenerator(browser='firefox', os='windows')
fingerprint = generator.generate()

# Use custom fingerprint
with Camoufox(fingerprint=fingerprint) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
```

**Manual Configuration:**

```python
from camoufox import Camoufox

# Override specific properties
with Camoufox(
    config={
        "navigator.userAgent": "Mozilla/5.0 ...",
        "navigator.language": "en-US",
        "navigator.hardwareConcurrency": 8,
        "screen.width": 1920,
        "screen.height": 1080,
        "timezone": "America/New_York",
    }
) as browser:
    page = browser.new_page()
```

**Virtual Display (Linux):**

```python
from camoufox import Camoufox

# Run in virtual display (requires Xvfb)
with Camoufox(headless='virtual') as browser:
    page = browser.new_page()
    page.goto("https://example.com")

    # Full GPU/WebGL support
    # Undetectable (uses windowed Firefox)
```

**Persistent Context:**

```python
from camoufox import NewBrowser
from playwright.sync_api import sync_playwright

# Save cookies, localStorage between sessions
with sync_playwright() as p:
    context = NewBrowser(
        p,
        persistent_context=True,
        user_data_dir="./profile"
    )

    page = context.new_page()
    page.goto("https://example.com")
    # Login, cookies saved

    context.close()

# Next session: cookies still present
with sync_playwright() as p:
    context = NewBrowser(
        p,
        persistent_context=True,
        user_data_dir="./profile"
    )

    page = context.new_page()
    page.goto("https://example.com")
    # Still logged in!
```

### Production Patterns

**Concurrent Scraping:**

```python
import asyncio
from camoufox import AsyncCamoufox

async def scrape_page(url: str) -> str:
    async with AsyncCamoufox(geoip=True) as browser:
        page = await browser.new_page()
        await page.goto(url)
        content = await page.content()
        return content

async def main():
    urls = ["https://example1.com", "https://example2.com", ...]

    # Scrape 10 pages concurrently
    tasks = [scrape_page(url) for url in urls[:10]]
    results = await asyncio.gather(*tasks)

    return results

asyncio.run(main())
```

**Retry Logic:**

```python
from camoufox import Camoufox
from playwright.sync_api import TimeoutError
import time

def scrape_with_retry(url: str, max_retries: int = 3) -> str:
    for attempt in range(max_retries):
        try:
            with Camoufox(geoip=True) as browser:
                page = browser.new_page()
                page.goto(url, timeout=30000)
                return page.content()

        except TimeoutError:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            raise

    raise Exception(f"Failed after {max_retries} retries")
```

**Distributed Architecture:**

```python
# Server (machine A)
# Run: camoufox server
# Outputs: ws://machine-a:3000/ws

# Client (machine B)
from playwright.sync_api import sync_playwright

def scrape_remote(url: str, ws_endpoint: str):
    with sync_playwright() as p:
        browser = p.firefox.connect(ws_endpoint)
        page = browser.new_page()
        page.goto(url)
        content = page.content()
        browser.close()
        return content

# Use
scrape_remote("https://example.com", "ws://machine-a:3000/ws")
```

**Logging & Monitoring:**

```python
from camoufox import Camoufox
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def scrape_with_logging(url: str):
    logger.info(f"Starting scrape: {url}")

    try:
        with Camoufox(geoip=True, debug=True) as browser:
            page = browser.new_page()

            # Monitor console logs
            page.on("console", lambda msg: logger.debug(f"Console: {msg.text}"))

            # Monitor network
            page.on("request", lambda req: logger.debug(f"Request: {req.url}"))
            page.on("response", lambda resp: logger.debug(f"Response: {resp.status} {resp.url}"))

            page.goto(url)
            logger.info(f"Page loaded: {page.title()}")

            return page.content()

    except Exception as e:
        logger.error(f"Error scraping {url}: {e}")
        raise
```

### Testing with Playwright

**Unit Testing:**

```python
import pytest
from camoufox import Camoufox

@pytest.fixture
def browser():
    """Fixture providing a Camoufox browser"""
    with Camoufox(headless=True) as browser:
        yield browser

def test_page_title(browser):
    page = browser.new_page()
    page.goto("https://example.com")
    assert "Example Domain" in page.title()

def test_fingerprint(browser):
    page = browser.new_page()
    page.goto("https://browserleaks.com/javascript")

    # Check that fingerprint is applied
    user_agent = page.evaluate("navigator.userAgent")
    assert "Firefox" in user_agent

    # Check timezone
    timezone = page.evaluate("Intl.DateTimeFormat().resolvedOptions().timeZone")
    assert timezone  # Should be set
```

**Integration Testing:**

```python
import pytest
from camoufox import Camoufox

@pytest.mark.integration
def test_real_website():
    """Test against real website"""
    with Camoufox(geoip=True) as browser:
        page = browser.new_page()

        # Navigate
        page.goto("https://example.com")

        # Interact
        page.fill("input[name='search']", "test")
        page.click("button[type='submit']")

        # Wait for response
        page.wait_for_selector(".results")

        # Verify
        results = page.query_selector_all(".result")
        assert len(results) > 0
```

**Screenshot Testing:**

```python
from camoufox import Camoufox

def test_rendering():
    with Camoufox() as browser:
        page = browser.new_page()
        page.goto("https://example.com")

        # Take screenshot
        page.screenshot(path="screenshot.png")

        # Compare with baseline (using image diff library)
        # assert images_match("screenshot.png", "baseline.png")
```

---

## Challenges & Solutions

### Python 3.8 Compatibility

**Challenge:**

Python 3.8 was released in 2019 and lacks several modern features:

1. `Path.is_relative_to()` (added in 3.9)
2. `str.removeprefix()` / `str.removesuffix()` (added in 3.9)
3. Type hint improvements (PEP 604: `X | Y` syntax)

**Why Support 3.8:**

- Ubuntu 20.04 LTS (EOL: April 2025) ships with Python 3.8
- Enterprise environments often use older Python versions
- Broader compatibility without significant drawbacks

**Solution 1: Path Operations**

```python
# Python 3.9+
if Path(filename).is_relative_to(current_module):
    ...

# Python 3.8 compatible
try:
    Path(filename).relative_to(current_module)
    # Is relative
except ValueError:
    # Not relative
```

**Solution 2: Type Hints**

```python
# Python 3.10+
from typing import Union
def func(x: str | int) -> str | None:
    ...

# Python 3.8 compatible
from typing import Optional, Union
def func(x: Union[str, int]) -> Optional[str]:
    ...
```

**Solution 3: String Operations**

```python
# Python 3.9+
url = url.removeprefix("http://")

# Python 3.8 compatible
if url.startswith("http://"):
    url = url[7:]
```

**Testing Strategy:**

```yaml
# .github/workflows/test.yml
strategy:
  matrix:
    python-version: ['3.8', '3.9', '3.10', '3.11', '3.12']
```

### Cross-Platform Support

**Challenge:**

Windows, macOS, and Linux have different:
- File path conventions
- Environment variable limits
- Process management
- Binary formats

**Solution 1: Path Handling**

```python
from pathlib import Path

# Always use Path objects (cross-platform)
INSTALL_DIR = Path(user_cache_dir("camoufox"))

# Windows: C:\Users\user\AppData\Local\camoufox\
# macOS: ~/Library/Caches/camoufox/
# Linux: ~/.cache/camoufox/
```

**Solution 2: Environment Variables**

```python
# Windows: 2047 byte limit (CMD)
# Linux/macOS: 32767 byte limit (shell)

chunk_size = 2047 if OS_NAME == 'win' else 32767

for i in range(0, len(config_str), chunk_size):
    chunk = config_str[i : i + chunk_size]
    env_vars[f"CAMOU_CONFIG_{i // chunk_size + 1}"] = chunk
```

**Solution 3: Virtual Display (Linux Only)**

```python
# virtdisplay.py
@staticmethod
def assert_linux():
    if OS_NAME != 'lin':
        raise VirtualDisplayNotSupported(
            "Virtual display is only supported on Linux."
        )
```

**Solution 4: OS Detection**

```python
import platform
import sys

OS_MAP = {
    'darwin': 'mac',
    'linux': 'lin',
    'win32': 'win',
}

if sys.platform not in OS_MAP:
    raise UnsupportedOS(f"OS {sys.platform} is not supported")

OS_NAME = OS_MAP[sys.platform]
```

### Binary Distribution

**Challenge:**

Distributing pre-compiled Firefox binaries for multiple platforms:

- 3 operating systems (Windows, macOS, Linux)
- 3 architectures (x86_64, arm64, i686)
- 50+ MB per binary
- Frequent updates (every 2-4 weeks)

**Solution 1: GitHub Releases**

```python
# Don't bundle binaries with pip package
# Download on-demand from GitHub releases

class CamoufoxFetcher:
    def __init__(self):
        self.github_repo = "daijro/camoufox"
        self.api_url = f"https://api.github.com/repos/{self.github_repo}/releases"

    def get_latest_release(self) -> Dict:
        """Fetch latest release from GitHub API"""

    def find_asset(self, release: Dict) -> str:
        """Find matching asset for current OS/arch"""

    def download_asset(self, url: str) -> BytesIO:
        """Download with progress bar"""
```

**Solution 2: Asset Naming Convention**

```
# Format: camoufox-v{version}-{os}.{arch}.zip

camoufox-v1.0.0-windows.x86_64.zip
camoufox-v1.0.0-macos.arm64.zip
camoufox-v1.0.0-linux.x86_64.zip
camoufox-v1.0.0-linux.i686.zip
```

**Solution 3: Caching**

```python
# Cache downloaded binaries
INSTALL_DIR = Path(user_cache_dir("camoufox"))

# Only download if:
# 1. Not installed
# 2. Version mismatch
# 3. User runs `camoufox fetch`

def is_update_needed() -> bool:
    current_version = installed_verstr()
    latest_version = get_latest_version()
    return current_version != latest_version
```

**Solution 4: Fallback to Earlier Releases**

```python
# e3d3dcd - Look for assets in earlier releases
def find_asset_in_releases(target_version: str) -> str:
    """
    If current version's asset is missing, search previous releases.
    Useful for development or pinning specific versions.
    """
    releases = get_all_releases()
    for release in releases:
        if Version(release['tag_name']) >= Version(target_version):
            asset = find_asset(release)
            if asset:
                return asset
    raise MissingRelease(f"No asset found for version {target_version}")
```

### Version Management

**Challenge:**

Ensuring compatibility between:
- Pip package version (pythonlib)
- Camoufox browser version
- Playwright version
- Firefox version

**Solution 1: Version Constraints**

```python
# __version__.py
class CONSTRAINTS:
    MIN_VERSION = 'beta.19'     # Minimum browser version
    MAX_VERSION = '1'           # Maximum browser version

# Check at runtime
def validate_version():
    current = Version.from_path()
    if not current.is_supported():
        raise UnsupportedVersion(
            f"Camoufox version {current.release} is not supported.\n"
            f"Supported range: {CONSTRAINTS.as_range()}\n"
            f"Please run: camoufox fetch"
        )
```

**Solution 2: Dependency Pinning**

```toml
# pyproject.toml
[tool.poetry.dependencies]
playwright = "*"              # Latest (usually compatible)
browserforge = "^1.2.1"      # Minimum version required
```

**Solution 3: Automated Updates**

```bash
# User runs single command
camoufox fetch

# Updates:
# 1. Camoufox browser binaries
# 2. GeoIP database
# 3. Default addons
# 4. (Optional) BrowserForge fingerprints
```

**Solution 4: Version Checking CLI**

```bash
$ camoufox version
Pip package:    v0.4.11
Camoufox:       v1.0.0 (Up to date!)

$ camoufox version  # If outdated
Pip package:    v0.4.11
Camoufox:       vbeta.15 (Latest supported: v1.0.0)
```

---

## Hands-On: Using the Python Library

This section provides practical examples for common use cases.

### Installation

```bash
# Basic installation
pip install -U camoufox

# With GeoIP support (recommended)
pip install -U camoufox[geoip]

# Development installation
git clone https://github.com/daijro/camoufox
cd camoufox/pythonlib
pip install -e .
```

### Download Browser Binaries

```bash
# Download latest Camoufox
camoufox fetch

# Check installation
camoufox version

# View installation path
camoufox path
```

### Example 1: Basic Scraping

```python
from camoufox import Camoufox

def scrape_quotes():
    with Camoufox() as browser:
        page = browser.new_page()
        page.goto("https://quotes.toscrape.com")

        quotes = page.query_selector_all(".quote")
        for quote in quotes:
            text = quote.query_selector(".text").inner_text()
            author = quote.query_selector(".author").inner_text()
            print(f"{author}: {text}")

scrape_quotes()
```

### Example 2: Login and Session Management

```python
from camoufox import NewBrowser
from playwright.sync_api import sync_playwright

def login_example():
    with sync_playwright() as p:
        # Use persistent context to save login session
        context = NewBrowser(
            p,
            persistent_context=True,
            user_data_dir="./profile"
        )

        page = context.new_page()

        # Check if already logged in
        page.goto("https://example.com/dashboard")
        if page.url == "https://example.com/login":
            # Not logged in, perform login
            page.fill("input[name='username']", "myuser")
            page.fill("input[name='password']", "mypass")
            page.click("button[type='submit']")
            page.wait_for_url("https://example.com/dashboard")

        # Now on dashboard (logged in)
        print(f"Logged in! URL: {page.url}")

        context.close()

login_example()

# Next run: still logged in (cookies saved)
```

### Example 3: Handling CAPTCHAs

```python
from camoufox import Camoufox

def manual_captcha_solving():
    with Camoufox(headless=False) as browser:
        page = browser.new_page()
        page.goto("https://example.com/protected")

        # Wait for CAPTCHA element
        if page.is_visible(".captcha"):
            print("CAPTCHA detected! Please solve manually...")

            # Pause script execution
            # User solves CAPTCHA in the visible browser
            page.pause()

        # Continue after CAPTCHA solved
        print("CAPTCHA solved! Continuing...")
        page.click("button[type='submit']")

manual_captcha_solving()
```

### Example 4: Proxy Rotation

```python
from camoufox import Camoufox
from camoufox.ip import Proxy
import random

def scrape_with_proxy_rotation(urls: list, proxies: list):
    results = []

    for url in urls:
        # Random proxy selection
        proxy = random.choice(proxies)

        with Camoufox(
            geoip=True,  # Auto-configure based on proxy location
            proxy=Proxy(server=proxy)
        ) as browser:
            page = browser.new_page()
            page.goto(url)
            results.append(page.content())

    return results

# Usage
proxies = [
    "socks5://us-proxy-1.example.com:1080",
    "socks5://us-proxy-2.example.com:1080",
    "socks5://us-proxy-3.example.com:1080",
]
urls = ["https://example1.com", "https://example2.com"]
scrape_with_proxy_rotation(urls, proxies)
```

### Example 5: Parallel Scraping

```python
import asyncio
from camoufox import AsyncCamoufox

async def scrape_page(url: str) -> dict:
    async with AsyncCamoufox(geoip=True) as browser:
        page = await browser.new_page()
        await page.goto(url)

        title = await page.title()
        content = await page.content()

        return {
            "url": url,
            "title": title,
            "content": content
        }

async def scrape_parallel(urls: list, concurrency: int = 5):
    # Process URLs in batches
    results = []

    for i in range(0, len(urls), concurrency):
        batch = urls[i:i+concurrency]
        tasks = [scrape_page(url) for url in batch]
        batch_results = await asyncio.gather(*tasks)
        results.extend(batch_results)

    return results

# Usage
urls = [f"https://example.com/page/{i}" for i in range(100)]
results = asyncio.run(scrape_parallel(urls, concurrency=10))
```

### Example 6: Taking Screenshots

```python
from camoufox import Camoufox

def screenshot_example():
    with Camoufox() as browser:
        page = browser.new_page()
        page.goto("https://example.com")

        # Full page screenshot
        page.screenshot(path="fullpage.png", full_page=True)

        # Element screenshot
        element = page.query_selector("h1")
        element.screenshot(path="heading.png")

        # Screenshot as bytes (for S3, etc.)
        screenshot_bytes = page.screenshot()

screenshot_example()
```

### Example 7: Handling Infinite Scroll

```python
from camoufox import Camoufox

def scrape_infinite_scroll():
    with Camoufox() as browser:
        page = browser.new_page()
        page.goto("https://example.com/infinite-scroll")

        # Scroll to bottom multiple times
        for i in range(10):
            # Scroll to bottom
            page.evaluate("window.scrollTo(0, document.body.scrollHeight)")

            # Wait for new content to load
            page.wait_for_timeout(2000)

            print(f"Scroll {i+1}: Loaded more content")

        # Extract all items
        items = page.query_selector_all(".item")
        print(f"Total items: {len(items)}")

scrape_infinite_scroll()
```

### Example 8: Form Automation

```python
from camoufox import Camoufox

def fill_form_example():
    with Camoufox(humanize=True) as browser:
        page = browser.new_page()
        page.goto("https://example.com/form")

        # Fill text inputs
        page.fill("input[name='name']", "John Doe")
        page.fill("input[name='email']", "john@example.com")

        # Select dropdown
        page.select_option("select[name='country']", "US")

        # Check checkbox
        page.check("input[name='terms']")

        # Upload file
        page.set_input_files("input[type='file']", "document.pdf")

        # Submit form
        page.click("button[type='submit']")

        # Wait for success message
        page.wait_for_selector(".success-message")
        print("Form submitted successfully!")

fill_form_example()
```

### Example 9: PDF Generation

```python
from camoufox import Camoufox

def generate_pdf():
    with Camoufox() as browser:
        page = browser.new_page()
        page.goto("https://example.com")

        # Generate PDF
        page.pdf(
            path="output.pdf",
            format="A4",
            print_background=True,
            margin={
                "top": "1cm",
                "right": "1cm",
                "bottom": "1cm",
                "left": "1cm"
            }
        )

generate_pdf()
```

### Example 10: Remote Server Usage

```bash
# On server machine (192.168.1.100)
$ camoufox server
Listening on ws://0.0.0.0:3000/ws
```

```python
# On client machine
from playwright.sync_api import sync_playwright

def connect_remote():
    with sync_playwright() as p:
        # Connect to remote server
        browser = p.firefox.connect("ws://192.168.1.100:3000/ws")

        page = browser.new_page()
        page.goto("https://example.com")
        print(page.title())

        browser.close()

connect_remote()
```

---

## External References

### Official Documentation

- **Camoufox Python Library:** https://camoufox.com/python
- **Camoufox GitHub:** https://github.com/daijro/camoufox
- **PyPI Package:** https://pypi.org/project/camoufox/

### Dependencies

- **Playwright Python:** https://playwright.dev/python/
  - Browser automation framework
  - Firefox support via Juggler protocol
  - Used as base for Camoufox API

- **BrowserForge:** https://github.com/daijro/browserforge
  - Fingerprint generation engine
  - Statistical fingerprint distribution
  - Header and fingerprint generation

- **MaxMind GeoLite2:** https://dev.maxmind.com/geoip/geolite2-free-geolocation-data
  - Free GeoIP database
  - City-level location accuracy
  - Used via GitHub mirror

### Related Projects

- **hrequests:** https://github.com/daijro/hrequests
  - HTTP client with browser-like fingerprints
  - Shares CLI infrastructure code with Camoufox

- **Undetected ChromeDriver:** https://github.com/ultrafunkamsterdam/undetected-chromedriver
  - Similar goals for Chrome
  - Inspiration for some anti-detection techniques

### Learning Resources

- **Browser Fingerprinting Guide:** https://browserleaks.com/
- **Playwright Documentation:** https://playwright.dev/docs/intro
- **Firefox Juggler Protocol:** https://github.com/microsoft/playwright/tree/main/browser_patches/firefox
- **CLDR Data:** http://cldr.unicode.org/

### Community

- **GitHub Issues:** https://github.com/daijro/camoufox/issues
- **GitHub Discussions:** https://github.com/daijro/camoufox/discussions

---

## Conclusion

The Camoufox Python library has evolved from a simple Playwright wrapper into a sophisticated anti-detection toolkit with 52+ commits, 47+ releases, and thousands of users. Key achievements include:

1. **Fingerprint Automation:** Automatic generation and injection of 50+ properties
2. **GeoIP Intelligence:** Statistical locale selection and automatic timezone/geolocation
3. **Cross-Platform:** Windows, macOS, Linux support with virtual display for headless servers
4. **WebGL Rotation:** 1,000+ real-world GPU fingerprints with statistical sampling
5. **Developer Experience:** Clean API, excellent documentation, type hints, helpful warnings

The library has made Camoufox accessible to Python developers while maintaining the power and flexibility needed for production use. It continues to evolve with new features, bug fixes, and community contributions.

**Future Directions:**
- Mobile fingerprint support (Android/iOS)
- Enhanced WebRTC IP spoofing
- Machine learning-based behavioral simulation
- Distributed fingerprint management
- Enhanced testing infrastructure

The Python library represents the most active development area of the Camoufox project, with continuous improvements driven by real-world usage and community feedback.
