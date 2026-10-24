# GSTechSys Sales App
**Full System Documentation — Structure, Business Logic, Functional Reference, Database Schema & Reports**

---

## 1. Overview

GSTechSys is a desktop Point-of-Sale / Sales Management application for a **general-items retail / distribution business** — electronics, electrical appliances, hardware, sanitary ware, computer and mobile accessories, solar panels, batteries, inverters, wires & cables, breakers, and similar goods. (It began life branded for a cosmetics shop and was generalised; the application structure was always domain-neutral — only the seeded category/unit catalogue is domain-specific, and it is fully editable in-app.) It manages the full sales lifecycle: inventory, purchasing, customer accounts, sales invoicing, credit/outstanding payment collection, returns, and management reporting — with multi-company (branch) support, role-based access control, a 30-day licensing trial, a trilingual (English / Arabic / Urdu) interface, and printable A4/POS documents.

The system offers **two ways to sell**: the detailed **Sales Invoice** screen (full control over discounts, schemes, tax, notes, print format) and a touch-friendly **Quick Sale (POS)** screen (product tiles with images, barcode scanning, a running cart, delivery charges, cash-tendered/change calculation, park-and-resume, and an 80 mm thermal voucher). Both write to the same invoice/stock/customer-balance data, so a sale made either way is identical in the books and reports.

### 1.1 Technology Stack

| Layer | Technology |
|---|---|
| UI Framework | PyQt5 (desktop, Windows) |
| Database | SQLite (file-based, `sqlite3` module, `Row` factory) |
| PDF Generation | ReportLab (`SimpleDocTemplate`, custom `Canvas` subclasses for watermarks) |
| PDF Inspection/Testing | PyMuPDF (`fitz`) — used during development to visually verify PDFs |
| Encryption | `cryptography.Fernet` (symmetric encryption for company contact data) |
| Thermal Printing | `pywin32` (`win32print`) for raw POS receipt printing, with text-file fallback |

### 1.2 Entry Point & Startup Sequence

`main.py` is the application entry point:
1. Adds the project root to `sys.path`.
2. Patches `QComboBox`/`QCompleter` popups with a custom delegate to fix a Windows/Fusion-style white-on-white text rendering bug.
3. Applies a global QSS stylesheet (dark-blue `#1a3a5c` primary, orange `#e8a020` accent, plus semantic button colors: `#btnSuccess` green, `#btnDanger` red, `#btnWarning` orange, `#btnSecondary` gray).
4. Runs `database.migrate_old_database()`, `database.init_db()`, and `database.migrate_encrypt_companies()` — schema creation, column migrations, and one-time data reconciliation all happen here, idempotently, on every launch (see §5.9).
5. Installs `sys.excepthook` to log all unhandled exceptions via `app_logger.log_unhandled`.
6. Ensures a default logo exists, spawns a background daemon thread for scheduled auto-backup, then shows `LoginWindow` and enters the Qt event loop.

---

## 2. Project Structure

```
gstechsys/
├── main.py                  # App entry point, global stylesheet, auto-backup launcher
├── database.py               # All schema, migrations, and data-access functions (single data-access layer)
├── config.py                  # Fernet ENCRYPTION_KEY for company data encryption
├── assets/
│   ├── logo.png / logo.ico    # Default branding
│   ├── logos/                 # Per-company uploaded logos
│   ├── item_images/           # Product images shown on POS tiles (see §4.1 / §5.21)
│   └── help/                  # Auto-captured screenshots for the in-app Help screen
├── backups/                   # Auto/manual SQLite backups (created at runtime)
└── modules/
    ├── login.py                # Authentication screen
    ├── dashboard.py             # Main shell: header, sidebar, page routing, role-based nav
    ├── items.py                 # Item / Stock master CRUD
    ├── categories.py            # Item category CRUD
    ├── units.py                 # Unit-of-measure CRUD
    ├── suppliers.py              # Supplier master CRUD
    ├── customers.py              # Customer master CRUD + balance display
    ├── purchase.py               # Stock-in / purchase entry
    ├── invoice.py                 # Sales Invoice screen (the core transaction screen)
    ├── pos_quick_sale.py          # Quick Sale (POS) — touch terminal, cart, tiles, voucher
    ├── business_dashboard.py      # Dashboard — KPI cards, profit, top items, last-7-days
    ├── outstanding_payment.py     # Outstanding Payment collection form (sidebar + invoice-screen dialog)
    ├── returns.py                  # Sales return processing
    ├── schemes.py                  # Buy-X-get-Y-free promotional scheme CRUD
    ├── reports.py                   # All reporting tabs
    ├── users.py                      # User account / role management (admin)
    ├── company.py                    # Multi-company (branch) management (admin)
    ├── settings.py                    # Global application settings (admin)
    ├── backup_restore.py              # Backup/restore + activity log viewer (admin)
    ├── import_data.py                  # Bulk CSV import for master data
    ├── help_window.py                   # Bilingual (English/Urdu) in-app user guide
    ├── pos_preview.py                    # POS 3-inch thermal receipt preview/print dialog
    ├── app_logger.py                      # Centralized activity logging (never raises)
    ├── i18n.py                             # Trilingual engine (EN/AR/UR) + widget-tree translator
    ├── license_manager.py                 # 30-day trial state, key generation & activation
    ├── license_dialog.py                  # Trial-reminder / trial-expired / activation dialogs
    └── utils.py                            # PDF generation, currency formatting, encryption helpers
```

---

## 3. Roles & Access Control

The system has three roles, enforced **both** at the UI level (sidebar items not created/buttons hidden) and, for the most sensitive screens, again at the widget level (`is_admin(user)` checks that disable the whole screen) — defense in depth.

| Capability | Staff | Manager | Admin |
|---|:---:|:---:|:---:|
| Items / Stock (view) | ✅ | ✅ | ✅ |
| Items / Stock (add) | ✅ | ✅ | ✅ |
| Items / Stock (edit/delete) | ❌ | ✅ | ✅ |
| Categories & Units management | ❌ | ✅ | ✅ |
| Suppliers, Purchase, Customers | ✅ | ✅ | ✅ |
| Quick Sale (POS) | ✅ | ✅ | ✅ |
| Dashboard (sales/profit overview) | ❌ | ✅ | ✅ |
| Sales Invoice (create) | ✅ | ✅ | ✅ |
| Sales Invoice (**edit** a saved invoice) | ❌ | ✅ | ✅ |
| Outstanding Payments | ✅ | ✅ | ✅ |
| Returns | ✅ | ✅ | ✅ |
| Schemes (add/edit/delete) | view-only | ✅ | ✅ |
| Reports (all tabs) | ❌ | ✅ | ✅ |
| User Management | ❌ | ❌ | ✅ |
| Company (multi-branch) management | ❌ | ❌ | ✅ |
| Backup & Restore | ❌ | ❌ | ✅ |
| Import Data | ❌ | ❌ | ✅ |
| Settings | ❌ | ❌ | ✅ |
| Help / مدد | ✅ | ✅ | ✅ |

