# OpenEMR Codebase Audit — Raw Findings

**Audit Date:** 2026-04-27
**Auditor:** Claude Code (static analysis only — no runtime execution)
**Scope:** See header section. Excludes vendor/, node_modules/, tests/, ci/, .github/, Documentation/, public/assets/, sites/default/documents/, sites/default/cache/, sites/default/logs/

---

## 1. Architecture

### 1.1 HTTP Entry Point and Request Flow

**Web entry point:** `index.php`
- Reads `?site=` or falls back to `HTTP_HOST` / `default` to select the site
- Validates site ID against `[A-Za-z0-9\-.]` (injection guard present)
- Loads `sites/$site_id/sqlconf.php` (hardcodes DB credentials — see Security section)
- Redirects to `interface/login/login.php?site=$site_id` if configured (`$config == 1`), or `setup.php` if not

**Request flows — three distinct paths:**

| Path | Entry | Notes |
|---|---|---|
| Legacy web UI | `interface/login/login.php` → `interface/globals.php` → controller PHP files | Session-based, ACL gated |
| REST/FHIR API | `apis/dispatch.php` → `ApiApplication::run()` → middleware pipeline | OAuth2 Bearer token |
| OAuth2 authorization server | `oauth2/authorize.php` → `ApiApplication::run()` | OAuth2 session cookie |

**`controller.php`** (legacy positional router):
- Wraps `Controller->dispatch()` / `Controller->act()` based on presence of `?controller=` param
- Catches `AccessDeniedHttpException`, `HttpExceptionInterface`, and `\Throwable`; returns reference IDs on 500 errors

**`_rest_routes.inc.php`** delegates to three separate route arrays loaded from:
- `apis/routes/_rest_routes_standard.inc.php` → `RestConfig::$ROUTE_MAP`
- `apis/routes/_rest_routes_fhir_r4_us_core_3_1_0.inc.php` → `RestConfig::$FHIR_ROUTE_MAP`
- `apis/routes/_rest_routes_portal.inc.php` → `RestConfig::$PORTAL_ROUTE_MAP`

### 1.2 Session Initialization

Session initialization is centralized in `src/Common/Session/SessionUtil.php`.

- Four named sessions exist: `OpenEMR` (core), `authserverOpenEMR` (OAuth2), `apiOpenEMR` (API), `PortalOpenEMR` (portal)
- `cookie_httponly` is **false** for core OpenEMR (JavaScript needs session ID access for multi-patient window management), **true** for portal and OAuth2
- `cookie_samesite` is `Strict` for core and portal, `Lax` for OAuth2, `None` (with `Secure=true`) for OAuth2 cross-origin SMART App scenarios
- `use_strict_mode`, `use_only_cookies`, `use_cookies` are all set
- Session ID length: 48 chars with `sid_bits_per_character=6` (deprecated in PHP 8.4; falls back to system default)
- `gc_maxlifetime`: 14400 (4 hours)
- Session locking is intentionally avoided for performance via `ReadAndCloseNativeSessionStorage`; writes reopen the session explicitly
- Optional Redis Sentinel backend via `src/Common/Session/Predis/SentinelUtil.php`

Session is initiated early in `interface/globals.php` via `SessionUtil::setAppCookie(SessionUtil::CORE_SESSION_ID)`.

### 1.3 Routing Approach

**Three routing strategies coexist:**

1. **Legacy positional routing** (`controller.php`): `?act=DoSomething&id=1` mapped to `Controller->act()`
2. **Named controller routing** (`controller.php`): `?controller=EncounterController&action=index` mapped to `Controller->dispatch()`
3. **REST router** (`ApiApplication`): Middleware pipeline (Symfony HttpKernel), routes defined as closures in `_rest_routes_*.inc.php` files, matched by `{METHOD} {path}` string keys with colon-prefixed path params (`:uuid`, `:pid`)

The `src/BC/FallbackRouter.php` provides routing-test support for development.

### 1.4 Service Layer Location and Patterns

Modern service layer: `src/Services/` — 60+ service classes extending `BaseService` (`src/Services/BaseService.php`).

Key services for clinical data:
- `PatientService` — `patient_data` table, pagination via `QueryPagination`, sort-column whitelist
- `EncounterService` — `form_encounter` table, UUID-based lookup, `puuidBind` patient-scoping
- `ClinicalNotesService` — `form_clinical_notes`
- `AllergyIntoleranceService`, `ImmunizationService`, `ConditionService` — `lists` / `immunizations` tables
- `DocumentService` — `documents` table, delegates to `Document.class.php`
- `PrescriptionService` — `prescriptions` table

Symfony `EventDispatcher` fires typed events before/after patient create/update (`BeforePatientCreatedEvent`, `PatientCreatedEvent`, `PatientUpdatedEvent`, etc.). 79 event classes in `src/Events/`.

Legacy library code in `library/` uses procedural functions (`sqlStatement`, `sqlQuery`, `sqlFetchArray`) via ADODB (`adodb/adodb-php`).

### 1.5 Database Access Patterns

Four access layers present simultaneously:

| Layer | Where Used | Notes |
|---|---|---|
| `QueryUtils` (`src/Common/Database/QueryUtils.php`) | Modern services | Table-whitelist via `SHOW TABLES`; column whitelist via `escape_sql_column_name` |
| ADODB surface (`library/sql.inc.php`) | Legacy interface code | `sqlStatement`, `sqlQuery`, `sqlInsert` wrappers |
| Doctrine DBAL (`doctrine/dbal ^4.4`) | Audit logger, migrations | Used in `EventAuditLogger` via `DatabaseConnectionFactory` |
| Doctrine Migrations (`doctrine/migrations ^3.9`) | Schema upgrades | `sql/` upgrade scripts also present for pre-migration versions |

