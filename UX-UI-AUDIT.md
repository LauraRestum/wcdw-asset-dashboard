# UX / UI Audit — WCDW Asset Dashboard & Campaign Studio

**Date:** July 3, 2026
**Method:** Full code review of `index.html` / `studio.html` (Campaign Studio), `dashboard.html` (Asset Dashboard), `manifest.json`, and `vercel.json`, plus hands-on testing in Chromium at desktop (1440×900) and mobile (375×740) sizes, in light and dark themes, with keyboard-only interaction.

**Priority key:** 🟥 P1 = fix first (broken or blocking) · 🟧 P2 = high value · 🟨 P3 = polish

---

## 1. Information architecture & navigation

### 🟥 1.1 The two tools don't know about each other
There is **no link between the Studio and the Dashboard in either direction**. `index.html` (the Studio) never mentions the dashboard; `dashboard.html` never mentions the Studio. A user who lands on the site home page has no way to discover the asset library, and vice versa — the dashboard is only reachable if you already know the `/dashboard` URL.

**Fix:** Add a small persistent nav (or at least a header link) to both pages: "Asset library ↔ Campaign Studio".

### 🟥 1.2 `index.html`, `studio.html`, and the README disagree
- `index.html` and `studio.html` are **byte-identical copies of the Campaign Studio** (1.36 MB each).
- The README says `index.html` is the dashboard and that you can "double-click `index.html`" to use it from disk. Neither is true anymore: the home page is the Studio, and the Studio explicitly requires an HTTP server (it fetches `manifest.json`).
- The real dashboard moved to `dashboard.html`, reachable on Vercel via the `/dashboard` rewrite — but that rewrite doesn't exist on GitHub Pages, so the two documented hosting paths behave differently.

**Fix:** Pick one canonical URL per tool. Keep `studio.html` as the only copy of the Studio and make `index.html` either the dashboard again (matching the README) or a tiny landing/redirect page. Update the README to match reality.

### 🟧 1.3 No shareable URLs / no state in the URL
Neither tool reflects state in the URL. You can't send a teammate "all giraffe photos" or "this specific asset", and refreshing the Studio loses everything you've built (tab, mode, copy edits, chosen photo — all gone with no warning).

