# Use Cases — Dynamic Form

## Overview

This document describes the use cases for the Dynamic Form application, a web-based system for creating, publishing, and submitting dynamic forms. Users interact with the application through a browser-based SPA secured with OAuth2/OIDC authentication.

### Actors

| Actor | Description |
|-------|-------------|
| **Unauthenticated User** | A visitor who has not yet logged in. Can only access the login flow and the language switcher. |
| **User** | Any authenticated person without an admin role. Can browse forms, submit forms, and manage their own submissions. |
| **Admin** | An authenticated person with the `ROLE_ADMIN` authority granted by the identity provider. Has all User capabilities plus exclusive rights to create, update, and delete form definitions, view all submissions across all users, and delete any submission. |

### Authorization Summary

| Operation | User | Admin |
|-----------|------|-------|
| Browse available forms | Yes | Yes |
| View a form definition | Yes | Yes |
| Submit a form | Yes | Yes |
| View own submissions | Yes | Yes |
| View all submissions | No | Yes |
| View own submission details | Yes | Yes |
| View any submission details | No | Yes |
| Edit own submission | Yes | Yes |
| Delete a submission | No | Yes |
| Create a form definition | No | Yes |
| Edit a form definition | No | Yes |
| Delete a form definition | No | Yes |

---

## UC-01: Log In

**Actor:** Unauthenticated User

**Preconditions:** The user is not logged in and has navigated to the application.

**Main Flow:**
1. The application detects that the user is not authenticated and displays a login prompt on the home page.
2. The user clicks the **Login** button in the navigation bar.
3. The application redirects the user to the OAuth2/OIDC identity provider (e.g., Keycloak) using the Authorization Code flow.
4. The user enters their credentials on the identity provider's login page.
5. Upon successful authentication, the identity provider redirects the user back to the application with an authorization code.
6. The application exchanges the code for an access token and stores it in memory.
7. The user is now authenticated. If the token contains the `ROLE_ADMIN` authority, the user gains Admin privileges; otherwise the User role applies.

**Postconditions:** The user is logged in. The navigation bar shows their identity and a logout button. All routes appropriate to their role are accessible.

**Alternative Flow — Authentication Failure:**
- If the identity provider rejects the credentials, the login page shows an error and the user remains unauthenticated.
- If the redirect back to the application fails or the token exchange fails, the application displays an authentication error message.

---

## UC-02: Log Out

**Actor:** User, Admin

**Preconditions:** The user is logged in.

**Main Flow:**
1. The user clicks the **Logout** button in the navigation bar.
2. The application clears the in-memory access token and invalidates all cached query data.
3. The application redirects the user to the identity provider's logout endpoint.
4. The identity provider ends the session and redirects the user back to the application's home page.
5. The home page shows the unauthenticated state with a login prompt.

**Postconditions:** The user's session is terminated. No access token remains in memory. Protected routes are no longer accessible.

---

## UC-03: Browse Available Forms

**Actor:** User, Admin

**Preconditions:** The user is logged in.

**Main Flow:**
1. The user navigates to the **Forms** page (default home page after login, also accessible via the navigation bar).
2. The application fetches the list of all published form definitions from the backend (`GET /api/forms`, requires authentication).
3. A list of forms is displayed, each showing the form title and description.
4. The user can select any form to proceed to submission.

**Postconditions:** The user sees all available forms and can choose one to fill out.

**Alternative Flow — No Forms Available:**
- If no form definitions exist, the page displays an empty state message.

---

## UC-04: Submit a Form

**Actor:** User, Admin

**Preconditions:** The user is logged in and has selected a form from the list.

**Main Flow:**
1. The user selects a form on the Forms page.
2. The application navigates to the form submission page and fetches the form definition for the selected form key (`GET /api/forms/{key}`, requires authentication).
3. The form is rendered dynamically with all fields defined in the form definition. Supported field types include: text, email, telephone, number, date, textarea, select (dropdown), radio buttons, and checkboxes.
4. The user fills in the fields. Required fields are marked and enforced.
5. The user clicks **Submit**.
6. The application validates all field values on the client side.
7. If validation passes, the application sends the form data to the backend (`POST /api/form-data/{key}`, requires authentication). The backend records the submission with the current timestamp and the authenticated user's username extracted from the JWT `preferred_username` claim.
8. On success, a confirmation message is displayed and the user is redirected to their submissions list.