`src/BC/DatabaseConnectionFactory.php` is the canonical connection factory. Direct `new PDO` or `new mysqli` instantiation is not present in `src/`.

### 1.6 API Boundaries

| API | Path | Auth | Notes |
|---|---|---|---|
| Standard REST | `/apis/api/*` | OAuth2 Bearer | User role only; default off (`rest_api = 0`) |
| FHIR R4 US Core | `/apis/fhir/*` | OAuth2 Bearer | User + Patient roles; default off (`rest_fhir_api = 0`) |
| Portal REST | `/apis/portal/*` | OAuth2 Bearer | Patient role; experimental; default off |
| OAuth2 AS | `/oauth2/*` | N/A | Auth code, client credentials, password grants |
| Patient portal (session) | `portal/` | Session cookie (`PortalOpenEMR`) | Separate session from main app |

All REST/FHIR routes enforce OAuth2 via `OAuth2AuthorizationListener` middleware. Per-route ACL is enforced by `RestConfig::request_authorization_check()`.

### 1.7 Legacy library/ and Modern src/ Coexistence

- `interface/globals.php` sets up globals, loads ADODB, initializes session, and includes legacy library files
- Modern code in `src/` uses `OEGlobalsBag` (Symfony `ParameterBag` wrapper) instead of `$GLOBALS`
- `src/BC/ServiceContainer.php` is a static service locator bridging the two worlds (acknowledged as a transitional pattern)
- Legacy files in `interface/` `require_once('../globals.php')` as their first act; new controllers inject dependencies
- `bootstrap.php` (CLI only) sets up autoloader + PSR-11 container without touching sessions/DB

### 1.8 Template Engines

| Engine | Files | Where Used |
|---|---|---|
| Twig 3.x | `templates/**/*.twig` (183 files) | Login, portal, modern UI pages |
| Smarty 4.5 | Legacy `.html` / `.tpl` in `interface/` | Older forms |
| Raw PHP echo | `interface/**/*.php` | Majority of legacy UI |

`src/Common/Twig/TwigContainer.php` is the entry point for Twig rendering. Twig auto-escapes HTML by default.

---

## 2. Security / Auth / Authorization

### 2.1 Session Handling

- Session starts in `interface/login/login.php` via `SessionUtil::setAppCookie(SessionUtil::CORE_SESSION_ID)` before globals are loaded
- `$ignoreAuth = true` is declared in `login.php` and `interface/smart/register-app.php` to bypass authentication for the login form itself — **expected and intentional**
- Session fixation mitigation: Session regeneration is handled in `AuthUtils::confirmUserPassword()` (login flow regenerates session ID after successful auth — needs runtime verification)
- `cookie_httponly = false` for core session (documented tradeoff for multi-patient window support) — creates residual XSS risk for session token theft in legacy pages
- `cookie_samesite = Strict` for core; `SameSite=None; Secure` for OAuth2 cross-site cookies

### 2.2 Password/Auth Flow

**File:** `library/auth.inc.php` (controller), `src/Common/Auth/AuthUtils.php` (logic)

- Supports: standard username/password, LDAP/AD, Google Sign-In (OAuth OpenID)
- Password hashing: `src/Common/Auth/AuthHash.php` — configurable via global `gbl_auth_hash_algo`: BCRYPT (default), ARGON2I, ARGON2ID, SHA512HASH (legacy)
- Timing attack prevention: dummy hash stored in `globals` table used to equalize timing for unknown usernames
- Password expiry: configurable via `password_expiration_days` global
- Login failure: sets `loginfailure` in session; no brute-force lockout visible in this code (may be in `users_secure` table — not confirmed from static analysis)
- `$_POST['clearPass']` is cleared via `sodium_memzero()` on failure; stored temporarily in `$passTemp` variable
- **MFA:** `src/Common/Auth/MfaUtils.php` present (TOTP, U2F, WebAuthn via `robthree/twofactorauth`, `yubico/u2flib-server`)
- One-time auth tokens: `src/Common/Auth/OneTimeAuth.php`

### 2.3 Role Checks and ACL

- ACL library: `src/Gacl/` (phpGACL wrapper), `src/Common/Acl/AclMain.php`
- `AclMain::aclCheckCore($section, $aco, $user, $return_value)` is the primary check function
- Static GACL object cached in `AclMain::$gaclObject` for performance
- ACO sections: `admin`, `acct`, `patients`, `encounters`, `squads`, `sensitivities`, `lists`, `placeholder`, `nationnotes`, `patientportal`, `menus`, `groups`, `inventory`
- REST API uses `RestConfig::request_authorization_check($request, $section, $aco)` which delegates to `AclMain`

**Coverage gaps found:**
- `interface/patient_file/` has 100 PHP files; 49 check ACL directly; ~51 do not contain an explicit `aclCheck` call
- Examples of files without explicit ACL check: `download_template.php`, `letter.php`, `label.php`, `barcode_label.php`, `pos_checkout.php`, `front_payment_terminal.php`, `addr_label.php`, `education.php`
- Many of these rely on session authentication (via `globals.php`) rather than role-based authorization — any authenticated user can access them

