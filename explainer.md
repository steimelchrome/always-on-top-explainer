# Explainer: `alwaysOnTop` option for `window.open()`

## 1. Introduction & Abstract

Currently, web applications cannot create standalone, independent windows that stay above other desktop applications. While the `Document Picture-in-Picture API` does allow for always-on-top windows, they are strictly tied to the lifecycle of the initiating tab and close automatically if that tab navigates or closes. This explainer proposes a new boolean option for `window.open()`, called `alwaysOnTop`. When set to `true`, the browser requests that the operating system keep the newly created window pinned above other non-always-on-top windows, enabling fully standalone, always-on-top utility windows that can outlive the tab that opened them.

---

## 2. Use Cases

There are several productivity and utility use cases for this feature:

* **Video Conferencing Controls:** A mini-controller window for a meeting (mute, unmute, end call) that stays visible while the user takes notes in another native app.
* **Developer Tools / Dashboards:** Live-reloading logs or performance monitors that developers want to keep an eye on while coding in their IDE.
* **Persistent Note-Taking / Utilities:** A standalone notepad or scratchpad that remains visible while working in other applications.

---

## 3. Proposed API

We propose adding `alwaysOnTop` to the `windowFeatures` parameter of `window.open()`, as well as introducing a corresponding read-only boolean `alwaysOnTop` property directly onto the `Window` interface.

```javascript
// Proposed syntax using the traditional comma-separated string
const utilityWindow = window.open(
  'https://example.com/controls', 
  'Controls', 
  'width=300,height=200,alwaysOnTop=true'
);

// Read-only boolean property reflection
console.log(utilityWindow.alwaysOnTop); // true (if successfully opened as always-on-top)
```

### Expected Behavior

* **Permission Requirement:** This capability is gated behind the `window-management` permission. If this permission has not been granted by the user, the `alwaysOnTop` request is silently ignored, and a standard popup window is created instead.
* If `true` and the required permission is granted, the OS-level window manager is instructed to keep this window pinned above other non-always-on-top windows.
* **UA-Defined Limits:** To preserve usability, browsers (User Agents) may enforce implementation-specific limits. For example, UAs may restrict origins to a single active `alwaysOnTop` window at a time, or make it mutually exclusive with other types of Picture-in-Picture (PiP) windows.
* **Popup Window Context Requirement:** The `alwaysOnTop` hint only applies when `window.open()` creates a new, standalone popup window context. If the call results in a new tab within an existing multi-tab browser window, or targets and navigates an existing window (e.g., via a named `target`), the `alwaysOnTop` request is silently ignored.
* If the user manually minimizes the window, it should obey.
* The `window.alwaysOnTop` property is read-only after creation and reflects whether the window is currently pinned above other windows (returning `true`) or running in standard/fallback mode (returning `false`).

---

## 4. Feature & Permission Detection

Exposing the read-only `alwaysOnTop` property on the `Window` interface addresses two critical detection challenges for developers:

1. **Feature Detection:** Developers can synchronously check for browser API support by verifying the existence of `'alwaysOnTop' in Window.prototype`.
2. **State Verification:** A script running inside the new popup (or the opening application holding a reference to it) can confidently determine if the window was legitimately successfully opened with the always-on-top capability by checking the `window.alwaysOnTop` boolean.

```javascript
async function openUtilityWindow() {
  // 1. Synchronous Feature Detection
  if (!('alwaysOnTop' in Window.prototype)) {
    // Fallback: browser does not support the alwaysOnTop API
    window.open('/tool', 'Tool', 'width=300,height=200');
    return;
  }

  // 2. Permission Detection
  try {
    const permissionStatus = await navigator.permissions.query({ name: 'window-management' });
    if (permissionStatus.state === 'granted') {
      // Permission granted: open as an always-on-top window
      const win = window.open('/tool', 'Tool', 'width=300,height=200,alwaysOnTop=true');
      console.log("Is always-on-top:", win.alwaysOnTop);
    } else {
      // Fallback behavior: standard window
      const win = window.open('/tool', 'Tool', 'width=300,height=200');
      console.log("Is always-on-top:", win.alwaysOnTop); // false
    }
  } catch (e) {
    // Fallback for UAs without the Permissions API for this permission
    window.open('/tool', 'Tool', 'width=300,height=200');
  }
}
```

---

## 5. Security & Privacy Considerations

Because an "always on top" window can easily be abused for malicious purposes (e.g., spoofing system UI, un-dismissible ads, phishing), strict mitigations are required.

* **Transient User Activation & Popup Blocker Integration:** `window.open(..., 'alwaysOnTop=true')` must require a user gesture (like a click or keypress), inheriting standard enforcement for "Popups and Redirects". Gating the behavior on the `window-management` permission empowers User Agents to seamlessly opt out of support, while allowing developers to reliably detect permission gating.
* **Move & Resize Restrictions:** To prevent abuse scenarios like a window that programmatically "follows the mouse" across the screen, synchronous APIs that manipulate window position and bounds (e.g., `moveTo`, `moveBy`, `resizeTo`, `resizeBy`) must be restricted. Similar to the existing restrictions on Document PiP windows, these calls should require a transient user gesture or be blocked by the UA.
* **Focus Restrictions:** To prevent "focus stealing" loops, an `alwaysOnTop` window should not be able to aggressively force focus back to itself if the user clicks away.
* **Easy Dismissal:** The user must always have an explicit, browser-controlled way to unpin or close the window (e.g., a standard window close button or an "unpin" toggle in the browser chrome).

---

## 6. Alternatives Considered

* **Document Picture-in-Picture API:** While similar, PiP windows have strict limitations regarding size, navigation, and capabilities. `alwaysOnTop` offers a true standard window experience.
* **Native Apps (Electron/NW.js):** Currently, developers are forced to wrap web apps in Electron just to get access to `win.setAlwaysOnTop(true)`. Bringing this to the web bridges a capability gap for developers.