# Membership store: builder's brief

Read this first. It covers the code, how to change it safely, and what must never break.

| File | What's in it |
|---|---|
| `README.md` | Operating and deploying it |
| `docs/DECISIONS.md` | Why it is built this way |
| `docs/DATABASE.md` | Schema and the live Supabase setup |
| `docs/BACKLOG.md` | What's next, cut on purpose, or unverified |

## What this is

A membership store for a small distribution business in the San Diego / Baja California area.

- **Visitors** see public products and public prices.
- **Members** sign up, pay a one-time fee by bank transfer, and get approved by staff. Then they see members-only products at member prices and order.
- **Orders** are paid on delivery, in cash or by transfer to the driver.
- **Staff** run everything from `/admin`: approvals, orders, a driver message to copy or open in WhatsApp, products with photos, transfer receipts, and CSV exports for Excel.

## Stack

- `app.py`: the whole app, about 1,000 lines, Python 3.12+.
- **Dependencies:** `psycopg[binary]` (Postgres) and `tzdata`. Nothing else server-side.
- **Hosting:** Vercel runs the WSGI callable `app` at the bottom of `app.py`; `pyproject.toml` sets `tool.vercel.entrypoint = "app:app"`. Locally, `python3 app.py` runs the same handler on `ThreadingHTTPServer`.
- **Database:** Postgres on Supabase, project **conectando-ensenada**, schema `store`, login role `store_app` (see `docs/DATABASE.md`). Any Postgres works.
- **Browser:** plain server-rendered HTML forms, with one small script served at `/app.js`. It handles Copy buttons, shrinks photos before upload, and runs the delivery map.
- **Map:** Leaflet 1.9.4 from unpkg, pinned with an SRI hash. Map tiles come from tile.openstreetmap.org, and addresses from Nominatim (OpenStreetMap). No API keys.

## Code map (`app.py`, top to bottom)

1. **Config.** Environment variables: `STORE_NAME`, `STORE_TZ`, `RECEIPT_DAYS`, `MAP_CENTER`, `COOKIE_SECURE`, `MEMBERSHIP_FEE`, `PAYMENT_DETAILS`, `DATABASE_URL`, `HOST`/`PORT` (local only). `STATUSES` and `PAYMENTS` are the allowed order states. `LEAFLET` holds the pinned map tags.
2. **`SCHEMA`.** Idempotent DDL (`CREATE … IF NOT EXISTS`, `ADD COLUMN IF NOT EXISTS`), ending with row-level security on for every table. `python3 app.py setup` runs it, and `schema.sql` is generated from it.
3. **Data and business rules.** These are pure functions over a `db` connection:
   - `db`: a small wrapper around psycopg. One connection per request. `with c:` commits, or rolls back on error. Placeholders are written `?`.
   - `create_user`, `is_member`, `visible_products`
   - `place_order`: decides prices and visibility server-side; stock is taken atomically.
   - `update_order`: cancelling returns stock; "delivered" needs a recorded payment.
   - `save_image`, `expire_receipts`
   - `dispatch_text`, `map_address`, `maps_link`, `wa_link`: build the driver message.
   - `order_rows`
4. **HTML helpers.** `CSS`, `note`, `select`, `field`.
5. **`class H`.** Request handling: routing, session, CSRF check, body size limit, security headers (`send`), page shell (`page`), `login_as`.
6. **Customer pages.** `APP_JS`/`app_js`, `image_input`, `image` (serves `/img/<id>` with access checks), `home` (photo grid and order form with map), `order`, `register*`, `login*`, `logout`, `account`.
7. **Staff pages.**
   - Shared pieces: `orders_table`, `approve_form`, `members_table`
   - Dashboard and orders: `admin_dashboard`, `admin_orders`, `admin_order`, `receipt_section`, `admin_receipt`, `admin_order_update`, `admin_whatsapp`
   - Members: `admin_accounts`, `admin_account_create`, `admin_account`, `admin_account_update`
   - Products: `product_form`, `admin_products`, `admin_product`, `admin_product_save`
   - Export: `EXPORTS`, `excel_safe`, `admin_export`, `admin_export_csv`
   - Drivers and audit log: `admin_drivers`, `admin_driver_save`, `admin_audit`
8. **`ROUTES`.** `(method, regex, handler, access)`, where access is `None`, `"user"` or `"staff"`. The router enforces access before the handler runs.
9. **`app(environ, start_response)`.** The Vercel entry point; it drives `H` without a socket. Below it is the CLI: `setup`, `add-staff`, and serve.

## Run and test

```bash
pip install "psycopg[binary]" tzdata
DATABASE_URL=postgresql://… python3 app.py setup      # create/upgrade tables
DATABASE_URL=postgresql://… python3 app.py add-staff you@x.com "Name" "+1 …"
DATABASE_URL=postgresql://… python3 app.py            # http://127.0.0.1:8000
TEST_DATABASE_URL=postgresql://postgres@localhost/postgres python3 -m unittest test_app
```