### 2.4 CSRF Protection

- `src/Common/Csrf/CsrfUtils.php` — HMAC-based tokens (SHA256 truncated to 40 chars); secret stored in session
- Two token families: `default` and `api`
- Verification via `CsrfUtils::checkCsrfInput(INPUT_POST, dieOnFail: true)` or `CsrfUtils::verifyCsrfToken()`
- 317 files in `interface/` import or use CSRF functions; 306 use `csrf_token_form`
- Files identified without CSRF in `interface/`: `care_plan/view.php`, `care_plan/report.php`, many `billing/` includes (`.inc.php`) — these may be includes not direct entry points

### 2.5 API Token Validation / OAuth2

- OAuth2 implemented via `league/oauth2-server ^8.4` + `steverhoades/oauth2-openid-connect-server`
- Grant types: Authorization Code, Client Credentials, Password (configurable), Refresh Token
- Token TTL: access token 1 hour (`PT1H`), auth code 5 min (`PT300S`), refresh token 3 months (`P3M`)
- OAuth2 keys stored in `sites/$site_id/documents/certificates/` (RSA)
- **OAuth2 password grant** defaults to `0` (off) in production globals but is set to `3` (both roles) in `docker/development-easy/docker-compose.yml`
- JWKS endpoint at `/oauth2/<site>/.well-known/jwks.json`
- Scope enforcement: `ScopeRepository::getScopeEntityByIdentifier()` validates against server-supported scopes; per-route ACL checked in route closures

### 2.6 Webhook / External Receiver Auth

- `interface/modules/custom_modules/oe-module-faxsms/library/webhook_receiver.php` sets `$ignoreAuth = true`
- It does use `SignalWireWebhookValidator` to validate input fields (format/length), but there is **no cryptographic webhook signature verification** visible in the file — the receiver accepts any POST that has valid field formats
- This is a potential unauthenticated inbound processing endpoint

### 2.7 CSRF/XSS Risks in Legacy Templates

- `interface/billing/sl_eob_search.php:855` — `echo (!empty($_REQUEST['only_with_debt'])) ? 'checked=checked' : ''` inside HTML attribute — safe (attribute keyword only, not a value injection), but pattern is fragile
- `interface/reports/ippf_cyp_report.php:78,372,391` — `echo $_POST['form_details'] ? 3 : 1` — numeric output, safe
- `interface/billing/ub04_helpers.php:33` — uses `js_escape()` helper before injecting into JS — correctly escaped
- No clearly exploitable unescaped `echo $_GET/POST` found after filtering for known safe escape helpers (`text()`, `attr()`, `xlt()`, `js_escape()`, `htmlspecialchars()`)
- The `text()` / `attr()` / `xlt()` helper functions in `library/` provide escaping for output
- Twig templates auto-escape by default; no `|raw` audit performed

### 2.8 PHI Exposure in Logs/Errors

- `apis/dispatch.php:38` — on uncaught exception: `error_log($e->getMessage()); error_log($e->getTraceAsString())` — stack traces may contain SQL or data fragments; response body exposes `$e->getMessage()` directly: `die(json_encode(['error' => ..., 'message' => $e->getMessage()]))` — **direct exception message exposure in API response**
- `oauth2/authorize.php` has similar: `die("An error occurred while processing the request...")` (no message leak) — OK
- `interface/globals.php` — `shouldDisplayErrors` only true when `OPENEMR__ENVIRONMENT=dev`
- `EventAuditLogger` uses a **dedicated separate DB connection** to isolate audit writes

### 2.9 Database Credentials Storage

- `sites/default/sqlconf.php` stores credentials in plaintext PHP variables (`$host`, `$login`, `$pass`, `$dbase`)
- Development docker-compose.yml exposes `MYSQL_ROOT_PASSWORD: root` and `MYSQL_PASS: openemr`
- **Critical**: `docker-compose.yml` contains hardcoded GitHub Composer tokens (lines 62-64): `GITHUB_COMPOSER_TOKEN: c313de1ed5a00eb6ff9309559ec9ad01fcc553f0` and base64/ASCII-encoded alternate tokens — these should be rotated immediately if not already expired

### 2.10 Patient Portal Isolation

- Portal uses a fully separate session name (`PortalOpenEMR`) with `httpOnly=true`
- `portal/verify_session.php` enforces `$session->get('pid')` and `$session->get('patient_portal_onsite_two')` before allowing access
- `$ignoreAuth_onsite_portal = true` is set to prevent main-app auth check; portal auth is independent
- Portal patient cannot access main app data paths — namespace separation is architecturally enforced

---

## 3. PHI and Compliance (HIPAA)

### 3.1 Where Patient Data Is Read/Written

**Key tables (from `sql/database.sql`):**
- `patient_data` — demographics, SSN (`ss`), drivers license, DOB, contact info, providerID
- `form_encounter` — encounter records linked to pid
- `form_soap` — SOAP notes (subjective, objective, assessment, plan) as `text` fields
- `form_clinical_notes` — structured clinical notes
- `lists` — problems, allergies, medications (type field differentiates)
- `prescriptions` — medication orders with `rxnorm_drugcode`
- `immunizations` — vaccination records
- `documents` — file metadata; blobs stored on filesystem or CouchDB
- `billing` — CPT/procedure codes, fees
- `pnotes` — patient notes / messages
- `history_data` — patient history
- `insurance_data` — insurance information

