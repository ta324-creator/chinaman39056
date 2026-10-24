# Backlog

Organized by how urgent it is. Move an item to `docs/DECISIONS.md` once it's decided.

## Verify after the first live deploy (never tested against the real services)

The build environment couldn't reach the internet. Everything below was tested locally against Postgres 16 and a real browser with stand-ins, but not live.

- [ ] **Vercel can connect to Supabase as `store_app` through the transaction pooler** (user `store_app.<project ref>`, port 6543). If that fails, Vercel's **Logs** tab shows the error.
- [ ] **Real map tiles and Nominatim address lookups load on the live site, on a phone.** "Use my current location" needs HTTPS, which Vercel provides.
- [ ] **Copy buttons work on the live HTTPS site.** The clipboard needs HTTPS; the fallback opens the preview to copy by hand.
- [ ] **A photo from an iPhone uploads correctly.** Safari normally converts HEIC to JPEG; desktop Chrome can't read HEIC files and says so.

## Do soon (security housekeeping)

- [ ] **Rotate both passwords that were shared in the build chat:** the `store_app` database password and the first staff login. See `docs/DATABASE.md`; staff change their own under **Members → their name**.
- [ ] **Add login rate limiting.** Login has none yet: either a per-email and per-IP attempt counter in the database, or Vercel's firewall rules.
- [ ] **Unrelated to the store:** Supabase's advisor flagged the other app in conectando-ensenada. Several `SECURITY DEFINER` functions can be called by the public API roles, three extensions are installed in `public`, and leaked-password protection is off.

## Cut on purpose (add when a real need shows up)

| Item | What happens today |
|---|---|
| Online card payment | Payment is on delivery, or by manual deposit for the membership. Needs a processor that approves these products in writing. |
| Membership renewal or expiry | The fee is one-time. |
| Price tiers or customer-specific prices | One public price and one member price per product. |
| Inventory movement ledger | Stock is set as a number. *shortcut:* editing stock overwrites any order placed while the form was open; add a "received +N" field if that bites. |
| Driver cash reconciliation report | Export orders and filter by driver and payment in Excel. |
| WhatsApp Business API (automatic sending, delivery receipts) | A prefilled `wa.me` link plus Copy; staff press Send. |
| License or ID document uploads | Staff check documents outside the app before approving. |
| Reports and dashboards beyond counts | Use the CSV exports. |
| Self-service password reset by email | Staff set new passwords from the member's page. Needs an email provider. |
| Typed address → map pin (forward geocoding) | Members place the pin themselves; the typed address is kept regardless. |

## Open questions for the owner

- [ ] **Should some public products be hidden from members?** For example, single units shown only to visitors while members see the bulk pack. This would be a three-way visibility per product: everyone, public only, or members only.
- [ ] **Should members see their own transfer receipt?** Today receipts are staff-only, because they can show bank details.
- [ ] **Should the site be in Spanish, or in both languages?** Today all text is in English.
- [ ] **Should staff accounts have limited roles** (for example, a dispatcher who can't approve members)? Today every staff login has full admin access.

## Known risks

- **Legal.** Selling vapes or other regulated products needs FDA authorization for each product, plus PACT Act and state and local compliance, whatever the membership model. The software only controls visibility. See `docs/DECISIONS.md` #17.
- **Receipts expire after 90 days.** Save any that matter before then. The orders export only records whether a receipt was on file.
- **Nominatim is a free service meant for light use.** At high volume, move address lookup to a paid provider (only the `fetch` in `APP_JS` and the `connect-src` in the CSP change).
- **Addresses come from OpenStreetMap.** Coverage of colonias varies, so members should check the address before ordering.
- **One shared Supabase project.** If the other app's database work ever drops or recreates schemas carelessly, `store` could be affected. Export regularly.
