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

* **Permission Requirement:** This capability is gated behind the `window-management` permission. If this permission has not been granted by the user, the `alwaysOnTop` request is silently ignored, and a standard popup window is created instead.
* If `true` and the required permission is granted, the OS-level window manager is instructed to keep this window pinned above other non-always-on-top windows.
* If the user manually minimizes the window, it should obey.
* The attribute could be read-only after creation, or mutable via a future property (e.g., `window.alwaysOnTop = false`).

---

## 4. Feature & Permission Detection

Web developers need a reliable way to detect support for `alwaysOnTop` and verify permission status so they can gracefully fall back to standard windows or inline UI modals on unsupported environments or when permissions are denied.

1. **Feature Detection:** Developers can verify synchronous support for the `alwaysOnTop` attribute by checking for its existence on the `Window.prototype` without opening a window.
2. **Permission Verification:** Because the feature requires user consent, developers can asynchronously query the `window-management` permission state using the Permissions API.

```javascript
async function openUtilityWindow() {
  // 1. Feature Detection
  if (!('alwaysOnTop' in Window.prototype)) {
    // Fallback: browser lacks support for the alwaysOnTop hint
    window.open('/tool', 'Tool', 'width=300,height=200');
    return;
  }

  // 2. Permission Detection
  try {
    const permissionStatus = await navigator.permissions.query({ name: 'window-management' });
    if (permissionStatus.state === 'granted') {
      // Permission granted: open as an always-on-top window
      window.open('/tool', 'Tool', 'width=300,height=200,alwaysOnTop=true');
    } else {
      // Fallback: permission denied (or prompt needed)
      window.open('/tool', 'Tool', 'width=300,height=200');
    }
  } catch (e) {
    // Graceful fallback for browsers without Permissions API support for this permission
    window.open('/tool', 'Tool', 'width=300,height=200');
  }
}
```

---

## 5. Security & Privacy Considerations

Because an "always on top" window can easily be abused for malicious purposes (e.g., spoofing system UI, un-dismissible ads, phishing), strict mitigations are required.

* **Transient User Activation:** `window.open(..., 'alwaysOnTop=true')` must require a "sticky" or transient user gesture (like a click or keypress). It cannot be triggered automatically on page load.
* **Permission Prompt & Integration:** Access to this capability relies on the established `window-management` permission API. Browsers should also display a clear, un-spoofable UI indicator (such as an icon or banner in the window frame) ensuring the user is always aware that the window is pinned to the top of the display.
* **Focus Restrictions:** To prevent "focus stealing" loops, an `alwaysOnTop` window should not be able to aggressively force focus back to itself if the user clicks away.
* **Easy Dismissal:** The user must always have an explicit, browser-controlled way to unpin or close the window (e.g., a standard window close button or an "unpin" toggle in the browser chrome).

---

## 6. Alternatives Considered

* **Document Picture-in-Picture API:** While similar, PiP windows have strict limitations regarding size, navigation, and capabilities. `alwaysOnTop` offers a true standard window experience.
* **Native Apps (Electron/NW.js):** Currently, developers are forced to wrap web apps in Electron just to get access to `win.setAlwaysOnTop(true)`. Bringing this to the web bridges a massive capability gap.