**Service classes:** `PatientService`, `EncounterService`, `AllergyIntoleranceService`, `ImmunizationService`, `ConditionService`, `PrescriptionService`, `DocumentService`

### 3.2 Encounter Notes Location

- SOAP notes: `form_soap` table (subjective/objective/assessment/plan as `text` fields, no separate index on pid — relies on `id` PK only)
- Clinical notes: `form_clinical_notes` + `form_clinical_notes` with `JOIN list_options` for category
- Progress notes / messages: `pnotes` table
- `forms` registry table links encounter ID to form table and form ID

### 3.3 Documents / Uploads

- Storage method: filesystem (`STORAGE_METHOD_FILESYSTEM = 0`) or CouchDB (`STORAGE_METHOD_COUCHDB = 1`), configurable via `document_storage_method` global
- Filesystem path: `sites/$site_id/documents/` (mapped by URL in `documents.url` column, stored as `file://...` path)
- Optional encryption: `Document.class.php` supports `encryptStandard()` via `KeySource::Database` when `drive_encryption` global is enabled
- Thumbnail support: encrypted alongside document
- **Default:** encryption is OFF (`drive_encryption = 0` not confirmed to default on)

### 3.4 Audit Logging

**`src/Common/Logging/EventAuditLogger.php`:**
- Enabled by default (`enable_auditlog = 1`)
- Patient-record events logged by default (`audit_events_patient-record = 1`)
- Scheduling, order, lab-results, security-administration events all defaulted to `1`
- Audit log encryption: **off by default** (`enable_auditlog_encryption = 0`)
- API logging: default `2` (full logging) — logs request URL, method, user, patient, response body
- Audit uses a separate DBAL connection from application DB to prevent application errors from blocking audit writes
- ATNA syslog support available (IHE audit trail)
- Tables tracked in `LOG_TABLES` constant: `billing`, `claims`, `form_encounter`, `form_soap`, `form_vitals`, `history_data`, `immunizations`, `insurance_data`, `lists`, `patient_data`, `pnotes`, `prescriptions`, `transactions`, `amendments`, `users`, etc.
- **Gap**: `form_clinical_notes` not listed in `LOG_TABLES` constant in `EventAuditLogger` — clinical note writes may not generate audit events automatically

### 3.5 Export/Download Paths

- `interface/patient_file/download_template.php` — substitutes patient variables into document templates; requires CSRF but **no explicit ACL role check** beyond session authentication
- `interface/patient_file/letter.php` — letter generation, requires CSRF but no explicit ACL
- `interface/reports/pat_ledger.php` — checks `acct/rep` ACL; has CSV export capability
- `interface/reports/collections_report.php` — no visible LIMIT on main encounter query — could return all encounters for all patients if no date range is entered; has CSV export
- `portal/get_patient_documents.php` — portal patient document access; protected by `verify_session.php`

### 3.6 Data Retention

- Deleter tool (`interface/patient_file/deleter.php`) performs **hard deletes** with audit logging (logs row content before delete to `log` table)
- Patient deletion: guarded by `admin/super` ACL + `allow_pat_delete` global setting
- No soft-delete pattern for clinical records — `activity` flag used for list/issue deactivation but not for physical record deletion
- Patient merge (`interface/patient_file/merge_patients.php`) — deletes source records after merge; requires `admin/super`

### 3.7 SSN / Sensitive Field Encryption

- `patient_data.ss` (SSN) is stored as `varchar(255)` **in plaintext** — no encryption wrapper applied in `PatientService` or legacy library
- `patient_data.drivers_license` also stored in plaintext
- Financial card data is handled separately via payment processors (Stripe, Rainforest) — not stored locally
- Document content is optionally encrypted but disabled by default

### 3.8 LLM/Third-Party Data Transfer Risks

Outbound HTTP calls identified:
- `src/Services/ProductRegistrationService.php:121` — `curl_init('https://reg.open-emr.org/api/registration')` — sends installation email and version info (not PHI)
- `src/Telemetry/TelemetryService.php` — click event telemetry stored locally in DB; `TelemetryRepository` — needs further runtime analysis to confirm if outbound
- `src/Cqm/CqmClient.php` — sends CQM measure data to local CQM service (node process, loopback)
- `src/USPS/USPSAddressVerifyV3.php` — patient address verification against USPS API (sends address data externally)
- `src/PaymentProcessing/Rainforest/Api.php`, `src/PaymentProcessing/Sphere/SphereRevert.php` — payment processing (not PHI)
- `src/Billing/EDI270.php` — insurance eligibility checks via clearinghouse
- **No LLM/AI API calls found in the codebase**

### 3.9 BAA Implications

- External calls: USPS address verification sends patient address to USPS API — requires BAA or must be disabled
- Product registration sends only email/version — no PHI
- Payment processors (Stripe, Rainforest, AuthorizeNet) are PCI-scoped, not HIPAA — no PHI transmitted
- CQM/QRDA reporting may transmit de-identified measure data

---

## 4. Performance

### 4.1 Key Table Indexes

**`patient_data`:**
- UNIQUE KEY `pid` (`pid`)
- UNIQUE KEY `uuid` (`uuid`)
- KEY `idx_patient_name` (`lname`, `fname`)
- KEY `idx_patient_dob` (`DOB`)
- KEY `id` (`id`)
- **Missing:** no index on `pubpid` (external MRN), `providerID`, `status`, `street`/`city`/`state` for geo searches