Sidebar visibility is driven by `ROLE_NAV` in `dashboard.py`, a dict of `role → [allowed page keys]`, cross-referenced against `ALL_NAV` (the master ordered list of `(key, icon+label)` pairs). Every sidebar item carries a distinct icon (🧮 Quick Sale (POS), 📦 Items, 🏷️ Categories, 📏 Units, 🚚 Suppliers, 🛒 Purchase, 👥 Customers, 🧾 Sales Invoice, 💳 Outstanding Payments, ↩️ Returns, 🎁 Schemes, 📊 Reports, 👤 Users, 🏢 Company, 💾 Backup & Restore, 📥 Import Data, ⚙ Settings, ❓ Help). **Quick Sale (POS)** is listed first for every role and is the default landing screen after login.

Authentication (`login.py`) is a plain username/password form; passwords are stored as SHA-256 hashes (`users.password_hash`). There is no lockout/rate-limiting.

---

## 4. Database Schema

SQLite, single file, accessed via `database.get_connection()` (`PRAGMA foreign_keys = ON` on every connection). All tables are created idempotently in `database.init_db()`, with `ALTER TABLE ... ADD COLUMN` migrations for columns added after the original release, followed by data-reconciliation passes (see §5.9).

### 4.1 Master / Reference Data

**`companies`** — multi-branch support; all contact fields are Fernet-encrypted at rest.
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| name, phone, email, website, address, city, country | TEXT | **encrypted** via `utils.encrypt_field()` |
| logo_path | TEXT | relative or absolute path under `assets/logos/` |
| is_active | INTEGER | exactly one company has `is_active=1` at a time |
| created_at | TIMESTAMP | |

**`users`**
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| username | TEXT UNIQUE | |
| password_hash | TEXT | SHA-256 |
| role | TEXT | `staff` \| `manager` \| `admin`, default `staff` |
| created_at | DATETIME | |

**`categories`** — `id`, `name`. Seeded from `database.DEFAULT_CATEGORIES` (general-items catalogue): General Items, Electronics, Electric Appliances, Hardware, Sanitary, Computer Accessories, Mobile Phone Accessories, Solar Panels, Batteries, Inverters, Circuit Breakers & Safety Devices, Wires & Cables, Conduit (Flexible Tubing), Screws, Clamps / Fasteners. Seeding runs once on a fresh database; the list is fully editable from the Categories screen.

**`units`** — `id`, `name UNIQUE`. Seeded from `database.DEFAULT_UNITS`: pcs, box, carton, dozen, set, pair, pack, meter, foot, roll, coil, kg, ltr. This is a **managed pick-list only** — `items.unit` remains a plain TEXT column (not a foreign key), so the Units screen exists to keep the dropdown curated and prevent typos, not to enforce referential integrity.

**`suppliers`** — `id`, `name`, `phone`, `email`, `address`, `created_at`.

**`customers`** — `id`, `name`, `phone`, `email`, `address`, `city`, **`balance`** (running outstanding/credit amount, updated by invoices and payments), `created_at`.

**`schemes`** — `id`, `label` (e.g. `"6/1"`), `buy_qty`, `free_qty`. Seeded: 6/1, 6/2, 12/1, 12/2, 12/3.

**`items`**
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| name | TEXT | |
| category_id | INTEGER FK → categories | |
| unit | TEXT | default `'pcs'`, free text (see Units note above) |
| barcode | TEXT | used by barcode-scanner input on Invoice/Purchase |
| purchase_price, sale_price | REAL | |
| stock_qty | REAL | adjusted by Purchase (+), Sales Invoice (−), Returns (+) |
| reorder_level | REAL | drives the "Low Stock" indicator in the Items screen |
| supplier_id | INTEGER FK → suppliers | |
| brand | TEXT | manufacturer/brand (e.g. Philips, Jinko) — added in the general-items migration |
| model_no | TEXT | manufacturer model / part number |
| specifications | TEXT | free text: watts, volts, Ah, mm², length |
| warranty_months | INTEGER default 0 | printed on invoice/POS lines when > 0; warranty-expiry = invoice date + months |
| sku | TEXT | optional internal code, separate from barcode |
| is_active | INTEGER default 1 | `0` hides a discontinued item from sale search (not deleted) |
| image_path | TEXT | relative path under `assets/item_images/` to the product image shown on POS tiles; empty → a 2-letter colored placeholder is drawn instead |
| created_at | DATETIME | |

> **General-items columns** (and `image_path`) are added idempotently in `init_db()` via `ALTER TABLE items ADD COLUMN` (nullable, safe defaults), and a one-time `backups/premigrate-items_general_cols-*.db` safety copy is taken before they are first introduced on an existing populated database. Item search on the Invoice/Purchase screens matches name, barcode, SKU, brand and model (`database.find_item`). Quantities of `meter`/`foot`/`kg`/`ltr` items accept decimals (`database.qty_decimals`/`format_qty`, places from `qty_decimal_places`).

> **Product images (`image_path`).** Attach an image from the Items Add/Edit dialog (**Browse Image…**); the file is copied into `assets/item_images/` (next to the `.exe` when frozen, so it survives upgrades) and the relative path is stored. The image appears on that item's **Quick Sale (POS)** tile. **Recommended format:** a **square PNG (or JPG), 256 × 256 px** (anything square from 128–512 px works), on a plain/transparent background, kept under ~200 KB. It is displayed at roughly 60 × 60 px on a tile and 64 × 64 px in the dialog preview, scaled with aspect ratio preserved, so a square image never distorts. Items without an image show a colored 2-letter placeholder, so images are optional.

### 4.2 Transactional Data

**`purchase_orders`** / **`purchase_items`** — stock-in records.
- `purchase_orders`: id, supplier_id, date, total_amount, notes, company_id, created_at.
- `purchase_items`: id, purchase_order_id, item_id, qty, price, total. On save, `items.stock_qty += qty`.