**Postconditions:** A new form submission record is saved in the system, associated with the submitting user and the form.

**Alternative Flow — Validation Error:**
- If required fields are missing or field values do not meet the field's type constraints, inline error messages are shown beneath the offending fields. The form is not submitted until all errors are resolved.

**Alternative Flow — Server Error:**
- If the backend returns an error, an error alert is displayed on the form page. The user can correct their input and retry.

---

## UC-05: View Submissions

**Actor:** User, Admin

**Preconditions:** The user is logged in.

**Main Flow:**
1. The user navigates to the **Submissions** page via the navigation bar.
2. The application fetches submissions from the backend (`GET /api/form-data`, requires authentication).
   - If the user has `ROLE_ADMIN`, the backend returns **all submissions** across all users.
   - Otherwise, the backend returns only submissions where `submittedBy` matches the authenticated user's username.
3. The submissions are displayed in a table showing the form key, submission date, submitter (visible to Admin), and available actions.
4. **User** sees actions: **View**, **Edit** for each of their own submissions.
5. **Admin** sees actions: **View**, **Edit**, **Delete** for every submission.

**Postconditions:** The user sees the submissions they are authorized to access.

**Alternative Flow — No Submissions:**
- If no submissions exist for the user (or at all, for Admin), the table displays an empty state message.

---

## UC-06: View Submission Details

**Actor:** User (own submissions only), Admin (any submission)

**Preconditions:** The user is logged in.

**Main Flow:**
1. On the Submissions page, the user clicks the **View** action for a submission.
2. The application navigates to the submission detail page and fetches the submission record by ID (`GET /api/form-data/submission/{id}`, requires authentication).
3. The backend verifies authorization:
   - **Admin**: access always granted.
   - **User**: access granted only if `submittedBy` matches their username; otherwise the backend returns 403 Forbidden.
4. The submission is rendered in a read-only format showing all field labels and submitted values. Boolean values (checkboxes) are displayed as Yes/No. Dates are formatted for readability.
5. The submission date and submitter identity are shown.
6. The user can navigate back to the submissions list.

**Postconditions:** The user has reviewed the full content of the submission. No data is modified.

**Alternative Flow — Unauthorized Access:**
- If a User attempts to view a submission that belongs to another user (e.g., via a direct URL), the backend returns 403 Forbidden and the application shows an error.

---

## UC-07: Edit a Submission

**Actor:** User (own submissions only), Admin (any submission)

**Preconditions:** The user is logged in and the target submission exists.

**Main Flow:**
1. On the Submissions page, the user clicks the **Edit** action for a submission.
2. The application navigates to the form submission page in edit mode, loading both the form definition and the existing submission data.
3. The form is rendered with all fields pre-populated with the previously submitted values.
4. The user modifies the desired fields.
5. The user clicks **Submit**.
6. The application validates the updated values on the client side.
7. If validation passes, the application sends an update request to the backend (`PUT /api/form-data/submission/{id}`, requires authentication) along with the authenticated user's username.
8. On success, a confirmation message is displayed and the user is redirected to their submissions list.

**Postconditions:** The existing submission record is updated with the new field values.

**Alternative Flow — Validation Error:**
- Inline field errors are shown for any invalid values. The submission is not saved until all errors are resolved.

---

## UC-08: Delete a Submission

**Actor:** Admin

**Preconditions:** The Admin is logged in and the target submission exists.

**Main Flow:**
1. On the Submissions page, the Admin initiates the **Delete** action for a submission.
2. The application sends a delete request to the backend (`DELETE /api/form-data/submission/{id}`, requires `ROLE_ADMIN`).
3. On success, the submission is permanently removed and the table refreshes.

**Postconditions:** The submission record is permanently deleted from the system.

