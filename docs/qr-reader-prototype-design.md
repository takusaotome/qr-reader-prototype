# QR Code Reader Prototype — Design Document

| Item | Details |
|------|---------|
| Document ID | QR-PROTO-DESIGN-001 |
| Scope | `experiment/qr-reader-prototype/` |
| Created | 2026-02-27 |
| Author | Development Team |
| Document Type | Implementation specification (normative reference for future production integration and maintenance) |

### Revision History

| Rev | Date | Changes |
|-----|------|---------|
| 1.0 | 2026-02-27 | Initial version |
| 1.1 | 2026-02-27 | Code review: try/finally, _startGeneration, destroy() instance capture |
| 1.2 | 2026-02-27 | Design review: async race condition spec, destroy() state machine rules, UI Controller layer separation, HTTPS prerequisite |
| 1.3 | 2026-02-27 | Re-review: stop() failure recovery path in flow, destroy() sync exception note, removed unused error→idle transition |

---

## 1. Overview

Prior to developing the QR code reading feature, a standalone prototype is built to validate camera-based QR code scanning.
The prototype uses the same tech stack as the production project (CakePHP 3.7 + jQuery 3.3.1, targeting iOS Safari) to facilitate future integration.

## 2. Technology Selection and Prerequisites

| Item | Selection | Version | Notes |
|------|-----------|---------|-------|
| QR scanning library | html5-qrcode | 2.3.8 | iOS Safari compatible, uses `Html5Qrcode` low-level API |
| jQuery | jQuery | 3.3.1 | Same version as production |
| SRI (html5-qrcode) | — | — | `sha256-ZgsSQ3sddH4+aLi+BoXAjLcoFAEQrSE/FnsUtm+LHY4=` |
| SRI (jQuery) | — | — | `sha256-FgpCb/KJQlLNfOu91ta32o/NMZxltwRo8QtmkMRdAu8=` |

- **Prototype**: CDN delivery (pinned version + SRI)
- **Production**: Local placement at `webroot/js/lib/html5-qrcode.min.js`

### HTTPS Requirement (Mandatory)

iOS Safari requires **HTTPS** for camera access (`getUserMedia`), except on localhost.
The production server must have an SSL certificate configured.
For prototype device testing, use ngrok to create an HTTPS tunnel.

## 3. File Structure

```text
experiment/qr-reader-prototype/
├── index.html                            # Prototype (single-file HTML + CSS + JS)
├── README.md                             # Usage, ngrok device testing, production integration notes
└── docs/
    └── qr-reader-prototype-design.md     # This design document
```

## 4. Camera Initialization

The rear camera is acquired with a 2-step fallback. No camera selection UI is provided (rear camera is the only option in production use).

```text
1. facingMode: { exact: "environment" }    ← Strict rear camera request
     ↓ on failure
   Recreate instance (Html5Qrcode.clear() → new Html5Qrcode())
     ↓
2. facingMode: { ideal: "environment" }    ← Retry with rear camera preferred
     ↓ on failure
   Fire onError(message, type) → transition to error state
```

### Why Instance Recreation is Needed

The html5-qrcode library may leave its internal state in a transitioning state after a `start()` failure.
Calling `start()` again on the same instance causes a "Cannot transition to a new state, already under transition" error.
Therefore, before fallback, the instance is recreated via `clear()` → `new Html5Qrcode()`.

## 5. QrScannerModule Design

### 5.1 State Machine

```text
idle ──→ starting ──→ scanning ──→ stopping ──→ idle
  ↑          │            │            │
  │          ↓            ↓            ↓
  └─────── error ←────── error ←───── error

  * destroy() forces transition to idle from any state (see "destroy() Special Rules" below)
```

#### Normal Transition Table

| Current State | Allowed Transitions | Trigger |
|---------------|---------------------|---------|
| `idle` | `starting` | `start()` called |
| `starting` | `scanning` | Camera started successfully |
| `starting` | `error` | Camera start failed |
| `scanning` | `stopping` | `stop()` called / auto-stop after QR detection |
| `scanning` | `error` | Unexpected error |
| `stopping` | `idle` | Camera stop completed |
| `stopping` | `error` | Camera stop failed |
| `error` | `starting` | `start()` called (retry from error) |