**`invoices`** — the central sales transaction table.
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| invoice_no | TEXT UNIQUE | format `{prefix}{0000}`, e.g. `INV-0001`; prefix configurable in Settings |
| customer_id | INTEGER FK → customers | nullable (Walk-in/Guest) |
| date | TEXT | |
| payment_type | TEXT | `Cash` \| `Bank Transfer` \| `Credit` \| `Partial` |
| amount_paid | REAL | amount actually settled (kept current by `record_payment()` as further payments arrive) |
| total_before_discount, total_discount, net_total | REAL | `net_total` is the amount owed/collected — items net of discount **plus delivery charges** (tax is tracked separately in `tax_amount`) |
| delivery_charges | REAL default 0.0 | optional delivery/shipping fee entered on the POS screen; folded into `net_total` and printed as its own line on the voucher |
| notes | TEXT | optional, max 500 chars in the UI; prints in a gray box above totals |
| company_id | INTEGER FK → companies | which branch issued the invoice |
| tax_rate, tax_amount | REAL | **historical snapshot** at save time — never recalculated on reprint |
| previous_balance, total_due | REAL | **historical snapshot** of the customer's balance/total-due shown on the printed invoice at save time |
| amount_due | REAL | **derived/reconciled**: `max(net_total − amount_paid, 0)` |
| payment_status | TEXT | **derived/reconciled**: `Unpaid` \| `Partial` \| `Paid` |
| created_at | DATETIME | |

> **Snapshot principle:** `tax_rate`/`tax_amount`/`previous_balance`/`total_due` are frozen at the moment the invoice is saved and are **never** recalculated from live data when the invoice is reprinted later — this guarantees a reprinted invoice always matches what the customer originally received. By contrast, `amount_due`/`payment_status` are *intentionally* live-derived, because they represent the invoice's current settlement state as Outstanding Payments are recorded against it over time.

**`invoice_items`** — id, invoice_id, item_id, price, qty, disc_pct, scheme_id, disc_amt, amount, scheme_free_qty. Stock effect on save: `items.stock_qty -= (qty + scheme_free_qty)`.

**`held_sales`** — parked (suspended) Quick Sale carts, so a cashier can set a transaction aside and serve the next customer, then resume it later. **No stock or balance is touched while a sale is held** — holding only snapshots the in-progress cart as JSON; stock is deducted and balances move only when the resumed sale is finally completed as a real invoice.
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| label | TEXT | shown in the Resume list (e.g. the would-be invoice no + item count) |
| customer_id | INTEGER | optional |
| total | REAL | payable at hold time, shown in the Resume list |
| payload | TEXT (JSON) | the full cart: line items, discount, delivery |
| created_by | INTEGER FK → users | cashier who parked it |
| created_at | TIMESTAMP | |

Helpers: `save_held_sale()`, `list_held_sales()`, `get_held_sale()`, `delete_held_sale()`. Resuming a held sale loads its payload back into the terminal and deletes the held row.

**`returns`** / **`return_items`** — sales returns, always linked to an original invoice.
- `returns`: id, invoice_id, date, reason, total_amount, company_id, created_at.
- `return_items`: id, return_id, item_id, qty, price, amount. On save: `items.stock_qty += qty` (stock restored) and, for Credit/Partial invoices, `customers.balance` is reduced directly.
- ⚠️ **Known limitation:** unlike invoices and payments, returns currently adjust `customers.balance` directly and do **not** post a `customer_ledger` entry, so a return will not appear as a line item in the Customer Ledger report even though the balance itself updates correctly.

### 4.3 Outstanding Payment & Ledger System

**`payments`** — one row per payment *session* (a customer may pay against several invoices at once).
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| payment_no | TEXT UNIQUE | format `PAY-0000` |
| customer_id | INTEGER FK → customers | |
| payment_date | DATE | |
| payment_method | TEXT | `Cash` \| `Bank Transfer` \| `Cheque` \| `Online` |
| cheque_no, bank_ref | TEXT | conditionally shown/required in the UI by method |
| total_paid | REAL | |
| notes | TEXT | saved to the customer ledger as an informational entry |
| created_by | INTEGER FK → users | |
| created_at | TIMESTAMP | |