**Fix:**
- Dashboard: encode `?q=…&cat=…&sort=…` and `#asset=<path>` so search results and individual assets deep-link.
- Studio: persist generator state to `localStorage` (it's all serializable) and restore on load; consider a `beforeunload` warning while the generator has unsaved edits.

---

## 2. Accessibility

This is a tool built *for a blindness-services organization* — the dashboard largely walks the walk (skip link, `<dialog>`, focus rings, live regions, reduced-motion support), but the Studio doesn't meet the same bar.

### 🟥 2.1 Studio modal is not accessible (Escape, focus trap)
Verified in-browser: the premade-asset preview modal (`#modal`) **does not close on Escape**, does not trap focus (Tab moves through the page behind the modal), doesn't move focus into the modal on open, and doesn't restore focus on close. The dashboard already does all of this correctly with `<dialog showModal()>` — the Studio uses a plain `div.modal`.

**Fix:** Convert `#modal` to a `<dialog>` like `dashboard.html:369` does, or add Escape handling + focus trapping.

### 🟥 2.2 Brand green used as text on light backgrounds fails WCAG
`#78BE21` on white measures **2.29:1** (AA requires 4.5:1; 3:1 even for large text). It's used for all the eyebrow labels ("ASSET LIBRARY", "BUILD SOMETHING NEW", card-type labels "EMAIL"/"MEDIA", "NO BANNER"), and — more importantly — **inside the generated emails themselves**: the eyebrow line ("Walk with us for independence") and the event-meta line ("Saturday, October 17 · Dallas Zoo") render in `#78BE21` / accent at 12–13 px on white. The Studio's default output fails contrast checks.

Also: muted text `#69768a` on the `#eaf0f8` page background is **4.02:1**, just under AA for the many ≤13 px labels using it.

**Fix:** Keep `#78BE21` for decorative bars/buttons-on-navy (it's 7.3:1 on navy — fine), but introduce a darker "text green" (~`#3f7a10` or darker) for text on light surfaces, in both the Studio UI and the email/graphic templates. Darken `--muted` slightly.

### 🟧 2.3 Studio tabs are only half-implemented ARIA tabs
`role="tablist"`/`role="tab"` are present but there's no `aria-controls`, no `aria-labelledby` on the panels, no `tabindex` roving, and no arrow-key navigation (same for the Email/Graphic/Social mode switcher). Screen-reader users are told "tab" but get none of the expected behavior.

**Fix:** Either complete the pattern (arrow keys + `aria-controls`) or simplify honestly to buttons with `aria-pressed`.

### 🟧 2.4 Brand check results are invisible to screen readers
The brand-check badge and list re-render on every keystroke with no `aria-live` region anywhere in the Studio (`grep aria-live` → 0 hits). A screen-reader user gets no feedback that their copy went off-brand.

**Fix:** `aria-live="polite"` on `#guard` (or a debounced status announcement, e.g. "2 issues to fix").

### 🟧 2.5 Graphic preview canvas has no text alternative
`#bannerCanvas` has no `role="img"` / `aria-label`, so the entire graphic/social preview is silent. The photo tiles in the background-image picker also rely on `title` alone for names.

**Fix:** Set `role="img"` and a generated label ("Preview: 1080×1080 graphic, headline '…' over photo …") updated on render; give library tiles `aria-label` from the manifest description.

### 🟧 2.6 Dashboard lightbox: focus jumps to Close on every navigation
`paintLb()` calls `$("#lbClose").focus()` on every repaint (`dashboard.html:606`), so clicking "Next" yanks focus off the Next button; keyboard/SR users lose their place each step.

**Fix:** Only focus Close on open; on prev/next, keep focus where it is and let the existing `aria-live` panel announce the new asset.

### 🟨 2.7 Studio is missing the basics the dashboard has
No skip link, no `<meta name="description">`, no favicon (`grep` → 0 hits for all three). Minor, but it makes the two pages feel like different products (see §5).

---

## 3. Campaign Studio — workflow & correctness

### 🟥 3.1 Exported emails with photos will break when actually sent
The generator inserts the hero photo as `src="event-photos/…"` (a relative path) or, for uploads, as a `data:` URI. Both fail in real email clients: relative paths resolve to nothing once the HTML leaves the site, and Gmail/Outlook strip `data:` images. The premade templates correctly use hosted `mcusercontent.com` URLs — the generator doesn't. A marketer can pass the brand check, export, load the file into Mailchimp, and send a broken hero.

**Fix (any of):**
- Rewrite photo `src` on export to the site's absolute published URL (origin + path) — works today since assets are on Pages/Vercel.
- Add a "hosted image URL" field with a warning when the selected image isn't hosted.
- At minimum, show a pre-export checklist ("Replace the hero src with a hosted URL before sending").

### 🟧 3.2 Copying in Graphic/Social mode silently resets your canvas size
`copyBtn` restores its label via `setTimeout(() => setMode(mode), 1200)`, and `setMode('graphic'|'social')` force-resets the size to 1080×1080. Build a 1080×1920 Story, hit Copy, and 1.2 s later your artboard snaps back to square. Also, `setMode` rebuilds the whole control panel just to restore a button label.

**Fix:** Restore the label directly (`copyBtn.textContent = original`) instead of calling `setMode`; don't reset `bw/bh` inside `setMode` when the mode didn't change.

### 🟧 3.3 Brand check only reads headline + body
`brandCheck()` concatenates `f-head` and `f-body` only — the CTA, sub-body, and event-meta fields are never checked. "Click here", "impaired", or an em dash in the CTA sails through with an "On-brand" badge, undermining trust in the feature.

**Fix:** Include all free-text fields (`f-cta`, `f-sub`, `f-meta`) in the check.

### 🟧 3.4 Copy failure falls back to a raw `alert()`
`copyBtn` catch shows `alert('Copy failed, use Export.')` — jarring and inconsistent with the dashboard's toast. The `legacyCopy` fallback that exists elsewhere in the same file isn't used here.

### 🟨 3.5 No reset / undo in the generator
Once you edit copy there's no "start over" or "back to approved copy" — you have to re-select the audience to reload defaults, which isn't obvious. A small "Reset to approved copy" affordance would help.

### 🟨 3.6 Post-event state
The "days out" pill will read "0 days out" forever after Oct 17. Cheap fix: "It's walk day!" / "See you next year".

---

## 4. Asset Dashboard — findings

### 🟧 4.1 `email-banners` category renders as a raw slug
The manifest now contains a 5-asset `email-banners` category, but `CAT_META` (`dashboard.html:390`) doesn't know it, so the chip and card labels show literal **"email-banners"** in lowercase with the fallback gray dot (verified in-browser). Any future category will silently do the same.

**Fix:** Add `"email-banners": { label: "Email banners", color: … }` — and consider having `build.py` fail or warn when a manifest category is missing from `CAT_META`.

### 🟧 4.2 Lightbox loads full-resolution originals with no loading state
Opening a card immediately swaps in the original (e.g. 7087×4725 JPEG). Verified: the stage shows a partially-decoded strip of the image with no spinner or placeholder while the rest streams in. On event Wi-Fi this looks broken.

**Fix:** Show the (already-cached) thumbnail immediately, blurred or letterboxed, and swap in the original on its `load` event with a small progress indicator.

### 🟧 4.3 No file sizes anywhere
The originals total ~350 MB, but neither the card, the lightbox, nor the manifest expose per-file size — people on hotspots download blind, and marketers can't tell which photo is email-friendly.

**Fix:** Have `build.py` write `bytes` into the manifest; show "8.2 MB" next to dimensions and on the Download button.

### 🟧 4.4 "Copy path" copies a relative path — but the audience needs URLs
`Copy path` yields `event-photos/the-walk/….jpg`. That's useful for developers editing the repo, but the likelier need (per the repo's own Mailchimp history) is a **hosted URL to paste into an email tool or doc**.

**Fix:** Copy the absolute published URL (`location.origin + '/' + path`), or offer both ("Copy URL" primary, "Copy repo path" secondary).

### 🟨 4.5 Stat tiles look interactive but aren't
"Event photos 22 / Brand 2 / Email 8" are natural filter entry points (and on mobile they occupy the entire first screen before the search box appears — verified at 375 px). Making them buttons that apply the matching filter would turn dead space into navigation; alternatively compress them on mobile so search is above the fold.

### 🟨 4.6 Tags in the lightbox aren't clickable
Every asset shows its keyword tags as inert pills. Clicking a tag should close the lightbox and run that search — it's the cheapest "more like this" you can build, and the data is already there.

### 🟨 4.7 Small keyboard/QoL gaps
- `/` (or `Ctrl+K`) to focus search.
- The chips group is a filter but the buttons include a redundant `role="button"` (`dashboard.html:442`).
- Video cards could show duration (`duration_seconds` is already in the manifest).
- The empty state says "try All collections" but doesn't offer a one-click "Clear filters" button.
- The 🔒 "Locked" badge is styled like an error (red); it's information, not a failure — consider neutral/gray.

---

## 5. Consistency between the two tools

They read as two unrelated products, which costs users the transfer of learning (and costs maintainers double the CSS):

| | Dashboard | Studio |
|---|---|---|
| Font | system-ui | Montserrat |
| Dark mode | ✅ (toggle + auto) | ❌ none |
| Focus ring | blue | green |
| Modal | native `<dialog>`, Esc, focus mgmt | custom div, none of it |
| Copy feedback | toast | button-label swap / `alert()` |
| Skip link / favicon / meta description | ✅ | ❌ |
| Brand palette | Envision blue/green remap | Navy/green/gold |

The July "standardize fonts to Montserrat" pass covered the Studio and emails but not `dashboard.html`.

**Fix:** Extract one small shared stylesheet (tokens: colors, focus ring, radius, font) and one shared modal/toast pattern; add dark mode to the Studio or, at minimum, `color-scheme: light` so form controls don't half-invert on dark-mode systems.

---

## 6. Performance

### 🟧 6.1 The Studio page is 1.36 MB of HTML — twice
- The same Envision logo PNG is embedded as base64 **six times** (once per email template) plus other repeated base64 images — that's the bulk of the 1.36 MB.
- The whole file is then **duplicated** as both `index.html` and `studio.html` (2.7 MB in the repo, and users who visit both URLs download it twice — different cache keys).
- 1.36 MB of HTML blocks parsing on the home page; on a phone on event-day Wi-Fi that's a slow first paint.

**Fix:** De-duplicate the two files (see §1.2); move the six email templates into `email-templates/*.html` fetched on demand (they're only needed on View/Download/Copy); reference the logo once as a shared hosted file.

### 🟨 6.2 Render-blocking Google Fonts
Both the Studio and every generated email pull Montserrat from Google Fonts with no `font-display: swap` and no preconnect; offline/file:// use and slow networks show a flash of Arial or nothing. Self-hosting a woff2 subset (or adding `&display=swap` + `<link rel="preconnect">`) is a one-line improvement. Note the canvas exporter already waits on `document.fonts.load` — good.

---

## 7. What's already good (keep it)

- Dashboard's a11y foundation: skip link, `:focus-visible` rings, `<dialog>` lightbox with focus restore, `aria-live` result counts, `prefers-reduced-motion` support, real alt text sourced from manifest descriptions.
- `manifest.json` as a single source of truth feeding both tools, with thumbnails keeping the gallery at ~1.5 MB.
- The brand check is a genuinely great idea — the fixes in §3.3 make it trustworthy.
- Approved-copy pickers (audience → prefilled headline/body/CTA) are a strong "pit of success" design.
- The graphic generator's shrink-to-fit text layout and canvas export work well, including the accessible-contrast auto text color (`textOn()`).

## Suggested order of attack

1. **P1s:** link the two tools together; resolve `index.html`/README mismatch; Studio modal Escape+focus; text-green contrast token; email-export image URLs.
2. **P2s:** brand check covers all fields; copy-button size reset; `email-banners` label; lightbox loading state; file sizes; copy-URL; tab ARIA; live-region for brand check; split the 1.36 MB page.
3. **P3s:** clickable stats/tags, keyboard shortcuts, shared design tokens, dark-mode Studio, post-event countdown state.
