# Project Foundation: The Birth of Camoufox

## Overview

On July 26, 2024, Camoufox was born with commit `1090f6a` titled "Initial release v128.0-1". This wasn't just a simple Firefox fork—it was a **sophisticated anti-fingerprinting browser** built from the ground up with a novel architecture for injecting fake fingerprints while maintaining perfect consistency across all detection vectors.

This document analyzes what Camoufox looked like at its inception, examining the technical decisions, architectural patterns, and implementation details that formed the foundation for everything that followed.

## The First Commit

**Commit**: `1090f6a2128d0ab3ec533e4fc48166873caa8010`
**Date**: Friday, July 26, 2024, 06:34:50 -0500
**Message**: "Initial release v128.0-1"
**Files Changed**: 485 files
**Firefox Base**: Mozilla Firefox 128.0

### What Was Included?

The first commit was surprisingly complete, including:

- ✅ Complete fingerprint injection system (C++ and patches)
- ✅ Font fingerprinting protection with bundled fonts
- ✅ Full Playwright/Juggler integration (~15,000 lines of JS)
- ✅ LibreWolf privacy patches
- ✅ Build system (Makefile, Docker, multi-platform support)
- ✅ Custom branding and theming
- ✅ Network header spoofing
- ✅ Comprehensive documentation

This wasn't an MVP—it was a **fully functional anti-detection browser** on day one.

## Core Architecture: The MaskConfig System

The genius of Camoufox's architecture lies in its **centralized configuration injection system** called MaskConfig.

### How It Works

```
┌─────────────────────────────────────────────────────────────┐
│  User/Python Library                                         │
│  Sets CAMOU_CONFIG environment variable                      │
│  {"navigator.userAgent": "Mozilla/5.0 ...", ...}            │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Firefox Process Start                                       │
│  MaskConfig::Init() reads CAMOU_CONFIG                       │
│  Parses JSON using nlohmann/json library                     │
│  Stores in static member variable                            │
└────────────────────┬────────────────────────────────────────┘
                     │
      ┌──────────────┴───────────────┬──────────────────┐
      ▼                              ▼                  ▼
┌─────────────┐            ┌──────────────┐   ┌──────────────┐
│ Navigator.cpp│            │ nsScreen.cpp │   │ Window.cpp   │
│ GetString(  │            │ GetInt32(    │   │ GetRect(     │
│ "navigator. │            │ "screen.     │   │ "window.     │
│ userAgent") │            │ width")      │   │ innerWidth") │
└─────────────┘            └──────────────┘   └──────────────┘
      │                              │                  │
      ▼                              ▼                  ▼
Return spoofed value        Return spoofed value  Return spoofed
or std::nullopt (use       or std::nullopt        value or
default Firefox behavior)                          std::nullopt
```

### The MaskConfig Class

Located in `additions/dom/mask/MaskConfig.hpp`, this C++ header provides the entire configuration API:

```cpp
class MaskConfig {
 public:
  // Initialize from CAMOU_CONFIG environment variable
  static void Init();

  // Type-safe getters with optional return
  static std::optional<std::string> GetString(const std::string& key);
  static std::optional<int32_t> GetInt32(const std::string& key);
  static std::optional<double> GetDouble(const std::string& key);
  static std::optional<bool> GetBool(const std::string& key);
  static std::optional<std::vector<std::string>> GetStringList(
      const std::string& key);
  static std::optional<std::vector<std::string>> GetStringListLower(
      const std::string& key);
  static std::optional<nsIntRect> GetRect(const std::string& key);

  // Check if a key exists
  static bool HasKey(const std::string& key);

 private:
  static nlohmann::json config_;  // The parsed JSON configuration
  static bool initialized_;       // Has Init() been called?
};
```

### Why This Architecture is Brilliant

1. **Zero JavaScript injection**: No detectable JavaScript runs in page context
2. **Atomic initialization**: Configuration is set once at browser startup
3. **Type safety**: Each property has compile-time type checking
4. **Performance**: No runtime parsing overhead—config is parsed once
5. **Maintainability**: Adding a new spoofable property requires only:
   - Modifying the C++ file that controls that property
   - Adding one `GetString()` call
6. **Consistency**: Impossible to have mismatches between different parts of the browser

### Example: Spoofing navigator.userAgent