- All normal transitions go through `_setState()`, which fires `onStateChange(newState)`
- Disallowed transitions are rejected by `_setState()` with a `console.warn`

#### destroy() Special Rules

`destroy()` is an emergency cleanup for page unload / background transitions and is treated as a **forced reset operation independent of the normal state machine**.

- **Forces transition to `idle` from any state** (does not go through `_setState()`)
- Directly assigns `_state = 'idle'`, then explicitly fires `onStateChange('idle')`
- Reason for bypassing `_setState()`: `destroy()` must be callable from states like `stopping` or `starting`, where the normal transition table does not allow a direct transition to `idle`

### 5.2 Async Race Condition Prevention

#### 5.2.1 _startGeneration Counter (Promise Invalidation)

A counter incremented on each `start()` call. Also incremented by `destroy()`.
Used in async `.then()` / `.catch()` callbacks to check `isStale()`,
preventing stale promise results from overwriting state after `destroy()`.

```javascript
var generation = ++this._startGeneration;
function isStale() { return generation !== self._startGeneration; }

// Inside promise callback
.then(function() {
  if (isStale()) return;  // skip if destroy() has been called
  self._setState('scanning');
})
```

#### 5.2.2 destroy() Instance Capture (Design Requirement)

The `stop().then(…clear…)` chain inside `destroy()` runs asynchronously.
If `start()` is called during this window, `this._html5QrCode` gets replaced with a new instance,
and the stale async callback calling `this._html5QrCode.clear()` would **incorrectly destroy the new instance**.

**Rule**: In `destroy()`, capture the target instance in a local variable before starting async operations.
Callbacks must operate on the old instance only through the local variable.

```javascript
destroy: function() {
  this._startGeneration++;

  // [REQUIRED] Capture current instance in a local variable
  var instance = this._html5QrCode;
  if (instance) {
    if (this._state === 'scanning' || this._state === 'starting') {
      instance.stop().then(function() {
        instance.clear();     // ← use 'instance', not 'this._html5QrCode'
      }).catch(function() {
        instance.clear();
      });
    } else {
      instance.clear();
    }
  }
  this._state = 'idle';
  this.onStateChange('idle');
  this._html5QrCode = null;
  this._hasHandledScan = false;
}
```

> **Implementation note**: In the `starting` state, the library may not support `stop()`, and `stop()` may throw a synchronous exception. The implementation wraps the `instance.stop()` call itself in a `try/catch` to ensure `instance.clear()` executes even on synchronous exceptions. Each `clear()` call is also individually wrapped in `try/catch`.

#### 5.2.3 _hasHandledScan Lock (Duplicate Detection Prevention)

After a successful scan, subsequent QR detection notifications are ignored until `stop()` completes.

- Return immediately at the top of `_handleScanResult()` if `_hasHandledScan === true`
- Return immediately at the top of `_handleScanResult()` if `_state !== 'scanning'`
- Reset to `false` when `stop()` completes (on transition to `idle`)

### 5.3 Callback Exception Safety

In `_handleScanResult()`, `onScanSuccess()` is wrapped in `try/finally`.
Even if the external callback throws an exception, `stop()` is guaranteed to execute in the `finally` block.

```javascript
_handleScanResult: function(decodedText) {
  if (this._hasHandledScan) return;
  if (this._state !== 'scanning') return;

  this._hasHandledScan = true;
  try {
    this.onScanSuccess(decodedText);
    // enableEnterFallback processing
  } catch (e) {
    console.error('[QrScanner] onScanSuccess threw:', e);
  } finally {
    this.stop();  // always executed even on exception
  }
}
```

**Recovery flow on exception**:
1. `onScanSuccess()` throws → `catch` logs the error
2. `finally` executes `stop()` → `scanning` → `stopping`
3. Promise inside `stop()` resolves → `_hasHandledScan = false` reset → `stopping` → `idle`
4. Returns to `idle` state → ready for rescan

### 5.4 Public API

