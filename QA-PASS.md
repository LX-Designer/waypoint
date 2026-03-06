# Targeted QA and defect-fix pass

## 1. Defect list

| # | Severity | Defect | Cause | Fix applied |
|---|----------|--------|--------|-------------|
| 1 | Low | Script can throw on load if `#year` is missing (e.g. footer markup changed). | `document.getElementById('year').textContent = ...` is called without a null check. | Resolve `#year` once, set `textContent` only when the element exists. |

**No other defects were found** in the traced flows:

- **Hero flow:** Sector click adds `.active` to objective step, populates and enables the dropdown, then calls `scrollToGoalDropdown()` after rAF/rAF + 100ms. Objective change shows panel (removes `.hidden`, adds `.visible`), focuses panel, then `scrollToRecommendationPanel()` after 150ms. CTA is shown only via `.recommendation-panel.visible .recommendation-cta` (opacity 1). Changing sector removes panel visibility, adds `.hidden` after 250ms, clears results, repopulates dropdown, and runs `scrollToGoalDropdown()` again. Behaviour matches requirements.
- **Form flow:** Form has `name="contact"`, `method="POST"`, `data-netlify="true"`, hidden `form-name`, honeypot. `handleSubmit` prevents default, sends POST with `application/x-www-form-urlencoded`, updates note on success/error only when the note element exists, re-enables the button in `finally` when the button exists. Returns `false` so the form does not submit natively; no duplicate submission.
- **Responsive:** Sector grid uses explicit breakpoints (5 cols ≥1024px, 3 cols 768–1023px, 1 col ≤767px). Nav is horizontal at ≥768px with tighter spacing 768–1023px; hamburger and dropdown at ≤767px. No overlap or missing rules found at 1280, 1024, 900, 768, 430, 390.
- **Accessibility:** Dropdown is focused only after `.active` is set on the step. Panel is focused only after `.visible` is set (and `.hidden` removed). Reduced motion is used in scroll helpers and in the global `prefers-reduced-motion` media query. No focus on hidden elements and no keyboard trap in the traced paths.

---

## 2. Minimal code change

**File:** `index.html`

**Change:** Guard the footer year update so missing `#year` does not throw.

```diff
-  document.getElementById('year').textContent = new Date().getFullYear();
+  const yearEl = document.getElementById('year');
+  if (yearEl) yearEl.textContent = new Date().getFullYear();
```

No other code was changed. No refactor, no selector renames, no behaviour changes beyond this guard.

---

## 3. Manual verification checklist

Use this to confirm the flows and responsive behaviour in the browser.

### Hero flow
- [ ] Select a sector → objective dropdown and “What are you trying to achieve?” appear.
- [ ] View scrolls and focus moves to the dropdown (desktop: dropdown centered; narrow: scroll with top offset).
- [ ] Select an objective → recommendation panel appears; view scrolls to panel and focus moves to panel.
- [ ] “Enquire now” CTA is visible only when the recommendation panel is visible.
- [ ] Change sector again → panel and CTA hide, dropdown resets and repopulates, view scrolls/focus goes to dropdown.

### Form flow
- [ ] Submit with valid data → “Thanks! Your message…” (or similar) appears; form resets; no console error.
- [ ] Confirm submission in Netlify dashboard (or test notification).
- [ ] Trigger a failure (e.g. disconnect network or use a failing endpoint) → “Sorry, there was an error…” appears; button returns to “Send message.”
- [ ] No duplicate requests in network tab when submitting once.
- [ ] (Optional) Remove `#form-note` in devtools and submit → request still fires; no console error.

### Responsive (check at 1280, 1024, 900, 768, 430, 390 px width)
- [ ] Sector buttons: at 1280/1024 one row of five; at 900/768 two rows (3+2); at 430/390 one column; equal height, no odd wrapping.
- [ ] Nav: at 768 and above, all items in one row (or wrapped cleanly); at 767 and below, hamburger opens/closes dropdown; no overlap with logo or content.
- [ ] Recommendation panel: padding and text readable at all widths; no horizontal overflow.
- [ ] Form: labels and inputs readable; “Send message” and fields tappable (min ~44px height on small widths).

### Accessibility / basic UX
- [ ] Tab through hero: sector buttons → dropdown (when visible) → panel (when visible) → CTA; order is logical.
- [ ] Focus ring visible on sector buttons, dropdown, panel, “Enquire now,” and form controls.
- [ ] With “Reduce motion” on (OS or browser): scroll to dropdown/panel is instant; animations minimal.
- [ ] Focus is not placed on the dropdown or panel while they are still hidden (e.g. no focus on `display: none` or before `.visible`).
- [ ] Open hamburger menu, tab through links, close with Escape or outside click; focus not trapped.