Here's how the first commit modified `dom/base/Navigator.cpp`:

```cpp
void Navigator::GetUserAgent(nsAString& aUserAgent, CallerType aCallerType,
                             ErrorResult& aRv) const {
  // NEW CODE: Check if MaskConfig has a custom user agent
  auto spoofedUserAgent = MaskConfig::GetString("navigator.userAgent");
  if (spoofedUserAgent.has_value()) {
    // Return the spoofed value
    CopyUTF8toUTF16(mozilla::MakeStringSpan(spoofedUserAgent.value()),
                    aUserAgent);
    return;
  }

  // ORIGINAL FIREFOX CODE: Return real user agent
  nsresult rv = GetWindow()->GetBrowserDOMWindow()->GetUserAgent(aUserAgent);
  if (NS_WARN_IF(NS_FAILED(rv))) {
    aRv.Throw(rv);
  }
}
```

**Key Points:**
- The modification is **minimal and non-invasive**
- Original Firefox code remains intact as fallback
- If no spoof is configured, Firefox behaves normally
- **No JavaScript can detect this modification** because it happens in C++

##

 Fingerprint Injection: The Core Patches

### 1. Navigator Properties (`patches/fingerprint-injection.patch`)

This massive patch (~600 lines) modifies 8 critical Firefox C++ files to spoof 25+ navigator properties.

#### Files Modified

| File | Purpose | Properties Modified |
|------|---------|---------------------|
| `dom/base/Navigator.cpp` | Main Navigator object | userAgent, appVersion, platform, oscpu, product, productSub, language, languages, hardwareConcurrency, maxTouchPoints, doNotTrack, globalPrivacyControl, cookieEnabled, onLine, buildID, pdfViewerEnabled |
| `dom/base/NavigatorBinding.cpp` | WebIDL bindings | Ensures WebDriver always returns false |
| `dom/workers/WorkerNavigator.cpp` | Worker context navigator | Same properties for Web Workers |
| `dom/base/nsGlobalWindowInner.cpp` | Window object | innerWidth, innerHeight, outerWidth, outerHeight, screenX, screenY, scrollMinX, scrollMinY, scrollMaxX, scrollMaxY, pageXOffset, pageYOffset, devicePixelRatio |
| `dom/base/nsScreen.cpp` | Screen object | width, height, availWidth, availHeight, availTop, availLeft, colorDepth, pixelDepth |
| `dom/base/Element.cpp` | Document body | clientWidth, clientHeight, clientTop, clientLeft |
| `dom/base/nsHistory.cpp` | History object | length |
| `dom/battery/BatteryManager.cpp` | Battery API | charging, chargingTime, dischargingTime, level |

#### Navigator WebDriver: Always False

One of the most important anti-detection features:

```cpp
// dom/base/NavigatorBinding.cpp
bool Navigator::GetWebdriver() const {
  // ORIGINAL CODE:
  // return mWindow && mWindow->GetDocumentShell() &&
  //        mWindow->GetDocumentShell()->GetIsUnderWebDriver();

  // NEW CODE: Always return false to hide automation
  return false;
}
```

**Why This Matters**: The `navigator.webdriver` property is the **#1 way** websites detect Selenium, Playwright, and Puppeteer. By hardcoding it to `false` in C++, Camoufox makes this detection impossible.

#### Language Properties: Fallback Chain

The patch implements a smart fallback system for language properties:

```cpp
void Navigator::GetLanguage(nsAString& aLanguage) {
  // 1. Check if explicitly set in config
  auto spoofedLanguage = MaskConfig::GetString("navigator.language");
  if (spoofedLanguage.has_value()) {
    CopyUTF8toUTF16(mozilla::MakeStringSpan(spoofedLanguage.value()),
                    aLanguage);
    return;
  }

  // 2. Fall back to locale:language if set
  auto localeLanguage = MaskConfig::GetString("locale:language");
  if (localeLanguage.has_value()) {
    // Combine with locale:region if available
    auto localeRegion = MaskConfig::GetString("locale:region");
    if (localeRegion.has_value()) {
      std::string combined = localeLanguage.value() + "-" +
                            localeRegion.value();
      CopyUTF8toUTF16(mozilla::MakeStringSpan(combined), aLanguage);
      return;
    }
    CopyUTF8toUTF16(mozilla::MakeStringSpan(localeLanguage.value()),
                    aLanguage);
    return;
  }

  // 3. Use Firefox default
  GetAcceptLanguages(aLanguage);
}
```

