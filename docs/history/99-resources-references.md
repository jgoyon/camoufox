# Learning Resources & External References

## Overview

This document provides a curated collection of resources for learning about browser fingerprinting, anti-detection techniques, Firefox development, and contributing to privacy-preserving browser projects. Whether you're a beginner or an experienced developer, these resources will help you understand the foundations and advanced concepts behind Camoufox.

## 📚 Browser Fingerprinting Fundamentals

### Academic Papers

#### Essential Reading

1. **[(Cross-)Browser Fingerprinting via OS and Hardware Level Features](https://yinzhicao.org/TrackingFree/crossbrowsertracking_NDSS17.pdf)**
   - Authors: Yinzhi Cao, Song Li, Erik Wijmans
   - Conference: NDSS 2017
   - **Why read**: Comprehensive overview of hardware-level fingerprinting
   - **Key takeaways**: CPU, GPU, and OS fingerprinting techniques

2. **[Online Tracking: A 1-million-site Measurement and Analysis](https://webtransparency.cs.princeton.edu/webcensus/)**
   - Authors: Steven Englehardt, Arvind Narayanan
   - Institution: Princeton University
   - **Why read**: Real-world analysis of tracking techniques
   - **Key takeaways**: Canvas and font fingerprinting prevalence

3. **[FP-Scanner: The Privacy Implications of Browser Fingerprint Inconsistencies](https://www.usenix.org/system/files/sec18-shusterman.pdf)**
   - Authors: Antoine Vastel, et al.
   - Conference: USENIX Security 2018
   - **Why read**: How to detect anti-fingerprinting tools
   - **Key takeaways**: Consistency checking, leak detection

4. **[Beauty and the Beast: Diverting Modern Web Browsers to Build Unique Browser Fingerprints](https://hal.inria.fr/hal-01285470/file/beauty-sp16.pdf)**
   - Authors: Pierre Laperdrix, Walter Rudametkin, Benoit Baudry
   - Conference: IEEE S&P 2016
   - **Why read**: Comprehensive fingerprinting taxonomy
   - **Key takeaways**: 17+ different fingerprinting vectors

#### Advanced Topics

5. **[Amerge: Advanced WebAssembly Fingerprinting](https://mdn.dev/archives/media/attachments/2020/06/22/advanced-tor-browser-fingerprinting.pdf)**
   - **Topic**: WebAssembly as fingerprinting vector
   - **Key takeaways**: New attack surfaces beyond JavaScript

6. **[Clock Around the Clock: Time-Based Device Fingerprinting](https://www.securitee.org/files/clockaroundtheclock_ccs2018.pdf)**
   - Conference: ACM CCS 2018
   - **Topic**: Timing attacks for fingerprinting
   - **Key takeaways**: Clock skew fingerprinting

7. **[HSTS Super Cookies](https://www.radicalresearch.co.uk/lab/hstssupercookies/)**
   - **Topic**: Network-level fingerprinting
   - **Key takeaways**: HTTP Strict Transport Security abuse

### Technical Blogs & Articles

1. **[Fingerprinting Guidance - MDN](https://developer.mozilla.org/en-US/docs/Web/Privacy/Fingerprinting)**
   - Mozilla's official guidance on fingerprinting
   - Covers what website developers should know
   - Good overview of mitigation strategies

2. **[How to Make Your Site Privacy-Friendly](https://developer.mozilla.org/en-US/docs/Web/Privacy)**
   - MDN's comprehensive privacy guide
   - Ethical considerations
   - Privacy-preserving alternatives

3. **[Browser Fingerprinting: What Is It and What Should You Do About It?](https://blog.mozilla.org/en/products/firefox/browser-fingerprinting-what-is-it-and-what-should-you-do-about-it/)**
   - Mozilla's explanation for end-users
   - Good for understanding user perspective
   - Links to Firefox's protection features

4. **[Canvas Fingerprinting](https://hovav.net/ucsd/dist/canvas.pdf)**
   - Princeton research on canvas fingerprinting
   - First major canvas fingerprinting paper (2012)
   - Historical perspective on the technique

5. **[Audio Fingerprinting](https://audiofingerprint.openwpm.com/faq)**
   - AudioContext fingerprinting explained
   - Interactive demo and FAQ
   - Technical implementation details

### Books

1. **"Web Browser Engineering" by Pavel Panchekha & Chris Harrelson**
   - URL: https://browser.engineering/
   - **Why read**: Understand browser internals
   - **Relevant chapters**: Layout, Rendering, JavaScript execution
   - **Note**: Free online book

2. **"High Performance Browser Networking" by Ilya Grigorik**
   - Publisher: O'Reilly
   - **Why read**: Network-level browser behavior
   - **Relevant chapters**: HTTP, WebRTC, TLS
   - **Note**: Available free at https://hpbn.co/

---

## 🦊 Firefox Development

### Official Documentation

1. **[Firefox Source Docs](https://firefox-source-docs.mozilla.org/)**
   - Complete Firefox developer documentation
   - Build instructions, coding standards
   - Architecture overviews

2. **[Building Firefox](https://firefox-source-docs.mozilla.org/setup/)**
   - Step-by-step build guide
   - Platform-specific instructions
   - Troubleshooting common issues

3. **[Searchfox](https://searchfox.org/)**
   - **Essential tool**: Firefox source code search
   - Cross-reference navigation
   - Blame view, history
   - **Usage tip**: Search for function names to find implementations

4. **[Firefox Platform API](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox)**
   - API documentation
   - WebIDL interfaces
   - XPCOM components

### Firefox Internals

5. **[Gecko Overview](https://wiki.mozilla.org/Gecko:Overview)**
   - Rendering engine architecture
   - Process model
   - Component interactions

6. **[WebIDL in Gecko](https://developer.mozilla.org/en-US/docs/Mozilla/WebIDL_bindings)**
   - How JavaScript APIs are defined
   - C++ to JavaScript binding
   - Adding new APIs

7. **[XPCOM (Cross Platform Component Object Model)](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM)**
   - Firefox's component architecture
   - Creating components
   - Interface definition

8. **[Firefox Preferences System](https://firefox-source-docs.mozilla.org/modules/libpref/index.html)**
   - about:config system
   - Preference types
   - Default prefs, user prefs

### Development Tools

9. **[Mozilla Build System (mach)](https://firefox-source-docs.mozilla.org/mach/index.html)**
   - Build automation tool
   - Common commands
   - Extending mach

10. **[Firefox DevTools](https://firefox-source-docs.mozilla.org/devtools/)**
    - Browser DevTools architecture
    - Remote debugging protocol
    - DevTools API

---

## 🎭 Anti-Detection & Privacy

### Privacy-Focused Browsers

1. **[TOR Browser Design Document](https://2019.www.torproject.org/projects/torbrowser/design/)**
   - **Must read**: Comprehensive anti-fingerprinting design
   - Resistance goals and non-goals
   - Detailed defenses for each fingerprinting vector
   - **Key sections**: Navigator, screen, fonts, WebGL

2. **[Brave Privacy Features](https://brave.com/privacy-features/)**
   - Alternative anti-fingerprinting approach
   - Fingerprint randomization vs blocking
   - Privacy Badger integration

3. **[LibreWolf](https://librewolf.net/)**
   - Privacy-focused Firefox fork
   - Telemetry removal
   - Default settings for privacy

### Anti-Bot Frameworks

4. **[Puppeteer Extra Stealth Plugin](https://github.com/berstend/puppeteer-extra/tree/master/packages/puppeteer-extra-plugin-stealth)**
   - JavaScript-based anti-detection
   - **Comparison point**: See why C++ is better
   - Learn detection vectors from the evasions

5. **[undetected-chromedriver](https://github.com/ultrafunkamsterdam/undetected-chromedriver)**
   - Chrome automation evasion
   - Python-based approach
   - Patch management system

### Detection Testing Tools

6. **[CreepJS](https://abrahamjuliot.github.io/creepjs/)**
   - **Essential testing tool**: Comprehensive fingerprint test
   - [Source code](https://github.com/abrahamjuliot/creepjs) to learn detection
   - Multiple fingerprinting vectors
   - Lie detection (inconsistency checking)

7. **[BrowserLeaks](https://browserleaks.com/)**
   - Multiple specialized tests:
     - [WebRTC Leak Test](https://browserleaks.net/webrtc)
     - [Canvas Fingerprinting](https://browserleaks.net/canvas)
     - [WebGL Fingerprinting](https://browserleaks.net/webgl)
     - [Font Detection](https://browserleaks.net/fonts)
     - [IP Leak Test](https://browserleaks.com/ip)

8. **[BrowserScan](https://www.browserscan.net/)**
   - Overall fingerprint score
   - Proxy/VPN detection
   - WebRTC leak detection
   - Timezone and geolocation consistency

9. **[Rebrowser Bot Detector](https://bot-detector.rebrowser.net/)**
   - Specialized bot detection test
   - Pass/fail for specific checks
   - Useful for validation

10. **[AudioFingerprint OpenWPM](https://audiofingerprint.openwpm.com/)**
    - Audio context fingerprinting test
    - Visualization of audio processing
    - Technical explanation

---

## 🤖 Browser Automation

### Playwright

1. **[Playwright Documentation](https://playwright.dev/)**
   - Official Playwright docs
   - API reference
   - Best practices

2. **[Playwright Firefox Patches](https://github.com/microsoft/playwright/tree/main/browser_patches/firefox)**
   - **Essential for understanding Juggler**: Playwright's Firefox modifications
   - Patch files in the repository
   - Build scripts

3. **[Juggler (Mozilla's Protocol)](https://github.com/puppeteer/juggler)**
   - Original Juggler repository
   - Protocol documentation
   - Comparison with Chrome DevTools Protocol

### Protocol Specifications

4. **[Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)**
   - Industry-standard automation protocol
   - Domain organization
   - Event/command structure

5. **[WebDriver Specification](https://www.w3.org/TR/webdriver/)**
   - W3C standard for browser automation
   - Selenium uses this
   - Command definitions

---

## 🔧 C++ & Systems Programming

### C++ Fundamentals

1. **[C++ Reference](https://en.cppreference.com/)**
   - Standard library reference
   - Language features
   - Code examples

2. **[Modern C++ (C++11/14/17/20)](https://github.com/AnthonyCalandra/modern-cpp-features)**
   - New features in modern C++
   - `std::optional`, `std::variant`, etc.
   - Used extensively in Firefox

### Firefox C++ Patterns

3. **[Firefox C++ Coding Style](https://firefox-source-docs.mozilla.org/code-quality/coding-style/index.html)**
   - Naming conventions
   - Code organization
   - Review guidelines

4. **[Smart Pointers in Gecko](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Guide/Internal_strings)**
   - `RefPtr`, `nsCOMPtr`
   - Memory management
   - Avoiding leaks

5. **[nsString Guide](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Guide/Internal_strings)**
   - String types in Firefox
   - `nsString`, `nsCString`, `nsAString`
   - Conversion functions

---

## 🌐 Web Standards & APIs

### W3C Specifications

1. **[HTML Living Standard](https://html.spec.whatwog.org/)**
   - Canvas API: https://html.spec.whatwg.org/multipage/canvas.html
   - Navigator API: https://html.spec.whatwg.org/multipage/system-state.html#the-navigator-object
   - Media Capture: Handled by separate spec

2. **[WebGL Specification](https://www.khronos.org/registry/webgl/specs/latest/)**
   - WebGL 1.0 and 2.0
   - Extension registry
   - Conformance tests

3. **[Web Audio API](https://www.w3.org/TR/webaudio/)**
   - AudioContext specification
   - Nodes and processing
   - Security considerations

4. **[Media Capture and Streams](https://www.w3.org/TR/mediacapture-streams/)**
   - `getUserMedia()` API
   - MediaDevices enumeration
   - Constraints and capabilities

5. **[Geolocation API](https://www.w3.org/TR/geolocation-API/)**
   - Position interface
   - Accuracy and timestamps
   - Privacy considerations

### JavaScript APIs

6. **[MDN Web APIs](https://developer.mozilla.org/en-US/docs/Web/API)**
   - Comprehensive API documentation
   - Browser compatibility tables
   - Live examples

---

## 🛠️ Development Tools & Libraries

### Build & Compilation

1. **[Mozilla Build Tools](https://firefox-source-docs.mozilla.org/setup/index.html)**
   - Platform-specific toolchains
   - Compiler requirements
   - Bootstrap process

2. **[Mozconfig Files](https://firefox-source-docs.mozilla.org/build/buildsystem/mozconfigs.html)**
   - Build configuration syntax
   - Common options
   - Platform-specific settings

3. **[Docker for Firefox Development](https://firefox-source-docs.mozilla.org/setup/docker.html)**
   - Containerized builds
   - Cross-compilation
   - CI/CD integration

### Patch Management

4. **[GNU Patch](https://www.gnu.org/software/patch/manual/)**
   - Applying patches
   - Creating patches
   - Unified diff format

5. **[Git Patches](https://git-scm.com/docs/git-format-patch)**
   - Creating patch files from commits
   - Applying patch series
   - Patch management

### Testing

6. **[Firefox Test Frameworks](https://firefox-source-docs.mozilla.org/testing/)**
   - Mochitest (browser tests)
   - xpcshell (unit tests)
   - Web Platform Tests

---

## 🔐 Security & Privacy Research

### Research Groups

1. **[Brave Privacy Research](https://brave.com/research/)**
   - Browser privacy research
   - Fingerprinting studies
   - Open-source tools

2. **[Princeton Web Transparency Project](https://webtransparency.cs.princeton.edu/)**
   - Tracking measurement studies
   - OpenWPM crawler
   - Census datasets

3. **[INRIA Privatics Team](https://team.inria.fr/privatics/)**
   - European privacy research
   - Browser fingerprinting focus
   - FP-Collect project

### Security Communities

4. **[r/netsec](https://www.reddit.com/r/netsec/)**
   - Security news and research
   - Technical discussions
   - Tool releases

5. **[r/privacy](https://www.reddit.com/r/privacy/)**
   - Privacy tools and techniques
   - Browser comparison discussions
   - Privacy news

6. **[Bugcrowd & HackerOne](https://www.bugcrowd.com/)**
   - Bug bounty platforms
   - Security vulnerability disclosure
   - Learning from real exploits

---

## 📊 Fingerprinting Detection Services

### Commercial Services

1. **[FingerprintJS](https://fingerprintjs.com/)**
   - Commercial fingerprinting service
   - [Open-source version](https://github.com/fingerprintjs/fingerprintjs) available
   - Learn detection techniques from source

2. **[DataDome](https://datadome.co/)**
   - Bot detection service
   - [Bot bounty program](https://yeswehack.com/programs/datadome-bot-bounty)
   - Testing ground for evasion

3. **[PerimeterX](https://www.humansecurity.com/)**
   - Advanced bot detection
   - Behavioral analysis
   - CAPTCHA alternatives

### Open-Source Tools

4. **[Fingerprint2](https://github.com/fingerprintjs/fingerprintjs/tree/v2)**
   - Legacy open-source version (v2)
   - Simpler codebase for learning
   - Still widely used

5. **[Client.js](https://github.com/jackspirou/clientjs)**
   - Browser fingerprinting library
   - Educational example
   - Simple implementation

---

## 🐍 Python Automation Libraries

### Core Libraries

1. **[Playwright Python](https://playwright.dev/python/)**
   - Official Python bindings
   - Async and sync APIs
   - Page Object Model patterns

2. **[Selenium](https://selenium-python.readthedocs.io/)**
   - Classic automation framework
   - WebDriver protocol
   - Browser compatibility

3. **[Pyppeteer](https://github.com/pyppeteer/pyppeteer)**
   - Unofficial Puppeteer port
   - Chrome/Chromium only
   - Async-first design

### Helper Libraries

4. **[Faker](https://faker.readthedocs.io/)**
   - Fake data generation
   - Locales and providers
   - Useful for fingerprint generation

5. **[user-agents](https://github.com/intoli/user-agents)**
   - Real user agent database
   - Random user agent generation
   - Browser statistics

6. **[Browserforge](https://github.com/daijro/browserforge)**
   - **Camoufox's fingerprint database**
   - Real browser fingerprints
   - Statistical distributions
   - Header generation

---

## 🎓 Educational Courses & Tutorials

### Free Courses

1. **[CS253 Web Security - Stanford](https://web.stanford.edu/class/cs253/)**
   - Browser security fundamentals
   - XSS, CSRF, and other attacks
   - Modern web security

2. **[Browser Internals for Web Developers](https://frontendmasters.com/courses/web-browser-internals/)**
   - Frontend Masters course
   - How browsers work
   - Rendering pipeline

3. **[Mozilla Developer Network (MDN) Learning Area](https://developer.mozilla.org/en-US/docs/Learn)**
   - Web development fundamentals
   - JavaScript, HTML, CSS
   - Web APIs

### YouTube Channels

4. **[LiveOverflow](https://www.youtube.com/c/LiveOverflow)**
   - Security research and CTF
   - Browser exploitation
   - Reverse engineering

5. **[Hussein Nasser](https://www.youtube.com/c/HusseinNasser-software-engineering)**
   - Systems design and networking
   - WebRTC, HTTP/2, TLS
   - Backend engineering

6. **[Computerphile](https://www.youtube.com/user/Computerphile)**
   - Computer science concepts
   - Security and privacy topics
   - Academic perspectives

---

## 💬 Communities & Forums

### Discussion Forums

1. **[r/firefox](https://www.reddit.com/r/firefox/)**
   - Firefox community
   - Feature discussions
   - Troubleshooting help

2. **[Mozilla Discourse](https://discourse.mozilla.org/)**
   - Official Mozilla community forum
   - Add-on development
   - Firefox development

3. **[Stack Overflow - Firefox Tag](https://stackoverflow.com/questions/tagged/firefox)**
   - Technical Q&A
   - Development issues
   - Code examples

### Chat Communities

4. **[Mozilla Matrix Channels](https://wiki.mozilla.org/Matrix)**
   - Real-time chat
   - Developer channels
   - Official Mozilla communication

5. **[Playwright Discord](https://discord.gg/playwright)**
   - Playwright community
   - Support and discussions
   - Feature requests

---

## 📖 Contributing to Camoufox

### Getting Started

1. **Read the Documentation**
   - Start with `00-project-foundation.md`
   - Understand the MaskConfig system
   - Review existing patches

2. **Set Up Development Environment**
   - Follow build instructions in `06-build-system.md`
   - Use Docker for consistent environment
   - Test with `make run`

3. **Find an Issue to Work On**
   - Check [GitHub Issues](https://github.com/daijro/camoufox/issues)
   - Look for "good first issue" labels
   - Ask in discussions for guidance

### Development Workflow

4. **Creating a Patch**
   - Use the developer UI: `make edits`
   - Make changes in `camoufox-*/` directory
   - Test thoroughly with `make build && make run`
   - Generate patch file

5. **Testing Your Changes**
   - Test on all fingerprinting sites
   - Check for consistency leaks
   - Verify visual appearance
   - Test in headless mode

6. **Submitting a Pull Request**
   - Fork the repository
   - Create a feature branch
   - Commit with clear messages
   - Open PR with description

### Code Quality

7. **Firefox Coding Standards**
   - Follow [Firefox C++ style](https://firefox-source-docs.mozilla.org/code-quality/coding-style/index.html)
   - Use `./mach clang-format` for formatting
   - Add comments for complex logic

8. **Testing Best Practices**
   - Test on Linux, Windows, and macOS
   - Test both headless and headful modes
   - Check memory leaks with Valgrind
   - Performance test with benchmarks

---

## 🔗 Quick Reference Links

### Essential Bookmarks

| Resource | URL | Purpose |
|----------|-----|---------|
| **Searchfox** | https://searchfox.org/ | Firefox source search |
| **CreepJS** | https://abrahamjuliot.github.io/creepjs/ | Fingerprint testing |
| **BrowserLeaks** | https://browserleaks.com/ | Leak detection |
| **MDN** | https://developer.mozilla.org/ | Web API docs |
| **Firefox Source Docs** | https://firefox-source-docs.mozilla.org/ | Build & development |
| **TOR Design Doc** | https://2019.www.torproject.org/projects/torbrowser/design/ | Anti-fingerprinting design |
| **Playwright Docs** | https://playwright.dev/ | Automation framework |
| **Browserforge** | https://github.com/daijro/browserforge | Fingerprint database |

### Camoufox Repository Links

- **Main Repository**: https://github.com/daijro/camoufox
- **Python Library**: https://github.com/daijro/camoufox/tree/main/pythonlib
- **Issue Tracker**: https://github.com/daijro/camoufox/issues
- **Discussions**: https://github.com/daijro/camoufox/discussions

---

## 📝 Recommended Reading Order

### For Beginners

1. Start with MDN Web APIs to understand browser APIs
2. Read "Web Browser Engineering" for browser internals
3. Study TOR Browser Design Doc for anti-fingerprinting
4. Explore Firefox Source Docs for development setup
5. Read Camoufox `00-project-foundation.md` for architecture

### For Intermediate Developers

1. Study academic papers on fingerprinting techniques
2. Deep dive into Firefox C++ codebase with Searchfox
3. Analyze Playwright Firefox patches
4. Read Camoufox patch files to understand modifications
5. Experiment with creating your own patches

### For Advanced Contributors

1. Research latest fingerprinting papers
2. Analyze commercial fingerprinting services
3. Study WAF detection techniques
4. Develop novel anti-fingerprinting approaches
5. Contribute to Camoufox core features

---

## 🎯 Learning Paths

### Path 1: Browser Fingerprinting Expert

**Goal**: Understand all fingerprinting vectors and defenses

1. **Month 1**: Read fingerprinting papers, understand attack vectors
2. **Month 2**: Test fingerprinting sites, analyze detection code
3. **Month 3**: Study TOR and Brave anti-fingerprinting approaches
4. **Month 4**: Implement basic fingerprint spoofing in a test browser
5. **Month 5**: Contribute fingerprint protection patches to Camoufox

**Resources**: Academic papers, CreepJS source, TOR design doc

### Path 2: Firefox Core Developer

**Goal**: Contribute to Firefox/Camoufox codebase

1. **Month 1**: Build Firefox from source, explore codebase
2. **Month 2**: Study Firefox architecture and XPCOM
3. **Month 3**: Make small patches, learn review process
4. **Month 4**: Work on medium-sized features
5. **Month 5**: Mentor other contributors

**Resources**: Firefox Source Docs, Searchfox, Mozilla Discourse

### Path 3: Automation & Anti-Detection

**Goal**: Build undetectable automation systems

1. **Month 1**: Master Playwright/Selenium
2. **Month 2**: Study bot detection techniques
3. **Month 3**: Learn Camoufox Python library
4. **Month 4**: Build production automation with Camoufox
5. **Month 5**: Create reusable automation frameworks

**Resources**: Playwright docs, FingerprintJS source, Camoufox Python lib

---

## ❓ Frequently Consulted Resources

### When Writing C++ Patches

1. **Searchfox** - Find where properties are implemented
2. **Firefox C++ Style Guide** - Naming and formatting
3. **XPCOM Guide** - Component architecture
4. **MaskConfig.hpp** - Configuration API reference

### When Testing Anti-Fingerprinting

1. **CreepJS** - Comprehensive fingerprint test
2. **BrowserLeaks** - Specialized leak tests
3. **BrowserScan** - Overall fingerprint score
4. **Rebrowser** - Bot detection validation

### When Debugging Builds

1. **Firefox Build Docs** - Troubleshooting build errors
2. **Mozilla Discourse** - Ask for help
3. **Stack Overflow** - Search for similar errors
4. **Searchfox** - Find implementation details

### When Contributing

1. **GitHub Issues** - Find issues to work on
2. **CONTRIBUTING.md** - Contribution guidelines
3. **Code Review Guidelines** - Review standards
4. **Test Framework Docs** - Writing tests

---

## 🌟 Staying Updated

### News & Updates

1. **Follow [@daijro](https://github.com/daijro)** - Camoufox author
2. **Watch [Camoufox Repository](https://github.com/daijro/camoufox)** - Get notifications
3. **Subscribe to [r/privacy](https://www.reddit.com/r/privacy/)** - Privacy news
4. **Follow [Mozilla Blog](https://blog.mozilla.org/)** - Firefox updates

### Academic Research

1. **[arXiv CS.CR](https://arxiv.org/list/cs.CR/recent)** - Security papers
2. **[NDSS Symposium](https://www.ndss-symposium.org/)** - Security conference
3. **[USENIX Security](https://www.usenix.org/conferences)** - Systems security
4. **[IEEE S&P](https://www.ieee-security.org/)** - Security & privacy

### Browser Updates

1. **[Firefox Release Notes](https://www.mozilla.org/en-US/firefox/releases/)** - New Firefox versions
2. **[Chrome Release Blog](https://chromereleases.googleblog.com/)** - Chrome updates
3. **[WebKit Blog](https://webkit.org/blog/)** - Safari engine updates

---

## 📞 Getting Help

### When Stuck

1. **Search existing issues** - Problem might be solved already
2. **Check documentation** - Answer might be in the docs
3. **Ask in Discussions** - Community can help
4. **Read source code** - Ultimate source of truth
5. **Use debugger** - Step through code execution

### Best Places to Ask

- **Technical questions**: GitHub Discussions
- **Build issues**: Mozilla Discourse, Stack Overflow
- **Feature requests**: GitHub Issues
- **General chat**: Matrix/Discord channels

---

**Previous**: [10-community-contributions.md](./10-community-contributions.md) ← Community contributions

**Back to**: [README.md](./README.md) ← Documentation index