**`form_encounter`:**
- PRIMARY KEY `id`
- UNIQUE KEY `uuid`
- KEY `pid_encounter` (`pid`, `encounter`)
- KEY `encounter_date` (`date`)
- **Missing:** no index on `provider_id`, `facility_id`, `sensitivity`

**`billing`:**
- PRIMARY KEY `id`
- KEY `pid` (`pid`)
- **Missing:** no composite index on `(pid, encounter)`, no index on `activity`, `code_type` — critical for billing reports that always filter by these

**`form_soap`:**
- PRIMARY KEY `id` only — **no index on `pid`** — full table scan required for any per-patient SOAP note lookup

**`lists` (problems/allergies):**
- KEY `pid` (`pid`), KEY `type` (`type`)
- Missing composite `(pid, type)` index — every patient allergy/problem query hits separate indexes

### 4.2 N+1 Query Patterns

**`interface/reports/collections_report.php` (lines 790–975):**
- Main query retrieves all encounters matching criteria (unbounded by patient count if no date range)
- Inside the `while ($erow = sqlFetchArray($eres))` loop (line 790), per-encounter calls to `SLEOB::arGetPayerID()` (up to 3 times per encounter) and `getInsName()` — classic N+1 pattern
- No pagination on the main result set — could return thousands of rows

**`library/classes/Document.class.php` line 437:**
- `documents_factory()` fetches document IDs in one query but then instantiates `new Document($result->fields['id'])` per row — each construction triggers a separate DB query to load document metadata

**`interface/reports/daily_summary_report.php` lines 268–312:**
- Runs 4 separate unrelated queries (new patients, total visits, total payments, total paid amounts) that could be combined with CTEs or a single multi-join query

### 4.3 Unbounded Report Queries

- `interface/reports/collections_report.php` — main encounter query at line 786 has no LIMIT; filtered only by date range if user provides one; on large datasets this returns all encounters
- `src/Services/PatientService.php::getChartTrackerInformation()` at line 119 — no LIMIT, no WHERE clause — returns all chart tracker entries (potential full-table scan)
- `interface/reports/amc_full_report.php:82` — uses `SELECT * FROM report_itemized` with `LIMIT $index, $batchSize` (batched, but `$batchSize` not capped)

### 4.4 Patient Search

- `PatientService::getAll()` uses `FhirSearchWhereClauseBuilder` to build parameterized WHERE clauses
- Sort column whitelist via `ALLOWED_SORT_COLUMNS` array prevents SQL injection via `_sort` parameter
- Pagination via `QueryPagination` and `SearchQueryConfig` — pagination is supported but calling code must pass a config with a limit
- No full-text search index on `fname`, `lname` — uses `KEY idx_patient_name (lname, fname)` for prefix queries; `LIKE '%x%'` searches will not use the index

### 4.5 Document/File Access Patterns

- `documents_factory($foreign_id)` loads all documents for a patient, then instantiates each as a `Document` object triggering per-document queries — O(n) queries per patient document load
- CouchDB option exists but requires separate infrastructure

### 4.6 Constraints Relevant to AI/LLM Latency

- No built-in response caching layer (no Redis cache for clinical queries)
- `SessionUtil` supports Redis Sentinel for session storage but not query caching
- Largest latency risk: `collections_report.php` and `daily_summary_report.php` — unbounded aggregate queries
- FHIR bulk export: `src/FHIR/Export/ExportJob.php` — async export job, designed for large data sets
- API pagination is available but default page size not globally enforced at the middleware layer

---

## 5. Data Quality

### 5.1 Required vs Optional Fields in Key Tables

**`patient_data`:**
- `fname`, `lname`, `DOB`, `sex` — `NOT NULL DEFAULT ''` — technically required but empty string is valid (no DB-level enforcement beyond `NOT NULL`)
- `pid` — UNIQUE, auto-assigned
- `pubpid` — `NOT NULL DEFAULT ''` — external MRN; defaults to `pid` value if empty at insert time (`PatientService::databaseInsert()` line 183)
- `uuid` — `binary(16) DEFAULT NULL` — nullable; UUIDRegistry assigns on insert
- No DB-level `CHECK` constraints enforcing non-empty names or valid DOB format

**`form_encounter`:**
- `pid`, `encounter` — nullable (`DEFAULT NULL`), no NOT NULL constraint
- `date` — nullable
- `reason` — nullable longtext

**`lists` (problems/allergies/medications):**
- `type` — nullable varchar — differentiates allergies (`allergy`), problems (`medical_problem`), medications (`medication`)
- `diagnosis` — varchar(255), nullable — stores ICD/SNOMED codes as free text
- No enforced vocabulary constraint on `diagnosis` field

### 5.2 Duplicate Patient Risk

- `pubpid` (external MRN): **no UNIQUE constraint** in `patient_data` — multiple patients can have the same external MRN
- `pid` (internal ID): UNIQUE KEY enforced
- Duplicate detection: `interface/patient_file/manage_dup_patients.php` provides UI for finding/merging duplicates but does not prevent creation
- Merge tool at `interface/patient_file/merge_patients.php` — hard-deletes source records after merge; protected by `admin/super` ACL

### 5.3 Date Format Consistency

- `patient_data.DOB` stored as `date` SQL type — consistent
- `form_encounter.date` stored as `datetime` — consistent
- `form_soap.date` stored as `datetime`
- `lists.begdate` / `lists.enddate` stored as `datetime`
- Legacy string date handling: some legacy library code uses `date("Y-m-d H:i:s")` directly (not clock-injected) — creates timezone-sensitivity issues
- `PatientService::databaseInsert()` uses `date("Y-m-d H:i:s")` for `date` and `regdate` fields — not `DateTimeImmutable` or injected clock

