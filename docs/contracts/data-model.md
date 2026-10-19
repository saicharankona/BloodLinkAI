# Data model contract

- **Status:** Reviewed contract baseline — pending affected-member and administrator approval
- **Primary owner:** Member 2 — Database, Synthetic Data and Machine Learning
- **Reviewers:** Member 1 for identity, organization, and access boundaries; Member 3 for recommendation inputs and gate states

## Contract rules

- Every record used by the prototype, including development, tests, demos, and ML experiments, is synthetic. Do not import real donor, patient, health, contact, or location data.
- Persisted records are scoped to an `organization_id`. References between organization-owned records must belong to the same organization.
- MongoDB `_id` values are internal. API identifiers are opaque strings and must not expose database internals or unrelated organization records.
- Dates and times are UTC instants. A missing, unknown, or invalid value must not be converted into a passing gate state.
- Store only fields needed for the prototype. Contact and precise location fields are restricted and must not be included in general recommendation responses.
- This is an application data contract, not a clinical data standard. It does not define compatibility or eligibility rules.

## Collections

All fields below are required unless marked optional. Each organization-owned document has `_id` and `organization_id`. The `organizations` collection is the ownership root: its documents have `_id` but do not have an `organization_id` field.

### `organizations`

| Field | Type | Rule |
|---|---|---|
| `name` | string | Synthetic display name |
| `created_at` | UTC datetime | Creation time |
| `is_demo` | boolean | True for synthetic demo organizations |

### `staff_users`

| Field | Type | Rule |
|---|---|---|
| `email` | string | Synthetic login identifier; unique within an organization |
| `password_hash` | string | Secure password hash only; never store a plaintext password |
| `role` | string | `hospital_staff` or `blood_bank_staff`; role grants are enforced server-side |
| `is_active` | boolean | Inactive users cannot authenticate |
| `created_at` | UTC datetime | Creation time |

No public registration, password recovery, or administrator role is defined by this contract. Those require a separate approved requirement. Because `email` is unique only within an organization, the login identifier must be approved as either an organization-qualified email or another unambiguous synthetic identifier before authentication is implemented.

### `donors`

| Field | Type | Rule |
|---|---|---|
| `synthetic_label` | string | Fake display label; no real-person identifiers |
| `rbc_type` | string or null | Synthetic value interpreted only by the configured compatibility source; null/unknown cannot pass compatibility |
| `approx_location` | object or null | Generated approximate coordinates or an approved coarser representation; never real location data |
| `contact_reference` | string or null | Fake contact value/reference; restricted and not returned in ordinary API responses |
| `created_at` | UTC datetime | Creation time |

Contact access, if ever needed for a future workflow, requires explicit authorization and is outside the current recommendation contract. No contacting is performed.

### `emergency_requests`

| Field | Type | Rule |
|---|---|---|
| `created_by_user_id` | identifier | Same-organization staff user |
| `component` | string | Must be `RBC` |
| `requested_rbc_type` | string | Interpreted and validated by the explicitly configured compatibility source; the schema does not define blood-type rules |
| `urgency` | string | Allowed values and ranking semantics remain unresolved; validate against the approved configuration |
| `created_at` | UTC datetime | Creation time |

The contract does not define or store patient identity or patient records.

### `eligibility_verifications`

| Field | Type | Rule |
|---|---|---|
| `donor_id` | identifier | Same-organization donor |
| `status` | string | `verified`, `unverified`, `unknown`, or `expired` |
| `verified_by_user_id` | identifier or null | Same-organization authorized verifier when verified |
| `verified_at` | UTC datetime or null | Required for `verified` |
| `expires_at` | UTC datetime or null | Required to determine whether a verification is still current |
| `updated_at` | UTC datetime | Last status change |

Only explicitly verified, non-expired status can pass the eligibility gate. `verified_by_user_id` must refer to an active, same-organization staff user authorized under the approved role/action matrix. Verification authority, process, and expiry policy must be agreed before implementation. Donation history and model output are never eligibility evidence.

### `availability_opt_ins`

| Field | Type | Rule |
|---|---|---|
| `donor_id` | identifier | Same-organization donor |
| `status` | string | `opted_in`, `withdrawn`, or `unknown` |
| `updated_at` | UTC datetime | Last affirmative update or withdrawal |
| `valid_until` | UTC datetime or null | Freshness deadline; policy remains unresolved |

Only an affirmative opt-in that is not withdrawn and is within the approved freshness window can pass the availability gate. Historical responses do not imply current availability.

### `donation_history`

| Field | Type | Rule |
|---|---|---|
| `donor_id` | identifier | Same-organization donor |
| `occurred_at` | UTC datetime | Synthetic historical event time |
| `event_type` | string | Synthetic, non-clinical prototype event category; approved categories remain to be specified |

This history may be a ranking input only if its categories and use are reviewed. It never establishes eligibility, compatibility, or current availability.

### `historical_response_events` (optional ML dataset)

| Field | Type | Rule |
|---|---|---|
| `synthetic_donor_id` | identifier | Synthetic donor reference |
| `request_features` | object | Synthetic, pre-outcome features only |
| `responded` | boolean | Historical response target for the experiment |
| `recorded_at` | UTC datetime | Synthetic event time |

The feature set, cohort assumptions, and leakage checks require review before training. No real historical data is permitted. Predictions describe historical response behavior only.

## Validation and indexes

- Reject missing required fields, invalid types, unsupported component values, invalid references, and cross-organization references.
- Do not accept client-supplied `organization_id`, `verified_by_user_id`, or other authorization facts as authority; derive or validate them from the authenticated server-side context.
- Enforce uniqueness of staff email within an organization. Add organization-scoped indexes for donor, request, gate-status, and history lookups.
- Any freshness rule must compare UTC timestamps against an explicitly approved policy. Missing timestamps, missing expiry, and clock-invalid states fail closed.
- Synthetic initialization must be repeatable and development-only. Reset/initialization behavior and command details are an implementation decision; no public reset route is defined.

## Dependencies and unresolved decisions

- Member 2 implements the persistence representation after this contract is reviewed.
- Member 1 must confirm authentication, roles, organization context, and verification permissions with the team.
- Member 3 must confirm which fields are needed for gates/ranking and the minimum safe response shape.
- The API contract defines authentication/session behavior, role/action authorization, error envelopes, and evaluation-status semantics; this data contract does not override those rules.
- Unresolved: authoritative compatibility source and reviewer; eligibility verifier and expiry policy; availability freshness; urgency categories; location precision; donation-history categories; synthetic ML feature definitions; and whether any contact reference is needed at all.
