# Internal HR modules — Design

Date: 2026-11-22
Status: Draft (agreed scope, awaiting build go-ahead)

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

## Permissions
New permissions following the allow-list model: `ManagePolicies`,
`ManageProjects`, `ApproveCommission`, `ImportGrab`. PM approval is scoped to
projects where the approver is `manager_id`. `exec` sees projects but not
commission amounts.

## Build order
1. Policy/NDA/memo signing (click + MyDigital ID)
2. Projects + tasks + import
3. Outstation + claims project/trip link + Grab claim type
4. Commission
5. Grab for Business import, project cost report
6. Leave fixes, profile self-edit, hide payroll menu