### 5.4 Free-Text Note Structure

- SOAP notes: `form_soap` — four `text` fields (subjective, objective, assessment, plan) — unstructured prose
- Clinical notes: `form_clinical_notes` — categorized via `list_options` but note body is free text
- Patient notes: `pnotes.body` — `text` field — free text
- No NLP normalization, no section tagging — raw clinician text

### 5.5 Coding Systems Present

| System | Where | Notes |
|---|---|---|
| ICD-10 / ICD-9 | `billing`, `lists.diagnosis`, `codes` table | `codes.code_type` differentiates |
| CPT4 | `billing`, `codes` table | Used in fee sheets |
| SNOMED CT | `lists` (allergy/problem) | Via `list_options` + coded fields |
| LOINC | `categories` (document categories), `form_clinical_notes` | Category codes present |
| RxNorm | `prescriptions.rxnorm_drugcode` | Present, nullable |
| NDC | Potentially via drug tables | Not confirmed from schema alone |
| CVX | `sql/cvx_codes.sql` | Immunization vaccine codes |
| HCPCS | `codes` table | Via `code_type` |

ICD-9 legacy columns still present in `billing` table (`ICD9_01` through `ICD9_12`).

### 5.6 Medication/Problem/Allergy Data Reliability

**Allergies (`lists` WHERE `type='allergy'`):**
- `title` — varchar(255) — allergen name as text, no enforced vocabulary
- `diagnosis` — stores allergy code, nullable, no enforced format
- `reaction` — varchar(255) — free text reaction description
- `severity_al` — nullable varchar(50)
- `verification` — references `allergyintolerance-verification` list option — structured but optional

**Problems (`lists` WHERE `type='medical_problem'`):**
- `diagnosis` — ICD/SNOMED code in free text format — no lookup table FK constraint

**Medications (`prescriptions`):**
- `rxnorm_drugcode` — present and nullable — not enforced
- `drug` — varchar(150) free text name
- `active` — int, 1=active — not boolean typed

**Key data quality gap:** Allergy and problem codes are stored as free text in `lists.diagnosis` with no foreign key to a canonical code table — allows non-standard or malformed codes.

---

## 6. AI Clinical Co-Pilot Integration Points

The following are identified seams for an AI Clinical Co-Pilot — no design proposals, only observed facts about existing interfaces.

### 6.1 REST API Endpoints

All require OAuth2 Bearer token; APIs must be enabled via globals.

| Endpoint | Method | Notes |
|---|---|---|
| `/api/patient` | GET, POST, PUT | Search/create/update patients |
| `/api/patient/:puuid/encounter` | GET, POST | List/create encounters |
| `/api/patient/:pid/encounter/:eid/soap_note` | GET | Read SOAP notes |
| `/api/patient/:pid/encounter/:eid/vital` | GET, POST, PUT | Vitals |
| `/fhir/Patient` | GET | FHIR patient demographics |
| `/fhir/AllergyIntolerance` | GET | Allergy list |
| `/fhir/MedicationRequest` | GET | Medication orders |
| `/fhir/Condition` | GET | Problem list |
| `/fhir/Observation` | GET | Lab results and vitals |
| `/fhir/DocumentReference` | GET | Clinical documents |
| `/fhir/Encounter` | GET | Encounter list |
| `/fhir/DiagnosticReport` | GET | Lab reports |
| `/fhir/Immunization` | GET | Immunization records |
| `/fhir/CarePlan` | GET | Care plans |
| `/fhir/CareTeam` | GET | Care team members |
| `/fhir/Procedure` | GET | Procedure records |

FHIR Bulk Export available at `/fhir/Patient/$export` via `FhirOperationExportRestController` — async job, good for batch processing.

### 6.2 Event Hooks (Symfony EventDispatcher)

| Event | File | Trigger |
|---|---|---|
| `BeforePatientCreatedEvent` | `src/Events/Patient/BeforePatientCreatedEvent.php` | Before new patient insert |
| `PatientCreatedEvent` | `src/Events/Patient/PatientCreatedEvent.php` | After patient created |
| `PatientUpdatedEvent` | `src/Events/Patient/PatientUpdatedEvent.php` | After patient updated |
| `BeforePatientUpdatedEvent` | `src/Events/Patient/BeforePatientUpdatedEvent.php` | Before patient update |
| `ServiceSaveEvent` | `src/Events/Services/ServiceSaveEvent.php` | Before/after service-layer saves |
| `ServiceDeleteEvent` | `src/Events/Services/ServiceDeleteEvent.php` | Before/after service deletes |
| `EncounterButtonEvent` | `src/Events/Encounter/EncounterButtonEvent.php` | Encounter UI button render |
| `EncounterMenuEvent` | `src/Events/Encounter/EncounterMenuEvent.php` | Encounter menu items |
| `LoadEncounterFormFilterEvent` | `src/Events/Encounter/LoadEncounterFormFilterEvent.php` | Form load filter |
| `PatientReportFilterEvent` | `src/Events/PatientReport/PatientReportFilterEvent.php` | Patient report render |
| `PatientDemographics/RenderEvent` | `src/Events/PatientDemographics/RenderEvent.php` | Demographics page |
| `CDAPreParseEvent` / `CDAPostParseEvent` | `src/Events/CDA/` | CDA document import |
| `RestApiExtend` | `src/Events/RestApiExtend/` | API route extension |
| `SendNotificationEvent` | `src/Events/Messaging/SendNotificationEvent.php` | Outbound notifications |

