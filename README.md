# WCDW Asset Dashboard

A browsable **asset dashboard** for **Envision Dallas — "Celebrating Independence: White Cane Day Walk"** held at the **Dallas Zoo (2026)**: every photo, brand asset, and email image in one searchable gallery, plus the post-visit thank-you email (`email/VisitUs-EMAIL.html`).

## The dashboard

Open **[`index.html`](index.html)** — a self-contained, accessible dashboard that lets you:

- **Search** by description, keyword tag, collection, or file name (e.g. `giraffe`, `braille`, `banner`, `tunnel`).
- **Filter** by collection and **sort** by collection, name, or resolution.
- **Preview** any asset full size in a lightbox (photos, the logo video, and the email HTML), then **copy its path** or **download the original**.
- Toggle **light / dark** mode. Built keyboard-first with alt text, focus management, and live region announcements — appropriate for a blindness-services organization.

**Three ways to view it:**

| How | Steps |
| --- | --- |
| **GitHub Pages** | Enable Pages for this branch (Settings → Pages) — the dashboard is the site's home page. |
| **Local server** | `python3 -m http.server` in the repo root, then open `http://localhost:8000/`. |
| **Straight from disk** | Double-click `index.html`. It reads the pre-generated `dashboard/data.js`, so it works with no server. |

The gallery uses small thumbnails in `dashboard/thumbnails/` (≈1.5 MB total) instead of the full 350 MB of originals, so it stays fast; the originals are only fetched when you open or download an asset.

## Finding assets without the dashboard

1. **Browse the folders** below (or the per-collection tables further down).
2. **Read [`manifest.json`](manifest.json)** — a machine-readable index of every asset with a description and keyword tags, built for search tools and AI assistants. Each entry lists the file's path, type, dimensions, description, and tags.

## Folder map

```
.
├── index.html                    The asset dashboard (open this)
├── manifest.json                 Machine-readable index of every asset (source of truth)
├── dashboard/                    Dashboard build output
│   ├── build.py                  Regenerates thumbnails + data.js from the assets & manifest
│   ├── data.js                   Generated: manifest embedded for the offline (file://) dashboard
│   └── thumbnails/               Generated: web-sized previews mirroring the asset paths
├── event-photos/                 Photos from the White Cane Day Walk at the Dallas Zoo
│   ├── the-walk/                 Participants walking/marching, canes raised, start banner
│   ├── ceremony-and-program/     Seated audience and the outdoor program
│   ├── animal-encounters/        Giraffe feeding, tortoise & opossum keeper presentations
│   ├── registration-and-booths/  Check-in and the Envision info/braille booth
│   ├── group-photos-and-portraits/  Posed group photos and portraits
│   └── candid-moments/           Candid, close-up, and human-interest shots
├── brand/                        Brand/marketing imagery (building exterior, logo video)
└── email/                        The thank-you email template + its 7 image assets
```

## Adding or changing assets

`manifest.json` is the single source of truth. After you add, remove, or move a file:

1. Add or edit its entry in `manifest.json` (path, category, description, tags).
2. Run the build to regenerate thumbnails and the embedded data:

   ```bash
   pip install Pillow      # once
   python3 dashboard/build.py
   ```

   This writes `dashboard/thumbnails/…`, refreshes `dashboard/data.js`, and backfills each asset's `type`, `filename`, `thumbnail`, and `dimensions` in `manifest.json`. Commit the results.

## event-photos/the-walk

| File | What it shows |
| --- | --- |
| `walk-participant-white-cane-tree-path.jpg` | A smiling woman holding a white cane walks a tree-lined zoo path beside a younger attendee in a "Rollin' to the Beat / Bowl-A-Thon" tee. |
| `guided-walk-cane-user-with-escort.jpg` | A cane user in an Envision Dallas polo is sighted-guided arm-in-arm by a companion while another woman films on her phone. |
| `two-women-walking-cane-user.jpg` | A blind woman with a white cane (navy skirt, clip-on fan) walks and chats with a younger woman companion. |
| `participants-walking-boardwalk.jpg` | A stream of participants in lanyards and event shirts walking the zoo's wooden boardwalk. |
| `crowd-behind-independence-banner.jpg` | A dense crowd, many raising white canes, marching behind the large green "…INDEPENDENCE" banner; a "White Cane Day" tee in front. |
| `walk-start-banner-group-with-sponsors.jpg` | Wide marquee shot: the whole group behind the "Celebrating Independence — White Cane Day Walk — Dallas Zoo" start banner with sponsor logos (Southwest, Comerica, Atmos, ABC). |

## event-photos/ceremony-and-program

