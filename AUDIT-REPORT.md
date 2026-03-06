# Codebase hygiene & quality audit — single-page marketing site

---

## Functional Contract (hero + form)

**DOM hooks and expected behaviour** — script resolves these at load (primary ID/class first, then fallbacks). Behaviour must match below.

| Role | Primary selector | Fallbacks | Expected behaviour |
|------|------------------|-----------|---------------------|
| Sector buttons container | `#sector-cards` | `.sector-cards-grid`, `.sector-card` | Wraps sector buttons; sector click runs only if objective step + select exist. |
| Sector buttons | `.sector-card` | — | Click: highlight card, set `selectedSector`, show objective step, populate + enable dropdown, scroll/focus to dropdown. |
| Sector instruction | `#sector-instruction` | `.hero-instruction` | "Select your sector"; gets `.visible` on load. |
| Objective step container | `#objective-step` | `.objective-step` | Wraps dropdown + label; gets `.active` when sector selected. |
| Objective instruction | `#objective-instruction` | `.objective-instruction` | "What are you trying to achieve?"; shown when step is active. |
| Objective dropdown | `#objective-select` | `.objective-step select`, `select[name="objective"]` | Populated from OBJECTIVES[selectedSector]; change shows recommendation panel, scroll/focus to panel. |
| Recommendation panel | `#recommendation-panel` | `.recommendation-panel` | Shows title/text from RECOMMENDATION_MAP; gets focus when shown; CTA inside. |
| Recommendation label/title/text | `#recommendation-label`, `#recommendation-title`, `#recommendation-text` | — | Populated on objective change; guarded so missing elements don’t throw. |
| Results area | `#results-area` | `.results-container` | Legacy results container; hidden when panel shows. |
| CTA | `.recommendation-cta` (inside panel) | — | "Enquire now"; visible only when panel has `.visible`. |
| Form | `form[name="contact"]` | — | `method="POST"`, `data-netlify="true"`, `name="contact"`; hidden `form-name` value `contact`; honeypot `bot-field`. Submit: always POST (fetch); UI updates (button/note) only if `#submit-btn` / `#form-note` exist. |

**Netlify form requirements (must stay)**  
- `method="POST"`, `data-netlify="true"`, `name="contact"`.  
- Hidden input `name="form-name"` value `contact`.  
- Honeypot: `netlify-honeypot="bot-field"` and input `name="bot-field"`.  
- All inputs have `name`.  
- Client submit: `fetch('/', { method: 'POST', headers: { 'Content-Type': 'application/x-www-form-urlencoded' }, body: new URLSearchParams(formData).toString() })`.

---

## 1) Findings (by area)

### A) HTML structure & semantics
- **Headings:** One H1 (hero). H2s used in sections (What We Do, About Driftline, Start a Conversation), in popup (`.popup-title`), and in results (`.results-title`). Hierarchy is valid.
- **Landmarks:** `<header>`, `<nav>`, `<main>`, `<footer>` present. Sections have `id`s for anchor links (`#interactive-content`, `#services`, `#about`, `#contact`). Nav links match (`#contact`, `#services`, etc.).
- **Duplicate IDs:** None in HTML. SVG `linearGradient` ids (`waveFade1`–`waveFade4`, `waveFadePopup`) are unique.
- **Invalid nesting:** Hero has a stray `</div>` closure — `sector-cards-grid` closes at 1382 then `objective-step` and `recommendation-panel` sit inside a parent that closes at 1416; structure is slightly ambiguous but valid.
- **Images:** Logo has `alt="Driftline Learning Design"`. No other images in hero.
- **Forms:** All inputs have `name`, `id`, and associated `<label for="...">`. Honeypot and `form-name` present for Netlify.

### B) CSS quality and conflicts
- **Duplicate rules:** `.hero-inner` defined twice (lines 262–265 and 267–275). `.btn` has two `transition` declarations (lines 188 and 230). Duplicate `@keyframes spin` (lines 821–824).
- **!important:** Used in `#services .service-cards .svc` (gap, margin, wave-divider) and `.hidden { display: none !important }`. Acceptable for overrides but could be reduced by increasing specificity without !important where possible.
- **Over-specific selectors:** `#services .service-cards .svc` and `#services .service-cards .svc strong` etc. are high-specificity; consider a single class on the services section cards if refactoring.
- **Repeated values:** Some hardcoded colors (`#4b5563`, `#e5e7eb`, `#f5f7fb`) could be moved to `:root` for consistency.
- **Scroll behaviour:** `html { scroll-behavior: smooth }` is overridden in `@media (prefers-reduced-motion: reduce)` — good.
- **Responsive:** Breakpoints at 980, 768, 767, and reduced-motion. Sector grid uses 1024/768/767. No obvious overflow or layout shift issues; tap targets (buttons, nav) use min-height 44px+.