Subscriber registration: via `src/Core/ModulesApplication.php` or module event listener registration.

### 6.3 Database Tables for Direct Query

| Table | Description |
|---|---|
| `patient_data` | Demographics, contact, provider assignment |
| `form_encounter` | Encounter records |
| `form_soap` | SOAP notes |
| `form_clinical_notes` | Structured clinical notes |
| `lists` | Allergies, problems, medications (differentiated by `type`) |
| `prescriptions` | Medication orders |
| `immunizations` | Vaccination history |
| `documents` | Document metadata (content on disk or CouchDB) |
| `billing` | Charges, CPT codes |
| `pnotes` | Patient messages/notes |
| `history_data` | Patient history |
| `form_vitals` | Vital signs |
| `openemr_postcalendar_events` | Appointments |
| `procedure_order`, `procedure_result` | Lab orders and results |
| `onotes` | Office notes |

### 6.4 SMART on FHIR / DSI Integration

- SMART App launch supported via `src/FHIR/SMART/SmartLaunchController.php`
- Decision Support Interventions (DSI): `src/Services/DecisionSupportInterventionService.php` + `src/FHIR/SMART/ExternalClinicalDecisionSupport/`
- Predictive DSI and Evidence DSI entity types supported via registered OAuth2 clients
- `dsi_source_attributes` table stores DSI configuration per client
- OAuth2 client scopes in `api_token` / `oauth_clients` tables

### 6.5 Module System

- `interface/modules/custom_modules/` — PHP/JS modules registered in `modules` table
- Module event listener pattern via `ModuleManagerListener.php` in each module
- Module hook points: `EncounterButtonEvent`, `EncounterMenuEvent`, `PatientDemographics/RenderEvent`, `TemplatePageEvent`

---

## 7. Risk Table

| Severity | Finding | Evidence | Affected Files | Recommendation |
|---|---|---|---|---|
| Critical | Hardcoded GitHub PAT tokens in docker-compose | `docker/development-easy/docker-compose.yml` lines 62-64 | `docker/development-easy/docker-compose.yml` | Rotate tokens immediately; use `${{secrets.}}` or `.env` files excluded from VCS |
| Critical | Exception message exposed in API error response | `apis/dispatch.php:38` — `'message' => $e->getMessage()` in JSON error body | `apis/dispatch.php` | Return generic error; log detail server-side only |
| High | Webhook receiver accepts unauthenticated POST requests | `interface/modules/custom_modules/oe-module-faxsms/library/webhook_receiver.php:25` — `$ignoreAuth = true`; no HMAC signature verification visible | `webhook_receiver.php` | Implement SignalWire webhook signature validation (HMAC-SHA1 header check) |
| High | SSN and drivers license stored in plaintext | `sql/database.sql` — `patient_data.ss varchar(255)`, `patient_data.drivers_license varchar(255)`; `PatientService.php` — no encrypt call | `sql/database.sql`, `src/Services/PatientService.php` | Encrypt at-rest using `CryptoGen::encryptStandard()` matching document encryption pattern |
| High | OAuth2 password grant enabled for all roles in dev compose | `docker/development-easy/docker-compose.yml:71` — `OPENEMR_SETTING_oauth_password_grant: 3` | `docker-compose.yml` | Verify production deploy has `oauth_password_grant = 0`; add documentation warning |
| High | `pubpid` (external MRN) has no UNIQUE constraint | `sql/database.sql` — `patient_data` table indexes — only `pid` and `uuid` are UNIQUE | `sql/database.sql`, `src/Services/PatientService.php` | Add `UNIQUE KEY pubpid (pubpid)` migration; handle duplicates at application layer |
| High | `form_soap` table has no index on `pid` | `sql/database.sql` — `form_soap` table has only PRIMARY KEY `id` | `sql/database.sql` | Add `KEY pid (pid)` to `form_soap` table; add `KEY pid (pid)` to other form tables similarly |
| High | Collections report main query has no row limit | `interface/reports/collections_report.php:786` — `ORDER BY f.pid, f.encounter` without LIMIT | `interface/reports/collections_report.php` | Add pagination or mandatory date range with indexed bounds |
| Medium | `billing` table missing composite index on `(pid, encounter, activity)` | `sql/database.sql` — billing only has `KEY pid (pid)` | `sql/database.sql` | Add `KEY billing_encounter (pid, encounter, activity)` for billing report performance |
| Medium | `form_clinical_notes` not in `EventAuditLogger::LOG_TABLES` | `src/Common/Logging/EventAuditLogger.php` — `LOG_TABLES` constant does not include `form_clinical_notes` | `src/Common/Logging/EventAuditLogger.php` | Add `form_clinical_notes` to `LOG_TABLES`; verify audit coverage for all clinical note types |
| Medium | Audit log encryption disabled by default | `library/globals.inc.php:2906` — `enable_auditlog_encryption = 0` | `library/globals.inc.php`, `src/Common/Logging/EventAuditLogger.php` | Enable audit log encryption in production deployments; document requirement |
| Medium | ~50 `interface/patient_file/` pages lack explicit ACL role check | `download_template.php`, `letter.php`, `label.php`, `barcode_label.php`, `pos_checkout.php`, etc. | `interface/patient_file/*.php` | Audit each file; add `AclMain::aclCheckCore()` for appropriate section/ACO; minimum authenticated-only access review |
| Medium | Patient name search via `LIKE '%x%'` will not use `idx_patient_name` index | `PatientService.php` — search builds parameterized WHERE; substring match bypasses B-tree index | `src/Services/PatientService.php`, `sql/database.sql` | Add `FULLTEXT INDEX` on `(fname, lname)` for substring patient search; or use prefix-only search |
| Medium | `USPS::addressVerify()` sends patient street address externally | `src/USPS/USPSAddressVerifyV3.php` — Guzzle POST to USPS API | `src/USPS/USPSAddressVerifyV3.php` | Requires BAA with USPS or must be disabled; document in deployment guide |
| Medium | Document encryption off by default | `library/classes/Document.class.php:990` — `storagemethod` conditionally encrypts; `drive_encryption` global defaults are not confirmed to be `1` | `library/classes/Document.class.php`, `library/globals.inc.php` | Enable `drive_encryption` by default; document this as required for HIPAA compliance |
| Medium | `patient_data.ss` visible in template and report output without masking | `library/report.inc.php:27` — SSN included in report field list; `library/custom_template/custom_template.php:326` — `listitemCode(xl('SSN'), $row['ss'])` | `library/report.inc.php`, `library/custom_template/custom_template.php` | Mask SSN output (show last 4 only) in UI; add display-layer protection |
| Low | `getChartTrackerInformation()` is an unbounded query | `src/Services/PatientService.php:119-131` — `SELECT ... FROM chart_tracker ... ORDER BY p.pubpid` with no LIMIT | `src/Services/PatientService.php` | Add LIMIT + pagination; this is a static method called in chart tracker views |
| Low | Angular 1.8.3 is end-of-life (EOL December 2021) | `package.json` — `angular: 1.8.3` | `package.json` | Migrate to modern framework; AngularJS EOL creates long-term security risk for XSS defenses |
| Low | `select2 4.0.13` — older version | `package.json` | `package.json` | Review for known CVEs; update to 4.1.x |
| Low | `adodb/adodb-php ^5.22.11` — legacy DB abstraction layer | `composer.json` | `composer.json` | Continue migration toward Doctrine DBAL / QueryUtils for new code |
| Low | Session cookie `httponly=false` for core OpenEMR session | `src/Common/Session/SessionUtil.php` — documented tradeoff | `src/Common/Session/SessionUtil.php` | Mitigated by SameSite=Strict; document risk; explore alternative multi-patient session approach |
| Low | `docker-compose.yml` enables XDEBUG and profiler by default in dev | `docker/development-easy/docker-compose.yml:56-57` | `docker-compose.yml` | Confirm production deploys do not include dev compose or have XDEBUG disabled |