This allows users to either:
- Set `navigator.language` directly: `{"navigator.language": "fr-FR"}`
- Set locale once: `{"locale:language": "fr", "locale:region": "FR"}` and have it apply to multiple properties

### 2. Window & Screen Hijacking (`patches/window-hijacker.patch`)

Visual window dimensions must match the JavaScript-reported dimensions, or websites can detect the spoof. This patch solves that elegantly.

#### The Challenge

When you set `window.innerWidth = 1920` in JavaScript, the actual browser window might be 1280 pixels wide. This mismatch is **easily detectable**:

```javascript
// Detection code websites use
if (window.innerWidth !== document.documentElement.clientWidth) {
  console.log("Dimension spoofing detected!");
}
```

#### The Solution: CSS Injection

Camoufox injects custom CSS rules that force the browser chrome to match the spoofed dimensions:

```javascript
// File: browser/base/content/browser.js
function applyCamouflageConfig() {
  const config = ChromeUtils.camouGetConfig(); // Access MaskConfig from JS

  // Get spoofed dimensions from config
  const innerWidth = config["window.innerWidth"];
  const innerHeight = config["window.innerHeight"];
  const outerWidth = config["window.outerWidth"];
  const outerHeight = config["window.outerHeight"];

  if (innerWidth && innerHeight) {
    // Create custom CSS to set content area size
    const style = document.createElementNS(
      "http://www.w3.org/1999/xhtml", "style"
    );
    style.textContent = `
      .browserStack {
        width: ${innerWidth}px !important;
        height: ${innerHeight}px !important;
        overflow: auto !important;
        scrollbar-width: none !important;
        contain: size !important;
      }
    `;
    document.documentElement.appendChild(style);
  }

  if (outerWidth && outerHeight) {
    // Set the actual window size
    window.resizeTo(outerWidth, outerHeight);

    // Lock the size (prevent user resizing)
    document.documentElement.style.width = outerWidth + "px";
    document.documentElement.style.height = outerHeight + "px";
  }
}
```

**Key Techniques:**
- `scrollbar-width: none` - Hides scrollbars without affecting layout
- `contain: size` - CSS containment for performance
- `!important` - Overrides any other CSS rules
- `overflow: auto` - Allows scrolling if content exceeds viewport

#### Automatic Sizing

If only `innerWidth`/`innerHeight` are set, the patch automatically calculates the required window size:

```javascript
if (!outerWidth && !outerHeight && innerWidth && innerHeight) {
  // Measure the toolbar height
  const toolbarHeight = document.querySelector("#navigator-toolbox")
                               .getBoundingClientRect().height;

  // Set window size = inner size + toolbar + titlebar
  window.resizeTo(
    innerWidth,
    innerHeight + toolbarHeight + 30  // 30px for titlebar
  );
}
```

### 3. Font Fingerprinting Protection (`patches/font-hijacker.patch`)

Font fingerprinting is one of the most powerful browser fingerprinting techniques. Websites can detect which fonts are installed on your system by testing if text rendered in a specific font has a different size than a fallback font.

#### The Attack

```javascript
// Simplified font fingerprinting technique
function isFontAvailable(fontName) {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');

  // Measure text with fallback font
  ctx.font = '72px monospace';
  const fallbackWidth = ctx.measureText('mmmmmmmmmmlli').width;

  // Measure text with target font (falls back if not available)
  ctx.font = `72px '${fontName}', monospace`;
  const testWidth = ctx.measureText('mmmmmmmmmmlli').width;

  // If widths differ, font is available
  return testWidth !== fallbackWidth;
}

// Test for 1000+ fonts to create unique fingerprint
const fonts = ['Arial', 'Helvetica', 'Times New Roman', 'Comic Sans MS', ...];
const installed = fonts.filter(isFontAvailable);
console.log('Fingerprint:', installed.join(','));
```

#### The Defense: Whitelist System

Camoufox implements a **font whitelist** that blocks access to non-allowed fonts:

