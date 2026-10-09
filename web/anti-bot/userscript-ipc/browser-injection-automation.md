# Anti-Bot Bypass via Userscript Injection and Local IPC

Tags: anti-bot, bot-bypass, cloudflare, datadome, akamai, tampermonkey, userscript, ipc, websocket, csp-bypass, fingerprint-evasion, react-hooking, event-spoofing, browser-automation, bug-bounty

## Applicability

Test when a target application relies on client-side anti-bot protections (Cloudflare Turnstile, DataDome, Akamai Bot Manager) and traditional automation frameworks (Selenium, Puppeteer, Playwright) are detected via CDP flags, TLS fingerprinting, or WebDriver properties. This architecture uses a real browser with injected userscripts communicating over local IPC, eliminating all framework-level detection signals.

## Architecture

The approach decouples automation logic from the browser instance entirely:

1. User opens their daily-driver browser and authenticates manually (passes CAPTCHAs, behavioral checks, IP reputation)
2. A Tampermonkey userscript injects a payload into the target page's DOM
3. The payload establishes a polling or WebSocket connection to a local controller (e.g. `http://127.0.0.1:8081`)
4. The controller sends JSON commands; the injected JS executes them against the real DOM and returns results

This defeats the primary detection vectors: no CDP/WebDriver flags, genuine TLS and canvas fingerprints, and a legitimately authenticated session.

## Core Techniques

### 1. IPC bridge via GM_xmlhttpRequest

Userscript managers operate in a privileged extension context. `GM_xmlhttpRequest` bypasses the page's CSP `connect-src` directives entirely — the browser allows the extension to make cross-origin requests to `127.0.0.1` without triggering security violations.

```javascript
// Tampermonkey header (key grants)
// @grant        GM_xmlhttpRequest
// @run-at       document-start

function tmFetch(url, options = {}) {
    return new Promise((resolve, reject) => {
        GM_xmlhttpRequest({
            method: options.method || 'GET',
            url: url,
            data: options.body,
            headers: { 'Content-Type': 'application/json' },
            onload: (res) => resolve(JSON.parse(res.responseText)),
            onerror: reject
        });
    });
}
```

Standard `fetch()` or `XMLHttpRequest` would be blocked by CSP; the extension context sidesteps this completely.

### 2. CSP Trusted Types bypass for dynamic code execution

When a site enforces Trusted Types (blocking `eval()` and inline scripts), create a custom policy to wrap the payload:

```javascript
let secureCode = "console.log('Injected payload running');";
if (window.trustedTypes && window.trustedTypes.createPolicy) {
    const policy = window.trustedTypes.createPolicy(
        'tm-loader-' + Math.random().toString(36).substring(2, 10),
        { createScript: (s) => s }
    );
    secureCode = policy.createScript(secureCode);
}
eval(secureCode);
```

### 3. Background tab throttling defeat

Browsers throttle `setTimeout`/`setInterval` to 1000ms+ in background tabs. Two workarounds maintain high-speed polling:

**Web Workers via Blob URLs** — workers run in a separate thread, exempt from background throttling.

**Web Audio API** — create an `AudioContext` playing an inaudible oscillator (1Hz, gain 0.01). The browser treats the tab as actively playing media and disables throttling.

### 4. Visibility and focus spoofing

SPAs pause rendering or disconnect WebSockets when the tab loses focus. Override the native APIs:

```javascript
Object.defineProperties(Document.prototype, {
    'hidden': { get: () => false, configurable: true },
    'visibilityState': { get: () => 'visible', configurable: true },
    'hasFocus': { value: () => true, configurable: true }
});
```

### 5. Framework memory hooking (React/Angular)

Instead of fragile DOM selectors, read application state directly from framework internals:

- **React**: traverse `__reactFiber$` and `__reactProps$` properties on DOM elements to access the component tree, state, and props
- **Angular**: access `__ngContext__` on DOM nodes for component state

This extracts raw JSON state from memory, bypassing the rendered UI entirely.

### 6. Event trust spoofing via Proxy

Synthetic events dispatched with `element.click()` have `isTrusted: false` (read-only). Bypass by invoking framework event handlers directly with a proxied event:

```javascript
const reactKey = Object.keys(element).find(k => k.startsWith('__reactProps$'));
const props = element[reactKey];

const nativeEvent = new MouseEvent('click', { bubbles: true, cancelable: true });

const fakeNativeEvent = new Proxy(nativeEvent, {
    get(target, prop) {
        if (prop === 'isTrusted') return true;
        const value = Reflect.get(target, prop);
        return typeof value === 'function' ? value.bind(target) : value;
    }
});

props.onClick({
    bubbles: true,
    cancelable: true,
    target: element,
    nativeEvent: fakeNativeEvent
});
```

This calls React's internal handler directly — the application sees a trusted user click.

### 7. Prototype tampering concealment

Sites detect overridden functions by checking `fn.toString()` for `[native code]`. Hook `Function.prototype.toString` to return the native string for any hijacked function:

```javascript
const originalToString = Function.prototype.toString;
Function.prototype.toString = function() {
    // Return native representation for overridden functions
    if (this === Document.prototype.hidden || /* other overrides */) {
        return 'function hidden() { [native code] }';
    }
    return originalToString.call(this);
};
```

## Verify

Confirm the architecture works by validating each layer:

1. Open the target site in a standard browser — verify CAPTCHAs and bot checks pass without intervention
2. Install the userscript and confirm the IPC bridge connects to the local controller
3. Send a command from the controller and verify the injected JS executes it against the live DOM
4. Confirm extracted data (via framework hooking or DOM scraping) arrives at the controller
5. Verify background tab operation — switch tabs and confirm polling continues at full speed
6. Check that `isTrusted` spoofing and visibility overrides survive the site's detection scripts

## Report

Architecture leverages a legitimate browser session with userscript injection and local IPC to bypass anti-bot systems that detect traditional automation frameworks. Six reinforcing evasion layers:

1. **No automation fingerprint** — real browser, no CDP/WebDriver flags, genuine TLS and canvas fingerprints
2. **CSP bypass** — extension-privileged `GM_xmlhttpRequest` ignores `connect-src` restrictions
3. **Framework state extraction** — reads React/Angular internal state directly from memory, bypassing DOM
4. **Event trust forgery** — `Proxy`-wrapped synthetic events spoof `isTrusted` at the framework handler level
5. **Persistence under throttling** — Web Workers and Audio API keep polling active in background tabs
6. **Detection evasion** — `toString` hooking and visibility spoofing defeat prototype and focus checks

Impact: complete bypass of client-side bot detection, enabling automated interaction with an authenticated session indistinguishable from genuine user activity.
