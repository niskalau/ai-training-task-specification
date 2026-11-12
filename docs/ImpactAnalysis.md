# Impact Analysis — Form Availability Window (Start/End Date)

## 1. Request

> "Hey, we have some forms that should only be open for a limited time but right now there's no way to control that — people keep submitting to old forms. Would it be possible to add some kind of start/end date thing to forms?"

**Interpretation (working assumption, to be confirmed):** An Admin can give a form definition an optional *opens at* and/or *closes at* moment. Outside that window, the backend rejects new submissions to the form, and the UI tells users the form is not open.

**Underlying problem:** "People keep submitting to old forms." The date window is the customer's proposed solution. A manual "close now" switch or an archive state might solve the problem as well or better, so this must be confirmed (see Q1).

---

## 2. Impact Analysis

### 2.1 Database

| Change | Notes |
|---|---|
| `form` table: add `available_from` and `available_until` columns (nullable) | Null = no limit on that side. Type choice depends on timezone decision (§3.2): `TIMESTAMP WITH TIME ZONE` / `Instant` recommended over the plain `TIMESTAMP` the schema uses today. |
| Optional: `form.status` / `closed_manually` column | Only if the customer also wants a manual open/close toggle (Q1). |
| Migration of existing rows | Schema is managed by Hibernate `ddl-auto: update`. Nullable columns are added safely; existing forms stay "always open". No migration tool (Flyway/Liquibase) exists, which is worth noting if a non-null default or a backfill is ever needed. |
| `form_data` table | Probably no change. `submitted_at` is `TIMESTAMP` without time zone. If the window is compared against it in reports, the timezone question (§3.2) applies here too. |

### 2.2 Backend

| Area | Change |
|---|---|
| `entity/Form` | New fields `availableFrom`, `availableUntil`. |
| `dto/*` (form create/update/response DTOs) | New fields. Bean Validation: `availableUntil` must be after `availableFrom` (custom class-level constraint, see TechSpec §11.3). |
| `mapper/*` (MapStruct) | Map the new fields. Mostly automatic, but verify the type conversion. |
| `service/FormDataService` (submit) | **Core rule:** reject `POST /api/form-data/{formKey}` when `now` is outside the window. This must be enforced server-side (Req §4.3: the backend enforces rules, frontend checks are supplementary). |
| `service/FormDataService` (update) | Decide whether `PUT /api/form-data/submission/{id}` is also blocked after close (Q6). If so, look up the form by the submission's `formKey`. The form may have been deleted meanwhile (soft reference, TechSpec §13.1). |
| `service/FormService` | Validate the window on create/update. Optionally compute a derived `status` (`UPCOMING` / `OPEN` / `CLOSED`) so the frontend does not have to compare clocks itself. |
| Clock | Inject a `java.time.Clock` bean instead of calling `now()` directly, so the window logic is testable and timezone-explicit. |
| Exception handling (`@RestControllerAdvice`) | A new error case: the form is not open. Map it to a distinct status and error code (e.g. `409 Conflict` or `403` with `{"code":"FORM_NOT_OPEN", "availableFrom":…, "availableUntil":…}`) so the frontend can show a translated message. The backend has no i18n, so the frontend needs a machine-readable code. |
| `config/DatabaseInitializer` + `resources/forms/*.json` | The JSON loader must accept the new optional properties. Decide whether seed forms get dates. |
| `GET /api/forms` (public list) | Currently returns only `formKey` and `title`. Decide whether it should include the window/status, filter out closed forms, or stay unchanged (Q4, Q5). This endpoint is **public** (TechSpec §6.2), so it would expose schedule information to unauthenticated callers. |
| Logging | Log rejected out-of-window submissions at INFO/WARN without leaking submission data (Req §4.3). |

### 2.3 API Contract

- `POST /api/forms`, `PUT /api/forms/{key}`: request body gains `availableFrom` and `availableUntil` (ISO-8601, optional).
- `GET /api/forms/{key}`, `GET /api/forms/all`: responses include the new fields, plus `status` if one is computed.
- `GET /api/forms`: possibly gains `status` and dates (Q4).
- `POST /api/form-data/{formKey}`: new error response when the form is closed or not yet open.
- `PUT /api/form-data/submission/{id}`: possibly the same new error response (Q6).
- Document the agreed date format and timezone semantics in TechSpec §5.

### 2.4 Frontend