```cpp
// File: layout/style/FontFace.cpp
already_AddRefed<Promise> FontFace::Load(ErrorResult& aRv) {
  // NEW CODE: Check if font is allowed
  nsString family;
  GetFamily(family);
  NS_ConvertUTF16toUTF8 familyUTF8(family);

  auto allowedFonts = MaskConfig::GetStringListLower("fonts");
  if (allowedFonts.has_value()) {
    std::string lowercaseFamily = familyUTF8.get();
    std::transform(lowercaseFamily.begin(), lowercaseFamily.end(),
                   lowercaseFamily.begin(), ::tolower);

    bool found = false;
    for (const auto& font : allowedFonts.value()) {
      if (font == lowercaseFamily) {
        found = true;
        break;
      }
    }

    if (!found) {
      // Font not in whitelist - return error
      SetStatus(FontFaceLoadStatus::Error);
      promise->MaybeReject(NS_ERROR_DOM_SYNTAX_ERR);
      return promise.forget();
    }
  }

  // ORIGINAL CODE: Load the font
  // ... rest of Firefox's font loading code
}
```

#### Bundled Font Collections

The first commit included **390 font files** bundled with the browser:

**Linux Fonts** (from TOR Browser):
- Arimo, Cousine, Tinos - Metrically compatible with Arial, Courier, Times
- Noto Sans/Serif families - Comprehensive Unicode coverage
- 171 total font files (~50MB)

**Windows Fonts** (from Windows 11 22H2):
- Arial, Calibri, Cambria, Consolas, Courier New, Georgia, etc.
- Segoe UI family (including Emoji and Symbol fonts)
- CJK fonts: MS Gothic, Microsoft YaHei, Malgun Gothic
- 145 total font files

**macOS Fonts** (from macOS Sonoma):
- SF Pro, SF Compact, SF Mono - System fonts
- Helvetica Neue, Avenir, Menlo, Monaco
- Apple Color Emoji
- Hiragino, PingFang - CJK support
- 74 total font files

**Font Configuration Files:**
Each platform has a custom `fonts.conf` file (Fontconfig format) that maps font family names to the bundled font files.

### 4. Network Header Spoofing (`patches/network-patches.patch`)

HTTP headers must match JavaScript-reported values, or websites can detect inconsistencies:

```javascript
// JavaScript reports one thing
console.log(navigator.userAgent);  // "Mozilla/5.0 (Windows NT 10.0; ...)"

// But server sees something different
// Server log: User-Agent: Mozilla/5.0 (X11; Linux x86_64; ...)
// ^ MISMATCH DETECTED!
```

#### The Patch

```cpp
// File: netwerk/protocol/http/nsHttpHandler.cpp
nsresult nsHttpHandler::NewProxiedChannel(...) {
  // Set User-Agent header
  auto customUserAgent = MaskConfig::GetString("headers.User-Agent");
  if (!customUserAgent.has_value()) {
    // Fall back to navigator.userAgent if not explicitly set
    customUserAgent = MaskConfig::GetString("navigator.userAgent");
  }
  if (customUserAgent.has_value()) {
    nsAutoCString userAgent(customUserAgent.value().c_str());
    rv = httpChannelInternal->SetRequestHeader(
        NS_LITERAL_CSTRING("User-Agent"), userAgent, false);
  }

  // Set Accept-Language header
  auto customAcceptLanguage = MaskConfig::GetString("headers.Accept-Language");
  if (customAcceptLanguage.has_value()) {
    nsAutoCString acceptLanguage(customAcceptLanguage.value().c_str());
    rv = httpChannelInternal->SetRequestHeader(
        NS_LITERAL_CSTRING("Accept-Language"), acceptLanguage, false);
  }

  // Set Accept-Encoding header
  auto customAcceptEncoding = MaskConfig::GetString("headers.Accept-Encoding");
  if (customAcceptEncoding.has_value()) {
    nsAutoCString acceptEncoding(customAcceptEncoding.value().c_str());
    rv = httpChannelInternal->SetRequestHeader(
        NS_LITERAL_CSTRING("Accept-Encoding"), acceptEncoding, false);
  }

  // Continue with original Firefox code...
}
```

**Headers Controlled:**
- `User-Agent` - Browser identification
- `Accept-Language` - Preferred languages
- `Accept-Encoding` - Supported compressions (gzip, br, etc.)

## Playwright Integration: Juggler

The first commit included a **complete Playwright integration** through Mozilla's Juggler protocol implementation. This wasn't a basic port—it was ~15,000 lines of carefully modified JavaScript.