### C) JavaScript quality and resilience
- **Script placement:** Inline at end of `<body>` — DOM is ready; no `DOMContentLoaded` needed.
- **Null safety:** `handleSubmit` uses `document.getElementById('submit-btn')` and `document.getElementById('form-note')` with no checks — if IDs were removed, would throw. Sector handler uses `recommendationPanel` (defined later in script; OK by execution order) but `objectiveStep`, `objectiveSelect`, etc. are not checked before use. `objectiveSelect.addEventListener('change', ...)` runs unconditionally — if `objectiveSelect` is null, throws. Recommendation panel elements (`recommendationPanel`, `recommendationLabel`, `recommendationTitle`, `recommendationText`) are used in the change handler with no guards.
- **Video modal:** `modal.querySelector('.video-modal-close')` and `.video-modal-backdrop` used without null checks before `addEventListener` and `focus`.
- **Menu toggle:** `menuToggle` and `nav` are guarded (`if (menuToggle && nav)`); good.
- **Duplicate listeners:** Sector cards and objective select each get one listener; no duplicate binding.
- **scrollToGoalDropdown / scrollToRecommendationPanel:** Use `getElementById` with early return; robust.
- **recommendationPanel reference in sector handler:** Defined later in same script; by the time the click runs, binding exists. No bug.

### D) Forms & Netlify integration
- **Form:** `name="contact"`, `method="POST"`, `data-netlify="true"`, `netlify-honeypot="bot-field"`, hidden `form-name` value `contact`. Satisfies Netlify requirements.
- **Submit:** `handleSubmit` uses `fetch('/', { method: 'POST', headers: { 'Content-Type': 'application/x-www-form-urlencoded' }, body: new URLSearchParams(formData).toString() })` — correct for Netlify form detection.
- **Success/error:** Message shown in `#form-note` with classes `form-success` / `form-error`. Button re-enabled and text reset in `finally`.
- **No conflict** with Netlify parsing; no client-side only submission that would bypass form-name.

### E) Accessibility
- **Focus:** Visible focus styles on `.btn`, `select`, `.itab`, `.recommendation-panel`, `.svc-video`. `var(--focus)` used consistently.
- **Keyboard:** No trap; menu closes on link click and outside click. Recommendation panel has `tabindex="-1"` and receives focus when shown.
- **Labels:** Form fields have `<label for="...">`. Objective select has `aria-describedby="objective-description"` and `aria-disabled` toggled.
- **Reduced motion:** `prefers-reduced-motion` media query disables transitions/animations and sets `scroll-behavior: auto`. Scroll helpers use `behavior: prefersReducedMotion ? 'auto' : 'smooth'`.
- **ARIA:** Nav has `aria-expanded` on toggle and nav; `aria-label` on nav and regions. Popup has `aria-live="polite"`, `role="region"`, `aria-label="Recommendation"`.
- **Contrast:** Not measured; grey text on light background may need verification for WCAG AA.

### F) Performance and security
- **Fonts:** Google Fonts with preconnect; `display=swap`. Blocking until load; could add `font-display: swap` in link (already in URL).
- **Images:** Logo SVG; no dimensions in HTML (height in CSS). No lazy-loading needed for single logo.
- **No third-party scripts** other than fonts.
- **innerHTML:** Used for `recommendationText.innerHTML = recommendation.text` (content from RECOMMENDATION_MAP — author-controlled) and `resultsText.innerHTML`, `objectiveSelect.innerHTML` (options from OBJECTIVES). No user input; low XSS risk. Video modal uses template literal with `videoId` — if `openVideoModal` is ever called with user input, must sanitize; currently not used in markup.
- **Secrets:** None in file.

### G) Responsiveness
- **Breakpoints:** 980, 768, 767 used consistently. Sector grid: 1024 (5 col), 768–1023 (3 col), &lt;768 (1 col).
- **Nav:** Hamburger at ≤767px; solid background; focus and tap targets OK.
- **Hero:** Sector grid and scroll/focus behaviour work across viewports.
- **Containers:** `min(var(--max), calc(100% - 40px))`; no horizontal overflow observed.

---

## 2) Prioritized fixes

### P0 (functionality / security / major UX)
- **JS null safety:** Guard `handleSubmit` so it does not throw if `#submit-btn` or `#form-note` is missing. Guard sector card handler so it does not use `objectiveStep` / `objectiveSelect` when null. Guard objective `change` listener so it is only attached when `objectiveSelect`, `recommendationPanel`, `recommendationLabel`, `recommendationTitle`, `recommendationText` exist; add null checks inside handler. Guard video modal so `querySelector` results are checked before `addEventListener` and `focus`.
- **Form submit:** If `submitBtn` or `note` is null, avoid calling methods on them (early return or no-op).

### P1 (quality / accessibility / responsiveness)
- Remove duplicate CSS: second `.hero-inner` block, duplicate `.btn` transition, duplicate `@keyframes spin`.
- Move repeated hex colors to `:root` where it improves maintainability (optional).
- Consider reducing `!important` in `#services .service-cards .svc` by using a dedicated class (optional refactor).
- Add `scope` or comment to form so Netlify form detection is unambiguous (optional; form-name already set).