**Alternative Flow — Insufficient Privileges:**
- If a non-Admin user somehow triggers this action, the backend returns 403 Forbidden. The operation does not proceed.

---

## UC-09: Create a Form Definition

**Actor:** Admin

**Preconditions:** The Admin is logged in.

**Main Flow:**
1. The Admin clicks **Create Form** in the navigation bar.
2. The application navigates to the form editor page.
3. The Admin enters the following top-level properties:
   - **Form Key** — a unique identifier using only lowercase letters, numbers, and hyphens.
   - **Title** — a human-readable name for the form.
   - **Description** — a brief explanation of the form's purpose.
4. The Admin adds one or more fields using the **Add Field** button. For each field, the Admin specifies:
   - **Name** — an identifier starting with a letter, using alphanumeric characters only; must be unique within the form.
   - **Label** — the display text shown to users filling out the form.
   - **Type** — one of: text, email, tel, number, date, textarea, select, radio, checkbox.
   - **Required** — whether the field must be filled in.
   - **Placeholder** (optional) — hint text shown inside the input.
   - **Options** (for select, radio, and checkbox types) — a list of selectable values, with at least one option required.
5. The Admin may reorder fields using the up/down controls next to each field.
6. The Admin may remove a field using the remove control (at least one field must remain).
7. The Admin clicks **Save**.
8. The application validates the form definition on the client side.
9. If validation passes, the new form definition is sent to the backend (`POST /api/forms`, requires `ROLE_ADMIN`).
10. On success, the form is published and immediately available on the Forms list page for all authenticated users.

**Postconditions:** A new form definition is stored in the system and visible to all authenticated users.

**Alternative Flow — Duplicate Form Key:**
- If the chosen form key already exists, the backend returns an error and the editor displays a validation message. The Admin must change the form key and resubmit.

**Alternative Flow — Validation Error:**
- Missing required metadata, invalid field names, or missing options for select/radio types result in inline error messages. The form is not saved until all issues are resolved.

**Alternative Flow — Insufficient Privileges:**
- If a non-Admin user attempts to submit a new form definition, the backend returns 403 Forbidden.

---

## UC-10: Edit a Form Definition

**Actor:** Admin

**Preconditions:** The Admin is logged in and the target form definition exists.

**Main Flow:**
1. The Admin navigates to the form editor for an existing form.
2. The application fetches the existing form definition and pre-populates the editor with its current properties and fields.
3. The Admin modifies the title, description, fields, or field properties as needed, following the same rules as in UC-09.
4. The Admin saves the updated form definition.
5. The application validates and sends the update to the backend (`PUT /api/forms/{key}`, requires `ROLE_ADMIN`).
6. On success, the form definition is updated. All future submissions to this form will use the new field structure.

**Postconditions:** The form definition is updated in the system with the new configuration.

**Alternative Flow — Insufficient Privileges:**
- If a non-Admin user attempts this operation, the backend returns 403 Forbidden.

---

## UC-11: Delete a Form Definition

**Actor:** Admin

**Preconditions:** The Admin is logged in and the target form definition exists.

**Main Flow:**
1. The Admin triggers the delete action for a form definition.
2. The application sends a delete request to the backend (`DELETE /api/forms/{key}`, requires `ROLE_ADMIN`).
3. On success, the form definition is permanently removed. It no longer appears on the Forms list page.

**Postconditions:** The form definition is permanently deleted from the system. Existing submissions for that form key are unaffected.

**Alternative Flow — Insufficient Privileges:**
- If a non-Admin user attempts this operation, the backend returns 403 Forbidden.

---

## UC-12: Change Application Language

**Actor:** Unauthenticated User, User, Admin

**Preconditions:** The user is on any page of the application.

**Main Flow:**
1. The user opens the language selector in the navigation bar.
2. The user selects a preferred language from the available options.
3. The application switches all UI text, labels, error messages, and date formats to the chosen language without a full page reload.

**Postconditions:** The application is displayed in the selected language. The language preference is persisted for subsequent visits via the browser's language detection.
