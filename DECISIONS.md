# Decisions

This file records why the platform is built the way it is. Decisions are in the order they were made, all on 2026-10-24 unless dated. When one changes, add a new entry rather than editing history.

## Business model (the owner's choices)

1. **One company, one app, three areas.** The public shop, the member area and the staff admin share one app and one database, separated by permissions, not by separate sites.
2. **Membership is the gate to ordering.**
   - Anyone can register and log in. Until approved, they see only how to pay the membership.
   - The fee is **one-time**, paid by bank transfer to the account in `PAYMENT_DETAILS`, with their `APP-<id>` reference in the memo.
   - Staff approve manually, after the deposit arrives, by typing the bank's deposit reference. That reference goes into the audit log.
   - Staff can also create logins directly, for members who paid in person and for staff.
3. **Products have two visibilities.**
   - **Public:** shown to everyone at the *public price*.
   - **Members only:** hidden from everyone except approved members and staff.
   - Members see everything at the *member price*, with the public price crossed out.
   - *Open question:* should some public products be hidden from members? That would be a three-way choice; see the backlog.
4. **Payment happens at delivery.**
   - Every order starts as *due on delivery*, and the amount to collect equals the total.
   - The customer pays the driver in cash or by transfer. Staff record which.
   - An order can't be marked *delivered* until the payment is recorded.
5. **Drivers have no accounts.** Staff send each delivery to the driver over WhatsApp, either with the **Copy** button or the **Open WhatsApp** button, which opens a prefilled `wa.me` link and logs the handoff.
   - The WhatsApp Business API was cut: it costs money and setup, and the manual flow covers the need.
   - When the driver forwards the customer's transfer screenshot, staff attach it to the order as a *transfer receipt*.
6. **The business is in the US, near the border.** Delivery addresses may be in the US or Mexico. "SPEI" in early notes just meant "bank transfer", not a Mexican-only payment system.

## Technical

7. **One Python file, two dependencies.** The need is a small operation, so it's one readable file with no framework, and no layers for later. Standard library plus `psycopg` and `tzdata`.
8. **The server enforces everything.** Prices, visibility, access to images and receipts, and admin pages are all checked on the server on every request. This was the blueprint's main engineering rule.
9. **Hosting: GitHub → Vercel**, chosen by the owner.
   - Vercel has no persistent disk, so the original SQLite version was moved to Postgres.
   - The app exposes a WSGI `app`, the current zero-config Python entry point on Vercel, and drives the same handler class used locally.
10. **Database: Supabase, inside the existing project conectando-ensenada.**
    - The free plan allows 2 projects and both were in use, so a new project wasn't possible. Neon via Vercel was considered and set aside once the owner chose this.
    - To share the project safely, the tables live in their own schema `store`, owned by their own login role `store_app`. That role can't read the other app's tables, and Supabase's public API roles (anon, authenticated) can't reach `store`.
    - Row-level security is on with no policies, as a second lock; the app connects as the owner, which RLS doesn't restrict.
    - The role's password was set as a SCRAM hash, so plaintext never went into migration history.
11. **Images are stored in Postgres (`bytea`).**
    - This avoids a second service, keys and bucket setup.
    - The browser shrinks photos to 1600 px JPEG before upload: a 3,000-pixel phone photo is stored at about 30 KB. That also keeps uploads under Vercel's request size limit.
12. **Space is managed automatically** (owner's request).
    - Each product keeps only its current photo.
    - Each order keeps only its latest receipt, and receipts are deleted `RECEIPT_DAYS` (90) after upload.
    - Cleanup runs when staff open the dashboard or export page, so there's no scheduler.
    - Orders, payments and the audit log are never deleted.
13. **Excel export as CSV**, so the owner can keep raw copies.
    - Files carry a UTF-8 byte-order mark so Excel reads accents correctly.
    - Cells starting with `= + - @` are neutralised (CSV injection), and phone numbers are written as `="+1 …"` so they show as typed.
    - Exports never include password hashes, sessions or image bytes. Each download is logged.
14. **Delivery map: OpenStreetMap instead of Google** (owner's choice).
    - It's free: no API key, card or Google Cloud account.
    - The driver still gets a **Google Maps link** to the exact pin.
    - Leaflet loads from unpkg, pinned to 1.9.4 with an SRI hash. Address lookup uses Nominatim, debounced to one lookup per pin placement.
    - It's not limited to the US. Mexican addresses include the colonia ("Col. …"), with the street before the number. US addresses include the neighborhood.
15. **Typed address and map address are both kept** (owner's request).
    - The address box is the member's own words. The pin fills it in only while it's empty or unchanged.
    - The map's address is shown under the map and saved separately as `pin_address`.
    - When the two differ, the member, staff and driver all see both.
16. **Security baseline.**
    - scrypt password hashes; sessions stored hashed; HttpOnly, SameSite=Lax cookies (Secure automatically on Vercel); a CSRF token on every logged-in form.
    - A strict CSP that allows only the map hosts; `nosniff`; and a 100 KB body limit, raised to 4 MB for staff uploads.

## Compliance (flagged, not solved by software)

17. **Regulated products.** The original blueprint was for importing vapes for US sale. That's heavily regulated (FDA authorization, PACT Act, state and local rules), and a membership doesn't change it.
    - The software only controls who sees and orders what.
    - Mark regulated items **Members only**, add them only after legal review, and check ID (and the license, for businesses) before approving members.
