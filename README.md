# QR Code Reader Prototype

A technical verification prototype for QR code reading via camera.
Built to validate integration with a CakePHP 3.7 + jQuery 3.3.1 environment.

## Live Demo

Try it out: **https://takusaotome.github.io/qr-reader-prototype/**

> Open the URL on an iPhone/iPad to test with the rear camera. HTTPS is provided by GitHub Pages, so camera access works without additional setup.

## Tech Stack

| Item | Version | Notes |
|------|---------|-------|
| html5-qrcode | 2.3.8 | QR scanning library (CDN) |
| jQuery | 3.3.1 | Same version as production (CDN) |
| Target Browser | iOS Safari | Rear camera only |

## Local Setup

### PC (Browser)

```bash
cd experiment/qr-reader-prototype
python3 -m http.server 8000
```

Open `http://localhost:8000` in your browser.

> The built-in webcam will be used. You can display a QR code image on a phone to test scanning.

### Device Testing (iPhone / iPad) — ngrok

iOS Safari requires HTTPS for camera access, so use ngrok.

#### 1. Install ngrok (if not installed)

```bash
brew install ngrok
ngrok config add-authtoken <YOUR_TOKEN>
```

> Get your token at https://dashboard.ngrok.com.

#### 2. Start the local server

```bash
cd experiment/qr-reader-prototype
python3 -m http.server 8000
```

#### 3. Create an HTTPS tunnel with ngrok

In a separate terminal:

```bash
ngrok http 8000
```

Open the displayed `https://xxxx.ngrok-free.app` URL in Safari on your iPhone.

#### Notes

- The free ngrok plan has connection and bandwidth limits (sufficient for testing)
- The URL changes each time; sending the URL via QR code to your device is convenient

## Camera Initialization

The rear camera is acquired with a 2-step fallback:

1. `facingMode: { exact: "environment" }` — strictly request rear camera
2. On failure → `facingMode: { ideal: "environment" }` — retry with rear camera preferred

No camera selection UI is provided (rear camera is the only option in production use).

## Result Text Field

- Auto-populated when a QR code is successfully scanned
- Manual input is also supported (as a fallback when QR scanning fails)
- Green highlight provides visual feedback on successful scan

## QrScannerModule API

### Callbacks

```javascript
// On successful scan (primary path)
QrScannerModule.onScanSuccess = function(decodedText) {
  console.log('Scanned value:', decodedText);
};

// On error
// type: 'permission_denied' | 'camera_not_found' | 'unknown'
QrScannerModule.onError = function(errorMessage, type) {
  console.error(type, errorMessage);
};

// On state change
// state: 'idle' | 'starting' | 'scanning' | 'stopping' | 'error'
QrScannerModule.onStateChange = function(state) {
  console.log('State:', state);
};
```

### Methods

```javascript
QrScannerModule.start('qr-reader');  // Start scanning (only when idle)
QrScannerModule.stop();              // Stop scanning (only when scanning)
QrScannerModule.isActive();          // true if scanning or starting
QrScannerModule.destroy();           // Full cleanup
```

### Configuration

```javascript
QrScannerModule.config.enableEnterFallback = true;  // Enable Enter key event firing
```

### State Diagram

```
idle → starting → scanning → stopping → idle
  ↓                  ↓          ↓
error ←─────────── error ←─── error
```

- Invalid transitions are ignored (e.g., calling start() while scanning does nothing)
- `onStateChange` fires on every transition

## Verification Checklist

| # | Item | How to Verify |
|---|------|---------------|
| 1 | Camera startup | Tap "Start Scan" → preview displays + status shows "Scanning" |
| 2 | QR reading | Hold QR code → value in text field + green highlight + console log |
| 3 | Manual input | Directly type into the text field |
| 4 | Stop scanning | Tap "Stop Scan" → camera stops + status shows "Idle" |
| 5 | Rapid tap test | Start → immediately stop → immediately start → no errors |
| 6 | Duplicate detection prevention | Keep same QR in view → `onScanSuccess` fires only once |
| 7 | Camera permission denied | Deny permission → error message displayed |
| 8 | Tab switch | Go to background → camera released |
| 9 | Page navigation | Navigate away → camera released |

## Production Integration Notes

### Library Placement

```
webroot/js/lib/html5-qrcode.min.js  ← Download from CDN and place locally
```

- Pin version 2.3.8. SRI (Subresource Integrity) recommended
- Do not use CDN references in production

### jQuery

Use the existing production jQuery 3.3.1. No additional loading required.

### Integrating QrScannerModule

1. Extract `QrScannerModule` code into a standalone `.js` file
2. Load it in the CakePHP View template
3. Set page-specific handlers on `onScanSuccess` (e.g., Ajax submission, page navigation)

### Enter Key Fallback

If the existing screen uses Enter key for form submission:

```javascript
QrScannerModule.config.enableEnterFallback = true;
```

This causes a `keydown(Enter)` event to fire in addition to `onScanSuccess` on successful QR scan.

### Scan History

Debug-only feature for the prototype. In production, remove the history section HTML and delete `addHistory()` calls.

### Camera Release

The following event handlers must be configured in production as well:

- `pagehide` — reliable camera release on iOS Safari
- `visibilitychange` — supplementary for background transitions

Do not use `beforeunload` as it is unreliable on iOS Safari.
