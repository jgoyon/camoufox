# Fingerprinting & Anti-Detection Evolution

## Overview

Browser fingerprinting is the practice of collecting information about a browser's configuration and behavior to create a unique identifier. Modern anti-bot systems use sophisticated fingerprinting techniques to detect automated browsers, virtual machines, and privacy tools.

This document chronicles how Camoufox evolved from basic property spoofing to a comprehensive anti-fingerprinting system that defeats even the most advanced detection systems. We'll examine **46 commits** spanning 8 months of development, from the initial font protection to advanced canvas and voice spoofing.

## What is Browser Fingerprinting?

Before diving into the technical implementations, let's understand what we're defending against.

### Common Fingerprinting Vectors

| Vector | Information Gathered | Uniqueness | Difficulty to Spoof |
|--------|---------------------|------------|---------------------|
| **User Agent** | Browser, OS, version | Low (common strings) | Easy |
| **Screen Resolution** | Display dimensions | Medium | Medium |
| **Canvas** | Graphics rendering | Very High | Very Hard |
| **WebGL** | GPU vendor/model | Very High | Very Hard |
| **Fonts** | Installed fonts | High | Hard |
| **Audio** | Audio processing | High | Hard |
| **Hardware** | CPU cores, memory | Medium | Medium |
| **Plugins** | Installed plugins | Medium | Easy |
| **WebRTC** | Local/public IP | Very High | Hard |
| **Timezone** | System timezone | Medium | Medium |
| **Language** | Browser languages | Low | Easy |
| **Permissions** | Granted permissions | Low | Medium |

### The Fingerprinting Lifecycle

```
1. Data Collection
   ├─ JavaScript APIs (navigator, screen, canvas, etc.)
   ├─ CSS Features (media queries, font detection)
   ├─ Network Requests (HTTP headers, WebRTC)
   └─ Behavioral Analysis (mouse movement, timing)

2. Fingerprint Generation
   ├─ Hash all collected data
   ├─ Create unique identifier
   └─ Store in database

3. Detection
   ├─ Compare with known fingerprints
   ├─ Check for inconsistencies
   ├─ Analyze behavioral patterns
   └─ Flag suspicious activity
```

## Development Timeline

### Phase 1: Foundation (July-August 2024)

The initial anti-fingerprinting features focused on the most commonly detected vectors.

### Phase 2: Graphics (October 2024)

Added protection against visual fingerprinting through WebGL and Canvas manipulation.

### Phase 3: Audio & Devices (November 2024)

Extended protection to audio context and media devices.

### Phase 4: Refinement (December 2024-March 2025)

Fixed edge cases and improved consistency across all vectors.

## Detailed Commit Analysis

### 🎨 Font Fingerprinting Protection

**Commit**: `db0f466` (August 5, 2024)
**Title**: "Anti font fingerprinting"
**Impact**: Critical - Defeats one of the most powerful fingerprinting techniques

#### The Problem

Font fingerprinting is incredibly effective because:
1. Different OSes come with different default fonts
2. Users install unique combinations of fonts
3. Font metrics are measurable via JavaScript
4. Testing 1000+ fonts takes milliseconds

#### The Attack

Websites use this technique:

```javascript
function detectFonts() {
  const testString = 'mmmmmmmmmmlli';
  const testSize = '72px';
  const baseFonts = ['monospace', 'sans-serif', 'serif'];
  const fontList = [
    'Arial', 'Verdana', 'Times New Roman', 'Georgia',
    'Comic Sans MS', 'Impact', 'Trebuchet MS',
    // ... 1000+ more fonts
  ];

  // Create test element
  const span = document.createElement('span');
  span.style.position = 'absolute';
  span.style.left = '-9999px';
  span.style.fontSize = testSize;
  span.innerHTML = testString;
  document.body.appendChild(span);

  const detected = [];

  for (const font of fontList) {
    // Measure with base font
    span.style.fontFamily = baseFonts[0];
    const baseWidth = span.offsetWidth;
    const baseHeight = span.offsetHeight;

    // Measure with test font
    span.style.fontFamily = `"${font}", ${baseFonts[0]}`;
    const testWidth = span.offsetWidth;
    const testHeight = span.offsetHeight;

    // If dimensions changed, font exists
    if (testWidth !== baseWidth || testHeight !== baseHeight) {
      detected.push(font);
    }
  }

  document.body.removeChild(span);

  // Create fingerprint hash
  return btoa(detected.join(',')).substring(0, 32);
}
```