**Tests.** `test_app.py` has 20 tests. They create a throwaway database on the `TEST_DATABASE_URL` server and drop it afterwards, so never point them at a server where you can't create databases. They cover:
- business rules
- every HTTP page
- the Vercel entry point
- CSRF and staff-only access
- image visibility
- receipt expiry
- CSV safety
- the map form

**Disposable local Postgres** (Ubuntu paths; initdb refuses to run as root):
```bash
B=/usr/lib/postgresql/16/bin; D=~/.pgtest
$B/initdb -D $D/data -A trust -U postgres && $B/pg_ctl -D $D/data -o "-p 54329 -k $D" -l $D/log start
TEST_DATABASE_URL="postgresql://postgres@/postgres?host=$D&port=54329" python3 -m unittest test_app
```

**Browser checks.** For anything touching `APP_JS`, test in a real browser (Playwright works). Under the page's CSP:
- string predicates like `page.wait_for_function("…")` are blocked; wait on elements instead (`locator(...).wait_for()`).
- with no internet, serve unpkg from `npm pack leaflet@1.9.4` (same bytes, so SRI still passes), and stub the tiles and Nominatim with `context.route`.

## Rules that must not break

- **Server decides access.** Prices, members-only products and their photos, receipts, and admin pages are all checked on the server for every request. Never rely on hiding something in HTML.
- **Only staff approval grants membership.** Registering, depositing or paying never does. `is_member()` is `biz_status == 'approved'`.
- **Every POST from a logged-in session carries the CSRF token** (`h.csrf()` in every form). The router rejects it otherwise.
- **Escape all output** with `e()`, which is `html.escape`. Use double-quoted attributes.
- **Money is integer cents.** Use `cents()` to parse and `money()` to show. Never use floats for money.
- **Stock can't go negative.** There's a DB `CHECK`, and `place_order` uses a conditional `UPDATE … WHERE stock>=?` inside one transaction.
- **Audit everything.** Every staff action and order change writes to `audit` via `log()`. Keep this for new features. Order-related details start with `#<order id> ` and account-related ones with `APP-<id>`; the order and member pages filter on those prefixes.
- **Uploads.** Images arrive as `data:` URLs (the browser has already shrunk them). `save_image` checks the type, the magic bytes and a 3 MB cap. Bodies over 100 KB are refused unless the sender is staff (4 MB).
- **Exports.** Never export password hashes, sessions or image bytes. Pass every cell through `excel_safe`.
- **CSP lives in `H.send`.** If you add an outside resource, add exactly its host there, and pin scripts with an SRI hash.
- **Never commit secrets.** `DATABASE_URL` lives only in Vercel's environment variables.

## Gotchas

- **Python 3.12+ is required** for the nested f-strings with reused quotes (`pyproject.toml` says so).
- **Write SQL with `?`.** The `db` wrapper turns `?` into `%s`, so never put a literal `?` in SQL. Pass `None` (not `()`) when there are no parameters, so `%` isn't parsed.
- **Use `WHERE true`, not `WHERE 1`.** That's Postgres.
- **`prepare_threshold=None`** is required, because the Supabase/Neon transaction poolers can't hold prepared statements.
- **Rows are dicts** (`dict_row`). Use `row["col"]`, never `row[0]`. For counts, alias them: `count(*) AS n`.
- **No `lastrowid`.** Use `INSERT … RETURNING id` and `.fetchone()["id"]`.
- **Vercel has no persistent disk.** All state must live in Postgres: sessions, images, everything.
- **Cleanup piggybacks on page loads.** There's no scheduler: `expire_receipts` runs when staff open `/admin` or `/admin/export`.
- **`APP_JS` is a Python string.** Escape backslashes in JS regexes (`\\.`), or Python warns.
- **Times are Unix seconds** (`BIGINT`), shown in `STORE_TZ` via `when()`.

## How to change things

- **New column or table.**
  1. Add an idempotent statement to `SCHEMA`, and add new tables to the RLS list.
  2. Regenerate `schema.sql`:
     ```bash
     python3 -c "import os,sys;os.environ['DATABASE_URL']='x';sys.path.insert(0,'.');import app;open('schema.sql','w').write('-- Run once …\n'+app.SCHEMA)"
     ```
  3. Apply it to the live database the way `docs/DATABASE.md` shows: as `store_app`, with `SET ROLE store_app; SET search_path = store;`.
  4. Add a test.
- **New page.** Write a handler `def x(h, *groups)`, render it with `h.page(title, body)`, and add a `ROUTES` entry with the right access level. If it's a staff page, add it to the nav in `H.page`.
- **New order state or payment type.** Update `STATUSES`/`PAYMENTS`, check `update_order` and `dispatch_text`, and add a test.
- After any change, run the tests. If it's visual, take a screenshot at 1280 px and 390 px (phone) wide.

## Working style used so far

Smallest change that fully solves the need ("ponytail" style). Push back on features the need doesn't justify, and say what was skipped. Never cut security, validation, accessibility or audit logging. New logic gets a test. A known shortcut gets a code comment of the form `shortcut: <limit>, <when to upgrade>`.
