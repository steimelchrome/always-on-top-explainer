# Explainer: `alwaysOnTop` option for `window.open()`

## 1. Introduction & Abstract

Currently, web applications cannot create windows that stay above other desktop applications. This explainer proposes a new boolean option for `window.open()`, called `alwaysOnTop`. When set to `true`, the browser requests that the operating system keep the newly created window pinned above other non-always-on-top windows.

---

## 2. Use Cases

There are several legitimate productivity and utility use cases for this feature:

* **Picture-in-Picture (PiP) for Custom Content:** While the `Document Picture-in-Picture API` exists, it is highly restricted. This option would allow full-featured, standalone utility windows (like a video player, a stock ticker, or a calculator) to remain visible.
* **Video Conferencing Controls:** A mini-controller window for a meeting (mute, unmute, end call) that stays visible while the user takes notes in another native app.
* **Developer Tools / Dashboards:** Live-reloading logs or performance monitors that developers want to keep an eye on while coding in their IDE.
* **Persistent Note-Taking / Utilities:** A standalone notepad or scratchpad that remains visible while working in other applications. Crucially, unlike the `Document Picture-in-Picture API` (where the auxiliary window is strictly tied to the initiating tab and automatically closes if that tab navigates or closes), an `alwaysOnTop` window can remain open independently, outliving its opener.

---

## 3. Proposed API

We propose adding `alwaysOnTop` to the `windowFeatures` parameter of `window.open()`.

```javascript
// Proposed syntax using the traditional comma-separated string
const utilityWindow = window.open(
  'https://example.com/controls', 
  'Controls', 
  'width=300,height=200,alwaysOnTop=true'
);

```

### Expected Behavior

* If `true`, the OS-level window manager is instructed to keep this window on top of others.
* If the user manually minimizes the window, it should obey.
* The attribute could be read-only after creation, or mutable via a future property (e.g., `window.alwaysOnTop = false`).

---

## 4. Feature Detection

Web developers need a reliable, synchronous way to detect support for `alwaysOnTop` so they can gracefully fall back to standard windows or inline UI modals on unsupported browsers.

We propose two complementary mechanisms for feature detection:

### 4.1 Prototype Property Check (Primary)

The most straightforward and standard-aligned method is ensuring the `alwaysOnTop` property exists on the `Window.prototype` (and consequently, on any `window` instance). This allows developers to check for support without actually opening a physical window.

```javascript
if ('alwaysOnTop' in Window.prototype) {
  // Browser supports the always-on-top capability
  window.open('/tool', 'Tool', 'width=300,height=200,alwaysOnTop=true');
} else {
  // Fallback behavior (e.g., opening a standard window or an inline overlay)
  window.open('/tool', 'Tool', 'width=300,height=200');
}

```

### 4.2 Modern `windowFeatures` Object Check (Alternative)

As the web platform moves toward passing a dictionary/object instead of a comma-separated string to `window.open()`, browsers can natively ignore unsupported object keys. By exposing a static `supportedFeatures` list or allowing dictionary validation, developers can check feature compatibility directly:

```javascript
// Checking if the token is recognized in a modern windowFeatures dictionary
if (window.open.supportedFeatures?.includes('alwaysOnTop')) {
  // Safe to request always-on-top behavior
}

```

---

## 5. Security & Privacy Considerations

Because an "always on top" window can easily be abused for malicious purposes (e.g., spoofing system UI, un-dismissible ads, phishing), strict mitigations are required.

* **Transient User Activation:** `window.open(..., 'alwaysOnTop=true')` must require a "sticky" or transient user gesture (like a click or keypress). It cannot be triggered automatically on page load.
* **Permission Prompt / Indicator:** Browsers should display a clear, un-spoofable UI indicator showing that the window is pinned. The browser may also require explicit user permission via a prompt.
* **Focus Restrictions:** To prevent "focus stealing" loops, an `alwaysOnTop` window should not be able to aggressively force focus back to itself if the user clicks away.
* **Easy Dismissal:** The user must always have an explicit, browser-controlled way to unpin or close the window (e.g., a standard window close button or an "unpin" toggle in the browser chrome).

---

## 6. Alternatives Considered

* **Document Picture-in-Picture API:** While similar, PiP windows have strict limitations regarding size, navigation, and capabilities. `alwaysOnTop` offers a true standard window experience.
* **Native Apps (Electron/NW.js):** Currently, developers are forced to wrap web apps in Electron just to get access to `win.setAlwaysOnTop(true)`. Bringing this to the web bridges a massive capability gap.