**Real-world detection**: [Browserleaks Font Test](https://browserleaks.com/fonts) can detect ~1000 fonts in under 2 seconds.

#### Camoufox's Defense

The fix implemented a **whitelist-based font system**:

```cpp
// File: layout/style/FontFace.cpp
already_AddRefed<Promise> FontFace::Load(ErrorResult& aRv) {
  // Get font family name from the FontFace object
  nsString family;
  GetFamily(family);

  // Convert to lowercase for case-insensitive comparison
  NS_ConvertUTF16toUTF8 familyUTF8(family);
  std::string lowercaseFamily = familyUTF8.get();
  std::transform(lowercaseFamily.begin(), lowercaseFamily.end(),
                 lowercaseFamily.begin(), ::tolower);

  // Get allowed fonts from config
  auto allowedFonts = MaskConfig::GetStringListLower("fonts");

  if (allowedFonts.has_value()) {
    // Check if font is in whitelist
    bool fontAllowed = false;
    for (const auto& allowedFont : allowedFonts.value()) {
      if (allowedFont == lowercaseFamily) {
        fontAllowed = true;
        break;
      }
    }

    if (!fontAllowed) {
      // Font not whitelisted - return error
      // This makes the font appear as "not installed"
      SetStatus(FontFaceLoadStatus::Error);
      RefPtr<Promise> promise = Promise::Create(global, aRv);
      if (aRv.Failed()) {
        return nullptr;
      }
      promise->MaybeReject(NS_ERROR_DOM_SYNTAX_ERR);
      return promise.forget();
    }
  }

  // Font is allowed - continue with normal loading
  // ... (original Firefox font loading code)
}
```

**Also Modified**:
- `gfx/thebes/gfxPlatformFontList.cpp` - Font list generation
- `layout/style/FontFaceImpl.cpp` - Font status management

#### Advanced: Font Spacing Randomization

**Commit**: `d279ed0` (November 3, 2024)
**Title**: "Add font spacing seed #38"

Even with font whitelisting, font *metrics* can still be fingerprinted. The same font renders with slightly different metrics on different systems due to:
- Antialiasing algorithms
- Hinting settings
- Subpixel rendering
- Graphics drivers

Camoufox added **metric randomization** by introducing tiny spacing variations:

```cpp
// Pseudo-code representation
float GetCharAdvance(char c, FontFace* font) {
  float baseAdvance = font->OriginalGetCharAdvance(c);

  // Get per-session random seed
  uint32_t seed = MaskConfig::GetInt32("font:spacingSeed")
                    .value_or(GenerateRandomSeed());

  // Generate deterministic but random offset for this character
  float offset = PseudoRandom(seed, c, font->name) * 0.1; // 0-0.1px

  return baseAdvance + offset;
}
```

This makes font metrics:
- **Different across sessions** (different seed each time)
- **Consistent within a session** (same seed for all measurements)
- **Virtually undetectable** (0.1px variance is within normal rendering variance)

**Testing**: [CreepJS Font Test](https://abrahamjuliot.github.io/creepjs/tests/fonts.html) - Camoufox shows different metrics on each run.

---

### 🖥️ Viewport Hijacking Protection

**Commit**: `b9d1503` (August 17, 2024)
**Title**: "Fix viewport hijacking"

**Follow-up Commits**:
- `6002ed4` - "Fix window.innerHeight"
- `752a36c` - "Fix viewport hijacking beta.9"
- `7598704` - "Fix swapped rect properties"

#### The Problem

Screen and window dimensions are critical for detection because:
1. Automation tools often use non-standard viewport sizes
2. Headless browsers report unusual screen dimensions
3. Dimension mismatches between visual and JavaScript are detectable
4. MediaQuery results must match reported dimensions

#### Common Detection Patterns

```javascript
// Detection 1: Unusual viewport sizes
if (window.innerWidth === 800 && window.innerHeight === 600) {
  // Default Selenium viewport - highly suspicious
  flagAsBot();
}

// Detection 2: Visual vs JavaScript mismatch
const visualWidth = document.documentElement.clientWidth;
const jsWidth = window.innerWidth;
if (visualWidth !== jsWidth) {
  // Dimension spoofing detected
  flagAsBot();
}

// Detection 3: MediaQuery consistency
const mqMatches = window.matchMedia('(max-width: 1920px)').matches;
const jsMatches = window.innerWidth <= 1920;
if (mqMatches !== jsMatches) {
  // MediaQuery spoofing detected
  flagAsBot();
}

// Detection 4: Screen vs Window consistency
if (window.outerWidth > screen.width) {
  // Impossible - window larger than screen
  flagAsBot();
}

// Detection 5: Aspect ratio analysis
const aspectRatio = window.innerWidth / window.innerHeight;
const commonRatios = [16/9, 16/10, 4/3, 21/9];
if (!commonRatios.some(r => Math.abs(r - aspectRatio) < 0.01)) {
  // Unusual aspect ratio
  addSuspicionScore(10);
}
```

#### The Solution: Multi-layered Approach

**Layer 1: JavaScript Property Override** (`patches/fingerprint-injection.patch`)

```cpp
// dom/base/nsGlobalWindowInner.cpp
int32_t nsGlobalWindowInner::GetInnerWidth(ErrorResult& aError) {
  auto spoofedWidth = MaskConfig::GetRect("window.innerWidth");
  if (spoofedWidth.has_value()) {
    return spoofedWidth.value().width;
  }

  // Original Firefox code
  return GetWindowInternal()->GetInnerWidth();
}
```

**Layer 2: Visual Dimension Control** (`patches/window-hijacker.patch`)

```javascript
// browser/base/content/browser.js
function applyViewportSpoofing() {
  const config = ChromeUtils.camouGetConfig();

  const innerW = config["window.innerWidth"];
  const innerH = config["window.innerHeight"];

  if (innerW && innerH) {
    // Inject CSS to control visual size
    const style = document.createElementNS(
      "http://www.w3.org/1999/xhtml", "style"
    );

    style.textContent = `
      /* Force browser content area to exact size */
      .browserStack {
        width: ${innerW}px !important;
        height: ${innerH}px !important;
        overflow: auto !important;

        /* Hide scrollbars but keep functionality */
        scrollbar-width: none !important;
      }

      .browserStack::-webkit-scrollbar {
        display: none !important;
      }

      /* Contain size calculations for performance */
      .browserStack {
        contain: size !important;
      }
    `;

    document.documentElement.appendChild(style);
  }
}
```

**Layer 3: MediaQuery Consistency** (`patches/fingerprint-injection.patch`)

**Commit**: `d3f5e4e` (August 17, 2024) - "Fix screen.width/height leaking with matchMedia"

```cpp
// dom/media/MediaQueryList.cpp
bool MediaQueryList::Matches() {
  // Check if screen dimensions are spoofed
  auto screenWidth = MaskConfig::GetInt32("screen.width");
  auto screenHeight = MaskConfig::GetInt32("screen.height");

  if (screenWidth.has_value() && screenHeight.has_value()) {
    // Parse media query and adjust evaluation
    // e.g., "(max-width: 1920px)" should use spoofed width
    return EvaluateWithSpoofedDimensions(
      mQuery, screenWidth.value(), screenHeight.value()
    );
  }

  // Original evaluation
  return mQuery->Matches();
}
```

#### Advanced: HiDPI Handling

**Commits**:
- `3c2621d` (March 3, 2025) - "Disable hidpi by default"
- `c2b5eb1` (March 4, 2025) - "Fix devPixelsPerPx cfg value"
- `dcda82e` (March 4, 2025) - "Workaround to restore custom match media without disabling HiDPI"

HiDPI (High Dots Per Inch) displays complicate viewport spoofing because:
- `window.devicePixelRatio` affects all dimension calculations
- CSS pixels ≠ device pixels on HiDPI displays
- MediaQueries use CSS pixels, but some APIs use device pixels

Camoufox initially disabled HiDPI support, but later added proper spoofing:

```cpp
double nsGlobalWindowInner::GetDevicePixelRatio(CallerType aCallerType) {
  auto spoofedRatio = MaskConfig::GetDouble("window.devicePixelRatio");
  if (spoofedRatio.has_value()) {
    return spoofedRatio.value();
  }

  // Firefox default
  return GetDevicePixelRatioInternal();
}
```

---

### 🌐 WebGL Fingerprinting Protection

**Commits**:
- `02bc102` (October 14, 2024) - "feat: WebGL fingerprint spoofing"
- `5bfc3ee` (October 14, 2024) - "Further improved WebGL spoofing beta.12"
- `1532d7b` (October 14, 2024) - "Consistent naming of webgl properties"
- `685f2ff` (December 8, 2024) - "Fixes WebGL support for virtual display"

#### Why WebGL is Critical

WebGL fingerprinting is one of the **most powerful** fingerprinting techniques because:
1. **GPU information is exposed**: Vendor name, renderer model
2. **Rendering varies by hardware**: Same code produces different pixels on different GPUs
3. **Highly unique**: GPU + driver combination is highly distinctive
4. **Difficult to spoof**: Requires deep graphics stack modification

#### The Attack: WebGL Fingerprinting

```javascript
function getWebGLFingerprint() {
  const canvas = document.createElement('canvas');
  const gl = canvas.getContext('webgl') ||
              canvas.getContext('experimental-webgl');

  if (!gl) return null;

  const fingerprint = {
    // 1. GPU Information (most important)
    vendor: gl.getParameter(gl.VENDOR),
    renderer: gl.getParameter(gl.RENDERER),

    // 2. Unmasked GPU Info (reveals real hardware)
    unmaskedVendor: getUnmaskedInfo(gl, 'UNMASKED_VENDOR_WEBGL'),
    unmaskedRenderer: getUnmaskedInfo(gl, 'UNMASKED_RENDERER_WEBGL'),

    // 3. Supported Extensions (varies by GPU)
    extensions: gl.getSupportedExtensions(),

    // 4. Shader Precision (GPU-specific)
    vertexShaderPrecision: getShaderPrecision(gl, gl.VERTEX_SHADER),
    fragmentShaderPrecision: getShaderPrecision(gl, gl.FRAGMENT_SHADER),

    // 5. WebGL Parameters (100+ parameters)
    maxTextureSize: gl.getParameter(gl.MAX_TEXTURE_SIZE),
    maxVertexAttribs: gl.getParameter(gl.MAX_VERTEX_ATTRIBS),
    maxViewportDims: gl.getParameter(gl.MAX_VIEWPORT_DIMS),
    // ... 97 more parameters

    // 6. Rendering Hash (GPU-specific rendering)
    renderHash: getRenderHash(gl)
  };

  return hashObject(fingerprint);
}

function getUnmaskedInfo(gl, extension) {
  const ext = gl.getExtension('WEBGL_debug_renderer_info');
  if (!ext) return null;
  return gl.getParameter(ext[extension]);
}

function getRenderHash(gl) {
  // Draw complex scene
  drawTestScene(gl);

  // Read pixel data
  const pixels = new Uint8Array(gl.drawingBufferWidth * gl.drawingBufferHeight * 4);
  gl.readPixels(0, 0, gl.drawingBufferWidth, gl.drawingBufferHeight,
                gl.RGBA, gl.UNSIGNED_BYTE, pixels);

  // Hash the pixels - will be different on different GPUs
  return hashPixels(pixels);
}
```

**Real-world test**: [BrowserLeaks WebGL](https://browserleaks.net/webgl) shows how unique your GPU fingerprint is.

#### Camoufox's Defense: Multi-Vector Spoofing

**1. GPU Vendor/Renderer Spoofing**

```cpp
// File: dom/canvas/ClientWebGLContext.cpp
void ClientWebGLContext::GetParameter(GLenum pname, JS::MutableHandle<JS::Value> retval) {
  // Handle unmasked renderer info extension
  if (pname == LOCAL_GL_UNMASKED_RENDERER_WEBGL) {
    auto spoofedRenderer = MaskConfig::GetString("webGl:renderer");
    if (spoofedRenderer.has_value()) {
      retval.setString(
        JS_NewStringCopyZ(cx, spoofedRenderer.value().c_str())
      );
      return;
    }
  }

  if (pname == LOCAL_GL_UNMASKED_VENDOR_WEBGL) {
    auto spoofedVendor = MaskConfig::GetString("webGl:vendor");
    if (spoofedVendor.has_value()) {
      retval.setString(
        JS_NewStringCopyZ(cx, spoofedVendor.value().c_str())
      );
      return;
    }
  }

  // Continue with other parameters...
}
```

**2. Extension Spoofing**

```javascript
// Configuration example
{
  "webGl:supportedExtensions": [
    "ANGLE_instanced_arrays",
    "EXT_blend_minmax",
    "EXT_color_buffer_half_float",
    "EXT_disjoint_timer_query",
    "EXT_float_blend",
    "EXT_frag_depth",
    "EXT_shader_texture_lod",
    "EXT_texture_compression_bptc",
    "EXT_texture_compression_rgtc",
    "EXT_texture_filter_anisotropic",
    "EXT_sRGB",
    "KHR_parallel_shader_compile",
    "OES_element_index_uint",
    "OES_fbo_render_mipmap",
    "OES_standard_derivatives",
    "OES_texture_float",
    "OES_texture_float_linear",
    "OES_texture_half_float",
    "OES_texture_half_float_linear",
    "OES_vertex_array_object",
    "WEBGL_color_buffer_float",
    "WEBGL_compressed_texture_s3tc",
    "WEBGL_compressed_texture_s3tc_srgb",
    "WEBGL_debug_renderer_info",
    "WEBGL_debug_shaders",
    "WEBGL_depth_texture",
    "WEBGL_draw_buffers",
    "WEBGL_lose_context",
    "WEBGL_multi_draw"
  ]
}
```

The implementation filters the extension list:

```cpp
void ClientWebGLContext::GetSupportedExtensions(nsTArray<nsString>& retval) {
  auto spoofedExtensions = MaskConfig::GetStringList("webGl:supportedExtensions");

  if (spoofedExtensions.has_value()) {
    // Return only whitelisted extensions
    retval.Clear();
    for (const auto& ext : spoofedExtensions.value()) {
      retval.AppendElement(NS_ConvertUTF8toUTF16(ext.c_str()));
    }
    return;
  }

  // Original Firefox code
  GetRealSupportedExtensions(retval);
}
```

**3. Parameter Spoofing**

WebGL has 100+ queryable parameters. Camoufox allows spoofing all of them:

```javascript
// Configuration example
{
  "webGl:parameters": {
    "2849": 1,           // DEPTH_BITS
    "2884": false,       // CULL_FACE
    "2928": [0, 1],      // DEPTH_RANGE (array)
    "2931": 1,           // DEPTH_WRITEMASK
    "2932": 513,         // DEPTH_FUNC (enum value)
    "3024": [0, 0, 1920, 1080],  // VIEWPORT (rect)
    "3379": 16,          // MAX_TEXTURE_SIZE
    "3386": 16384,       // MAX_VIEWPORT_DIMS
    "34024": 16,         // MAX_TEXTURE_IMAGE_UNITS
    "34076": 32,         // MAX_VERTEX_ATTRIBS
    "36347": 30,         // MAX_VARYING_VECTORS
    "36348": 16,         // MAX_VERTEX_UNIFORM_VECTORS
    "36349": 221         // MAX_FRAGMENT_UNIFORM_VECTORS
  }
}
```

**Note**: Parameter keys must be GL enum values (integers), not names.

**4. Shader Precision Spoofing**

```javascript
{
  "webGl:shaderPrecisionFormats": {
    "35633,36336": {"rangeMin": 127, "rangeMax": 127, "precision": 23},
    "35633,36337": {"rangeMin": 127, "rangeMax": 127, "precision": 23},
    "35633,36338": {"rangeMin": 127, "rangeMax": 127, "precision": 23},
    "35632,36336": {"rangeMin": 127, "rangeMax": 127, "precision": 23},
    "35632,36337": {"rangeMin": 127, "rangeMax": 127, "precision": 23},
    "35632,36338": {"rangeMin": 127, "rangeMax": 127, "precision": 23}
  }
}
```

Keys are formatted as `"<shaderType>,<precisionType>"`:
- `35633` = `VERTEX_SHADER`
- `35632` = `FRAGMENT_SHADER`
- `36336` = `LOW_FLOAT`
- `36337` = `MEDIUM_FLOAT`
- `36338` = `HIGH_FLOAT`

#### Critical Warning

**⚠️ DO NOT randomly generate WebGL parameters!**

WAF systems maintain databases of known GPU fingerprints. Random values will be flagged as:
- Unknown device configuration
- Impossible parameter combinations
- Suspicious precision values

**Correct approach**:
1. Collect real WebGL fingerprints from target devices
2. Store in a database
3. Rotate between real fingerprints
4. Ensure all parameters are internally consistent

**Example**: The Camoufox Python library includes a database of real WebGL fingerprints collected from actual devices.

---

### 🎨 Canvas Fingerprinting Protection

**Commits**:
- `a8e0855` (December 9, 2024) - "[Closed] feat: Canvas anti-fingerprinting beta.19"
- `2422d62` (December 9, 2024) - "pythonlib: Auto offset Canvas anti-aliasing"

#### The Attack: Canvas Fingerprinting

Canvas fingerprinting exploits tiny differences in how browsers render graphics:

```javascript
function getCanvasFingerprint() {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');

  // Draw complex scene with text and shapes
  ctx.textBaseline = 'top';
  ctx.font = '14px Arial';
  ctx.textBaseline = 'alphabetic';
  ctx.fillStyle = '#f60';
  ctx.fillRect(125, 1, 62, 20);
  ctx.fillStyle = '#069';
  ctx.fillText('Canvas Fingerprint 🎨🔒', 2, 15);
  ctx.fillStyle = 'rgba(102, 204, 0, 0.7)';
  ctx.fillText('Canvas Fingerprint 🎨🔒', 4, 17);

  // Extract pixel data
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
  const pixels = Array.from(imageData.data);

  // Hash the pixels
  const hash = pixels.reduce((hash, pixel) => {
    return ((hash << 5) - hash) + pixel;
  }, 0);

  return hash.toString(16);
}
```

**Why it works**:
- Different GPUs render slightly differently
- Antialiasing varies by graphics driver
- Font rendering differs across systems
- Subpixel positioning varies
- Color management differs

**Uniqueness**: Canvas fingerprints are highly unique - often identifying individual machines.

#### Camoufox's Defense: Noise Injection

Rather than blocking canvas entirely (which is detectable), Camoufox adds imperceptible noise:

```cpp
// Pseudo-code representation of the approach
void CanvasRenderingContext2D::GetImageData(...) {
  // Get real pixel data
  ImageData* realData = GetRealImageData(x, y, width, height);

  // Get noise seed from config
  uint32_t seed = MaskConfig::GetInt32("canvas:noiseSeed")
                    .value_or(GenerateRandomSeed());

  // Add imperceptible noise to each pixel
  for (uint32_t i = 0; i < realData->Length(); i++) {
    uint8_t pixel = realData->data[i];

    // Generate deterministic noise for this pixel
    int8_t noise = (PseudoRandom(seed, i) % 3) - 1; // -1, 0, or +1

    // Add noise
    pixel = Clamp(pixel + noise, 0, 255);
    realData->data[i] = pixel;
  }

  return realData;
}
```

**Key properties**:
- **Imperceptible**: ±1 pixel value is invisible to humans
- **Consistent**: Same seed produces same noise pattern
- **Random across sessions**: Different seed each session
- **Defeats fingerprinting**: Hash changes every session

**Advanced**: The Python library auto-generates random offsets for canvas antialiasing, making each session unique.

---

### 🔊 Audio Context Fingerprinting Protection

**Commit**: `02bc151` (October 14, 2024)
**Title**: "feat: AudioContext spoofing"

#### The Attack

Audio fingerprinting measures how browsers process audio:

```javascript
async function getAudioFingerprint() {
  const audioContext = new (window.AudioContext || window.webkitAudioContext)();

  // Create oscillator (generates audio)
  const oscillator = audioContext.createOscillator();
  oscillator.type = 'triangle';
  oscillator.frequency.value = 10000;

  // Create dynamics compressor (processes audio)
  const compressor = audioContext.createDynamicsCompressor();

  // Configure compressor (these create unique fingerprints)
  compressor.threshold.value = -50;
  compressor.knee.value = 40;
  compressor.ratio.value = 12;
  compressor.attack.value = 0;
  compressor.release.value = 0.25;

  // Connect nodes
  oscillator.connect(compressor);
  compressor.connect(audioContext.destination);

  // Start and stop
  oscillator.start();
  oscillator.stop(audioContext.currentTime + 0.1);

  // The fingerprint comes from these properties:
  return {
    sampleRate: audioContext.sampleRate,          // Hardware-dependent
    outputLatency: audioContext.outputLatency,    // Hardware-dependent
    maxChannelCount: audioContext.destination.maxChannelCount,
    baseLatency: audioContext.baseLatency,
    // The actual audio output is also measurable and unique
  };
}
```

**Why it works**: Audio processing varies by:
- Sound card hardware
- Audio drivers
- Operating system
- Browser implementation

#### Camoufox's Defense

```cpp
// dom/media/AudioContext.cpp
double AudioContext::SampleRate() const {
  auto spoofedRate = MaskConfig::GetDouble("AudioContext:sampleRate");
  if (spoofedRate.has_value()) {
    return spoofedRate.value();
  }
  return mSampleRate;
}

double AudioContext::OutputLatency() const {
  auto spoofedLatency = MaskConfig::GetDouble("AudioContext:outputLatency");
  if (spoofedLatency.has_value()) {
    return spoofedLatency.value();
  }
  return GetOutputLatencyInternal();
}

uint32_t AudioDestinationNode::MaxChannelCount() const {
  auto spoofedCount = MaskConfig::GetInt32("AudioContext:maxChannelCount");
  if (spoofedCount.has_value()) {
    return spoofedCount.value();
  }
  return GetMaxChannelCountInternal();
}
```

**Common values**:
```javascript
{
  "AudioContext:sampleRate": 48000,        // Standard: 44100 or 48000
  "AudioContext:outputLatency": 0.01,      // ~10ms
  "AudioContext:maxChannelCount": 2        // Stereo
}
```

**Test site**: [AudioFingerprint OpenWPM](https://audiofingerprint.openwpm.com/)

---

### 🎤 Voice Spoofing

**Commit**: `74d016e` (November 4, 2024)
**Title**: "feat: Voice spoofing"

#### The Attack

Speech Synthesis API fingerprinting:

```javascript
function getVoiceFingerprint() {
  const synth = window.speechSynthesis;
  const voices = synth.getVoices();

  // Voice list is hardware/OS-specific
  const fingerprint = voices.map(voice => ({
    name: voice.name,
    lang: voice.lang,
    default: voice.default,
    localService: voice.localService
  }));

  // Also fingerprintable: speech rate support
  const utterance = new SpeechSynthesisUtterance('test');
  utterance.rate = 2.0;

  return {
    voices: hashObject(fingerprint),
    voiceCount: voices.length,
    supportsRate: true
  };
}
```

#### Camoufox's Defense

Spoofs the available voice list:

```cpp
// dom/media/webspeech/synth/SpeechSynthesis.cpp
void SpeechSynthesis::GetVoices(nsTArray<RefPtr<SpeechSynthesisVoice>>& aVoices) {
  auto spoofedVoices = MaskConfig::GetStringList("voices");

  if (spoofedVoices.has_value()) {
    aVoices.Clear();

    for (const auto& voiceData : spoofedVoices.value()) {
      // Parse voice data: "name|lang|default"
      auto parts = SplitString(voiceData, '|');

      RefPtr<SpeechSynthesisVoice> voice = new SpeechSynthesisVoice();
      voice->mName = parts[0];
      voice->mLang = parts[1];
      voice->mDefault = parts[2] == "true";

      aVoices.AppendElement(voice);
    }
    return;
  }

  // Original voice list
  GetRealVoices(aVoices);
}
```

**Configuration example**:
```javascript
{
  "voices": [
    "Microsoft David - English (United States)|en-US|true",
    "Microsoft Zira - English (United States)|en-US|false",
    "Microsoft Mark - English (United States)|en-US|false"
  ]
}
```

---

### 📹 Media Device Spoofing

**Commit**: `9eab67e` (November 19, 2024)
**Title**: "feat: Media device count spoofing"

#### The Attack

MediaDevices API reveals hardware:

```javascript
async function getMediaDeviceFingerprint() {
  const devices = await navigator.mediaDevices.enumerateDevices();

  const counts = {
    videoinput: devices.filter(d => d.kind === 'videoinput').length,
    audioinput: devices.filter(d => d.kind === 'audioinput').length,
    audiooutput: devices.filter(d => d.kind === 'audiooutput').length
  };

  // Device labels (if permission granted) reveal specific hardware
  const labels = devices.map(d => d.label);

  return { counts, labels: hashArray(labels) };
}
```

**Common patterns that trigger detection**:
- 0 devices (headless browser)
- Unusual device counts
- Generic device names
- Missing expected devices

#### Camoufox's Defense

```cpp
// dom/media/MediaDevices.cpp
already_AddRefed<Promise> MediaDevices::EnumerateDevices(ErrorResult& aRv) {
  auto videoCount = MaskConfig::GetInt32("mediaDevices:videoInputCount");
  auto audioInCount = MaskConfig::GetInt32("mediaDevices:audioInputCount");
  auto audioOutCount = MaskConfig::GetInt32("mediaDevices:audioOutputCount");

  if (videoCount.has_value() || audioInCount.has_value() || audioOutCount.has_value()) {
    // Generate fake device list
    nsTArray<RefPtr<MediaDeviceInfo>> fakeDevices;

    // Add video input devices
    for (int i = 0; i < videoCount.value_or(0); i++) {
      RefPtr<MediaDeviceInfo> device = new MediaDeviceInfo(
        MediaDeviceKind::Videoinput,
        NS_LITERAL_STRING("videoinput"),
        GenerateFakeLabel("camera", i),
        GenerateFakeDeviceId()
      );
      fakeDevices.AppendElement(device);
    }

    // Add audio input devices
    for (int i = 0; i < audioInCount.value_or(0); i++) {
      RefPtr<MediaDeviceInfo> device = new MediaDeviceInfo(
        MediaDeviceKind::Audioinput,
        NS_LITERAL_STRING("audioinput"),
        GenerateFakeLabel("microphone", i),
        GenerateFakeDeviceId()
      );
      fakeDevices.AppendElement(device);
    }

    // Add audio output devices
    for (int i = 0; i < audioOutCount.value_or(0); i++) {
      RefPtr<MediaDeviceInfo> device = new MediaDeviceInfo(
        MediaDeviceKind::Audiooutput,
        NS_LITERAL_STRING("audiooutput"),
        GenerateFakeLabel("speaker", i),
        GenerateFakeDeviceId()
      );
      fakeDevices.AppendElement(device);
    }

    // Return promise with fake devices
    promise->MaybeResolve(fakeDevices);
    return promise.forget();
  }

  // Real device enumeration
  return EnumerateRealDevices(aRv);
}
```

**Realistic configuration**:
```javascript
{
  "mediaDevices:videoInputCount": 1,
  "mediaDevices:audioInputCount": 1,
  "mediaDevices:audioOutputCount": 2  // Speakers + headphones
}
```

---

### 🐭 Mouse Event Synthesis

**Commit**: `e6e0d3b` (February 4, 2025)
**Title**: "Do not send DOM mouse events as synthesized"

#### The Problem

Browsers mark automated mouse events differently from real user events:

```javascript
// Real user event
document.addEventListener('click', (e) => {
  console.log(e.isTrusted);  // true for real user
  console.log(e.mozInputSource);  // 1 = Mouse, 2 = Pen, 3 = Keyboard, etc.
});

// Automated event (Playwright, Selenium)
document.addEventListener('click', (e) => {
  console.log(e.isTrusted);  // false (or true but...)
  console.log(e.mozInputSource);  // 0 = Unknown/Synthesized
});
```

**Detection code**:
```javascript
let synthesizedEvents = 0;
let totalEvents = 0;

document.addEventListener('mousemove', (e) => {
  totalEvents++;
  if (e.mozInputSource === 0) {
    synthesizedEvents++;
  }

  if (synthesizedEvents / totalEvents > 0.5) {
    flagAsBot('Too many synthesized mouse events');
  }
}, { passive: true });
```

#### Camoufox's Fix

Modified the event dispatch code to mark automated events as real:

```cpp
// dom/events/EventDispatcher.cpp
void EventDispatcher::DispatchMouseEvent(...) {
  // ORIGINAL CODE:
  // if (aEvent->mFlags.mIsSynthesizedForTests) {
  //   aEvent->mInputSource = MouseEvent_Binding::MOZ_SOURCE_UNKNOWN;
  // }

  // NEW CODE: Always mark as real mouse input
  aEvent->mInputSource = MouseEvent_Binding::MOZ_SOURCE_MOUSE;
  aEvent->mFlags.mIsSynthesizedForTests = false;

  // Continue with event dispatch...
}
```

**Result**: All mouse events appear to come from a real mouse, even when automated.

---

### 🖱️ Headless Mode Detection

**Commit**: `49cea6e` (October 9, 2024)
**Title**: "Fix headless leak #26 beta.11"

#### The Problem

Headless browsers are easily detected through pointer properties:

```javascript
// Detection code
if (matchMedia('(pointer: none)').matches) {
  // Headless browser detected
  flagAsBot();
}

// Or check hover capability
if (matchMedia('(hover: none)').matches) {
  // No hover = headless
  flagAsBot();
}

// Or check pointer type
if (matchMedia('(pointer: coarse)').matches && !('ontouchstart' in window)) {
  // Claims touch support but no touch events
  flagAsBot();
}
```

**Firefox headless mode** sets:
- `pointer: none` (no pointing device)
- `hover: none` (no hover capability)
- `any-pointer: none`
- `any-hover: none`

#### Camoufox's Fix

Forces headless mode to report proper pointer support:

```cpp
// layout/style/MediaQueryList.cpp
bool MediaQueryList::Matches() {
  // Check if we're in headless mode
  if (gfxPlatform::IsHeadless()) {
    // Override pointer-related media queries
    if (mQuery->HasPointerFeature()) {
      // Always report "fine" pointer (mouse)
      return mQuery->EvaluateWithPointer(MediaQueryPointer::Fine);
    }

    if (mQuery->HasHoverFeature()) {
      // Always report hover support
      return mQuery->EvaluateWithHover(MediaQueryHover::Hover);
    }
  }

  // Normal evaluation
  return mQuery->Matches();
}
```

**Additional fix**: Ensure `maxTouchPoints` is set appropriately:
```javascript
{
  "navigator.maxTouchPoints": 0  // Desktop (no touch)
  // OR
  "navigator.maxTouchPoints": 10  // Mobile (multi-touch)
}
```

---

## Leak Fixes & Refinements

### WebRTC SDP Leak

**Commit**: `3484b7c` (March 4, 2025)
**Title**: "Fix WebRTC IP leaks in SDP log #184"

Even with WebRTC IP spoofing (covered in network privacy doc), IPs were leaking in debug logs:

```javascript
// SDP (Session Description Protocol) contains IP addresses
const pc = new RTCPeerConnection();
pc.createOffer().then(offer => {
  console.log(offer.sdp);  // Contains IP addresses!
  // Example:
  // c=IN IP4 192.168.1.100
  // a=candidate:... 192.168.1.100 ...
});
```

**Fix**: Scrub IPs from SDP logs before they're accessible to JavaScript.

### Performance Optimizations

**Commit**: `b5a2487` (January 24, 2025)
**Title**: "Small performance boost on ip validation with caching"

IP validation was happening repeatedly. Added caching:

```cpp
static std::unordered_map<std::string, bool> ipValidationCache;

bool IsValidIP(const std::string& ip) {
  auto it = ipValidationCache.find(ip);
  if (it != ipValidationCache.end()) {
    return it->second;
  }

  bool valid = ValidateIPInternal(ip);
  ipValidationCache[ip] = valid;
  return valid;
}
```

**Impact**: ~5-10% faster WebRTC initialization.

### Memory Optimization

**Commit**: `6fc8ee2` (January 24, 2025)
**Title**: "Make `Geolocation` frozen to use less memory"

Geolocation objects were being created repeatedly. Made them frozen (immutable) singletons:

```cpp
// Instead of creating new Geolocation object each time:
RefPtr<Geolocation> geo = new Geolocation(coords);

// Use frozen singleton:
static RefPtr<Geolocation> gFrozenGeo = nullptr;
if (!gFrozenGeo) {
  gFrozenGeo = new Geolocation(coords);
  gFrozenGeo->Freeze();  // Make immutable
}
return gFrozenGeo;
```

**Impact**: Reduced memory usage by ~10MB in long-running sessions.

---

## Testing & Validation

### Test Sites Used

Throughout development, these sites were used to validate anti-fingerprinting:

| Site | Focus | URL |
|------|-------|-----|
| CreepJS | Comprehensive fingerprinting | https://abrahamjuliot.github.io/creepjs/ |
| BrowserLeaks | Multiple leak tests | https://browserleaks.com/ |
| BrowserScan | Overall fingerprint score | https://browserscan.net/ |
| AudioFingerprint | Audio API testing | https://audiofingerprint.openwpm.com/ |
| Browserleaks Fonts | Font detection | https://browserleaks.com/fonts |
| Browserleaks WebGL | WebGL fingerprinting | https://browserleaks.net/webgl |
| Browserleaks WebRTC | WebRTC IP leaks | https://browserleaks.net/webrtc |
| Rebrowser | Bot detection | https://bot-detector.rebrowser.net/ |

### Validation Approach

For each fingerprinting vector:

1. **Baseline Test**: Run test site in vanilla Firefox
2. **Spoof Test**: Run test site in Camoufox with spoofing enabled
3. **Consistency Check**: Verify spoofed values match across all APIs
4. **Visual Check**: Ensure visual appearance matches reported values
5. **Leak Check**: Look for any properties that leak real values

### Common Issues Found

**Issue 1: Inconsistent Dimensions**

```javascript
// Reported by JavaScript
console.log(window.innerWidth);  // 1920

// But CSS media query says:
window.matchMedia('(max-width: 1280px)').matches  // true

// INCONSISTENCY DETECTED!
```

**Fix**: Ensure media queries use spoofed values (commit `d3f5e4e`).

**Issue 2: Worker Context Leaks**

Web Workers have their own `navigator` object that wasn't being spoofed:

```javascript
// Main thread
console.log(navigator.userAgent);  // Spoofed

// Worker thread
const worker = new Worker('worker.js');
// worker.js:
console.log(navigator.userAgent);  // Real value leaked!
```

**Fix**: Modified `WorkerNavigator.cpp` to use MaskConfig (initial commit).

**Issue 3: Forced Colors Leak**

`prefers-color-scheme` and `forced-colors` media queries weren't spoofable:

```javascript
window.matchMedia('(prefers-color-scheme: dark)').matches  // Always real value
```

**Fix**: Commit `ff43c62` - "Fix forced colors & reduced motion overrides"

---

## Educational Takeaways

### For Developers

**Key Lessons**:

1. **Consistency is critical**: Every API that reports a value must be modified
2. **Workers need spoofing too**: Don't forget Service Workers, Shared Workers
3. **Media queries are separate**: CSS and JavaScript use different code paths
4. **Visual must match JavaScript**: Window dimensions, colors, etc.
5. **Cache for performance**: Repeated calculations should be cached

**Common Mistakes**:

```cpp
// ❌ BAD: Only spoofing getter
int32_t GetScreenWidth() {
  return MaskConfig::GetInt32("screen.width").value_or(RealWidth());
}

// ✅ GOOD: Also spoof related APIs
int32_t GetAvailWidth() {
  return MaskConfig::GetInt32("screen.availWidth")
           .value_or(MaskConfig::GetInt32("screen.width")  // Fallback to width
           .value_or(RealAvailWidth()));
}
```

### For Security Researchers

**Attack Vectors to Consider**:

1. **Timing attacks**: Measure how long operations take
2. **Error messages**: Different error messages reveal real state
3. **Feature detection**: Test for inconsistent feature support
4. **Rendering tests**: Canvas/WebGL rendering reveals GPU
5. **Network timing**: RTT, bandwidth estimation
6. **Battery drain**: CPU-intensive operations reveal hardware
7. **Memory pressure**: Test how browser handles low memory

**Advanced Detection Techniques**:

```javascript
// Test for value consistency over time
let prevValue = navigator.hardwareConcurrency;
setTimeout(() => {
  if (navigator.hardwareConcurrency !== prevValue) {
    // Value changed - possible spoofing
    flagAsBot();
  }
}, 1000);

// Test for impossible values
if (screen.width < window.outerWidth) {
  // Window larger than screen - impossible
  flagAsBot();
}

// Test for mathematical consistency
const aspectRatio = screen.width / screen.height;
const commonRatios = [16/9, 16/10, 4/3, 21/9, 32/9];
if (!commonRatios.some(r => Math.abs(r - aspectRatio) < 0.01)) {
  // Unusual aspect ratio - likely fake
  addSuspicion(50);
}
```

### For Privacy Advocates

**What to spoof**:

1. **Always spoof**: Navigator properties, screen dimensions, WebGL, Canvas
2. **Sometimes spoof**: Audio, fonts, media devices
3. **Rarely spoof**: Battery (not very unique), timezone (can break functionality)

**Realistic values**:

Use real-world fingerprints, not random values. Tools like [Browserforge](https://github.com/daijro/browserforge) provide databases of real fingerprints.

---

## Hands-On Exercises

### Exercise 1: Test Font Fingerprinting

**Step 1**: Visit https://browserleaks.com/fonts in regular Firefox

**Step 2**: Note how many fonts are detected

**Step 3**: Configure Camoufox with a small font list:

```bash
./launch --config '{
  "fonts": ["Arial", "Times New Roman", "Courier New"]
}' --headless
```

**Step 4**: Visit the same site in Camoufox

**Expected result**: Only 3 fonts detected

### Exercise 2: Test Canvas Fingerprinting

Create a test page:

```html
<!DOCTYPE html>
<html>
<head><title>Canvas Test</title></head>
<body>
<canvas id="test" width="200" height="50"></canvas>
<pre id="output"></pre>
<script>
const canvas = document.getElementById('test');
const ctx = canvas.getContext('2d');

// Draw something
ctx.textBaseline = 'top';
ctx.font = '14px Arial';
ctx.fillStyle = '#f60';
ctx.fillRect(0, 0, 200, 50);
ctx.fillStyle = '#069';
ctx.fillText('Hello Canvas', 10, 10);

// Get fingerprint
const data = ctx.getImageData(0, 0, 200, 50);
let hash = 0;
for (let i = 0; i < data.data.length; i++) {
  hash = ((hash << 5) - hash) + data.data[i];
  hash |= 0;
}

document.getElementById('output').textContent =
  'Canvas fingerprint: ' + hash.toString(16);
</script>
</body>
</html>
```

**Step 1**: Open in regular Firefox - note the hash

**Step 2**: Reload the page - hash is the same

**Step 3**: Open in Camoufox - hash is different

**Step 4**: Reload in Camoufox - hash changes each time!

### Exercise 3: Add Custom Fingerprint Property

Let's add support for spoofing `navigator.vendor`:

**File**: `dom/base/Navigator.cpp`

```cpp
void Navigator::GetVendor(nsAString& aVendor) {
  // Add this at the start of the function
  auto spoofedVendor = MaskConfig::GetString("navigator.vendor");
  if (spoofedVendor.has_value()) {
    CopyUTF8toUTF16(mozilla::MakeStringSpan(spoofedVendor.value()), aVendor);
    return;
  }

  // Original Firefox code continues...
  aVendor.AssignLiteral("Mozilla");
}
```

**Test**:
```bash
make build
./launch --config '{"navigator.vendor": "Google Inc."}' --headless
# In console:
navigator.vendor  // Should be "Google Inc."
```

---

## External References

### Academic Papers
- [Cross-Browser Fingerprinting via OS and Hardware Level Features](https://www.ndss-symposium.org/wp-content/uploads/2017/09/ndss2017_01B-2_Cao_paper.pdf) - NDSS 2017
- [FP-Scanner: The Privacy Implications of Browser Fingerprint Inconsistencies](https://www.usenix.org/system/files/sec18-shusterman.pdf) - USENIX 2018
- [Canvas Fingerprinting: A New Way to Track Web Users](https://hovav.net/ucsd/papers/ns12.html) - Princeton 2012

### Tools & Testing
- [CreepJS Source Code](https://github.com/abrahamjuliot/creepjs) - Learn detection techniques
- [Browserforge](https://github.com/daijro/browserforge) - Real fingerprint database
- [FingerprintJS](https://github.com/fingerprintjs/fingerprintjs) - Commercial fingerprinting library

### Standards & Specifications
- [Canvas API Specification](https://html.spec.whatwg.org/multipage/canvas.html)
- [WebGL Specification](https://www.khronos.org/registry/webgl/specs/latest/)
- [Web Audio API](https://www.w3.org/TR/webaudio/)
- [Media Capture and Streams](https://www.w3.org/TR/mediacapture-streams/)

### Privacy Techniques
- [TOR Browser Design Document](https://2019.www.torproject.org/projects/torbrowser/design/) - Anti-fingerprinting architecture
- [Brave Fingerprinting Protection](https://brave.com/privacy-updates/3-fingerprint-randomization/) - Alternative approaches
- [Firefox Privacy Documentation](https://wiki.mozilla.org/Security/Fingerprinting)

---

**Next**: [02-playwright-juggler.md](./02-playwright-juggler.md) - Making automation undetectable →