| File | What it shows |
| --- | --- |
| `audience-member-raising-white-cane.jpg` | A woman in a black-and-white floral top raises a folded white cane amid the seated crowd; a "U.S. Navy Veteran" cap nearby. |
| `seated-audience-teal-shirts.jpg` | Wide shot of the seated audience in teal event shirts watching the program; wheelchair users in front, a Pepsi machine and PA speaker at right. |

## event-photos/animal-encounters

| File | What it shows |
| --- | --- |
| `giraffe-feeding-deck.jpg` | Attendees at the Dallas Zoo giraffe deck hold out lettuce to feed a giraffe leaning over the wooden enclosure. |
| `zoo-keeper-tortoise-presentation.jpg` | Two Dallas Zoo keepers in purple polos present a tortoise beside the "White Cane Day Walk — Animal Encounter Sponsor" banner. |
| `zoo-keeper-opossum-presentation.jpg` | A Dallas Zoo keeper with a microphone presents an opossum to the seated audience near the thatched-hut huts. |

## event-photos/registration-and-booths

| File | What it shows |
| --- | --- |
| `envision-info-booth-braille-demo.jpg` | The Envision Dallas info booth under a pavilion — staff demonstrate Perkins braille typewriters at a banner-draped table; visitors and children gather. |
| `indoor-registration-check-in.jpg` | Indoor check-in in a lodge room with elephant-silhouette decor — a man in a white polo hands out materials at a black-draped table. |

## event-photos/group-photos-and-portraits

| File | What it shows |
| --- | --- |
| `group-portrait-boardwalk-navy-veteran.jpg` | Posed group of four by the boardwalk railing — a U.S. Navy veteran in a cap and three participants in Envision walk shirts. |
| `delta-gamma-volunteers-group.jpg` | Seven smiling Delta Gamma sorority volunteers pose together on the zoo path. |
| `family-portrait-led-tunnel.jpg` | A family with children poses under the colorful LED-lit tunnel ceiling; White Cane Day shirts and a folded cane. |
| `large-group-portrait-led-tunnel.jpg` | A large multi-generational group (with a stroller) poses under the LED tunnel's blue/green light strips. |
| `group-portrait-tunnel-mural.jpg` | A group of attendees poses in the mural tunnel, including cane users and a child in a stroller. |

## event-photos/candid-moments

| File | What it shows |
| --- | --- |
| `guest-waving-lanyard.jpg` | A smiling, balding man in a dark polo and lanyard waves at the camera during the event. |
| `attendees-hug-at-concession-stand.jpg` | Two women share a warm hug in front of a Pepsi/Dole Whip concession stand; a baby in a stroller at left. |
| `volunteer-handing-treat-to-child.jpg` | A volunteer bends down to hand a treat to a delighted young boy in blue near a shaded path. |
| `white-cane-closeup-sneaker-keychain.jpg` | Close-up of a hand gripping a white cane, a small sneaker keychain tied to the grip; crowd blurred behind. |

## brand/

| File | What it shows |
| --- | --- |
| `envision-dallas-building-exterior.png` | Clean exterior photo of the Envision Dallas building (brown brick, blue/green logo signage) under a blue sky. |
| `white-cane-day-logo-animation.mp4` | Animated hero/background video (1280×720, ~52 s, H.264): the "Celebrating Independence — White Cane Day Walk — Dallas Zoo" logo animating in. |

## email/

The post-visit thank-you email template and the 7 image files it references.

| File | What it is |
| --- | --- |
| `VisitUs-EMAIL.html` | Responsive thank-you email for Envision Dallas. Image `src`s use a placeholder host (`REPLACE-WITH-YOUR-IMAGE-URL/`) — host the 7 files below and swap that base URL in before sending. |
| `hero.jpg` | Email hero/banner: the Envision Dallas building with a navy overlay (1200×420). |
| `icon-share.png` | Circular blue/green "Share" icon (share-node glyph). |
| `icon-kroger.png` | Circular blue/green "Kroger Community Rewards" icon (shopping cart). |
| `icon-donate.png` | Circular blue/green "Make a donation" icon (heart). |
| `soc-facebook.png` | Facebook app-style icon. |
| `soc-instagram.png` | Instagram app-style icon. |
| `soc-linkedin.png` | LinkedIn app-style icon. |

> **Do not rename the 7 image files in `email/`.** The email's `<img>` tags reference them by these exact filenames. The event and brand photos, by contrast, can be renamed freely — nothing links to them.

---

*Every photo was individually reviewed to write these descriptions and the tags in `manifest.json`. Faces of event attendees appear throughout; use in line with Envision Dallas's photo-consent and privacy practices.*
