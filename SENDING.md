# Before you send an email

The Campaign Studio produces clean, escaped, mostly compliant HTML — but a few things depend on your email tool (ESP) and can't be baked in. Run this checklist before any real send.

## Every send
- [ ] **Hosted hero image.** If you added a photo, it must be a **hosted `https://` URL** (paste one into "Or paste a hosted image URL", or upload the photo to your ESP and use its URL). Uploaded/`data:` images and relative paths are stripped by Gmail and Outlook — the hero will be blank. The Studio warns you in the brand check when the image won't survive.
- [ ] **Merge tags match your ESP.** The generator footer uses **Mailchimp** syntax: `*|UNSUB|*`, `*|UPDATE_PROFILE|*`. If you use a different ESP (Constant Contact, HubSpot, etc.), swap these for that tool's unsubscribe tags. A dead Unsubscribe link is a CAN-SPAM violation.
- [ ] **Physical mailing address** is correct. The footer is hardcoded to **1801 Valley View Lane, Farmers Branch, TX 75234** — update it if the campus address changes.
- [ ] **One primary CTA.** The brand check enforces this; don't add a second button.
- [ ] **Brand check is green** (no red "to fix" items).
- [ ] **Registration link** points to the right place (`https://whitecanedaywalk.com`).

## Spot-check the render
- [ ] Send a **seed test** to yourself and open it in **Gmail**, **Outlook desktop**, and **Apple Mail (dark mode)** — the three that break email most.
- [ ] Subject line + preview text read well in the inbox list.
- [ ] All images have alt text and the email is readable with images off.

## The `email/VisitUs-EMAIL.html` template
That thank-you template is a **draft, not send-ready**. Before using it:
- [ ] Replace the placeholder image host (`https://REPLACE-WITH-YOUR-IMAGE-URL/`) on all images with a real host.
- [ ] Replace the `{{Handlebars}}` merge tags (`{{UnsubscribeLink}}`, `{{DonateLink}}`, etc.) with your ESP's real tags — Mailchimp will **not** resolve `{{…}}` syntax.
- [ ] Confirm the footer postal address, and either wire up or remove the "Save your QR code" promise (the QR slot is currently commented out).