### P2 (nice-to-have)
- Use `window.scrollY` instead of `window.pageYOffset` (deprecation).
- Consolidate scroll helpers’ `headerOffset` (e.g. single constant).
- Optional: `loading="lazy"` on any future images.

---

## 3) Patch proposal (P0 only — implemented in file)

- **handleSubmit:** After getting `submitBtn` and `note`, if either is null return false and prevent default.
- **Sector cards:** Before `objectiveStep.classList.add('active')` (and use of `objectiveSelect`), check `objectiveStep` and `objectiveSelect`; if either is null, return from click handler (or skip adding listener).
- **Recommendation panel / objective change:** Only attach `objectiveSelect.addEventListener('change', ...)` when `objectiveSelect`, `recommendationPanel`, `recommendationLabel`, `recommendationTitle`, `recommendationText` are all non-null. Inside the handler, add early return if any of these are null before use.
- **Video modal:** After `modal.querySelector('.video-modal-close')` and `.video-modal-backdrop`, only add listeners and call `focus()` if the elements exist.
- **Duplicate @keyframes spin:** Remove one of the two identical `@keyframes spin` blocks.

---

## 4) Verification checklist

- [ ] **Form:** Submit contact form; confirm success message and no console errors. Confirm Netlify receives submission.
- [ ] **Hero:** Select sector → dropdown appears and scroll/focus works. Select goal → recommendation panel appears, scroll and focus on panel. Change sector again → dropdown updates and scroll/focus still work.
- [ ] **Menu:** On viewport &lt;768px, open hamburger, click link, confirm menu closes. Click outside, confirm menu closes.
- [ ] **Console:** No errors on load, sector select, goal select, form submit, menu toggle.
- [ ] **Reduced motion:** Enable OS “reduce motion”; confirm scroll is instant and animations are minimal.
- [ ] **Null guards:** Temporarily comment out `#form-note` in HTML; submit form — should not throw (button may stay “Sending...” if note is null; ideal is full guard so UI degrades safely).

---

## What changed in pass 2

- **Selector helper and fallbacks:** Added `$(selectors)` (single string or array; returns first match). Hero/form elements are resolved with primary ID/class then fallbacks. Behaviour unchanged when markup matches; still works if IDs/classes change to one of the fallbacks.
- **Runtime warnings only:** `warnMissing` runs at load; if an expected element is missing after trying all selectors, `console.warn`. No early return that would disable hero or form.
- **Form submit never blocked:** `handleSubmit` no longer returns when submit-btn/form-note are missing. Submission always runs; UI updates only when elements exist. Error message only on real failure.
- **Objective change listener:** Attached whenever `objectiveSelect` exists. Panel updates guarded; panel show/hide and scroll/focus run when `recommendationPanel` exists.
- **Single source for hero refs:** Recommendation panel refs resolved once at top; duplicate declaration removed. Scroll helpers use these refs.
- **Duplicate CSS removed:** `.hero-inner` merged; redundant `.btn` transition removed.
- **Sector grid and nav:** No layout change. Grid 5/3/1 columns; nav breakpoints unchanged.

---

## Security hardening recommendations (Netlify)

Do not add a strict CSP that blocks inline scripts/styles unless you migrate them. Suggested safe headers: X-Frame-Options: DENY, X-Content-Type-Options: nosniff, Referrer-Policy: strict-origin-when-cross-origin, Permissions-Policy: camera=(), microphone=(), geolocation=(). For CSP, start permissive (script-src 'self' 'unsafe-inline') so current inline script works; tighten after moving JS to a file if desired.

---

## Verification checklist (pass 2 — quick manual steps)

- [ ] **Sector select (mobile and desktop):** Choose a sector → dropdown appears, page smooth-scrolls to it, focus moves to dropdown. Change sector → dropdown updates, scroll/focus runs again.
- [ ] **Goal select:** Choose a goal from dropdown → recommendation panel appears, view scrolls to panel, focus moves to panel; CTA "Enquire now" is visible only after panel is shown.
- [ ] **CTA:** "Enquire now" appears only when recommendation panel is visible (not before goal selection).
- [ ] **Form:** Submit contact form → request sends to Netlify; success message or error message appears in `#form-note`. If `#form-note` is removed from DOM, submit still completes (no console error); only in-screen message is skipped.
- [ ] **Console:** No errors on load, sector select, goal select, form submit, menu open/close. Optional: if you remove a hero element (e.g. `#objective-step`), a single `[Hero] Expected element missing` warning may appear; hero should still work for elements that were found via fallbacks.
- [ ] **Reduced motion:** With OS "reduce motion" on, scroll to dropdown/panel is instant (no smooth scroll); animations minimal.