### What is Juggler?

Juggler is Mozilla's implementation of a browser automation protocol similar to Chrome DevTools Protocol. It provides:
- Page navigation and lifecycle management
- DOM interaction and JavaScript execution
- Network interception and modification
- Screenshot and video capture
- Frame and context isolation

### Architecture

```
┌────────────────────────────────────────────────────┐
│  Playwright Test Script (Node.js)                  │
│  const browser = await firefox.launch()            │
└──────────────────┬─────────────────────────────────┘
                   │ WebSocket or Pipe
                   ▼
┌────────────────────────────────────────────────────┐
│  Juggler Server (Firefox XPCOM Component)          │
│  additions/juggler/components/Juggler.js           │
│  - Protocol.js: Message routing                    │
│  - Dispatcher.js: Command dispatch                 │
│  - BrowserHandler.js: Browser-level commands       │
│  - PageHandler.js: Page-level commands             │
└──────────────────┬─────────────────────────────────┘
                   │
         ┌─────────┴──────────┐
         ▼                    ▼
┌──────────────────┐  ┌──────────────────┐
│ Content Process  │  │ Content Process  │
│ Frame Tree       │  │ Frame Tree       │
│ Page Agent       │  │ Page Agent       │
│ Runtime          │  │ Runtime          │
└──────────────────┘  └──────────────────┘
```

### Key Files in Initial Commit

| File | Lines | Purpose |
|------|-------|---------|
| `components/Juggler.js` | 157 | Main XPCOM component entry point |
| `protocol/Protocol.js` | 1,008 | Protocol type system and validation |
| `protocol/BrowserHandler.js` | 318 | Browser context management, cookies, permissions |
| `protocol/PageHandler.js` | 684 | Page navigation, DOM interaction, screenshots |
| `protocol/Dispatcher.js` | 138 | Message routing and command dispatch |
| `TargetRegistry.js` | 1,200 | Tab, frame, and worker tracking |
| `NetworkObserver.js` | 955 | HTTP request/response interception |
| `content/PageAgent.js` | 703 | JavaScript execution in page context |
| `content/FrameTree.js` | 709 | Frame hierarchy management |
| `content/Runtime.js` | 600 | Console, exceptions, and object inspection |

### Critical Stealth Modifications

The Juggler integration included several stealth patches from day one:

#### 1. Frame Execution Context Isolation

**Problem**: Playwright's PageAgent.js runs in the same context as page JavaScript, making it detectable.

**Solution**: The initial commit included context isolation:

```javascript
// content/PageAgent.js
class PageAgent {
  constructor(frame, session) {
    // Create isolated context for Playwright code
    this._context = Cu.getJSTestingFunctions().createContext({
      global: frame.contentWindow,
      sandboxPrototype: frame.contentWindow,
      // Key: Principal with system privileges
      principal: Cc['@mozilla.org/systemprincipal;1']
                   .createInstance(Ci.nsIPrincipal),
      // Isolated from page code
      freshCompartment: true,
    });
  }

  evaluateJavaScript(script) {
    // Runs in isolated context - invisible to page
    return Cu.evalInSandbox(script, this._context);
  }
}
```

**Result**: Page JavaScript cannot detect Playwright's presence through:
- `Object.getOwnPropertyNames(window)`
- Checking for injected properties
- Prototype chain inspection

#### 2. WebDriver Flag Fix

Beyond just returning `false` for `navigator.webdriver`, the Juggler integration ensures the WebDriver flag is never set in the first place:

```javascript
// protocol/BrowserHandler.js
async createBrowserContext(options) {
  // Create new browser context (like incognito mode)
  const context = Services.browserContexts.create();

  // CRITICAL: Never mark context as under automation
  // context.setIsUnderWebDriver(true);  // <-- This line is REMOVED

  return { browserContextId: context.id };
}
```

## Build System Foundation

The first commit included a complete, multi-platform build system.

### Makefile Targets