```javascript
var QrScannerModule = {
  // --- Configuration ---
  config: {
    enableEnterFallback: false,  // fires Enter key event only when true
    qrboxSize: 250,              // QR scanning frame size (px)
    fps: 10                      // scan frame rate
  },

  // --- Callbacks (replaceable from outside) ---
  onScanSuccess: function(decodedText) {},   // on successful scan (primary path)
  onError: function(errorMessage, type) {},  // error (permission_denied / camera_not_found / unknown)
  onStateChange: function(state) {},         // idle / starting / scanning / stopping / error

  // --- Methods ---
  start: function(targetElementId) {},  // accepted only in idle / error
  stop: function() {},                  // accepted only in scanning
  isActive: function() {},              // true if scanning or starting
  destroy: function() {}                // forced cleanup from any state (see 5.1 Special Rules)
};
```

### 5.5 Camera Release

| Event | Purpose | Notes |
|-------|---------|-------|
| `pagehide` | Primary camera release | Reliable on iOS Safari (required) |
| `visibilitychange` | Background transition detection | Supplementary |

Do not use `beforeunload` as it fires unreliably on iOS Safari.

## 6. Screen Layout

### 6.1 Screen Elements

| # | Element | Description |
|---|---------|-------------|
| 1 | Status bar | Color changes linked to state machine (5 colors) + text + pulse animation |
| 2 | Camera preview area | `<video>` stream rendered by `html5-qrcode` (`#qr-reader`) |
| 3 | Error message | Red background, fades out after 5 seconds |
| 4 | Start/Stop scan buttons | `disabled` controlled by state (`starting` / `stopping` disables both) |
| 5 | Scan result text field | Auto-populated on QR scan + manual input allowed (no `readonly`) |
| 6 | Scan history | Debug only. Max 20 items, timestamped, "Clear" button. Remove in production |

### 6.2 UI Controller Layer

QrScannerModule is designed as a pure scan engine with no UI dependencies.
The UI Controller layer is bundled within the prototype's `index.html` and connected to the module only through the following callbacks.

```text
QrScannerModule                      UI Controller
─────────────────                    ──────────────
onStateChange(state)  ──────────→  updateStatusUI(state) + updateButtonState(state)
onScanSuccess(text)   ──────────→  setResult(text) + addHistory(text)
onError(msg, type)    ──────────→  showError(userMessage)
```

#### UI Controller Functions

| Function | Responsibility |
|----------|----------------|
| `updateStatusUI(state)` | Toggle status bar CSS class + update text |
| `updateButtonState(state)` | Control `disabled` state of start/stop buttons |
| `setResult(text)` | Set value in text field + `trigger('input')` + green highlight (2s) |
| `addHistory(text)` | Add entry to history list (max 20, FIFO) |
| `showError(message)` | Display error message (fades out after 5s) |

For production integration, replace the UI Controller layer to match the CakePHP View template.
Bind production-specific handlers (Ajax submission, page navigation, etc.) to QrScannerModule callbacks.

### 6.3 Status Bar Color Definitions

| State | Background | Text Color | Dot Color | Animation |
|-------|------------|------------|-----------|-----------|
| `idle` | `#e8e8e8` | `#666` | `#999` | None |
| `starting` | `#fff3cd` | `#856404` | `#ffc107` | Pulse |
| `scanning` | `#d4edda` | `#155724` | `#28a745` | Pulse |
| `stopping` | `#fff3cd` | `#856404` | `#ffc107` | None |
| `error` | `#f8d7da` | `#721c24` | `#dc3545` | None |

### 6.4 Button State Control

| State | Start Scan | Stop Scan |
|-------|------------|-----------|
| `idle` | Enabled | Disabled |
| `starting` | Disabled | Disabled |
| `scanning` | Disabled | Enabled |
| `stopping` | Disabled | Disabled |
| `error` | Enabled | Disabled |

## 7. Operational Flows

### 7.1 Normal Flow

1. User taps "Start Scan"
   → QrScannerModule: `idle` → `starting`
   → UI Controller: status "Starting camera..." (yellow), both buttons disabled
2. Camera starts (`exact: "environment"` → fallback to `ideal: "environment"` on failure)
3. Camera started successfully
   → QrScannerModule: `starting` → `scanning`
   → UI Controller: status "Scanning" (green), stop button enabled
