# Improvement Audit — WCDW Asset Dashboard, Campaign Studio & Email Assets

**Date:** July 7, 2026 · **Baseline commit:** `8d8cf5f`
**Scope:** Everything in the repo — `studio.html`/`index.html`, `dashboard.html`, `dashboard/build.py`, `manifest.json`/`data.js`, `email/VisitUs-EMAIL.html`, `email-banners/`, `social/`, the photo library, `vercel.json`, and repo/deployment hygiene.
**Method:** Full code review of every non-generated file, cross-checks of manifest ↔ `data.js` ↔ disk ↔ thumbnails, WCAG contrast measurement, image/EXIF inspection, git-history analysis, and a headless-Chromium smoke test of all three pages over a local server.

This builds on **[`UX-UI-AUDIT.md`](UX-UI-AUDIT.md)** (July 3). That audit's findings are not repeated here except as a status scoreboard (§8) — the short version is that **almost none of it has been fixed yet**, and two things it praised have since regressed. Everything else below is new.

**Priority key:** 🟥 **P0** — broken, unsafe, or wrong if used today · 🟧 **P1** — high value / prevents real mistakes · 🟨 **P2** — quality & robustness · ⬜ **P3** — polish and maintainability.

---

## Top 10 (if you only fix ten things)

| # | Finding | Where |
|---|---------|-------|
| 1 | 🟥 Every premade email template's CTA links to **`whitecanedaywalk.org`** — every other surface (generator, README, PSA scripts, footers) says **`.com`**. A sent campaign points donors at the wrong domain. | `studio.html` (9 occurrences) |
| 2 | 🟥 Generator fields are interpolated **unescaped** into the email preview/export — pasted or typed HTML executes script in the studio's origin and ships in exported emails. | `studio.html:790–825` |
| 3 | 🟥 Templates mail03–06 embed hero photos as **190–314KB base64 `data:` URIs**, which Gmail/Outlook strip → blank heroes when actually sent. (Also ~0.9MB of the page's bulk.) | `studio.html:1082,1176,1252,1329` |
| 4 | 🟥 `email/VisitUs-EMAIL.html` cannot be sent as-is: placeholder image host on all 10 images **and** `{{Handlebars}}` merge tags (incl. `{{UnsubscribeLink}}`) that Mailchimp will not resolve — a dead unsubscribe link is a CAN-SPAM violation. | `email/VisitUs-EMAIL.html:40–173` |
| 5 | 🟥 No physical mailing address in any of the six studio templates or the generator footer (CAN-SPAM requires one; Mailchimp's `*|LIST_ADDRESS|*` is never used). | `studio.html:818–821, 965–968` |
| 6 | 🟥 `index.html` is still a byte-identical 1.36MB copy of `studio.html`, the README still says `index.html` is the dashboard, and the two tools still don't link to each other. | repo root |
| 7 | 🟧 Graphic export race: every keystroke blanks and asynchronously repaints the canvas; **Download PNG can capture a blank or stale graphic**. | `studio.html:884–896` |
| 8 | 🟧 Dashboard `CAT_META` is missing `email-banners` **and** the new `social` category — 10 of 42 assets (24%) render as raw slugs, and the stat tiles sum to 32, not 42. | `dashboard.html:390–430` |
| 9 | 🟧 The social-set carousel was only added to `studio.html`/`index.html`; the dashboard still shows the 3-slide set as three duplicate-looking cards. | `dashboard.html` gallery |
| 10 | 🟧 There is no download tier between the 640px thumbnail and the 11–25MB straight-off-camera original (7087×4725) — and no file sizes shown — so marketers grab 25MB files for 600px email slots. | photo library / `build.py` |

---

## 1. Send-blocking email correctness (P0)

These are the findings that produce a *visibly broken or non-compliant email* if today's assets are used for a real send.

### 1.1 🟥 Wrong CTA domain in all six premade templates
`whitecanedaywalk.org` appears 9× (`studio.html:957, 1021, 1117, 1191, 1193, 1268, 1270, 1344, 1346`); the generator default (`:350`), generated footer (`:817`), PSA scripts (`:468`), and README all use `whitecanedaywalk.com`. A marketer who downloads "Registration open email" and sends it points every recipient's Register button at a domain the org may not control.
**Fix:** confirm the real domain, then global-replace. Then prevent recurrence — see §6.3 (single `EVENT` constant) and §5.4 (CI grep).

### 1.2 🟥 Unescaped interpolation → script injection in preview and exported email
`emailHTML()` (`studio.html:790–825`) drops `f-head`, `f-body`, `f-sub`, `f-cta`, `f-meta`, `f-img`, and `f-link` into HTML with zero escaping — `clean()` (`:759`) only normalizes dashes. The result is assigned to `emailFrame.srcdoc` (same-origin, scripts run) and is what Export/Copy ships. `f-link` goes raw into `href="${link}"`: a `javascript:` URL is clickable in the preview and a `"` breaks the attribute.
**Failure:** paste `<img src=x onerror=…>` into the headline (or copy containing `<`/`&` from a doc) → script runs in the studio; the exported `.html` carries the payload to the ESP and recipients.
**Fix:** escape every interpolated value with the existing `ph_esc()` (`:567`); validate `f-link` with `new URL()` and allow only `http(s):`.

### 1.3 🟥 mail03–06 heroes are base64 `data:` URIs — stripped by real email clients
`studio.html:1082` (314K chars), `:1176`, `:1252`, `:1329`. Gmail, Outlook, and most ESPs strip or refuse `data:` images, so these templates preview perfectly in the studio and send with blank heroes. This is a **regression**: the July 3 audit explicitly praised the premade templates for using hosted URLs — only mail01/02 still do (`:945`, `:1015`).
**Fix:** host the four heroes (the site itself works — assets are already deployed) and reference by absolute URL, like mail01/02. This also removes ~0.9MB from the 1.36MB page.

### 1.4 🟥 `email/VisitUs-EMAIL.html` is a non-functional draft
Documented as intentional in the README, but worth stating plainly because it *looks* finished:
- All 10 `<img src>` use `https://REPLACE-WITH-YOUR-IMAGE-URL/` (lines 40, 71, 79–81, 96, 120, 159–161) — every client renders broken images until a real host is substituted.
- Eight `{{Handlebars}}`-style merge tags in `href`s (`{{KrogerLink}}` :104, `{{DonateLink}}` :128, `{{SignOutLink}}` :149, `{{UnsubscribeLink}}` :171, `{{PrivacyLink}}` :172, `{{BrowserLink}}` :173, plus `{{QRCode}}`/`{{WalletLink}}` in a comment :151). **Mailchimp uses `*|UNSUB|*` / `*|ARCHIVE|*` syntax and will pass `{{UnsubscribeLink}}` through literally** — recipients clicking Unsubscribe get a browser error, which is a CAN-SPAM violation (up to ~$53k per email) and a deliverability killer under Gmail/Yahoo bulk-sender rules.
- Line 152's visible copy says "Save your QR code for faster sign in" but the QR slot exists only inside an HTML comment — every recipient sees an instruction with no QR code.
- Line 169's postal address ("1801 Valley View Lane, Farmers Branch") should be verified — Envision Dallas's commonly published campus address is 4306 Capitol Ave, Dallas. A wrong footer address is itself a CAN-SPAM defect.
**Fix:** host the 7 images and swap the base URL; replace all merge tags with the target ESP's real syntax; render or remove the QR promise; confirm the address. Add a pre-send checklist to the README.

### 1.5 🟥 No postal address / unsubscribe in studio output
The six templates' footers have `*|UNSUB|*`/`*|UPDATE_PROFILE|*` but **no physical address** (`studio.html:965–968` and equivalents) — add `*|LIST_ADDRESS|*` or the real address. The *generator's* footer (`:818–821`) has neither address **nor** unsubscribe: anything built with it and sent as a campaign is non-compliant out of the box.

---

## 2. Architecture & repo health

### 2.1 🟥 One page, two files, wrong docs (carried over, still open — restated because it taxes every commit)
`index.html` ≡ `studio.html` (byte-identical, 1.36MB each). Every studio edit commits ~2.7MB of new blobs — the duplication has already added **~24MB across 15 blobs** to git history. The README still documents `index.html` as the dashboard ("double-click it — works from disk"), which is false twice over (it's the studio, and the studio requires HTTP). GitHub Pages and Vercel behave differently because the `/dashboard` rewrite only exists on Vercel.
**Fix:** keep `studio.html` as the only studio file; make `index.html` either the dashboard (matching the README) or a 2KB landing page linking both tools; add the missing cross-links between the two tools; update the README.

### 2.2 🟧 The studio page is ~90% two repeated images
Of the 1.36MB: one ~230KB JPEG embedded as base64 **4×** and one ~42KB logo PNG embedded **6×** (`studio.html:963, 1030, 1124, 1200, 1277, 1353` for the logo). De-duplicating to hosted/relative file references shrinks the page to roughly **150KB** — a ~9× faster first paint on event-day hotel/venue Wi-Fi. Moving the six email templates into `email-templates/*.html` fetched on demand (they're only needed on View/Download/Copy) gets most of the rest and makes templates diffable and individually editable.

### 2.3 🟧 No web-size image tier; originals are camera-native
Every event photo is a straight-off-camera 7087×4725 JPEG at 11–25MB (Sony A7 III/IV per EXIF). The dashboard's only choices are a 640px thumbnail or the full original — so the "Download" button hands a 25MB file to someone filling a 600px email slot, and the lightbox streams the 25MB original for a screen-sized preview (known 4.2).
**Fix in `build.py`:** emit a second derivative tier, e.g. `dashboard/web/…` at 2048px long edge, JPEG q82 (≈400–800KB each, ~15MB total). Use it as the lightbox image and the default download ("Download web size (0.6MB) / original (25MB)"), and as the studio's graphic background (the canvas never needs 7087px). Record `bytes` for every file in the manifest while you're in there (fixes known 4.3).

### 2.4 🟧 Repo weight & clone cost
Working tree is ~365MB (photos 344MB); a fresh clone moves ~2× that with `.git`. That's tolerable for a static-hosted repo, but be deliberate: don't add more camera-native originals; if the library grows, move originals to Releases/cloud storage and keep only web tiers in-repo. **Avoid Git LFS** here — GitHub Pages serves LFS pointer files, which would silently break the site's images on the Pages path.

### 2.5 🟧 EXIF metadata on originals
33 of 41 images carry EXIF (camera model, capture timestamps — Oct 26–27, 2025). **No GPS data was found** (good). Since attendees' photos are published, decide deliberately: strip EXIF from originals (`exiftool -all= …` or re-export via Pillow in `build.py`) or accept camera/date metadata as harmless. Thumbnails are already re-encoded clean. Note the capture dates say **2025** while the event branding says Oct 17 **2026** — presumably last year's photos promoting this year's walk; fine, but worth knowing the metadata contradicts the campaign year if anyone inspects a download.

### 2.6 🟨 Stray file: `wcdw mockup-10.png`
A 2000×1414 PNG sits at the repo root (space in filename), absent from `manifest.json` — invisible in both tools — yet `build.py` generated and committed `dashboard/thumbnails/wcdw mockup-10.jpg` for it. Either give it a manifest entry and a proper home, or delete it and its orphan thumbnail. (Root cause in §3.1.)

### 2.7 🟨 Missing site plumbing
No favicon in the studio (the dashboard has an inline SVG one; confirmed via headless smoke test — the only 404 on page load), no `404.html`, no `robots.txt`. Given the library contains identifiable photos of attendees **including children**, decide whether search engines should index the photo paths at all; a `robots.txt` (or `X-Robots-Tag`/`noindex` on Vercel) is a one-file decision the org's photo-consent policy should drive.

---

## 3. Build & data pipeline (`dashboard/build.py`, `manifest.json`)

The manifest layer is in good shape — `data.js` is byte-equivalent to `manifest.json`, all 42 asset paths exist on disk, no thumbnails are missing. The issues are about what the build *doesn't* check:

### 3.1 🟧 `build.py` thumbnails everything on disk, manifest or not, and never warns about drift
`build_thumbnails()` walks the whole tree and thumbnails every raster file regardless of manifest membership — that's how the stray mockup got a committed thumbnail. Conversely, a manifest entry whose file was deleted passes silently. **Fix:** drive thumbnail generation *from the manifest*, and end the run with a report: files on disk not in the manifest (warn), manifest paths missing on disk (**error, exit 1**), orphan thumbnails (delete). That turns the manifest into an enforced — not aspirational — source of truth.

### 3.2 🟨 Correctness nits in `build.py`
- `SKIP_DIRS` (`build.py:27`) is defined but never used; the walk re-implements the skip logic inline (`:46`) — delete one or use the other.
- **Stem collision:** `a.png` and `a.jpg` in the same folder both map to thumbnail `a.jpg` — last one wins silently. Include the original extension in the thumb name.
- MP4s get no poster: generate a poster frame for videos (or at least document that `thumbnail` is expected to be absent), so the dashboard lightbox isn't a black box while frames decode.
- `duration_seconds` is in the key-order list but never computed — it's hand-maintained; either compute it (ffprobe if available) or comment that it's manual.
- No `requirements.txt` / pinned Pillow version; add one line so the build is reproducible.
- Record `bytes` (and the new web-tier path, §2.3) per asset.

### 3.3 🟧 Nothing verifies generated files stay in sync
`data.js`, thumbnails, and the manifest enrichment are all committed artifacts with no check. One forgotten `build.py` run and the dashboard (which prefers `data.js`) and studio (which fetches `manifest.json`) render different libraries. **Fix:** a ~15-line GitHub Action: `pip install Pillow && python3 dashboard/build.py && git diff --exit-code`. This one job also catches the drift class in §3.1. (The repo currently has no `.github/` at all.)

### 3.4 🟨 `vercel.json` is a single rewrite
No caching headers (thumbnails and photos are immutable — `Cache-Control: public, max-age=31536000, immutable` would make repeat visits instant), no security headers (`X-Content-Type-Options: nosniff` at minimum; a basic CSP would have contained §1.2), and the `/dashboard` rewrite exists only on Vercel while the README also documents GitHub Pages (path behavior diverges). Add a `headers` block and use plain `/dashboard.html` links in-app so both hosts behave identically.

---

## 4. Campaign Studio — new functional findings

(Beyond the P0s in §1. Prior-audit items not repeated; see §8.)

### 4.1 🟧 Graphic renderer race → blank/stale exports
Every keystroke calls `renderGraphic()` (`studio.html:884–888`), which synchronously resets `c.width` (blanking the canvas) then paints asynchronously after `document.fonts.load()` + a fresh `new Image().onload`. Nothing sequences callbacks, and Export/Copy (`:895–896`) reads whatever pixels exist at click time. Type fast and hit Download PNG → previous headline, or a bare navy rectangle.
**Fix:** render-sequence token (ignore stale callbacks), cache loaded `Image`s per src, make export await the in-flight render. Debounce input → render ~100ms (see also §4.6).

### 4.2 🟧 Destructive dropdowns: audience/phase overwrite edits with no confirm and no way back
`applyAudience()` (`:726–727`) replaces headline/body/CTA unconditionally; `applyPhase()` overwrites the headline but "No phase" is a no-op so you can't undo (`:728`, `:442`). Ten minutes of custom copy dies by brushing a `<select>`. **Fix:** confirm when fields are dirty; pair with the still-missing "Reset to approved copy" button (known 3.5).

### 4.3 🟧 Newlines in the body are silently dropped in emails
`${b}` lands in a single `<td>` (`:814`) with no `\n → <br>`; multi-paragraph copy collapses to a run-on. (The social caption path handles this correctly with `pre-wrap`, `:191`.) Fix after escaping (§1.2): `esc(b).replace(/\n/g,'<br>')`.

### 4.4 🟧 Generated email HTML has no `<head>` at all
`:808` emits `<!DOCTYPE html><html><body …>` — no `charset`, no `title`, no `lang`. The approved copy is full of `'` and `·`, so any client/web-view not defaulting to UTF-8 shows `â€™` mojibake. The six handwritten templates get all of this right; the generator should emit the same head (plus the compliance footer, §1.5).

### 4.5 🟧 Error/empty states mislead
- A manifest that loads but contains no `event-photos/` images shows "serve this page over http (not file://)" even on a healthy HTTPS deploy (`:693`, `:704`, `:571`).
- When fetch *does* fail, only Photos gets an error tile — premade Banners/Social chips just show fewer tiles with no message (`:568–573`).
- Asset counter says "Showing 5 assets" over 3 social tiles: `updateAssetCount` counts slides, the grid groups them (`:505` vs `:527–539`).
**Fix:** track fetch-failure separately from empty-result; error tiles for all three filters; count groups, not slides.

### 4.6 🟨 Per-keystroke O(everything) re-render
Each input event rebuilds the option list via `innerHTML` (~16 buttons) *and* re-runs `render()` — in email mode that reassigns `iframe.srcdoc` (full document re-parse per character); in graphic mode it re-fires the font+image pipeline (§4.1). Debounce and update `aria-pressed` in place.

### 4.7 🟨 Smaller functional items
- **"Customize" on the 600×200 email-header asset opens a 1080×1080 canvas** — `customizeAsset` → `setMode('graphic')` force-resets size (`:602`, `:751`); same root cause as known 3.2.
- **Shrink-to-fit can still overflow**: the loop bails at `minBig` without fitting (`:859`); long headline+sub+CTA on 600×200 draws over the gold meta line and watermark (`:863–880`). Truncate or drop sub/CTA when it still doesn't fit.
- **`toBlob` can callback `null`** (tainted/unencodable) → `dl(null)` throws with no user feedback; `dl()` also revokes the object URL synchronously after `.click()`, which has raced Safari downloads (`:889–896`). Guard and revoke on a timeout.
- **Carousel:** grouping regex (`:530`) turns any lone `…-N.ext` file into a 1-slide "set" with self-cycling arrows and "1 images"; arrows don't `preventDefault` so the page scrolls behind the modal; slide changes are never announced (no live region on `#mTitle`/`#mCount`); "Download all" fires 3 programmatic downloads and browsers may block all but the first (`:564`). Demote groups <2; add Escape/preventDefault/live region; sequence downloads.
- **Off-palette green** `#1B5E20` (Material "Green 900") in mail02's impact band (`:1024`) vs the brand's `#00491E` (`:317`).
- **mail04/05/06 ship two CTAs** ("Join the Walk" + "Donate Instead") — violating the studio's own one-CTA guardrail that `brandCheck` enforces on users (`:771–772`, templates `:1190–1194` etc.).
- **WFAA PSA copy breaks the studio's own language rules** ("blind or low vision" instead of "blind or *have* low vision"; contains em dashes — a hard fail in `brandCheck`) (`:469`).
- **Hardcoded hero alt** "Walkers at the White Cane Day Walk" regardless of the actual image (`:795`) — manifest descriptions are available and unused.
- **Stale fallback "108 days out"** hardcoded in the hero (`:235`), correct only around July 1.
- **Dead code:** `wrap()` (`:827`), `ph()` (`:672`), `status:'live'` fields, unused `cat` on LIBRARY items.

---

## 5. Asset Dashboard — new findings

### 5.1 🟧 Category system didn't absorb the two new collections
- `CAT_META` (`dashboard.html:390–399`) lacks `email-banners` (known 4.1) **and** `social` → chips/cards/lightbox show raw lowercase slugs with the gray fallback dot for 10 of 42 assets.
- Stat tiles bucket only `event-photos`/`brand`/`email` (`:418–430`) → tiles sum to 32 under "Total 42"; banners and social live in no tile.
- `typeBadge` (`:501–505`) labels every image "Photo" — including 1200×300 banners and the Facebook cover.
**Fix:** derive tiles and chips from the manifest's `categories` object (it already exists in the JSON) instead of hardcoding; have `build.py` fail when a manifest category has no `CAT_META`/color entry (this is the third recurrence of this exact bug class).

### 5.2 🟧 Feature parity: the social-set carousel skipped the dashboard
Commits `d1e9ca5`/`fb2e613` touched only `index.html`/`studio.html`. The dashboard shows `save-the-date-instagram-1/2/3.jpg` as three near-identical cards with no set affordance — easy to post one slide of a 3-slide set. Port the grouping or add a "Slide N of 3" pill. (This is the cost of §2.1's duplication pattern in action.)

### 5.3 🟧 Lightbox media handling
- **Arrow keys hijack the video**: the document keydown handler `preventDefault`s ArrowLeft/Right into prev/next asset (`:623–627`), so you cannot keyboard-seek the 52s video (`:567–570`); bail when focus is inside the media element.
- **Video autoplays with controls** (`:568`) — 52s starts the instant the card opens (or when arrow-keying past it); WCAG 1.4.2 problem and it talks over screen-reader announcements. Drop `autoplay` or mute it.
- **The email preview iframe is same-origin and unsandboxed** (`:572`): any script in a current or future file in `email/` runs with the dashboard's origin (can touch `localStorage`, `window.parent`, the DOM). Email HTML is exactly the asset class most likely to be pasted in from third-party tools. Add `sandbox=""` — the preview needs no scripts.
- **Mobile overflow:** at ≤880px the stage media is sized against `100vh` rather than its grid track (`:243–250`), so portrait images (the 1080×1350 social slides) push Download/Copy off-screen with no scroll affordance. Cap stage height (~55vh) in the mobile query.

### 5.4 🟨 Smaller dashboard items
- **`aria-label` masks the good alt text**: thumb buttons announce "Preview *filename*" (`:522–523`), so the carefully written manifest descriptions are never heard in the grid. Label with the description.
- **No scroll lock / history integration** for the modal (`:560`): background scrolls under the lightbox (position lost on close); mobile Back exits the page instead of closing the preview.
- **Copy-path toasts "Copied" even on failure**: `legacyCopy` swallows the `execCommand` result and `done()` runs unconditionally (`:635–645`; same pattern `studio.html:618–621`).
- **Theme toggle destroys "auto"**: one click pins light/dark forever (`:672–679`), and the saved theme applies at end-of-body → white flash on load for dark-mode users. Cycle auto→light→dark; move the restore into a tiny inline head script.
- **Fetch fallback doesn't check `r.ok`** (`:687`) — a 404 page produces "Run build.py" misdiagnosis.
- **Dialog accessible name is always "Asset preview"** (`:369`); set it per-asset in `paintLb`.
- **Stale copy:** meta description (`:7`), hero lede (`:327`), and footer (`:362–366`) predate the banners/social collections, and the manifest's `usage_notes` (e.g. "hyperlink banners to whitecanedaywalk.com") surface nowhere in the UI.

---

## 6. Email template & banners (`email/`, `email-banners/`) — beyond the P0s

Weights are healthy: template HTML 16.7KB (no Gmail-clipping risk), referenced images ~96KB total, correct 2× retina sizing throughout. Structure is genuinely strong (13/13 layout tables have `role="presentation"`, MSO PixelsPerInch conditional, full CSS reset, preheader, real heading hierarchy, https links, no "click here"). Remaining issues:

### 6.1 🟧 Contrast failures concentrated exactly where this org can least afford them
Measured: `#78BE21` on white = **2.29:1**, on `#f6f8fc` = **2.15:1** (12px semi-bold eyebrows, lines 59/74/99/123 — needs 4.5); `#9aa1ad` QR-instruction text = **2.60:1** (line 152); footer `#6b7280` on `#f4f6fa` = **4.47:1**, just under AA. The same green-on-navy usage (5.18:1) is fine — keep it there. **Fix:** darker text-green (~`#3E7A00`) or navy for eyebrows on light; darken the grays; consider a 13–14px floor for footer text given the audience. This is the email-side twin of known finding 2.2, which also still applies to the studio UI and its generated emails.

### 6.2 🟧 Outlook desktop robustness
- Buttons put padding on the `<a>` inside a colored `<td>` (lines 102–106, 126–130, 148–150) — Word-engine Outlook collapses them to a text-sized sliver. Move padding to the `<td>`/use the bulletproof pattern.
- The Google Fonts `<link>` (line 11) is not wrapped in `<!--[if !mso]><!-->…<!--<![endif]-->` — this can trigger Outlook's Times New Roman fallback bug despite the correct inline Arial stacks.
- Vertical spacing rides on `<div>` padding (lines 45, 58) which Outlook drops → eyebrow/heading collisions. Move to `<td>` padding or table rows.

### 6.3 🟨 Dark mode, mobile, polish
- `color-scheme: light` (line 8) is honored by Apple Mail only; Gmail/Outlook apps will auto-invert the navy band and green accents. Either accept it or add `prefers-color-scheme` + `[data-ogsc]` overrides; the transparent-background icon PNGs need a solid disc/stroke to survive inversion.
- Clients that strip `<head><style>` (Gmail-app non-Google accounts, some Yahoo) never see the 600→100% media query — consider hybrid ("spongy") sizing if that audience matters.
- Preheader has no `&zwnj;&nbsp;` padding, so inbox preview text runs into the body copy (line 34).
- Social icon alts are bare platform names ("LinkedIn") — use "Envision on LinkedIn" (lines 79–81, 159–161).
- **Banner weights:** `every-step-counts.jpg` is 134KB for 1200×300 — re-export at q≈72 (the navy banner proves 63KB is achievable); two others are ~110KB.
- **Banner alt text:** all five banners bake headline, date, URL, and (one) a painted "Register" button into pixels, and the repo ships no recommended alt text with them. Add per-banner alt strings to the manifest/README, and the guidance to always repeat date + registration URL as live text near the banner.

---

## 7. Consistency, tooling & the highest-leverage refactors

1. **One `EVENT` constant** `{date, venue, url, registrationUrl}` consumed by the countdown, hero pill, `f-meta` default, `emailHTML`, filenames, and all templates. The event date/venue/URL is currently hardcoded in ~10 independent places across `studio.html` alone — this is how the `.org`/`.com` split (§1.1) and the eternal "0 days out" (known 3.6) happened, and it's what makes next year's event a find-and-replace minefield.
2. **Externalize the six email templates** to `email-templates/*.html`, fetched on demand. Kills the 6× logo and 4× hero duplication (§2.2), makes templates reviewable in diffs, and lets the email agent/ESP workflows consume them directly.
3. **Shared design tokens + shared widgets.** The two tools still disagree on font, focus color, modal implementation, copy feedback, and dark-mode support (prior §5 — unchanged). One small shared CSS file plus one `<dialog>`-based modal and one toast implementation would fix a half-dozen open findings in both files at once.
4. **CI as the drift-catcher** (§3.3): build-and-diff job, plus cheap greps that would have caught §1.1 (`grep -c whitecanedaywalk.org → must be 0`), §5.1 (manifest categories ⊆ CAT_META), and `data:image` in template heroes (§1.3).
5. **A `SENDING.md` pre-flight checklist** for anything email-shaped: real image host, real merge tags for the target ESP, unsubscribe + postal address present, contrast spot-check, Litmus/EOA or at minimum a seed-list test in Gmail + Outlook desktop + Apple Mail dark mode.

---

## 8. Status of the July 3 UX/UI audit (scoreboard)

Verified item-by-item at `8d8cf5f`:

| Status | Items |
|---|---|
| ✅ Fixed | — (none fully) |
| 🟡 Partial | 2.4 (`#assetCount` got `role="status"` — but the brand check, the actual complaint, is still silent) · 4.7c (video duration now shows in the lightbox, not on the card) · studio manifest-fetch now has a real error path (improvement noted since audit) |
| ❌ Not fixed | 1.1 cross-links · 1.2 index/studio/README mismatch · 1.3 shareable URLs/state persistence · 2.1 studio modal Escape/focus · 2.2 green-on-white contrast · 2.3 ARIA tabs · 2.5 canvas alt · 2.6 lightbox focus jump · 2.7 skip link/meta/favicon · 3.1 export image URLs (**regressed** — see §1.3) · 3.2 copy-button size reset · 3.3 brand-check coverage · 3.4 `alert()` · 3.5 reset affordance · 3.6 post-event countdown · 4.1 CAT_META (**worse** — now 2 categories) · 4.2 lightbox loading state · 4.3 file sizes · 4.4 copy-URL · 4.5 stat tiles · 4.6 clickable tags · 4.7a/b/d/e · §5 consistency · 6.1 page weight/duplication · 6.2 fonts |

The pattern to notice: **new features (social sets, new banner/social collections, new templates) keep landing while the known-broken foundations don't get touched, and each new feature then re-collides with the same foundations** (social → CAT_META; carousel → the non-`<dialog>` modal; new templates → the base64/hosted-URL problem). A short "fix sprint" before the next feature would pay for itself immediately.

---

## 9. What's genuinely good (keep and build on)

- `manifest.json` as a single enforced-ish source of truth feeding both tools; `data.js` is perfectly in sync today; descriptions and tags are thoughtfully written and used as real alt text.
- The dashboard's a11y baseline (skip link, `<dialog>`, `:focus-visible`, `aria-live` counts, reduced-motion) is still the best code in the repo.
- All manifest-derived rendering is XSS-escaped in **both** tools (`esc()` / `ph_esc()`) — the §1.2 hole is user-input-into-templates, not manifest data.
- `VisitUs-EMAIL.html`'s table architecture, `role="presentation"` discipline, and weight budget are professional-grade; it just needs its placeholders resolved.
- The brand-check concept, approved-copy pickers, and the canvas `textOn()` auto-contrast remain the product's best ideas.
- No GPS EXIF in any published photo; thumbnails pipeline keeps the gallery at ~2MB.

## Suggested order of attack

1. **This week (P0):** `.org`→`.com` (§1.1) · escape generator fields + validate links (§1.2) · host mail03–06 heroes (§1.3) · address + unsubscribe in all studio output (§1.5) · resolve or clearly quarantine `VisitUs-EMAIL.html` placeholders (§1.4) · de-duplicate `index.html`/`studio.html` + fix README + cross-link the tools (§2.1).
2. **Next (P1):** canvas export race (§4.1) · CAT_META/stat tiles from manifest + build-time check (§5.1) · web-size download tier + bytes in manifest (§2.3) · CI build-and-diff + greps (§3.3, §7.4) · email contrast + Outlook buttons (§6.1–6.2) · iframe sandbox + video autoplay/arrow keys (§5.3) · destructive dropdown confirm (§4.2).
3. **Then (P2/P3):** externalize templates + `EVENT` constant (§7.1–7.2) · shared tokens/modal/toast (§7.3) · the long tail in §4.7, §5.4, §6.3 · the still-open July 3 P2/P3 list (§8).
