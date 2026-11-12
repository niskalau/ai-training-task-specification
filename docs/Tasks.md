# Task Checklist — Form Availability (Manual Close + Start/End Dates)

Related: [ImplementationPlan.md](ImplementationPlan.md) (§ references below point there) · [ImpactAnalysis.md](ImpactAnalysis.md)

**How to use this file**

- Check off a task (`- [x]`) in the same PR that completes it.
- Tracks within a phase can run in parallel. Each track lists its owner and what it depends on.
- ★ = critical path. A delay here delays the whole feature, so pick these up first.
- A phase is done only when all of its **Exit criteria** boxes are checked.
- Stick to the file ownership in [§9.5](ImplementationPlan.md#95-shared-files-and-ownership) (summarised at the end of this file).

**Critical path:** 0B → 1A-2 → 2A → 3A → 3D → 4A → 4B → Done

---

## Phase 0 — Alignment and contract freeze

**Entry criteria**

- [ ] ImpactAnalysis.md and ImplementationPlan.md reviewed by the team

### Track 0A — Open questions
*Owner: Product owner · Depends on: —*

- [ ] A3 answered: can admins reopen a form whose end date has passed?
- [ ] A4 answered: is an **Open now** action needed for upcoming forms?
- [ ] A6 answered: is an availability history UI wanted, or is storing it enough?
- [ ] If A4 = yes: add the **Open now** tasks to tracks 2A, 2B and 3A in this file
- [ ] Note the non-blocking assumptions (A1, A2, A5, A8) as accepted defaults for now

### Track 0B — Contract freeze ★
*Owner: Tech lead, backend, frontend · Depends on: —*

- [ ] Freeze request/response shapes (§5.1–§5.3)
- [ ] Freeze the error-body shape (§5.4)
- [ ] Freeze the full list of error codes: `FORM_NOT_YET_OPEN`, `FORM_CLOSED`, `END_BEFORE_START`, `END_IN_PAST`, `STATUS_CHANGE_NOT_ALLOWED`, `DATES_LOCKED_WHILE_CLOSED`
- [ ] Freeze the list of `availability.*` translation key names
- [ ] Decide on Flyway (§3.3) and record the decision
- [ ] Agree on file ownership (§9.5)

**Exit criteria**

- [ ] §5 marked **frozen** in ImplementationPlan.md (later changes need tech lead approval)
- [ ] A3, A4, A6 answered or the assumptions explicitly accepted
- [ ] Flyway decision recorded

---

## Phase 1 — Foundations

**Entry criteria**

- [ ] Phase 0 exit criteria met

### Track 1A — Time and schema ★
*Owner: Backend · Depends on: 0B · 1A-1 and 1A-2 can run in parallel; 1A-3 needs both*

**1A-1 Configuration and clock**
- [ ] `config/FormAvailabilityProperties`: `timezone`, `opening-time`, `closing-time`, `default-lifespan`, `deadline-warning`
- [ ] Default values in `application.yml` (`Europe/Helsinki`, `00:00`, `17:00`, `P10Y`, `PT24H`)
- [ ] `config/ClockConfig`: `Clock` bean in the configured zone

**1A-2 Schema and audit entity ★**
- [ ] `entity/Form`: `availableFrom`, `availableUntil`, `manuallyClosed`, `closedAt`, `closedBy` (§3.1)
- [ ] `entity/FormAvailabilityAudit` (§3.2)
- [ ] `FormAvailabilityAuditRepository`
- [ ] Verify the schema is created on H2 and PostgreSQL

**1A-3 Backfill**
- [ ] Idempotent startup backfill: `available_from = created_at` date (Helsinki), `available_until = +10 years` (§3.3)
- [ ] Backfill runs before `DatabaseInitializer`
- [ ] Verify running it twice changes nothing

### Track 1B — Frontend foundation
*Owner: Frontend · Depends on: 0B*

- [ ] `types/Form.ts`: `FormStatus`, `FormAvailability`, `availability` on `Form`
- [ ] `utils/availability.ts`: date formatting with `Intl.DateTimeFormat` (`Europe/Helsinki`, current locale)
- [ ] `utils/availability.ts`: `isDeadlineNear()` using the `serverTime` offset
- [ ] Unit tests for `availability.ts` (§7.3)
- [ ] `components/common/AvailabilityBadge.tsx` (Open / Opens {date} / Closed)
- [ ] `components/common/AvailabilityInfo.tsx` (dates + deadline warning banner)
- [ ] **All** `availability.*` translation keys in every `public/locales/*/translation.json`
- [ ] `services/formClient.ts`: `closeForm`, `reopenForm`, `getAvailabilityHistory`
- [ ] `services/formClient.ts`: parse error `code` from `409` and `400` responses
- [ ] Mocked API responses that follow the frozen contract
- [ ] Add the missing `/forms/:formKey/edit` → `EditForm` route in `Content.tsx`

### Track 1C — Test scaffolding
*Owner: QA / backend · Depends on: 0B*

- [ ] Fixed-`Clock` test helper for Europe/Helsinki
- [ ] Testcontainers PostgreSQL base setup
- [ ] E2E fixtures: admin and user accounts
- [ ] E2E helper: create a form with given dates

### Track 1D — Documentation draft
*Owner: Any · Depends on: 0B*

- [ ] UseCases.md: new UC-13 Close a Form
- [ ] UseCases.md: new UC-14 Reopen a Form
- [ ] UseCases.md: new UC-15 View Availability History
- [ ] TechnicalSpecification.md §5: endpoints, `availability` object, error codes
- [ ] TechnicalSpecification.md §6.2: new admin endpoints
- [ ] TechnicalSpecification.md §7: new configuration properties
- [ ] TechnicalSpecification.md §9.2: `availability.*` translation keys

**Exit criteria**

- [ ] App starts on H2 and PostgreSQL with the new columns and audit table
- [ ] Existing forms are backfilled; a second backfill run changes nothing
- [ ] All existing backend and frontend tests pass
- [ ] Frontend foundation (1B) merged; `availability.ts` unit tests pass
- [ ] Fixed-`Clock` helper and Testcontainers setup merged

---

## Phase 2 — Core logic and UI build

**Entry criteria** (each track starts as soon as its own dependencies are met)

- [ ] For 2A: 1A-1 and 1A-2 merged
- [ ] For 2B and 2C: 1B merged

### Track 2A — `FormAvailabilityService` ★
*Owner: Backend · Depends on: 1A-1, 1A-2, 1C, 0A (A3) · Can be split into 2A-1 and 2A-2 once method signatures are agreed*

- [ ] Agree method signatures (needed before splitting the work)

**2A-1 Status, defaults, submit and edit checks**
- [ ] `opensAt(form)` / `closesAt(form)`
- [ ] `computeStatus(form)` per §2.1 (manual close takes precedence)
- [ ] Default dates: missing start → today, missing end → start + 10 years
- [ ] `assertCanSubmit(form, auth)` with admin bypass
- [ ] `assertCanEditSubmission(form, auth)` with admin bypass; deleted form = `CLOSED`
- [ ] Unit tests: status boundaries (23:59:59 / 00:00 / 16:59:59 / 17:00:00, start = end)
- [ ] Unit tests: DST transitions in March and October
- [ ] Unit tests: defaults, including leap day + 10 years
- [ ] Unit tests: submit/edit checks (user rejected with correct code, admin bypass, deleted form)

**2A-2 Date-edit rules, close, reopen, audit**
- [ ] `validateDateChange(old, new)`: every row of the §3.4 table
- [ ] `close(key, user)`: idempotent, sets `closedAt` / `closedBy`
- [ ] `reopen(key, newUntil?, user)`: requires future end date if dates alone would leave it `CLOSED`
- [ ] Audit writes for `CREATED`, `DATES_CHANGED`, `CLOSED`, `REOPENED`
- [ ] Unit tests: every §3.4 row, allowed and rejected, incl. unchanged past end date on closed form
- [ ] Unit tests: manual close overrides an open window; reopen with and without new end date
- [ ] Unit tests: audit entries written for each action

**Exceptions (owned by 2A)**
- [ ] `FormNotAvailableException(code, opensAt, closesAt)` → **409** with error body
- [ ] Date-rule exception → **400** with `END_BEFORE_START`, `END_IN_PAST`, `STATUS_CHANGE_NOT_ALLOWED`, `DATES_LOCKED_WHILE_CLOSED`
- [ ] `@RestControllerAdvice` mappings

### Track 2B — Admin UI
*Owner: Frontend · Depends on: 1B, 0A (A4, A6) · Built against mocks*

- [ ] `EditForm`: two date inputs, prefilled with defaults
- [ ] `EditForm`: disable/limit inputs per current status (§3.4)
- [ ] `EditForm`: Zod rules mirroring the server rules
- [ ] `EditForm`: **Close now** button with in-page confirmation
- [ ] `EditForm`: **Reopen** button with confirmation; asks for a new end date when required
- [ ] `EditForm`: availability history panel
- [ ] `EditForm`: map server error codes to translated field errors
- [ ] Unit tests for `EditForm` (§7.3)

### Track 2C — User UI
*Owner: Frontend · Depends on: 1B · Built against mocks*

**`Forms` list**
- [ ] Switch to authenticated `GET /api/forms/all`
- [ ] Show badge and dates on each form
- [ ] Closed/upcoming forms not clickable for users; clickable for admins
- [ ] Admin shortcuts: Edit / Close / Reopen

**`FormSubmission` (new submission)**
- [ ] Show `AvailabilityInfo` with deadline banner
- [ ] Users: "not yet open" / "closed" message instead of the form
- [ ] Admins: form shown with "outside availability window" notice
- [ ] On `409`: translated message, **do not reset** the form, refresh availability

**`FormSubmission` (edit mode)**
- [ ] Same rules; users see "submissions are locked because the form has closed"

**`FormSubmissions`**
- [ ] Look up form status from the `['forms']` query
- [ ] Hide **Edit** for users when the form is not `OPEN`; show lock indicator

**React Query**
- [ ] Invalidate `['forms']` and `['form', key]` after edit, close, reopen
- [ ] Short `staleTime` and `refetchOnWindowFocus` for form queries

**Tests**
- [ ] Unit tests for `Forms`, `FormSubmission`, `FormSubmissions` (§7.3)

### Track 2D — Documentation
*Owner: Any · Depends on: 1D*

- [ ] RequirementsSpecification.md §3.2: availability dates, Close/Reopen, audit
- [ ] RequirementsSpecification.md §3.3: submission and edit restrictions, admin bypass
- [ ] RequirementsSpecification.md §3.4: status display, deadline warning
- [ ] UseCases.md UC-03: status badges; fix the `GET /api/forms` authentication note
- [ ] UseCases.md UC-04 and UC-07: "form not open" and admin override flows
- [ ] UseCases.md UC-09 and UC-10: date fields and edit rules

**Exit criteria**

- [ ] All §7.1 unit tests pass (every §3.4 row, 00:00 / 17:00 boundaries, DST cases)
- [ ] Admin and user screens complete against mocks; their §7.3 unit tests pass
- [ ] Frontend uses only endpoints and fields from the frozen contract

---

## Phase 3 — API integration

**Entry criteria**

- [ ] 2A merged (needed by 3A, 3B, 3C)
- [ ] For 3D: 2B and 2C merged, and the first 3A endpoint merged

### Track 3A — Form API ★
*Owner: Backend · Depends on: 2A · Merge endpoint by endpoint so 3D can start early*

- [ ] Request DTOs: optional `availableFrom` / `availableUntil` (`LocalDate`)
- [ ] Response DTOs (authenticated endpoints): `availability` object (§5.3), incl. `serverTime`
- [ ] Public list DTO left unchanged
- [ ] Mappers: map new fields; status/instants come from the service
- [ ] `FormService`: apply defaults on create and update
- [ ] `FormService`: call `validateDateChange`; write audit entry on date change
- [ ] `POST /api/forms/{key}/close`
- [ ] `POST /api/forms/{key}/reopen`
- [ ] `GET /api/forms/{key}/availability-history` (newest first)
- [ ] `SecurityConfig`: new endpoints restricted to `ROLE_ADMIN`
- [ ] Integration tests: `availability` object on `GET /api/forms/{key}` and `/all`
- [ ] Integration tests: public `GET /api/forms` does **not** expose availability
- [ ] Integration tests: `400` date-rule codes on create/update
- [ ] Integration tests: `403` for non-admin close, reopen, history
- [ ] Integration tests: `DATE` / `TIMESTAMP WITH TIME ZONE` round-trip on PostgreSQL and H2

### Track 3B — Submission enforcement
*Owner: Backend · Depends on: 2A*

- [ ] `FormDataService` submit: `assertCanSubmit`
- [ ] `FormDataService` update: look up form by `formKey`, then `assertCanEditSubmission`
- [ ] Deleted form counts as `CLOSED` for non-admins
- [ ] INFO log for rejected attempts (form key + username, no submission data)
- [ ] Integration tests: `409` on submit (`FORM_NOT_YET_OPEN`, `FORM_CLOSED`) with error body
- [ ] Integration tests: `409 FORM_CLOSED` on edit; admin bypass on both

### Track 3C — Seed data
*Owner: Backend · Depends on: 2A, 1A-3*

- [ ] `resources/forms/*.json`: support optional `availableFrom` / `availableUntil`
- [ ] `DatabaseInitializer`: apply defaults to seeds
- [ ] `DatabaseInitializer`: insert only if the key does not exist (verify current behaviour)
- [ ] Backfill runs before seeds are loaded
- [ ] Initializer tests: seeds with and without dates; existing forms not overwritten

### Track 3D — Frontend wiring and E2E scripts ★
*Owner: Frontend, QA · Depends on: 2B, 2C, first 3A endpoint · Finishes after 3A and 3B*

- [ ] Wire `EditForm` (dates, close, reopen, history) to real endpoints
- [ ] Wire `Forms`, `FormSubmission`, `FormSubmissions` to real endpoints
- [ ] Fix any differences from the contract
- [ ] Remove mocks from the running app
- [ ] E2E script: admin creates form with future dates → user sees "Opens …" and cannot submit
- [ ] E2E script: admin **Close now** while user has form open → user gets error, input intact
- [ ] E2E script: admin reopens with new end date → user can submit again
- [ ] E2E script: user cannot edit a submission after close; admin can
- [ ] E2E scripts run locally

**Exit criteria**

- [ ] All §7.2 integration tests pass on PostgreSQL and H2
- [ ] Public `GET /api/forms` response verified unchanged
- [ ] Frontend runs against the real backend without mocks; all frontend unit tests pass
- [ ] §7.4 E2E scripts written and run locally

---

## Phase 4 — Verification and close-out

**Entry criteria**

- [ ] Phase 3 exit criteria met
- [ ] Feature deployed to the test environment

### Track 4A — E2E ★
*Owner: QA (developers fix defects)*

- [ ] Run all §7.4 scenarios in the test environment
- [ ] Log defects
- [ ] All defects fixed and re-tested

### Track 4B — Regression and coverage ★
*Owner: QA, backend, frontend · Can start in parallel with 4A; final pass after the last 4A fix*

- [ ] Regression UC-01 to UC-12 on backfilled data (forms in `OPEN` state behave as before)
- [ ] Backend coverage of new code > 70%
- [ ] Frontend coverage of new code > 70%
- [ ] Final regression pass after the last 4A fix

### Track 4C — Documentation close-out
*Owner: Any*

- [ ] Align RequirementsSpecification.md with delivered behaviour
- [ ] Align UseCases.md with delivered behaviour
- [ ] Align TechnicalSpecification.md with delivered behaviour
- [ ] Record the final answer for each of A1–A8 (§10):
  - [ ] A1 Opening time (00:00)
  - [ ] A2 Deadline warning threshold (24 h)
  - [ ] A3 Reopening a date-expired form
  - [ ] A4 Open now action
  - [ ] A5 10-year default from start date
  - [ ] A6 Audit history UI
  - [ ] A7 Submission limits out of scope
  - [ ] A8 Weekends and holidays

**Exit criteria — Definition of Done**

- [ ] All unit, integration and E2E tests pass
- [ ] Coverage of new code > 70%
- [ ] Documentation changes (§8) merged
- [ ] Each §10 item confirmed or explicitly accepted
- [ ] Product owner has seen a demo and accepted the feature

---

## Quick reference: shared file ownership (§9.5)

| File | Owner | Others |
|---|---|---|
| `entity/Form` | 1A | No changes after Phase 1 |
| Exception classes, `@RestControllerAdvice` | 2A | 3A/3B use, don't change |
| `service/FormService` / `service/FormDataService` | 3A / 3B | — |
| `config/SecurityConfig` | 3A | No other track |
| `types/Form.ts`, `services/formClient.ts` | 1B | Bug fixes only |
| `public/locales/*/translation.json` | 1B | Text changes only |
| `pages/FormSubmission.tsx` | 2C | 2B doesn't change it |
| `Content.tsx` (routing) | 1B | No other track |