4. QR code detected
   → QrScannerModule: `_hasHandledScan = true` → `onScanSuccess(decodedText)` → `stop()`
   → UI Controller: value set in text field + green highlight + history entry added
5. Camera stop completed
   → QrScannerModule: `scanning` → `stopping` → `idle`, `_hasHandledScan = false`
   → UI Controller: status "Idle" (gray), start button enabled

### 7.2 Error Flows

**Camera start failure** (`starting` → `error`):
- Permission denied → `onError(msg, 'permission_denied')` → UI Controller: error message displayed
- Camera not found → `onError(msg, 'camera_not_found')` → UI Controller: error message displayed
- Other start errors → `onError(msg, 'unknown')` → UI Controller: error message displayed

**Camera stop failure** (`stopping` → `error`):
- `Html5Qrcode.stop()` rejects inside `stop()` → `_hasHandledScan = false` reset → transition to `error` state
- UI Controller: status "Error" (red), start button enabled
- User taps "Start Scan" → `error` → `starting` to recover

### 7.3 Page Unload / Background Transition

- `pagehide` / `visibilitychange(hidden)` → `destroy()` → forced `idle` (Section 5.1 Special Rules)

## 8. Test Environment

| Environment | Steps |
|-------------|-------|
| PC testing | `python3 -m http.server 8000` → `http://localhost:8000` |
| Device testing | Above + `ngrok http 8000` → access HTTPS URL from device |

iOS Safari requires HTTPS for camera access (except localhost). See Section 2 prerequisites.

## 9. Verification Items

### 9.1 Basic Functionality

| # | Item | Expected Result |
|---|------|-----------------|
| 1 | Start scan | Camera starts + status "Scanning" (green) |
| 2 | QR reading | Value in text field + green highlight + `onScanSuccess` fired |
| 3 | Manual input | Text field accepts direct input |
| 4 | Stop scan | Camera stops + status "Idle" (gray) |

### 9.2 Edge Cases

| # | Item | Expected Result |
|---|------|-----------------|
| 5 | Rapid tap test | Start → immediately stop → immediately start with no errors |
| 6 | Duplicate detection prevention | Keep same QR in view → `onScanSuccess` fires only once |
| 7 | Camera permission denied | `onError(msg, 'permission_denied')` + error message displayed |
| 8 | Tab switch | `visibilitychange` triggers `destroy()` → status returns to "Idle" |
| 9 | Page navigation | `pagehide` triggers `destroy()` → camera released |

### 9.3 Exceptions and Race Conditions

| # | Item | Expected Result |
|---|------|-----------------|
| 10 | Callback exception | `stop()` executes even when `onScanSuccess` throws |
| 10a | — State recovery after exception | `state` returns to `idle`, `_hasHandledScan` resets to `false` |
| 10b | — Rescan after exception | "Start Scan" becomes enabled, scanning can be restarted |
| 11 | destroy() race condition | `destroy()` during `starting` → stale promise does not overwrite state |
| 12 | Restart after destroy() | `start()` after `destroy()` → new instance starts scanning normally |

## 10. Production Integration Notes

### Library Placement

```text
webroot/js/lib/html5-qrcode.min.js  ← Download from CDN and place locally (pin v2.3.8)
```

- SRI recommended. Do not use CDN references in production
- Use existing production jQuery 3.3.1

### Integrating QrScannerModule

1. Extract `QrScannerModule` code into a standalone `.js` file
2. Load it in the CakePHP View template
3. Set page-specific handlers on `onScanSuccess` (e.g., Ajax submission, page navigation)
4. Implement UI Controller layer individually per screen to match the View

### Enter Key Fallback

If the existing screen uses Enter key for form submission:

```javascript
QrScannerModule.config.enableEnterFallback = true;
```

On successful QR scan, a `keydown(Enter)` event fires in addition to `onScanSuccess`.

### Scan History

Debug-only feature for the prototype. In production, remove the history section HTML and delete `addHistory()` calls.

### Camera Release

Configure `pagehide` + `visibilitychange` event handlers in production as well.
Do not use `beforeunload` as it is unreliable on iOS Safari.