```makefile
# Fetch Firefox source code
fetch:
	python3 scripts/fetch.py

# Apply patches to source
dir:
	python3 scripts/setup_dir.py

# Bootstrap build dependencies (one-time setup)
bootstrap:
	cd camoufox-* && ./mach bootstrap

# Build the browser
build:
	cd camoufox-* && ./mach build

# Package for Linux
package-linux:
	python3 scripts/package.py --target linux

# Package for Windows
package-windows:
	python3 scripts/package.py --target windows

# Package for macOS
package-macos:
	python3 scripts/package.py --target macos

# Run the built browser
run:
	cd camoufox-* && ./mach run

# Developer UI for patch management
edits:
	python3 scripts/developer.py
```

### Multi-Platform Mozconfig Files

Each platform has a custom Mozilla build configuration:

**Linux** (`assets/linux.mozconfig`):
```bash
. ./assets/base.mozconfig
ac_add_options --target=x86_64-pc-linux-gnu
```

**Windows** (`assets/windows.mozconfig`):
```bash
. ./assets/base.mozconfig
ac_add_options --target=x86_64-pc-mingw32
export WINEPREFIX="$HOME/.wine-camoufox"
```

**macOS** (`assets/macos.mozconfig`):
```bash
. ./assets/base.mozconfig
ac_add_options --target=x86_64-apple-darwin
ac_add_options --with-macos-sdk=/path/to/MacOSX.sdk
```

**Base Config** (`assets/base.mozconfig`):
```bash
# Application
ac_add_options --enable-application=browser
ac_add_options --with-app-name=camoufox
ac_add_options --with-branding=browser/branding/camoufox

# Disable telemetry and data collection
ac_add_options --disable-crashreporter
ac_add_options --disable-backgroundtasks
ac_add_options --disable-updater
ac_add_options --disable-default-browser-agent
ac_add_options --disable-data-reporting

# Disable unnecessary features
ac_add_options --disable-accessibility
ac_add_options --disable-webspeech
ac_add_options --disable-system-policies

# Optimization
ac_add_options --enable-release
ac_add_options --enable-rust-simd
ac_add_options --enable-lto=cross

# Addons
ac_add_options --with-unsigned-addon-scopes=app,system
ac_add_options --allow-addon-sideload

# No tests
ac_add_options --disable-tests
```

### Docker Support

The initial commit included a complete Dockerfile for reproducible builds:

```dockerfile
FROM ubuntu:22.04

# Install build dependencies
RUN apt-get update && apt-get install -y \
    python3 python3-pip \
    build-essential \
    git mercurial \
    curl wget \
    && rm -rf /var/lib/apt/lists/*

# Install Rust
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

# Copy Camoufox source
WORKDIR /app
COPY . .

# Fetch and patch Firefox source
RUN make fetch && make dir

# Bootstrap (only needed once, but included for clean builds)
RUN make bootstrap

# Build
RUN make build

# Package
RUN make package-linux

CMD ["/bin/bash"]
```

## LibreWolf Integration: Privacy by Default

Camoufox incorporated privacy patches from the LibreWolf project from day one.

### Included LibreWolf Patches

1. **Context Menu Cleanup** (`librewolf/context-menu.patch`)
   - Removes "Email Link" and other tracking vectors

2. **DevTools Bypass** (`librewolf/devtools-bypass.patch`)
   - Removes restrictions on developer tools

3. **Remove Default Addons** (`librewolf/remove_addons.patch`)
   - Strips out Firefox's bundled addons (Pocket, etc.)

4. **Disable Pocket** (`librewolf/sed-patches/disable-pocket.patch`)
   - Removes Mozilla's Pocket integration completely

5. **Stop Undesired Requests** (`librewolf/sed-patches/stop-undesired-requests.patch`)
   - Blocks telemetry and update check URLs

6. **UI Cleanup Patches**:
   - `ui-patches/firefox-view.patch` - Removes Firefox View
   - `ui-patches/hide-default-browser.patch` - No default browser prompts
   - `ui-patches/remove-organization-policy-banner.patch` - No enterprise banners

### Ghostery Patches

Also included privacy patches from the Ghostery browser:

- `ghostery/Disable-Onboarding-Messages.patch` - No first-run prompts

## Initial Configuration System

The first commit allowed configuring these properties via JSON:

### Navigator
```json
{
  "navigator.userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:128.0) Gecko/20100101 Firefox/128.0",
  "navigator.appCodeName": "Mozilla",
  "navigator.appName": "Netscape",
  "navigator.appVersion": "5.0 (Windows)",
  "navigator.platform": "Win32",
  "navigator.oscpu": "Windows NT 10.0; Win64; x64",
  "navigator.language": "en-US",
  "navigator.languages": ["en-US", "en"],
  "navigator.hardwareConcurrency": 8,
  "navigator.maxTouchPoints": 10,
  "navigator.product": "Gecko",
  "navigator.productSub": "20030107",
  "navigator.doNotTrack": "1",
  "navigator.globalPrivacyControl": true,
  "navigator.cookieEnabled": true,
  "navigator.onLine": true,
  "navigator.buildID": "20230101000000",
  "navigator.pdfViewerEnabled": true
}
```

### Window & Screen
```json
{
  "window.innerWidth": 1920,
  "window.innerHeight": 1080,
  "window.outerWidth": 1920,
  "window.outerHeight": 1080,
  "window.screenX": 0,
  "window.screenY": 0,
  "window.devicePixelRatio": 1.0,
  "window.history.length": 2,

  "screen.width": 1920,
  "screen.height": 1080,
  "screen.availWidth": 1920,
  "screen.availHeight": 1040,
  "screen.availTop": 0,
  "screen.availLeft": 0,
  "screen.colorDepth": 24,
  "screen.pixelDepth": 24
}
```

### Fonts
```json
{
  "fonts": [
    "Arial", "Arial Black", "Calibri", "Cambria", "Comic Sans MS",
    "Consolas", "Courier New", "Georgia", "Impact", "Times New Roman",
    "Trebuchet MS", "Verdana"
  ]
}
```

### Battery
```json
{
  "battery.charging": false,
  "battery.chargingTime": 0,
  "battery.dischargingTime": 3600,
  "battery.level": 0.75
}
```

### HTTP Headers
```json
{
  "headers.User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:128.0) Gecko/20100101 Firefox/128.0",
  "headers.Accept-Language": "en-US,en;q=0.9",
  "headers.Accept-Encoding": "gzip, deflate, br"
}
```

## Why This Architecture Matters

### 1. No JavaScript Detection Surface

Traditional anti-detection browsers like Puppeteer Extra use JavaScript injection:

```javascript
// Detectable approach (what other tools do)
Object.defineProperty(navigator, 'webdriver', {
  get: () => false
});
```

This is **easily detectable**:

```javascript
// Detection code
const descriptor = Object.getOwnPropertyDescriptor(Navigator.prototype, 'webdriver');
if (descriptor.get.toString().includes('false')) {
  console.log('Detected JavaScript injection!');
}
```

Camoufox's C++ approach has **zero JavaScript footprint**—there's nothing to detect.

### 2. Perfect Consistency

All browser components share the same MaskConfig, ensuring:
- Navigator and HTTP headers match
- Window dimensions are consistent with visual appearance
- Worker threads have same navigator as main thread
- No timing races or initialization order issues

### 3. Performance

- Configuration parsed **once** at startup
- No runtime JSON parsing
- No JavaScript interceptors on critical paths
- Native C++ performance for all property accesses

### 4. Maintainability

Adding support for a new property requires:
1. Find the C++ file that implements it (use Firefox source search)
2. Add one `MaskConfig::GetXXX()` call
3. Update documentation

No complex JavaScript injection logic needed.

## Comparison with Firefox 128

Let's compare what was different from vanilla Firefox 128:

| Aspect | Firefox 128 | Camoufox v128.0-1 |
|--------|-------------|-------------------|
| `navigator.webdriver` | Returns true if under automation | Always returns false |
| Font access | All system fonts accessible | Whitelist-only access |
| Window dimensions | Real window size | Configurable, visually consistent |
| HTTP headers | Default Firefox headers | Configurable, matches navigator |
| Telemetry | Extensive data collection | Completely disabled |
| Pocket integration | Built-in | Removed |
| Default addons | Several pre-installed | None (except uBlock) |
| Search engines | Google, Bing, etc. | None by default |
| Branding | Firefox logo and name | Camoufox logo and name |
| Auto-updates | Enabled by default | Disabled |
| Crash reporting | Enabled | Disabled |
| Build time | ~2 hours | ~2.5 hours (extra patches) |
| Binary size | ~200 MB | ~250 MB (bundled fonts) |

## Educational Takeaways

### For Developers