| Area | Change |
|---|---|
| `types/Form.ts` | Add `availableFrom?`, `availableUntil?`, `status?`. |
| `pages/EditForm.tsx` (form editor) | Two new date (or date-time) inputs, with "clear" support. Zod rule: end after start. Show the timezone the admin is entering in. |
| `pages/Forms.tsx` (list) | Show a badge (Open / Opens on … / Closed), and hide or disable closed and upcoming forms per Q4. Possibly show the deadline ("Closes 31 Dec 23:59"). |
| `pages/FormSubmission.tsx` | Before rendering: if not open, show an informative message instead of the form, or render the form disabled. On submit: handle the new backend error gracefully. The user may have had the form open when it closed, so their typed input must not be silently lost. |
| `pages/FormSubmission.tsx` (edit mode) | Hide or disable editing for closed forms if Q6 says edits are blocked. |
| `pages/FormSubmissions.tsx` | Hide or disable the **Edit** action per Q6. Optionally show the form status. |
| `components/common/DynamicForm.tsx` | May need a `disabled` / read-only mode. `ReadOnlyDynamicForm` may be reusable instead. |
| React Query caching | `['forms']` and `['form', key]` can be stale across a window boundary. Use a sensible `staleTime` and refetch on focus, and never rely on the cached status for enforcement. |
| Client clock | Do not trust the browser clock for the open/closed decision. Prefer the server-computed `status`, or the server time from the response. |
| i18n (`public/locales/*/translation.json`) | New keys: status labels, "form not yet open/closed" messages, editor labels and validation messages. Date and time formatting must be locale-aware (UC-12). |
| Routing | The routing table (TechSpec §3.4) has no route for editing an existing form definition even though UC-10 exists. Confirm where the date fields are edited (likely `/create-form` reused, or a missing `/forms/:key/edit`). |

### 2.5 Security & Authorization

- Enforcement must be in the backend service layer, not only in the UI.
- Decide whether Admins bypass the window (Q8). If so, the bypass must use the `ROLE_ADMIN` check, not a client flag.
- The public `GET /api/forms` endpoint: decide what availability data unauthenticated users may see.

### 2.6 Testing

| Level | Cases |
|---|---|
| Backend unit (service, fixed `Clock`) | Before start → rejected. Exactly at start → accepted. Inside → accepted. Exactly at end → per the agreed inclusive/exclusive rule. After end → rejected. Only start set. Only end set. Neither set. End before start → validation error. Edit of an existing submission after close. Admin bypass (if any). |
| Backend integration (MockMvc / Testcontainers) | API contract for the new fields; correct HTTP status and error code; PostgreSQL timezone round-trip (H2 and PostgreSQL may differ on `TIMESTAMP WITH TIME ZONE`). |
| `DatabaseInitializer` | Seed JSON with and without dates loads correctly. |
| Frontend unit (vitest) | Editor validation; list badges; closed-form message; handling of the backend "not open" error; locale-aware date display. |
| UI / E2E (Selenium) | Admin sets a window and the user sees the correct state; a user submits after the form has closed while it was open in the browser. |
| Regression | Existing forms without dates behave exactly as before. |

Coverage target > 70% (Req §4.4) applies to the new code.

### 2.7 Documentation

- **RequirementsSpecification.md**: §3.2 (form management: new properties and validation), §3.3 (submission restricted to the window), and possibly §6 (remove a limitation or add new ones).
- **UseCases.md**: UC-03 (list shows availability), UC-04 (new alternative flow: form not open), UC-07 (edit after close), UC-09/UC-10 (new editor fields and validation), and possibly a new use case if a manual close is added.
- **TechnicalSpecification.md**: §4.1 schema, §4.2 domain model, §5 API (fields and error responses), §9.2 translation keys.

---

## 3. Ambiguities and Edge Cases

### 3.1 Scope of the restriction
1. **Is a date window what is really needed?** The complaint is about "old forms". A manual close button, or archiving, may be what Admins actually want, either instead of dates or in addition to them.
2. **What exactly is blocked?** New submissions only, or also editing existing submissions (UC-07)? Viewing past submissions should almost certainly remain allowed.
3. **Does it apply to Admins?** Admins may need to enter late or paper submissions on someone's behalf, or test a form before it opens.
4. **Are both dates required, optional, or independent?** Is a start without an end allowed, or an end without a start?

### 3.2 Time semantics
5. **Date vs. date-time.** "End date 31 Dec" could mean closing at 00:00 or at 23:59:59. Is per-minute precision needed (e.g. "closes at 16:00")?
6. **Inclusive vs. exclusive** boundaries.
7. **Timezone.** Whose timezone applies: the organization's, the server's, or the user's browser? The current schema uses `TIMESTAMP` without a timezone. Users in different timezones would see different "deadlines". DST transitions may also matter.
8. **Clock source.** A user's device clock may be wrong. The server must be the authority.

### 3.3 Timing edge cases
9. **Form closes while a user is filling it in.** Does the user lose their input? Is there a grace period? Is there a warning before the deadline ("closes in 10 minutes")?
10. **Stale page or cache.** A user loaded the list while the form was open and submits after it closed (or the reverse for a not-yet-open form).
11. **Request in flight at the boundary.** A submission sent at 23:59:59.900 arrives at 00:00:00.100. Which timestamp counts: receipt time or something else?

