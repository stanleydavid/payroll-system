# EarthBusiness Suite — Internal HR modules design

Date: 2026-11-22
Status: Draft (agreed scope incl. P&C role, encryption and immutable audit, awaiting build go-ahead)

Scope for the company's own internal HR use. Payroll stays in the codebase
but is hidden from the menu until verified statutory rule sets exist (see
`docs/payroll/statutory-rules.md`); commission is the one payroll touchpoint
and has an export fallback.

## Modules and current state

| Module | State | Work |
| --- | --- | --- |
| Staff profile | Exists (`employees`, portal `MyProfile`) | Staff self-edit with HR approval |
| Leave | Exists | Leave-type CRUD, service-year bands, carry-forward fix (plan #53, #55) |
| Daily clock-in | Exists (QR/kiosk/Face ID/geofence) | Outstation-aware (below) |
| Claims | Exists (`claims`) | Project + trip link, Grab type |
| Policy / NDA / memo signing | **New** | Section 1 |
| Projects + tasks + import | **New** | Section 2 |
| Commission (komisen) | **New** | Section 3 |
| Outstation | **New** | Section 4 |
| Grab | **New** | Section 5 |

## 1. Policy, NDA and memo signing

### Model
- `policy_documents` — company-scoped; `kind` (`policy` | `nda` | `memo` | `notice`),
  title, current version id, `sign_level` (`click` | `mydigital` | `startrack`),
  due date, audience (`all` | departments | employees), status (`draft` |
  `published` | `archived`).
- `policy_document_versions` — immutable; stored file (upload) + `sha256`
  of the exact bytes. A new version re-opens signing for everyone.
- `policy_signatures` — one row per (version, employee): `signed_at`, IP,
  user agent, `document_sha256` (copied at signing), `method`, and for
  `mydigital`: `earthid_jti` (unique), verified NRIC (masked at rest except
  last 4), name, and the raw `result_token` for evidence. Append-only.

### Signing levels
- **click** — "I have read and agree" + password re-confirmation.
- **mydigital** — EarthID MyKad Verification API
  (`EarthID-Enterprise/docs/mykad-verify-api-specs.md`):
  1. Backend creates `state` bound server-side to (employee, version, sha256), TTL 10 min.
  2. Portal opens `GET {EARTHID_BASE}/api/verify/mykad/start?client_id&redirect_uri&state&mode=popup`
     (no `nric` sent — legacy path).
  3. Callback page (same origin as the portal) posts `result_token` to the backend.
  4. Backend verifies via JWKS: signature, `iss`, `aud`, `exp`, `state`,
     one-time `jti`, `verified === true`, **and `nric` == `employees.ic_number`**
     (normalised to 12 digits). Mismatch → refuse, audited.
  5. Signature row + audit row in one transaction; signed PDF certificate page
     appended (name, masked IC, timestamp, document hash, EarthID `jti`).
  - Legal note shown to admins: this is an electronic signature (ECA 2006)
    with verified identity, not a DSA 1997 PKI digital signature.
- **startrack** — blocked on StarTrack e-sign API documentation. The repo
  only registers StarTrack as an EarthID OIDC client (`svc-startrack`).

### Config
`EARTHID_BASE_URL`, `EARTHID_VERIFY_CLIENT_ID`, `EARTHID_VERIFY_REDIRECT_URI`
(shown with a copy button on the admin settings screen, per the EarthID spec).
Unset → `mydigital` level is unavailable, not silently downgraded.

### UX
- HR: publish, audience picker, live signed/unsigned list, reminders, CSV export.
- Staff: portal "To sign" inbox + badge; history of signed documents.

## 2. Projects, tasks and import

- `projects` — code (unique per company), name, client, manager (employee),
  value (sen), start/end, status (`active` | `closed` | `cancelled`), `closed_at`.
- `project_tasks` — code (unique per project), name, value (sen), quantity
  unit (optional), status, assignees.
- `project_task_members` — task × employee, `share_bps` (split; sums to 10000).
- Codes are the tags used by claims, outstation, Grab expense codes and commission.

### Import (Excel/CSV template)
Reuses the `employee_import_service` pattern: upload → parse → validate →
preview with per-row errors → confirm in one transaction. Upsert by code.
Columns: project_code, project_name, client, manager_staff_no, project_value,
start_date, end_date, task_code, task_name, task_value, unit, quantity,
assignee_staff_nos (`;`-separated), share_pct, commission_rule_code.

## 3. Commission (komisen)

### Rules (`commission_rules`, attached to a project or task)
| Type | Parameters | Amount |
| --- | --- | --- |
| `percent` | `rate_bps` | value × rate |
| `fixed` | `amount` | flat per task |
| `tiered` | ordered `(up_to, rate_bps)` bands, marginal | banded over value |
| `per_unit` | `amount_per_unit` | quantity × amount |

All arithmetic in integer sen with a single, documented rounding point
(half-up at the final per-employee amount); split by `share_bps`, remainder sen
to the largest share so the split always sums exactly.

### Lifecycle — payable on **project close**
1. PM closes the project → engine computes a `commission_batches` row with
   `commission_lines` (employee, task, rule snapshot, base, amount).
2. PM approves → HR/finance approves.
3. Approved lines become `payroll_entries` with `item_type = 'commission'`
   for the chosen pay month (the engine already treats these as additional
   remuneration for PCB — `payroll_engine.rs:1516`). While payroll is hidden,
   the batch exports to CSV/Excel instead.
4. A closed+paid project cannot be re-paid; re-opening requires reversal of
   the batch. Rule snapshot stored on each line for provenance.

### Open question
Tiered: bands over the **project/task value** (assumed) vs. over a person's
cumulative sales for a period (target-based). The latter needs a period, not
a project close.

## 4. Outstation

- `outstation_trips` — employee(s), project, destination, purpose, start/end
  (local dates), status via the existing approval flow (PM → HR).
- Approved days: auto-absent skips them (`mark_absent` exclusion like approved
  leave); check-in off-geofence is accepted and tagged `outstation` rather
  than flagged.
- Per-diem: company rate table (by destination class), generated as claim
  lines on trip completion.
- Claims gain `project_id`, `trip_id` and claim types (mileage, hotel, meals,
  toll/parking, Grab, other). Mileage = km × company rate.

## 5. Grab

- **Staff-paid**: Grab claim type; upload e-receipt; booking ID unique per
  company (duplicate guard); project/trip required.
- **Grab for Business**: monthly ride-report CSV import; expense code =
  project code; rider email/phone matched to employee; unmatched rows queued
  for review; company-paid rides are cost records, not reimbursements.

## Reports
Project cost: claims + Grab + per-diem + commission, by project/month, Excel export.

## 6. Permissions and the P&C compensation role

New allow-list permissions: `ManagePolicies`, `ManageProjects`,
`ApproveCommission`, `ImportGrab`, and **`ViewCompensation`**.

- `ViewCompensation` gates every money figure about a person: basic salary,
  allowances, bonus, incentive (insentif), commission rules/rates, commission
  lines and totals, payslip amounts. Granted via a user group (e.g. "P&C").
- Enforcement is server-side: responses for callers without it carry no
  amount fields (`null` + `redacted: true`), not values hidden by the UI.
  Exports follow the same rule.
- PM approval is scoped to projects where the approver is `manager_id`; a PM
  without `ViewCompensation` approves the *work* (task done, project closed),
  not the amounts. Approving commission amounts needs `ApproveCommission` +
  `ViewCompensation`.
- `exec` keeps its existing denial; separation-of-duties invariants in
  `permission.rs` extended to the effective (role + group) set (plan #37).
- Every read/export of compensation data writes an audit row.

## 7. Application-level encryption at rest

Goal: a person with Postgres access, a dump or a backup cannot read personal
or compensation data. Decryption happens only in the API process's memory.

### Key hierarchy
- **Master key (KEK)** — never stored on the server. Derived at unlock time
  from a key-custodian passphrase (Argon2id) or recovery key; held only in
  process memory (`zeroize` on drop; `PR_SET_DUMPABLE=0`, no core dumps).
- **Per-company data keys (DEK)** — random 256-bit, stored in
  `encryption_keys` only wrapped by the KEK (AES-256-GCM), versioned for
  rotation.
- **Blind-index key** — per company, wrapped likewise; HMAC-SHA256 of
  normalised values for exact-match lookup (IC number, bank account).

### Custodian unlock (chosen)
- On boot the API starts **sealed**: login, health, attendance clock-in and
  other plaintext paths work; any route needing decryption returns 503
  `system_sealed`.
- `POST /api/admin/unseal` — super_admin session + custodian passphrase,
  strict rate limit, audited (who/when, never the secret). Successful unseal
  unwraps DEKs into memory.
- At least two custodians, each with their own wrapping of the KEK, plus a
  printed recovery key kept offline — losing every custodian secret loses the
  data, by design.
- On seal-at-boot, custodians are notified (email) so a deploy or crash
  doesn't leave the system sealed unnoticed.
- Residual risk, documented in the admin UI: someone with root on the running
  host can still read process memory. This design protects against DB-only
  access, dumps and backups, and against a sysadmin who lacks a custodian
  secret and does not tamper with the live host.

### Field encryption
- `core/field_crypto.rs`: AES-256-GCM, value stored as
  `version || nonce || ciphertext` (`bytea`). AAD = table, column and row id,
  so ciphertext cannot be moved to another row or column.
- Typed wrappers (`Encrypted<String>`, `Encrypted<Decimal>`, `Encrypted<i64>`)
  so repositories return ciphertext and services decrypt explicitly; the
  plaintext type never derives `Serialize` without the permission check.
- Money held encrypted is aggregated in Rust after decryption; SQL `SUM`
  over encrypted columns is not possible.
- Uploaded files (NDAs, signed PDFs, receipts, IC copies) encrypted on disk
  with the company DEK; served through the existing authorized upload route.

### What is encrypted
IC/passport, address, phone, personal email, bank account, emergency
contacts, date of birth; basic salary, allowances, bonus, incentive,
commission rules and lines; signature evidence (EarthID token, IP); claim
descriptions and receipts; Grab ride details.

### What stays plaintext (needed for joins, filters, indexes)
Ids, `company_id`, foreign keys, dates/timestamps, statuses, project/task
codes, login email (so sign-in and the sealed-state message still work).

### Rollout (chosen)
- Phase A: encryption foundation + unseal; all new modules and staff PII
  built encrypted from day one; one-shot migration encrypting existing PII
  columns (new migration, run after unseal).
- Phase B: existing payroll tables (`employees.basic_salary`, payroll items,
  details, entries) converted, with reports/EA form/dashboard rewritten to
  aggregate in Rust.

## 8. Full, immutable audit trail

Requirement: every **read, print/download, search, export, create, edit and
delete** is audited, and audit records cannot be altered or removed.

### Current state (gaps)
- `audit_logs` covers selected mutations and credential events (~125 call
  sites); reads, searches and prints are not recorded.
- Rows are mutable: `company_id` is `ON DELETE CASCADE` and `user_id` is
  `ON DELETE SET NULL` (`1008`), so deleting a company or user rewrites or
  erases history. No tamper evidence.
- `old_values` / `new_values` are plaintext `jsonb` — they would leak every
  field section 7 encrypts.
- Plan item #26 (retention purge of `audit_logs`) is superseded by this
  section for audit data.

### Capture
1. **Request layer (every authenticated API call).** An Axum middleware
   writes one event per request: actor (user, roles, effective permissions
   hash), company, method, route *template*, action class (`read` |
   `search` | `export` | `print` | `create` | `update` | `delete` |
   `auth`), entity type + ids touched, result count, HTTP status, client IP,
   user agent, session id, request id, duration. Search terms and filters
   are captured encrypted (they may contain IC numbers or names).
2. **Domain events (mutations).** Services write a field-level diff (old →
   new) for every create/update/delete **in the same transaction** as the
   change, encrypted with the company DEK. A mutation whose audit write fails
   rolls back.
3. **Reads fail closed.** If the request-level event cannot be written, the
   response is withheld (503) rather than served unaudited.
4. **Print / download.** Every PDF, CSV, Excel, payslip, EA form, signed
   document and file download is server-generated and logged as `print` /
   `export` with the document id and its SHA-256. The app's Print buttons go
   through the server. A browser Ctrl+P on an on-screen page is logged on a
   best-effort basis (`beforeprint` beacon); screenshots and photos of the
   screen cannot be detected by any web app.
5. **Background jobs and the unseal ceremony** are actors too (`system`,
   `custodian`).

### Immutability
- New `audit_events` table, partitioned by month. **No foreign keys** to
  users/companies (actor and company are stored by value), so deleting a user
  or company never touches audit rows. Existing `audit_logs` rows are copied
  in once and the table frozen.
- Database: the API role gets `INSERT` + `SELECT` only; `UPDATE`, `DELETE`
  and `TRUNCATE` revoked and additionally rejected by triggers. Company
  teardown no longer removes audit data.
- **Tamper evidence:** each event carries `seq`, `prev_hash` and `hash` =
  SHA-256 over its canonical bytes + `prev_hash` (hash chain per company).
  Every hour a **checkpoint** (company, last seq, chain head) is signed with an
  Ed25519 key unwrapped at unseal and shipped **off-host** to write-once
  storage (object lock) plus a daily digest email to the custodians.
- **Verification:** a daily job and an admin "Verify audit trail" action
  recompute the chain against the latest off-host checkpoint and alert on any
  gap, edit or deletion.
- Honest limit: a Postgres superuser can bypass grants and triggers. They
  cannot do so *undetectably*: a changed or deleted row breaks the chain
  against checkpoints already stored off-host.

### Encryption and access
- Event metadata (time, actor, action, entity type, route) is plaintext so it
  can be filtered; details (diffs, search terms, result ids lists) are
  encrypted per section 7.
- Viewing the audit trail needs `ViewAuditLog`; seeing compensation fields
  inside diffs also needs `ViewCompensation`. Reading the audit trail is
  itself audited.

### Retention
7 years (aligned with record-keeping under the Income Tax Act 1967 and the
Employment Act). Partitions older than that are exported to the off-host
write-once store with their checkpoints before detaching; nothing inside the
retention window can be deleted.

### Volume
Every read is an event: expect tens of thousands of rows per day for a
company of ~100 staff. Writes are batched per request (one row), monthly
partitions keep indexes small, and the admin audit view queries by
partition.

## 9. Online forms: job application, onboarding, internship

Three public-facing forms sharing one identity-verified candidate profile:
**job application**, **new-hire onboarding** (after offer accepted) and
**internship** (application + onboarding in one).

### Identity: EarthID with MyDigital ID (mandatory)
- The form is reachable only after signing in with **EarthID as an OIDC
  provider** (authorization code + PKCE; EarthID Flow 3) — a second provider
  beside Google in `oauth2_service`.
- Scopes: `openid profile email nric mydigital.verification individual_profile`.
- Accepted only when the ID token / userinfo carries
  `mydigital_verified === true` **and** a 12-digit `ic_no`. Otherwise the
  candidate is told to verify with MyDigital ID in EarthID first.
- Prefilled and locked from verified claims: name, NRIC (+ derived date of
  birth and gender from the IC). Prefilled but editable: email, phone,
  `mailingAddress`, `placeAddress`, `kodLokasi`.
- Candidates are a separate principal (`candidates`, keyed by EarthID `sub`
  and NRIC blind index), not users of the HR app; one candidate can apply to
  several openings without re-entering data.
- EarthID admin needs an OIDC client for the HR app with those scopes and
  `individual_profile` approval.

### Selfie
- Live camera capture in the browser only (`getUserMedia`); gallery upload
  disabled. Simple liveness prompt (e.g. turn head / blink, two frames) to
  deter a photo of a photo.
- Optional second shot holding the MyKad.
- Stored encrypted; HR compares visually. Automated face matching against an
  IC photo is out of scope (MyDigital ID returns no photo; would need a
  licensed eKYC vendor).
- Biometric data → explicit consent checkbox with purpose, per PDPA 2010
  (as amended 2024).

### Fields
| Group | Fields |
| --- | --- |
| Identity | name, NRIC (verified), DOB, gender, nationality, selfie |
| Contact | email, mobile phone |
| Home address | line 1–3, postcode, city, state, **kod lokasi** (from EarthID claim; format-validated, e.g. `KLTA.A1`; lookup via Kodlokasi API if its docs are provided) |
| Emergency contacts | 1–3: name, relationship, phone, address |
| Health | allergies, medical conditions, current medication (optional), blood type (optional) |
| Academic | institution, qualification level, field, year, CGPA, certificate upload |
| Professional certs | name, issuer, number, issue/expiry date, upload |
| Past employment | employer, position, from/to, last salary (optional), reason for leaving, release/service letter upload |
| Internship only | university, programme, internship start/end, university supervisor + contact, placement letter upload |
| Onboarding only | bank account, EPF/SOCSO/tax numbers, spouse/children (for PCB reliefs), signed offer letter, NDA (signing per section 1) |

### Health data handling
- Sensitive personal data (PDPA s.40): explicit consent, encrypted, and a
  separate `ViewMedical` permission (e.g. HR manager / first-aider) —
  `ViewCompensation` does not grant it.
- Recommendation: collect allergies and medical conditions at **onboarding**
  (after offer), not on the job application, to avoid the appearance of
  screening candidates on health grounds. Internship form asks them at the
  acceptance step.

### Pipeline
applied → screening → interview → offer → accepted → onboarding → hired
(or rejected / withdrawn). On **hired**, the candidate profile, verified NRIC,
address, emergency contacts, health data and documents are copied into the
employee record (encrypted), and the selfie becomes the staff photo.

### Retention and erasure
- Unsuccessful applicants: data kept 12 months (configurable) unless the
  candidate consents to the talent pool, then erased.
- Erasure is **crypto-shredding**: each candidate's details are encrypted
  under a per-candidate key; destroying the key makes the data unreadable while
  the immutable audit chain (section 8) stays intact and verifiable.

### Abuse controls
Turnstile on the public pages, per-sub rate limits, upload type/size limits,
malware scan of uploads, one active application per opening per candidate.

## 10. Product, UX and PWA

### Name and brand
- Product name: **EarthBusiness Suite** (replaces "PayrollMY": `index.html`
  title, `BrandLogo` assets under `public/branding/`, emails, PDFs, i18n
  strings, manifest). Module names inside the suite: People, Leave,
  Attendance, Claims, Projects, Documents, Recruitment, Payroll (hidden).
- New logo/lockup and colour tokens to be supplied by the company or
  proposed in a design pass; the app reads them from one token file.

### Platform
- One React codebase, **mobile-first responsive**, installable as a **PWA**
  (Android, iOS 16.4+, desktop Chrome/Edge) and used normally in any browser.
- `vite-plugin-pwa`: web app manifest (name, icons, maskable icon, theme
  colour, `display: standalone`, shortcuts: Clock in, Apply leave, New claim,
  To sign), service worker for the app shell and static assets.
- **No API responses are cached by the service worker**, and nothing
  sensitive goes to IndexedDB/localStorage — consistent with sections 7–8
  (encrypted at rest, every read audited). Offline: the shell opens and shows
  an offline state; drafts of forms (claim, leave, application) are kept
  locally *without* attachments or health/PII fields and submitted when back
  online.
- Clock-in is never queued offline: it depends on the server-observed network,
  geofence and QR TTL.
- **Web Push** notifications: approvals waiting, document to sign, leave /
  claim decision, outstation approved, system sealed (custodians).
- Device capabilities used: camera (QR scan, selfie, receipt photo),
  geolocation (geofence), WebAuthn / passkeys (Face ID / fingerprint login
  and check-in), share target for receipts (Android).

### Navigation
- **Mobile:** bottom tab bar — Home, Attendance, Requests (leave / claims /
  outstation), Inbox (approvals, to sign, notifications), Me. A primary
  floating action for the most common task of the role (Clock in for staff,
  Approve for managers).
- **Desktop/tablet:** collapsible left sidebar grouped by module, top bar
  with company switcher, global search (audited, permission-filtered),
  notifications and profile.
- Role-based home: staff see today's clock-in, leave balance, pending
  items; managers see approvals and team presence; P&C sees compensation
  tasks; admins see system health (sealed state, audit verification).

### Design system
- Design tokens (colour, type scale, spacing, radius, elevation, motion) in
  one place; light and dark themes; Tailwind v4 theme variables.
- Component library built on accessible primitives: Button, Input, Select,
  Combobox, DatePicker (local MYT), FileUpload with camera, Sheet/BottomSheet
  (mobile), Dialog, ConfirmDialog (replaces `window.confirm`/`alert`), Toast,
  Tabs, Badge/StatusPill, EmptyState, Skeleton, Stepper (multi-step forms),
  DataTable that becomes a card list under `md` breakpoint, keyboard-operable
  rows.
- Money fields render as redacted chips when the viewer lacks
  `ViewCompensation`; medical fields likewise with `ViewMedical`.
- Accessibility: WCAG 2.2 AA — labels on every control (plan #67), 44 px
  touch targets, focus rings, reduced-motion support, colour contrast checked
  in CI.
- Languages: the existing four-locale i18n (BM, EN, 中文, தமிழ்) covers all new
  screens.

### UX principles
- One primary action per screen; forms split into short steps with progress
  and autosave of non-sensitive drafts.
- Status always visible (pill + timeline) for every request.
- Explain blocks in plain language (e.g. "System is locked — waiting for a key
  custodian", "Outside office network — your check-in is flagged for HR").
- Fast: route-level code splitting, skeletons, optimistic UI only for
  non-financial actions; target LCP < 2.5 s on mid-range Android over 4G.

### Validation before build
Clickable prototype of the key flows (staff mobile: clock in, apply leave,
claim with Grab receipt, sign NDA; manager: approvals; candidate: EarthID
sign-in → application form with selfie; admin desktop: projects, commission,
audit) reviewed with real users before implementation of each module's UI.

## Build order
- UX: design system, rebrand to EarthBusiness Suite and PWA shell run
  alongside step 0; each module ships its mobile and desktop UI together.

0. Encryption foundation: key hierarchy, custodian unseal, field crypto,
   blind index, encrypted uploads, `ViewCompensation` redaction, and the
   immutable `audit_events` trail (request layer, hash chain, checkpoints)
1. Policy/NDA/memo signing (click + MyDigital ID)
2. Projects + tasks + import
3. Outstation + claims project/trip link + Grab claim type
4. Commission
5. Grab for Business import, project cost report
6. Leave fixes, profile self-edit (encrypted PII), hide payroll menu
6a. Online forms: EarthID OIDC login, candidate profile, job application,
    internship, onboarding, selfie, pipeline, hire → employee
7. Phase B: encrypt existing payroll data