1. **C++ modifications** are more powerful than JavaScript injection
2. **Centralized configuration** prevents consistency bugs
3. **Environment variables** are a simple IPC mechanism for configuration
4. **Type-safe interfaces** (like MaskConfig) prevent bugs
5. **Minimal patches** are easier to maintain across Firefox updates

### For Security Researchers

1. **JavaScript injection is detectable** - C++ modifications are not
2. **Visual consistency matters** - window dimensions must match
3. **Font fingerprinting is powerful** - whitelists are necessary
4. **HTTP/JS mismatches** are a common leak
5. **navigator.webdriver** is the easiest detection vector

### For Privacy Advocates

1. Telemetry can be **completely removed** at compile time
2. Mozilla services (Pocket, sync, etc.) are **modular and removable**
3. Search engine integration is **optional**
4. Browser fingerprinting requires **holistic defense** (no single fix)
5. Open source allows **full auditing** of privacy claims

## Hands-On: Try It Yourself

Want to understand how MaskConfig works? Here's a simple exercise:

###  Exercise 1: Add a New Spoofable Property

Let's add support for spoofing `navigator.vendor`.

**Step 1**: Find the file that implements `navigator.vendor`

```bash
cd camoufox-*
grep -r "GetVendor" dom/base/
# Result: dom/base/Navigator.cpp
```

**Step 2**: Modify `Navigator.cpp`

```cpp
void Navigator::GetVendor(nsAString& aVendor) {
  // Add MaskConfig check
  auto spoofedVendor = MaskConfig::GetString("navigator.vendor");
  if (spoofedVendor.has_value()) {
    CopyUTF8toUTF16(mozilla::MakeStringSpan(spoofedVendor.value()), aVendor);
    return;
  }

  // Original Firefox code
  aVendor.AssignLiteral("Mozilla");
}
```

**Step 3**: Rebuild and test

```bash
make build
./launch --config '{"navigator.vendor": "Google Inc."}' --headless
# Open DevTools console
navigator.vendor  // Should return "Google Inc."
```

### Exercise 2: Understand the Patch System

**Step 1**: Check out the first commit

```bash
git checkout 1090f6a
```

**Step 2**: Examine a patch file

```bash
cat patches/fingerprint-injection.patch | head -100
```

**Step 3**: Understand the patch format

Patches use unified diff format:
- Lines starting with `-` are removed
- Lines starting with `+` are added
- Lines with neither are context (unchanged)

**Step 4**: Apply a patch manually

```bash
# Get Firefox source
make fetch

# Apply just one patch
cd camoufox-*
patch -p1 < ../patches/fingerprint-injection.patch

# See what changed
git diff dom/base/Navigator.cpp
```

## Next Steps

Now that you understand Camoufox's foundation, you're ready to explore:

- **[01-fingerprinting-detection.md](./01-fingerprinting-detection.md)** - Deep dive into fingerprinting techniques
- **[02-playwright-juggler.md](./02-playwright-juggler.md)** - How automation stays undetectable
- **[06-build-system.md](./06-build-system.md)** - Building your own fork

## External References

### Browser Fingerprinting
- [Fingerprinting Guidance on MDN](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Pixel_manipulation_with_canvas#fingerprinting)
- [CreepJS - Fingerprint Testing Tool](https://abrahamjuliot.github.io/creepjs/)
- [Browser Fingerprinting: A Survey](https://arxiv.org/abs/1905.01051) - Academic paper

### Firefox Development
- [Firefox Source Docs](https://firefox-source-docs.mozilla.org/)
- [Building Firefox](https://firefox-source-docs.mozilla.org/setup/index.html)
- [Searchfox - Firefox Source Code Search](https://searchfox.org/)

### Anti-Detection
- [TOR Browser Design](https://2019.www.torproject.org/projects/torbrowser/design/) - Anti-fingerprinting architecture
- [Playwright Stealth Plugin](https://github.com/berstend/puppeteer-extra/tree/master/packages/puppeteer-extra-plugin-stealth) - JavaScript approach (comparison)

### C++ and Firefox Internals
- [Firefox Platform API](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox)
- [XPCOM Overview](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM)
- [WebIDL in Gecko](https://developer.mozilla.org/en-US/docs/Mozilla/WebIDL_bindings)

---

**Next**: [01-fingerprinting-detection.md](./01-fingerprinting-detection.md) - Evolution of anti-fingerprinting features →