**`payment_invoices`** — one row per invoice settled within a payment session; the authoritative audit trail for §5.9's ledger rebuild.
- id, payment_id, invoice_id, amount_paid, **invoice_due_before**, **invoice_due_after** (frozen snapshot of that invoice's due amount immediately before/after this payment).

**`customer_ledger`** — running statement-of-account per customer; **rebuilt from scratch on every app startup** (see §5.9), not hand-maintained incrementally for old data.
| Column | Type | Notes |
|---|---|---|
| id | INTEGER PK | |
| customer_id | INTEGER FK → customers | |
| entry_date | DATE | |
| entry_type | TEXT | `Invoice` (debit) \| `Payment` (credit) \| `Note` (zero-effect, informational) \| `Return` (reserved, not currently posted — see limitation above) |
| reference_no | TEXT | invoice_no or payment_no |
| description | TEXT | |
| debit, credit | REAL | |
| balance | REAL | running balance, computed sequentially as each row is inserted |
| created_at | TIMESTAMP | |

### 4.4 Configuration

**`settings`** — generic key/value store; never deletes rows, only inserts-if-missing then updates.
| key | type | default | meaning |
|---|---|---|---|
| discount_enabled | bool | 1 | show discount field on Sales Invoice |
| scheme_enabled | bool | 1 | show scheme field on Sales Invoice |
| tax_enabled | bool | 0 | enable Tax/GST on invoices |
| tax_rate | float | 0.0 | tax percentage applied when enabled |
| show_previous_balance | bool | 1 | show the customer's previous-balance panel on the invoice screen |
| invoice_prefix | str | `INV-` | |
| next_invoice_number | int | 1 | auto-incremented; self-healing against existing data |
| default_payment_type | str | `Cash` | |
| currency_symbol | str | `Rs.` | |
| report_format | str | `A4` | `A4_HALF` \| `A4` \| `POS_3INCH` |
| app_language | str | `en` | UI language: `en` \| `ar` \| `ur` |
| print_language | str | `en` | Printed-document language, independent of the UI language |
| qty_decimal_places | int | `2` | Decimal places for meter/foot/kg/ltr quantities |
| show_warranty_on_print | bool | 1 | Print "Warranty: N months (until …)" on invoice/POS lines |
| show_brand_on_print | bool | 1 | Show item brand/model under the line on printed documents |
| pos_footer_line1 / pos_footer_line2 | str | "Thank you for your business!" / "Visit us again!" | |
| pos_printer_name | str | "" | last-selected thermal printer |

---

## 5. Module-by-Module Functional Reference

### 5.1 Login (`login.py`)
Branded sign-in screen (company logo/name from the active company, falls back to the default logo). Validates non-empty username/password, calls `db.authenticate_user()` (SHA-256 comparison), and on success opens `DashboardWindow` with the authenticated user object (`id`, `username`, `role`). Also hosts the **language selector** (English / العربية / اردو — changing it rebuilds the login in that language and persists the choice) and a **trial/license status strip**.

### 5.2 Dashboard Shell (`dashboard.py`)
Builds the header (company branding + role badge + username), the icon-labeled sidebar (role-filtered per §3), and a `QStackedWidget` holding every page widget (instantiated once at login, not recreated on navigation). Clicking a sidebar button switches the stack and, for several pages, triggers a refresh (`invoice` → `refresh()`, `outstanding` → `_refresh_customers()`, `purchase`/`items`/`customers`/`suppliers`/`users`/`company`/`backup` → reload their tables). Also wires cross-screen navigation requests (e.g. Reports' "Edit Invoice" button jumps to the Invoice screen and loads that record).

### 5.3 Items / Stock (`items.py`)
CRUD for the item master. Table: Name, Category, Unit, Barcode, Purchase Price, Sale Price, Stock Qty, Reorder Level, Supplier — rows at/below `reorder_level` are flagged with a "Low Stock" indicator. Search box filters by name/barcode. Add/Edit dialog pulls Category and Unit from their respective managed tables. **Add** is available to all roles; **Edit/Delete** require manager/admin.

### 5.4 Categories (`categories.py`) & Units (`units.py`)
Both manager/admin-only CRUD screens following the same pattern: table + Add/Edit/Delete buttons. **Delete is blocked with a clear message** if the category/unit is currently referenced by any item — for Categories this is enforced naturally by the `items.category_id` foreign key (SQLite raises `IntegrityError`, caught and surfaced as a friendly message); for Units (free-text, no FK) the check is done manually by counting items with a matching `unit` string before allowing deletion.

### 5.5 Suppliers (`suppliers.py`) & Customers (`customers.py`)
Parallel CRUD screens. Customers additionally displays a **Balance** column (amber-highlighted when > 0) reflecting `customers.balance`. View is available to all roles; Add to all roles; Edit/Delete to manager/admin.

### 5.6 Purchase (`purchase.py`)
Stock-in entry: pick a supplier (optional) and date, add line items via an autocomplete item search + qty + price, save. On save, each item's `stock_qty` is incremented and a `purchase_orders`/`purchase_items` record is created. Purchase History table below shows past purchases; clicking a row shows its line items.

### 5.7 Sales Invoice (`invoice.py`) — the core screen
This is the most complex screen in the system. (For the faster, counter-style alternative, see **§5.21 Quick Sale (POS)** — both produce the same invoices and stock movements.)

**Entry fields:** Customer (searchable combo, "Walk-in / Guest" default), Date, Payment Type (Cash / Bank Transfer / Credit / Partial), item entry row (barcode/name search → auto-fills price, qty, discount %, scheme), items table with inline merge-on-duplicate-item logic, **Notes** (multi-line, 500-char-capped, prints in a gray box above the totals), Print Format selector (A4-Half / A4-Full / POS, with an auto-suggested format hint based on row count).

**Previous Balance panel:** if `show_previous_balance` is enabled and the selected customer has a non-zero balance, an amber (debt) or green (credit) panel shows it, and the totals area shows a computed "Total Due" (this invoice's net total ± previous balance) — this value is **snapshotted** into `invoices.previous_balance`/`total_due` at save time.

**"💳 Outstanding Payment" button:** opens `OutstandingPaymentDialog` (modal), pre-selecting the currently chosen customer; when the dialog closes, the Previous Balance panel re-fetches the customer's live balance.

**Save logic:** for Cash/Bank Transfer, the invoice is marked `Paid` immediately (`amount_paid = net_total`, `amount_due = 0`); for Credit/Partial, `amount_due`/`payment_status` are computed and a debit entry is posted to `customer_ledger`. Stock is deducted per line item (`qty + scheme_free_qty`). If Notes is non-empty, a zero-effect `Note` ledger entry is also posted.

**Edit Invoice (manager/admin only):** `load_invoice_for_edit(invoice_id)` reloads a previously-saved invoice into the form (entry point: Reports → Sales Invoice tab → "Edit Invoice" button, staff cannot see this button). Saving an edit follows a **reverse-then-reapply** pattern in `database.update_invoice()`: the original stock/balance/ledger effects are fully reversed (including a ledger reversal entry), then the new values are reapplied — this is safe even if the customer or payment type changed between the original save and the edit.

### 5.8 Outstanding Payment (`outstanding_payment.py`)
Available both as a sidebar page (`OutstandingPaymentWidget`) and as a modal dialog from the Invoice screen (`OutstandingPaymentDialog`).

- Header: auto-generated Payment No (`PAY-0000`), Date, Customer, Payment Method (Cash/Bank Transfer/Cheque/Online — Cheque No and Bank/Ref No fields show conditionally), running "Total Outstanding" label.
- Invoice table: checkbox, Invoice No, Date, Net Total, Prev Paid, **Amount Due** (yellow-highlighted), **Pay Now** (editable, green-tinted flush cell matching the Amount Due style, auto-fills the full due amount when checked, turns red/error-styled if the typed amount is invalid or exceeds the due amount), Status (Unpaid=red / Partial=amber / Paid=green).
- Running totals panel: Invoices Selected / Total Being Paid (green) / Remaining After (amber), recalculated live as checkboxes/amounts change.
- Notes field (saved to the customer ledger as an informational entry).
- Buttons: **Save Payment**, **Print Receipt**, **Save & Print**, **Clear/Reset**.
- On save, `database.record_payment()` validates no invoice is overpaid, updates each invoice's `amount_paid`/`amount_due`/`payment_status`, writes `payment_invoices` audit rows, decrements `customers.balance`, and posts one consolidated `Payment` credit entry to `customer_ledger`.

### 5.9 Data Integrity: Startup Reconciliation
Because `amount_due`/`payment_status` (on invoices) and the entire `customer_ledger` table were both added after the application had already been in use, `database.init_db()` performs two **idempotent reconciliation passes on every startup**, so historical data is never stuck with stale defaults:
1. **`amount_due`/`payment_status` backfill** — recomputed for every invoice from `net_total`/`amount_paid` (always safe, since `amount_paid` is kept current by `record_payment()`).
2. **`rebuild_customer_ledger()`** (via `_reconcile_customer_ledgers()`) — for every customer, the *entire* ledger is deleted and rebuilt from authoritative source data (`invoices`, `payments`, `payment_invoices`), sorted by each record's true creation timestamp. Each invoice's original debit is derived algebraically as `amount_due_now + total_paid_via_outstanding_payments_for_that_invoice` — correct regardless of how many partial payments have since been applied. This guarantees the running balance shown in the Customer Ledger report and the "Previous Outstanding" figure on Payment Receipts is always accurate, even for invoices created before the ledger feature existed.

### 5.10 Returns (`returns.py`)
Search an invoice by number, see its line items with a return-qty spinner per row (capped at the original quantity), enter a reason, save. Restores stock (`+qty`) and reduces `customers.balance` for Credit/Partial invoices (see the ledger limitation noted in §4.2). Return ID format: `RET-0000`. "Print Return Note" generates a PDF via `generate_return_pdf()`.

### 5.11 Schemes (`schemes.py`)
Buy-X-get-Y-free promotional definitions (e.g. "12/2" = buy 12 get 2 free), selectable per line item on the Sales Invoice. View-only for staff; full CRUD for manager/admin.

### 5.12 Reports (`reports.py`)
See §6 — dedicated catalogue below.

### 5.13 User Management (`users.py`) — admin only
CRUD for accounts: Add User (username, password, confirm, role), Reset Password, Change Role, Delete User. Self-protection: a user cannot change their own role or delete their own account. Includes a read-only permission matrix explaining what each role can do.

### 5.14 Company (`company.py`) — admin only
Multi-branch/company management. Add/Edit/Delete companies, upload a logo (copied into `assets/logos/`), and "Set as Active" (exactly one company is active at a time — this is the one whose branding appears on the login screen, header, and all printed documents). All contact fields are encrypted at rest (§4.1) and transparently decrypted for display/edit.

### 5.15 Settings (`settings.py`) — admin only
Global configuration, organized into: Interface Language (English / Arabic / Urdu — takes full effect after re-login), Invoice Options (discount/scheme/tax toggles, previous-balance visibility), Invoice Numbering & Defaults (prefix, default payment type, currency symbol), Report & Print Format (A4-Half / A4 / POS-3inch radio choice, with a POS-specific sub-panel for footer lines), all backed by the `settings` table (§4.4). Saving emits a `settings_changed` signal so other open screens (e.g. the Invoice screen) can refresh immediately.

### 5.16 Backup & Restore (`backup_restore.py`) — admin only
- **Backup:** one-click "Backup Now" (writes to `backups/`) or "Save Copy to Folder..." (arbitrary location), using SQLite's native `backup()` API. Auto-backups are pruned to the most recent 30.
- **Schedule:** Daily / Weekly / Monthly auto-backup, checked at app startup.
- **Restore:** strongly confirmed, two-step safe operation — a safety backup of the *current* database is always taken automatically before overwriting it with the chosen backup file; requires an app restart afterward (offered inline).
- **Activity Log:** a live viewer over `app_activity.log` (last 500 lines, newest first), with Clear/Refresh.

### 5.17 Import Data (`import_data.py`) — admin only (Companies sub-target), others manager/admin
Bulk CSV import for Items, Customers, Suppliers, Schemes, and (admin-only) Companies. Workflow: pick a target table → download a CSV template (with example row) → browse/select a CSV → preview the first 20 rows → Import Now. Each importer validates required fields, de-duplicates against existing data (by barcode/name for items, by name+phone for customers/suppliers, by label for schemes, by name for companies), auto-creates missing categories/suppliers referenced by name, and produces a summary dialog (imported / skipped / errors) with an exportable log.

### 5.18 Help / مدد (`help_window.py`)
A bilingual (English + Urdu, RTL-aware) in-app user guide, with one topic per major screen, each showing an overview, numbered steps, and (where captured) a real screenshot. Topics are filtered by the current user's role. Admins additionally see a "Capture Screenshots" button that grabs live renders of each screen for the guide.

### 5.19 POS Preview (`pos_preview.py`)
A modal dialog simulating an 80mm thermal receipt, with a printer-selection combo (populated from installed Windows printers, remembered in Settings) and a "Print" button that sends raw text via `pywin32` (falling back to a plain text file if `pywin32` isn't available).

### 5.20 Cross-Cutting: Logging (`app_logger.py`)
A dependency-free logger (never raises) writing timestamped `[INFO]`/`[ERROR]` lines to `app_activity.log`, with 2 MB rotation (to `.log.bak`). Captures all unhandled exceptions app-wide via `sys.excepthook`, plus explicit logging of every backup/restore/schedule-change action.

### 5.21 Quick Sale (POS) (`pos_quick_sale.py`)
A touch-friendly, single-screen point-of-sale terminal — the fast counter alternative to the full Sales Invoice screen. Available to **all roles** and shown first in the sidebar. Implemented as `QuickSaleWidget`, a `QStackedWidget` with two pages (the sell **terminal** and the **payment** panel).

**Left — the terminal:**
- **Barcode box** ("Insert item barcode") — scan or type a barcode + Enter to add that item to the cart.
- **Cart table** — Name, Price, Qty, Total, Tax, and three per-row touch buttons: **+** (increase, green), **−** (decrease, amber), **✕** (remove, red). Adding the same item again merges into the existing row and bumps its quantity.
- **Totals box** (yellow) — Total, **Discount** (editable), Sub Total, TAX Rate, **Delivery Charges** (editable), **Total Payable** (red, = net + tax + delivery), **Pay Amount** (editable — cash tendered by the client), **Change / Return** (computed live = Pay Amount − Total Payable), Total Items Type.
- **Action buttons** — **Cash** (pay by cash/bank now), **Sales Credit** (put on the customer's account), **Suspend** (park the cart), and **Resume Held…** (reopen a parked cart). The live next invoice number is shown.

**Right — the product picker:**
- **Search Items** box + a scrollable grid of **product tiles** (one per active item) showing the item image (or a colored 2-letter placeholder), name, live Stock, and Price. Tapping a tile adds it to the cart. Tiles are tinted by category.
- **Category filter bar** along the bottom (All + one chip per category) narrows the grid.

**Stock guard & restock shortcut.** Adding an item beyond its available stock is blocked. When stock is **zero/insufficient**, a prompt appears: *"<item> — Available Stock: N … Do you want to increase the stock?"* Choosing **Yes** emits `request_add_stock(item_id)`, which the dashboard handles by opening the **Purchase** screen with that item **pre-selected** (`PurchaseWidget.preload_item`) — so the cashier can add stock in a couple of clicks and return to the sale.

**Tax.** Invoice-wide, taken live from Settings (`tax_enabled`/`tax_rate`) — the POS does not do per-line tax.

**Delivery charges.** Any amount typed into **Delivery Charges** is added to Total Payable, carried into the payment screen, stored on the invoice (`invoices.delivery_charges`), folded into `net_total`, and printed as its own **Delivery** line on the voucher.

**Suspend / Resume.** **Suspend** saves the current cart (items, discount, delivery) to `held_sales` and clears the terminal — no stock or balances move. **Resume Held…** lists parked sales (When / Label / Total) with **Resume** and **Delete**; resuming loads the cart back and removes the held row.

**Payment panel** (opened by Cash or Sales Credit). A compact grouped layout: **Paid** (prefilled from Pay Amount, else the full payable for Cash / 0 for Credit), live **Change** and **Due**, Customer, Pay by (Cash / Bank Transfer), Sales Date, and a Comment field, beside a quick-amount shortcut strip and a numeric keypad. Three icon buttons finish the sale: **← Back**, **💾 Only Save** (save without printing), and **🖨 Complete Sale and Print** (save + print the voucher).

**Finalize.** Builds the invoice lines and calls the same `database.save_invoice()` the Sales Invoice screen uses — so stock deduction, invoice numbering, customer balances, the ledger, and reporting all behave identically. **Sales Credit requires a selected customer** (not Walk-in) and posts to their account; **Cash** must be tendered at least the full payable. Printing calls `utils.generate_pos_receipt_pdf(invoice_id, employee=<cashier>, tendered=<paid>)` (see §7) and opens the PDF. After a completed or suspended sale the terminal resets for the next customer.

> **Signals:** `request_navigate(str)` and `request_add_stock(int)` are connected in `dashboard.py` to route to other screens (e.g. open Purchase pre-loaded for restock).

### 5.22 Dashboard (`business_dashboard.py`) — manager/admin
`BusinessDashboardWidget` — an at-a-glance overview, second in the sidebar (after Quick Sale) for manager/admin only (it shows profit). Contents:
- **Period KPI cards** — Today / This Week / This Month / This Year, each showing **Net Sales** and **Profit** for that window.
- **Headline figure cards** — Outstanding (total owed), Low-Stock Items, Total Items, Customers, Invoices (all time).
- **Last 7 Days** table — date, invoices, net sales, profit (green/red).
- **Top Items — This Month** — the best sellers by revenue.
- A **Refresh** button; the dashboard also refreshes automatically each time it's navigated to.

All figures come from the shared analytics helpers (`get_sales_summary`, `get_sales_by_period`, `get_sales_by_item_report`, `get_total_outstanding`, `get_low_stock_count`, `get_counts`), so the Dashboard, the Sales & Income report, and the Profit & Loss report always agree.

> **Analytics data layer (database.py).** `get_sales_summary(start,end)` returns the full figure set (gross, discount, net sales, delivery, tax, received, COGS, returns, gross/net profit). `get_sales_by_period(start,end,period)` groups by `day`/`week`/`month` via SQLite `strftime`. COGS = Σ(sold qty × item `purchase_price`); returns are netted out of both revenue and COGS. **Profit here is a gross trading profit** — the app tracks no operating expenses — and tax is treated as a pass-through, never income.

---

## 6. Reports Catalogue (`reports.py`)

All report tabs follow a consistent pattern: date-range filter (where applicable) → "Run Report" → results table → "Export PDF" (via `generate_report_pdf()`, auto-fits A4 or A4-Half per the global `report_format` setting). Manager/Admin only.

| Tab | Contents |
|---|---|
| **Sales Invoice** | Date-range + text search (invoice no / customer) over all invoices; "Print Invoice" re-renders the original PDF; "Edit Invoice" (manager/admin) jumps to the Invoice screen in edit mode. |
| **Daily Sales** | Single-date summary of all invoices issued that day. |
| **Sales & Income** | Sales broken down **Daily / Weekly / Monthly** (a Group-by selector) over a date range: per-period Invoices, Gross Sales, Discount, Net Sales, Delivery, COGS, and **Profit** (green/red). Backed by `database.get_sales_by_period()`. |
| **Profit & Loss** | A statement for a date range (with **This Month / This Year** quick buttons): Gross Sales → Discount → Net Sales → Returns → Net Revenue → COGS → **Gross Profit** → Delivery Income → **Net Profit**, plus memo lines (tax collected, amount received, invoice/return counts). Backed by `database.get_sales_summary()`. Gross trading profit only — operating expenses are not tracked. |
| **Stock Report** | Current stock levels, low-stock flagging. |
| **Customer Ledger** | Full statement-of-account from `customer_ledger` (Date, Type, Reference, Description, Debit, Credit, Balance), color-coded by entry type (Invoice=red, Payment=green, Return=blue, Note=gray), date-range filterable. |
| **Outstanding** | Per-customer rollup (Customer, Total Invoices, Total Billed, Total Paid, Balance Due) for every customer with `balance_due > 0`; double-click a row to drill into that customer's invoices on the Sales Invoice tab; grand-total row on PDF export. |
| **Outstanding Paid Invoice** | One row per invoice-level payment recorded through the Outstanding Payment form, sorted by invoice number (Invoice No, Invoice Date, Customer, Net Total, Amount Paid, Balance After, Payment No, Payment Date, Method); **"Print Paid Invoice"** button regenerates that payment's receipt PDF for the selected row. |
| **Sales by Item** | Item-level sales aggregation over a date range. |
| **Returns Report** | All processed returns over a date range. |

---

## 7. Printable / PDF Outputs (`utils.py`)

| Function | Produces |
|---|---|
| `generate_invoice_pdf()` | The Sales Invoice document. Auto-switches A4-Half ↔ A4-Full based on item-row count (`MAX_ROWS_A4_HALF = 10`); dynamically draws a diagonal **CREDIT / PARTIAL / PAID / PARTIAL-DUE** watermark + badge based on `_resolve_invoice_stamp()`, which reconciles the invoice's original `payment_type` with its *current* `payment_status` (current status wins when known — e.g. a Credit invoice later settled in full via Outstanding Payment reprints as a green "PAID IN FULL" stamp, never the original red CREDIT stamp). Notes (if present) render in a light-gray bordered box above the totals. |
| `generate_payment_receipt()` | The Outstanding Payment receipt (`PAY-0000.pdf`): info box, per-invoice payment table, Total Amount Paid / Previous Outstanding / Remaining Balance totals box, optional notes, signature footer. Auto-switches A4-Half ↔ A4-Full at `MAX_ROWS_RECEIPT_A4_HALF = 8` rows. |
| `generate_return_pdf()` | The Return Note for a processed sales return. |
| `generate_report_pdf()` | Generic tabular report renderer used by every Reports tab's "Export PDF". |
| `build_pos_receipt_text()` / `print_pos_receipt()` | Plain-text 32-char-wide thermal receipt body + raw printer send (with notes wrapped between dashed dividers). |
| `generate_pos_receipt_pdf(invoice_id, save_path=None, employee=None, tendered=None)` | The **80 mm thermal voucher** printed by Quick Sale (POS) — also viewable/printable without a physical thermal printer. Retail layout: company header (name / address / phone / city), `Invoice: <no> \|\| EMP: <cashier>`, `C.nam: <customer> \|\| Ph: <phone>`, timestamp, a **Name \| Qty \| Price \| Total** line table (blue header), then **SubTotal / Discount / Delivery (if any) / VAT <rate>% (if tax on) / Total** (bold) **/ <pay type> / Change / Due**, and the footer lines + company email. `tendered` drives the Change figure (`max(tendered − total, 0)`); a settled Cash/Bank sale shows the full amount paid and zero due. Print-language-aware (English/Arabic/Urdu). |

All PDF generation reads the **active company**'s branding (name, address, logo) and the global `currency_symbol`/format settings, so output is automatically correct for whichever branch/company is currently active.

---

## 8. Business Process Flows

**Procure → Stock → Sell → Collect → (maybe) Return**, expressed as the tables/ledgers each step touches:

1. **Purchase** (`purchase.py`) → `purchase_orders`/`purchase_items` inserted → `items.stock_qty` **increases**.
2. **Sell** — either via **Sales Invoice** (`invoice.py`) or **Quick Sale (POS)** (`pos_quick_sale.py`); both insert `invoices`/`invoice_items` → `items.stock_qty` **decreases** (qty + scheme free qty) → if Credit/Partial: `customers.balance` increases and a debit `customer_ledger` entry is posted; if Cash/Bank Transfer: settled immediately, no ledger entry. The POS path additionally supports delivery charges, cash-tendered/change, and park-and-resume (`held_sales`), and can jump straight to **Purchase** to restock an out-of-stock item mid-sale.
3. **Outstanding Payment** (`outstanding_payment.py`) → `payments`/`payment_invoices` inserted → the targeted invoices' `amount_paid`/`amount_due`/`payment_status` updated → `customers.balance` **decreases** → one consolidated credit `customer_ledger` entry posted.
4. **Returns** (`returns.py`) → `returns`/`return_items` inserted → `items.stock_qty` **increases** (restored) → `customers.balance` decreases for Credit/Partial invoices (ledger entry not currently posted — see §4.2 limitation).
5. **Edit Invoice** (manager/admin) → fully reverses step 2's effects (stock, balance, ledger) then reapplies with the edited values — safe across customer/payment-type changes.
6. **Reporting** (`reports.py`) reads the cumulative state of all the above; the **Customer Ledger** and **Outstanding** tabs are the canonical view of customer-level financial state, kept correct on every startup by the reconciliation pass in §5.9.

---

## 9. Security & Data Protection

- **Passwords:** SHA-256 hashed, never stored or displayed in plain text.
- **Company contact data:** encrypted at rest using a symmetric Fernet key (`config.ENCRYPTION_KEY`) — this key must be kept secret and is required to decrypt existing company records; it should never be committed to a public repository or rotated without a re-encryption migration.
- **Role enforcement:** sidebar-level (UI never offers the control) plus widget-level (`is_admin()`/role checks) for the most sensitive screens (Users, Company, Settings, Backup, Import-Companies, Invoice-Edit).
- **Backup safety:** every restore operation automatically creates a safety backup of the live database immediately before overwriting it, and all backup/restore actions are logged.

---

## 10. Licensing & 30-Day Free Trial

A trial gate runs at startup (wired in `main.py` after `init_db()`, before the login window). Implemented by `modules/license_manager.py` (logic) and `modules/license_dialog.py` (UI).

- **Trial period:** `TRIAL_DAYS = 30`. During the final `REMIND_WITHIN_DAYS = 7` days a reminder dialog appears at launch (still allows continuing). The login screen shows a status strip ("Free trial — N days remaining" / "Trial expired …" / "Licensed version").
- **Expiry:** once expired, `enforce_license()` shows a blocking **Trial Expired** screen with purchase/contact details — Website `www.GSTechSys.com`, Email `sales@gstechsys.com`, WhatsApp `+92 332 8228209` — and the app will not continue until a valid license key is activated.
- **Storage (tamper-resistant):** the install/last-seen dates are Fernet-encrypted (reusing `config.ENCRYPTION_KEY`) and written to **both** a hidden file `.gstechsys.lic` next to the database **and** an `app_license` row inside the database. The two are reconciled to the **earliest** install date, so deleting one does not reset the trial. A "last seen" date that never moves backwards guards against clock-rollback.
- **Activation:** a 16-character key `XXXX-XXXX-XXXX-XXXX` validated by HMAC-SHA256 against `_LICENSE_SECRET` embedded in `license_manager.py`. Activation marks the install permanently licensed (never expires). Generate keys with `python modules/license_manager.py gen [seed]`. Changing `_LICENSE_SECRET` invalidates all previously issued keys.
- **Shipping note:** the distributed database must **not** already contain an `app_license` row / `.gstechsys.lic`, or the customer's 30 days count from the build date. The trial auto-initialises on first real launch.

---

## 11. Languages (English / Arabic / Urdu)

A lightweight in-house i18n layer (`modules/i18n.py`) localises the interface. English is the source language and the default.

- **Lookup:** `t("Save")` returns the Arabic/Urdu string for the current language from a central dictionary (`_TR`), falling back to the English source when no entry exists.
- **Widget-tree translation:** `translate_tree(widget)` walks an already-built screen and translates the text of labels, buttons, check/radio boxes, group-box titles, tab bars, `QTableWidget` headers and placeholders whose exact English text is in the dictionary — so screens are localised without wrapping every literal by hand. It is called in `login.py`, in `dashboard.py` (the whole shell, plus each page on navigation after its refresh), and in the license dialogs.
- **On-demand dialogs:** `install_dialog_autotranslate()` (called from `main.py`) patches `QDialog.exec_` so every pop-up (Add/Edit Item, Outstanding Payment, POS preview, Import summary, License dialogs) is translated when shown, with no per-dialog edits.
- **Logic-critical combos:** `translate_combo(combo)` shows the translated label but stores the English value as `itemData`; the payment-type (Invoice), method (Outstanding) and default-payment (Settings) combos use it and read `currentData()`, so **stored database values stay English** in every language.
- **Data is never translated:** customer/product names and numbers are left untouched — only known UI phrases in the dictionary are changed.
- **RTL:** Arabic and Urdu set `QApplication.setLayoutDirection(Qt.RightToLeft)`.
- **Selection & persistence:** choose the language on the **login screen** (rebuilds the login immediately) or in **Settings → Interface Language** (full effect after re-login). Stored in `app_language`, applied at startup via `i18n.init_from_db()` + `apply_direction()`. A bundled Arabic/Urdu UI font in `assets/fonts/` is loaded at startup when present.

### 11.1 Printed documents (print language & shaping)

- **`print_language`** (Settings → Print Language) is independent of the on-screen language — many shops use an Urdu UI but print English invoices. It drives the fixed labels, totals, watermark and footer on the invoice, POS receipt and reports.
- **Shaping:** ReportLab does not shape Arabic script, so `utils.shape_rtl()` reshapes + bidi-reorders text via **arabic-reshaper** + **python-bidi** (optional deps — see `requirements.txt`), rendered with a bundled TTF registered by `utils.pfont()`. `plabel()` translates+shapes fixed labels; `ptext()` shapes arbitrary data. Everything **degrades gracefully**: with the libraries/fonts absent, English printing is unaffected and ar/ur fall back rather than crash.
- **Fonts:** drop `NotoNaskhArabic-Regular.ttf` (Arabic) and `NotoNastaliqUrdu-Regular.ttf` (Urdu) into `assets/fonts/` — see that folder's `README.txt`. They are bundled by `GSTechSys.spec`.
- **Thermal POS:** raw ESC/POS text can't render Arabic/Urdu, so when the print language is ar/ur `print_pos_receipt()` prints the **POS PDF** (image-style) instead of raw text; English keeps the fast raw-text path.
- **Coverage status:** the **invoice**, **POS receipt** and **generic report** PDFs are print-language-aware. The **payment receipt** and **return note** PDFs currently print their fixed labels in English (data still prints); completing those two is a small follow-up.
- **Dictionary coverage:** ~95% of harvested UI strings are translated. To extend, add the exact English source string as a key in `_TR` (with `ar`/`ur`). The **Help** screen is English+Urdu; Arabic Help topics are not yet authored.

---

## 12. Known Limitations / Notes for Future Work

- **Returns do not post a `customer_ledger` entry** (they adjust `customers.balance` directly), so they won't appear as a line in the Customer Ledger report's history, even though the balance total itself is correct.
- **Units** is a managed pick-list, not a real foreign key — renaming a unit does not cascade to items that already used the old text (by design, per product decision — keeps the change low-risk; a full FK migration was considered and intentionally deferred).
- **Import Data** processes rows independently with no transaction wrapping — a CSV with some invalid rows will partially import (valid rows succeed, invalid ones are reported as errors) rather than failing atomically.
- **No login rate-limiting/lockout** — authentication is a simple credential check with no brute-force protection.
- **Translation coverage is dictionary-driven** — the Arabic/Urdu interface translates every UI phrase present in `i18n._TR`; strings not yet added (long explanatory notes, some pop-up dialogs opened as separate top-level windows) still render in English until their source text is added to the dictionary.
- **Trial protection is a deterrent, not hardened DRM** — a determined user with the Python source could bypass the 30-day gate; it is appropriate for a bundled `.exe` given to ordinary shop owners, not a cryptographic license server.

---

## 13. Build, Run & Test

**Run from source (Windows):**
```
<venv>\Scripts\python.exe main.py
```
First launch creates the SQLite database and seeds defaults; log in with the default admin account (`admin` / `admin123`) and change the password.

**Run the tests** (headless Qt; stdlib `unittest`, ~103 tests covering database, invoice logic, payments/returns, item search, licensing, i18n, PDF generation, migrations, widget smoke tests, the POS flow, the fresh/empty-database shipped state, and the Sales & Income / Profit & Loss analytics):
```
set QT_QPA_PLATFORM=offscreen
python -m unittest discover -s tests -p "test_*.py" -t tests
```

**Build the distributable `.exe`** (PyInstaller, one-file):
```
pyinstaller GSTechSys.spec --clean --noconfirm
```
The result is `dist/GSTechSys.exe` (~54 MB). It carries its own database, `backups/`, and `assets/item_images/` **next to the exe** on first run, so it is portable to any Windows machine. `GSTechSys.spec` lists `modules.pos_quick_sale` (and the other modules) in `hiddenimports`; add any new module there before building.

> **Shipping checklist:** ensure the distributed copy has **no** pre-existing `app_license` row / `.gstechsys.lic` (so the 30-day trial starts on the customer's first launch, not the build date), and that `config.ENCRYPTION_KEY` / `_LICENSE_SECRET` are the intended production values (see §9, §10).

---

## 14. Revision Note — Quick Sale (POS) Module

This revision documents the **Quick Sale (POS)** feature set and its supporting changes:
- New screen `modules/pos_quick_sale.py` (`QuickSaleWidget`) — touch terminal, product tiles with images, cart, delivery charges, cash-tendered/change, park-and-resume — see **§5.21**.
- New table **`held_sales`** for parked carts (**§4.2**); new columns **`items.image_path`** (**§4.1**) and **`invoices.delivery_charges`** (**§4.2**), both added by idempotent startup migrations.
- Product images on items (Browse Image… in the Items dialog; `assets/item_images/`; recommended **256 × 256 px square PNG**) — **§4.1**.
- Out-of-stock → one-click restock into **Purchase** (`request_add_stock` → `PurchaseWidget.preload_item`) — **§5.21**.
- Retail-format 80 mm **voucher** via `generate_pos_receipt_pdf()` (company header, EMP/customer lines, SubTotal/Discount/Delivery/VAT/Total/Change/Due) — **§7**.
- New i18n keys for the POS/voucher strings (EN/AR/UR) — **§11**.

## 15. Revision Note — Dashboard & Income/Profit Reports

- New **Dashboard** screen `modules/business_dashboard.py` (`BusinessDashboardWidget`) — manager/admin, KPI cards + top items + last-7-days — **§5.22**.
- Two new **Reports** tabs — **Sales & Income** (Daily/Weekly/Monthly) and **Profit & Loss** — **§6**.
- New analytics helpers in `database.py`: `get_sales_summary`, `get_sales_by_period`, `get_total_outstanding`, `get_low_stock_count`, `get_counts` — **§5.22**.
- Nav: `dashboard` added to `ROLE_NAV` (manager/admin) and `ALL_NAV`; i18n keys added for Dashboard / Sales & Income / Profit & Loss; `modules.business_dashboard` added to `GSTechSys.spec` hidden imports.

## 16. Revision Note — Branding & UI

- **GSTechSys default logo** replaced (`assets/logo.png` 256×256, regenerated multi-size `assets/logo.ico` used for the window / taskbar / `.exe` icon). Shown on the login screen and as the default on documents when no company logo is set; the active company's own uploaded logo is unaffected. Previous files kept as `assets/logo.png.prev.bak` / `logo.ico.prev.bak`.
- **Vendor contact** on the login footer and the printed-document fallback header set to **+92 332 8228209** (`modules/login.py`, `modules/utils.COMPANY_PHONE`), matching the licensing/contact details.
- **POS cart buttons** — the per-row **+ / − / ✕** controls enlarged (40×34 px, 18 px bold glyphs; cart row height 42 px) for touch legibility.
