# AEM GTM Martech Plugin — Agent Instructions

## Plugin Overview

The `aem-gtm-martech` plugin is a phase-based Google Analytics 4 (GA4) and Google Tag Manager (GTM) integration for AEM Edge Delivery Services. It loads analytics scripts and tag containers at optimal lifecycle points — eager, lazy, and delayed — to minimize impact on Core Web Vitals (LCP, CLS, TBT). The plugin defaults all user consent to `denied` and requires explicit opt-in via a consent callback.

## File Structure Context

After installation, the plugin source files live at `plugins/gtm-martech/` in the project repository. The main entry point is `plugins/gtm-martech/src/index.js`.

**NEVER modify any files inside `plugins/gtm-martech/`.** You will only create or modify project-level files.

## Required Project Files to Create or Modify

| File | Action | Purpose |
|------|--------|---------|
| `scripts/gtm-martech.js` | **CREATE** | Helper script that imports and configures the GtmMartech class |
| `head.html` | **MODIFY** | Add preload/preconnect hints at the end of the file |
| `scripts/scripts.js` | **MODIFY** | Import helper, wire into loadEager/loadLazy/loadDelayed |

## GtmMartech API Reference

### Constructor

```js
import GtmMartech from '../plugins/gtm-martech/src/index.js';
const martech = new GtmMartech(config);
```

### Config Object Shape

```js
{
  analytics: Boolean,          // Enable analytics (default: true)
  tags: String[],              // GA4 Measurement IDs
  containers: {
    lazy: String[],            // GTM Container IDs loaded during lazy phase
    delayed: String[],         // GTM Container IDs loaded during delayed phase
  },
  pageMetadata: Object,        // Key-value pairs passed to GA4 tag initializer
  consent: Boolean,            // Enable consent (default: true)
  consentCallback: Function,   // Returns Promise resolving to consent config object
  decorateCallback: Function,  // Called on each found/loaded Section or Block element
}
```

### Phase Methods

- `martech.eager()` — async. Call in `loadEager`.
- `martech.lazy()` — async. Call in `loadLazy`. **Must NOT be inside `Promise.all()`.**
- `martech.delayed()` — sync. Call via `window.setTimeout` in `loadDelayed`.

### Other Methods

- `martech.pushToDataLayer(payload)` — pushes an object to the GTM data layer.
- `martech.updateUserConsent(consent)` — updates Google consent state.

### Global

- `window.gtag()` — available after GtmMartech is instantiated.

## Instrumentation Steps

### Step 1: Create `scripts/gtm-martech.js`

Create the file `scripts/gtm-martech.js` with the GtmMartech import, configuration object with placeholder values, and a default export.

```js
// eslint-disable-next-line import/no-relative-packages
import GtmMartech from '../plugins/gtm-martech/src/index.js';

// For DA Preview support.
const disabled = window.location.search.includes('martech=off');

const martech = new GtmMartech({
  analytics: !disabled,
  tags: [/* TODO: Add one or more GA4 Measurement IDs (e.g. 'G-XXXXXXXXXX') */],
  containers: {
    lazy: [/* TODO: Add GTM Container IDs to load during lazy phase (e.g. 'GTM-XXXXXXX') */],
    delayed: [/* TODO: Add GTM Container IDs to load during delayed phase */],
  },
  pageMetadata: {},
  consent: !disabled,
  consentCallback: undefined, // TODO: Implement consent callback function if consent management is needed
  decorateCallback: undefined, // TODO: Implement decorateCallback for section/block event tracking
});

export default martech;
```

### Step 2: Add Preload Hints to `head.html`

Append the following three lines at the **end** of `head.html`:

```html
<script nonce="aem" src="/scripts/gtm-martech.js" type="module"></script>
<link rel="preload" as="script" crossorigin="anonymous" href="/plugins/gtm-martech/src/index.js"/>
<link rel="preconnect" href="https://www.googletagmanager.com"/>
```

### Step 3: Import the Plugin in `scripts/scripts.js`

Add this import at the **top** of `scripts/scripts.js` (with the other imports):

```js
import gtmMartech from './gtm-martech.js';
```

### Step 4: Call Eager Phase in `loadEager()`

In the `loadEager` function, wrap `gtmMartech.eager()` in `Promise.all` alongside the existing `loadSection()` call:

```js
async function loadEager(doc) {
  // ... existing code ...
  if (main) {
    decorateMain(main);
    doc.body.classList.add('appear');
    await Promise.all([
      gtmMartech.eager(),
      loadSection(main.querySelector('.section'), waitForFirstImage),
    ]);
  }
  // ... existing code ...
}
```

The `eager()` call is asynchronous and should be awaited. Adding it to the existing `Promise.all` with `loadSection` ensures both run concurrently.

### Step 5: Call Lazy Phase in `loadLazy()`

In the `loadLazy` function, add `await gtmMartech.lazy()` **after** `loadSections()` completes. **Do NOT place it inside a `Promise.all()`** — this is critical for correct decoration handling.

```js
async function loadLazy(doc) {
  // ... existing code ...
  await loadSections(main);
  await gtmMartech.lazy();
  // ... existing code ...
}
```

### Step 6: Call Delayed Phase in `loadDelayed()`

In the `loadDelayed` function, add `window.setTimeout(gtmMartech.delayed, 1000)` **before** the existing `delayed.js` import timeout:

```js
function loadDelayed() {
  // ... existing code ...
  window.setTimeout(gtmMartech.delayed, 1000);
  window.setTimeout(() => import('./delayed.js'), 3000);
  // ... existing code ...
}
```

### Step 7: Configure GA4 Measurement IDs (User Task)

The user must add their GA4 Measurement IDs (e.g. `'G-XXXXXXXXXX'`) to the `tags` array in `scripts/gtm-martech.js`.

### Step 8: Configure GTM Container IDs (User Task)

The user must add their GTM Container IDs (e.g. `'GTM-XXXXXXX'`) to `containers.lazy` and/or `containers.delayed` arrays in `scripts/gtm-martech.js`.

### Step 9: Implement Consent Callback (User Task)

If consent management is needed, the user must implement a `consentCallback` function and assign it in the config. The function must return a Promise that resolves to a consent config object. See the Consent Types Reference section below.

### Step 10: Implement Decorate Callback (User Task)

If section/block event tracking is desired, the user must implement a `decorateCallback` function to add DataLayer event pushes on sections and blocks.

```js
function decorateEvents(el) {
  if (el.classList.contains('block')) {
    // Check type of block and add DataLayer pushes as desired.
  } else if (el.classList.contains('section')) {
    // Do something on each section to push to DataLayer
  }
}
```

### Step 11: Review and Merge (User Task)

The user must review the changes on the `plugin/gtm-martech` branch and merge it into their main branch.

## Important Constraints

- All code must use ES module syntax (`import`/`export`).
- The plugin import path is always `../plugins/gtm-martech/src/index.js` (relative from `scripts/`).
- The `martech.lazy()` call must **NEVER** be inside a `Promise.all()` — this is critical for correct decoration via `decorateCallback`.
- The `nonce="aem"` attribute is required on script tags.
- Use `window.setTimeout` (not bare `setTimeout`) for the delayed phase.
- Default consent to `denied` — the legal disclaimer requires explicit opt-in.
- Include `const disabled = window.location.search.includes('martech=off');` in `scripts/gtm-martech.js` for DA Preview support.
- Never modify files inside `plugins/gtm-martech/`.

## Consent Types Reference

The `consentCallback` must return a Promise resolving to an object with these Google consent type keys, each set to `'granted'` or `'denied'`:

```js
{
  ad_storage: 'granted' | 'denied',
  ad_user_data: 'granted' | 'denied',
  ad_personalization: 'granted' | 'denied',
  analytics_storage: 'granted' | 'denied',
  functionality_storage: 'granted' | 'denied',
  personalization_storage: 'granted' | 'denied',
  security_storage: 'granted' | 'denied',
}
```

Example consent callback implementation:

```js
async function checkConsent() {
  return new Promise((resolve) => {
    // Perform the Consent popup check here.
    // Not using a CMP, therefore we must resolve to the desired Consent State.
    resolve({
      ad_storage: 'denied',
      ad_user_data: 'denied',
      ad_personalization: 'denied',
      analytics_storage: 'granted',
      functionality_storage: 'granted',
      personalization_storage: 'denied',
      security_storage: 'granted',
    });
  });
}
```