### 3.4 Administration edge cases
12. **Changing dates after submissions exist.** Extending a deadline, reopening a closed form, or moving the start date later after people have already submitted. Is all of that allowed? Is it audited (the system has no audit trail today, TechSpec §13.5)?
13. **Setting a window entirely in the past** on create. Allowed (e.g. creating an archived form) or an error?
14. **Editing a form's fields while it is open.** This is already an issue because there is no versioning, but it becomes more visible with time-boxed campaigns.
15. **Recurring or multiple windows** (e.g. "open every January"). Assume not supported unless the customer asks.
16. **Seed forms** loaded from JSON at startup. Should they support dates? Does re-running the initializer overwrite dates that an Admin changed?

### 3.5 User-facing behaviour
17. **Visibility.** Should closed or upcoming forms be hidden from the list, shown as disabled, or shown with a status? Should the opening and closing times be shown to users?
18. **Direct links.** A user follows a bookmarked or shared link to a closed form. What do they see?
19. **Public endpoint.** `GET /api/forms` is unauthenticated. Is it acceptable to reveal schedule information there?
20. **Submissions list.** Should it indicate that the form is now closed, and hide the Edit action?
21. **Language.** Messages and date formats must work in all supported languages.

### 3.6 Adjacent features the customer may assume are included
22. Notifications to users or Admins when a form opens or closes.
23. A maximum number of submissions, or one submission per user.
24. Automatic deletion or archiving of submissions after close.
25. Reporting on late or rejected attempts.

---

## 4. Clarifying Questions for the Customer

*Written to be sent as-is. Where useful, a suggested default is given in brackets so the customer can simply confirm it.*

**The core need**

1. Is the main goal to stop submissions after a deadline, or would a simple "Close form now" button for admins also solve the problem? Would you want both? *(Suggestion: dates plus a manual close.)*
2. Should a form be able to have only a start date, only an end date, or both? Should forms without dates stay open indefinitely, as today? *(Suggestion: both optional; no dates = always open.)*

**Timing**

3. Do you need to set exact times (e.g. "closes at 16:00"), or are whole days enough? If whole days: does the end date mean the form is still open for the whole of that day? *(Suggestion: date and time; for whole days, open until 23:59 on the end date.)*
4. Which timezone should the dates follow? Are all your users in one timezone, or could they be in several? *(Suggestion: one organization-wide timezone, shown next to the deadline.)*
5. If someone is in the middle of filling in a form when it closes, what should happen? Options: reject the submission with a clear message, accept submissions started before the deadline, or allow a short grace period (how long?).
6. Would it help to show users a warning when a deadline is near (e.g. "This form closes in 1 hour")?

**What users see and can do**

7. Should forms that are not yet open, or already closed, be **hidden** from the forms list, or **shown with their status** (e.g. "Opens 1 March", "Closed")?
8. Should users be able to **edit** their existing submissions after the form has closed, or should submissions become locked at the deadline? They will always be able to *view* them. *(Suggestion: locked at the deadline.)*
9. Should users see the opening and closing times before they start filling in a form?
10. The forms list is currently visible without logging in. Is it OK for schedule information (opening and closing dates) to be visible there too, or should it only be shown to logged-in users?

**Admins**

11. Should admins be able to submit or edit submissions outside the window, e.g. to enter a late or paper submission or to test a form before it opens?
12. Should admins be able to change the dates after a form has received submissions, for example to extend a deadline or reopen a closed form? Do you need a record of who changed the dates and when?
13. Should the system prevent setting a closing date that is already in the past, or is that a legitimate way to close a form immediately?
14. Do the forms that are pre-loaded when the system is installed need dates as well?

**Related needs (to agree scope)**

15. Do you need notifications, for example emailing admins when a form closes or reminding users before a deadline? *(Currently out of scope.)*
16. Do you need a form to open and close on a repeating schedule (e.g. every January), or is a single period per form enough?
17. Are there other limits you would like at the same time, such as a maximum number of submissions or one submission per person?
18. When a form closes, should anything happen to its existing submissions (e.g. archiving or export), or should they simply remain as they are?

---

## 5. Notes on the Existing Specifications

These inconsistencies were found during the analysis and affect this feature:

- **`GET /api/forms` authentication:** TechSpec §5.3 and §6.2 say it is **public**; UC-03 says it **requires authentication**. This matters for Q10.
- **No route for editing a form definition:** UC-10 exists, but the routing table (TechSpec §3.4) has only `/create-form`. The date inputs need a home in both create and edit flows.
- **Checkbox options:** UC-09 says checkbox fields take options, while TechSpec §4.3 says checkbox is boolean with no options. This is unrelated to this feature but worth fixing.
- **Backend does not validate submission data against the form definition** (Req §6.8). The window check will be the first form-definition-based rule enforced on submission, so it establishes the pattern for looking up the form in `FormDataService`.
- **Timestamps are stored without a timezone** (`TIMESTAMP`). This needs a decision before adding time-based business rules.