---

## 8. Open Questions / Assumptions

### 8.1 Cannot Verify from Static Analysis

1. **Login brute-force lockout**: `AuthUtils::confirmUserPassword()` mentions login counter but lockout threshold behavior requires database inspection of `users_secure` table and runtime trace
2. **Session regeneration on login**: Assumed to happen but the exact `session_regenerate_id()` call location was not confirmed in the auth flow — needs runtime test to verify session fixation mitigation is active
3. **`drive_encryption` global default**: The Document class supports encryption but the global default value (`0` or `1`) was not confirmed from `globals.inc.php` — needs DB inspection
4. **CouchDB PHI exposure**: CouchDB is an optional document store; if enabled, it receives patient document blobs — a BAA with the CouchDB provider (if cloud-hosted) would be required
5. **Telemetry outbound data**: `TelemetryService` stores events in a local DB table but `TelemetryRepository` requires runtime inspection to confirm whether events are ever transmitted externally or remain local
6. **Webhook HMAC check**: `SignalWireWebhookValidator` handles input format validation but whether it performs SignalWire's HMAC-SHA1 request signing verification requires reading the full class (not just the receiver)
7. **Rate limiting**: No rate limiting was found in source code for login attempts or API endpoints — may be handled at infrastructure (nginx/Apache) level
8. **PHP error log path**: `bootstrap.php` sets `error_log = /dev/stdout` for Docker; production error log path and content filtering is environment-dependent
9. **Password grant production state**: Docker compose shows `oauth_password_grant: 3` for dev; production configuration not included in repo — cannot confirm production default
10. **`form_soap` encryption**: SOAP note content (`subjective`, `objective`, `assessment`, `plan`) is stored as plaintext `text` fields — no encryption wrapper found; at-rest encryption relies on DB-level or disk encryption only
11. **ICD-10 vs ICD-9**: Legacy ICD-9 columns present in `billing` table — confirm whether active ICD-10 workflow still writes to legacy ICD9 fields or whether they are truly unused
12. **Patient search performance under load**: B-tree index `idx_patient_name (lname, fname)` will handle exact-prefix searches but LIKE `%x%` queries will degrade — real performance profile requires production data volumes
13. **Module security review**: `interface/modules/custom_modules/` contains third-party modules (weno, faxsms, telehealth, comlink) — each module's security posture was not fully audited; modules run with full application context
14. **ATNA audit**: ATNA syslog is conditionally enabled (`enable_atna_audit`) — whether it is enabled in target deployment is unknown
15. **Redis Sentinel session storage**: An alternate session backend; if enabled, session data is stored externally — BAA/security controls for Redis host are environment-dependent
