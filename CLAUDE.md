# Pantry — project instructions

A barcode-scanning pantry inventory app. Runs on an iPhone, added to the
Home Screen, served from GitHub Pages.

## Hard constraints — do not break these

- **The whole app is one file: `index.html`.** No build step, no bundler,
  no npm, no framework. HTML + CSS + vanilla JS in that single file.
  If you think the project needs a build step, say so and stop — don't add one.
- **No dependencies beyond the two CDN `<script>`/`<link>` tags already
  present** (html5-qrcode, Google Fonts). Don't add more without asking.
- **Target is iOS Safari**, both in-browser and as a Home Screen web app.
  Test your reasoning against WebKit, not Chrome. Notably:
  - The `BarcodeDetector` API does **not** exist on iOS. Barcode decoding
    must stay in the JS/WASM library. Don't "simplify" to BarcodeDetector.
  - Camera requires HTTPS and a user gesture. `<video>` needs `playsinline`.
  - `navigator.vibrate` does nothing on iOS. Feedback is the WebAudio beep.
- **Respect the safe-area insets** (`env(safe-area-inset-*)`) already used
  in the CSS. The app runs fullscreen under the notch and home indicator.

## Storage rules

- All state lives in `localStorage` under the single key `pantry.v1`.
- **Never introduce an un-namespaced key.** This app shares an origin
  (`doubleonick.github.io`) with the owner's other projects, so a generic
  key like `state` or `data` would collide with them. Prefix everything
  with `pantry.`.
- `normalize()` is the load-time guard: everything read from storage or
  imported from a backup goes through it. If you add a field to the data
  model, add it to `normalize()` too, and keep export→import lossless.
- `save()` returns a boolean. A failed write must stay user-visible —
  don't reintroduce a silent `catch {}`.
- Corrupt JSON is parked at `pantry.v1.unreadable` rather than discarded.
  Keep that behaviour.

## Data model

```
{ locations:[{id,name,hue}],
  categories:[string],
  items:[{id,name,brand,category,locationId,qty,barcode,updatedAt}],
  known:{ barcode -> {name,brand,category} },   // local barcode cache
  bannerOff:bool }
```

`known` is the local barcode→name cache, populated whenever the user names
an item by hand. It's what makes repeat scans instant when Open Food Facts
doesn't have the product. Don't clear it on import unless asked.

## Product behaviour worth preserving

- **Location is the entry point.** The user picks a place first; the
  scanner then knows where items go. Don't add a global "scan anything"
  path that bypasses this.
- Green = stocking in, amber = taking out. The colour carries through the
  scan header so the mode is readable at arm's length.
- "Take out" only decrements what's actually in the current location.
  It reports a miss rather than creating a negative or a phantom item.
- Reaching qty 0 removes the item with a 5-second Undo.
- Open Food Facts lookups must stay non-blocking and fail soft — a timeout
  or a miss falls through to "name it by hand", never an error state.

## Testing

There's no test runner. Verify changes by reasoning through the flows and,
where logic is non-trivial (anything touching `normalize()`), by extracting
the function and exercising it under `node`. Say what you checked.

## Deployment

`main` is served directly by GitHub Pages from the repo root. Merging a PR
ships it. There is no service worker, so a hard reload picks up changes.
