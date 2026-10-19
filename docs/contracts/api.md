# API contract

- **Status:** Draft for team review
- **Primary owner:** Member 1 — Backend, Security and Integration Coordinator
- **Reviewers:** Member 2 for persistence and validation behavior; Member 3 for gate and recommendation response semantics

## Conventions

- The API is served by Flask under `/api/v1`. JSON request and response bodies use UTF-8.
- All data is synthetic. Responses must include an explicit prototype/synthetic-data disclaimer where results or overview data are presented.
- Authentication is server-side and session-based. The server derives the user ID, approved role, and organization context from the authenticated session and trusted server-side records. Client-supplied identity, role, or organization fields never establish authorization.
- Every protected route must authenticate the caller, authorize the requested action, and enforce organization scope before returning or changing data. Apply these checks to individual records, lists, aggregates, nested resources, and recommendation results.
- Cookie-authenticated state-changing requests require CSRF protection. The session-creation flow must also use an approved login-CSRF mitigation. CSRF tokens must not be accepted from untrusted sources or treated as proof of authentication.
- Protected data requires an active authenticated staff account. Enforce role checks and organization isolation on every request. Return `404` for an out-of-scope object to avoid disclosing its existence.
- Do not include passwords, password hashes, session secrets, restricted contact references, precise location, or unrelated organization data in API responses or logs.
- No route automatically contacts donors, dispatches staff, or authorizes a transfusion. Demo initialization/reset is not a public API operation and must be restricted to development mode.
- A configured compatibility source and reviewer are prerequisites for any compatibility pass. Missing or unverified configuration returns `not_configured`; it must not be replaced with locally invented blood-type rules.
- Authentication and authorization failures must not reveal whether an out-of-scope resource or account exists. Error messages and logs must not expose credentials, session tokens, secret configuration, or sensitive synthetic record fields.
- The API is a student prototype, not a clinically validated system. Successful HTTP responses do not establish clinical suitability or authorize contact, dispatch, or transfusion.

## Authentication and staff context

| Method and path | Access | Contract |
|---|---|---|
| `POST /api/v1/session` | Unauthenticated; login-CSRF mitigation required | Validate synthetic demo credentials, apply login throttling appropriate to the prototype, and establish a new authenticated session on success |
| `DELETE /api/v1/session` | Authenticated, CSRF-protected | Invalidate the server-side session and clear the session cookie using matching cookie attributes |
| `GET /api/v1/session` | Authenticated | Return the current user's opaque ID, approved role, and organization display context; never return credential data |

Login and logout errors use the common error envelope. Public self-registration is not provided.

Session requirements:

- Hash passwords using an established password-hashing implementation; never store or return plaintext passwords.
- Rotate or replace the session identifier after successful authentication. A pre-authentication session identifier must not remain the authenticated session identifier.
- Configure cookies with `HttpOnly`, an appropriate `SameSite` policy, and `Secure` when served over HTTPS. Do not expose session identifiers to client-side JavaScript.
- Define an explicit session lifetime and expiration behavior before implementation. Logout invalidates the session server-side; clearing a browser cookie alone is insufficient.
- Protect cookie-authenticated state-changing requests with CSRF validation. The session-creation route must use a separately specified login-CSRF mitigation.
- Resolve the login identifier unambiguously. Email is unique only within an organization under the data-model draft, so the team must approve whether login uses a synthetic global username, an organization-qualified identifier, or another non-ambiguous approach.
- Derive role and organization membership from trusted server-side account records on authenticated requests. Do not trust role or organization values submitted by the client.
- Use a generic authentication failure response that does not disclose whether a username, email, or organization exists.

The exact session lifetime, cookie policy for each deployment mode, login identifier, and login throttling behavior are unresolved configuration/security decisions. They must be agreed before the session implementation is accepted.

## Authorization and organization boundaries

The initial user categories are `hospital_staff` and `blood_bank_staff`. The exact permissions assigned to each category remain subject to team approval.

Before implementation, approve a role-to-action matrix covering at least:

| Action | Required contract decision |
|---|---|
| View organization-scoped overview | Which staff roles may access aggregate counts |
| Create an emergency request | Which roles may create requests for their organization |
| List and view emergency requests | Which roles may read each request and its recommendation results |
| Evaluate recommendations | Which roles may request an evaluation and view its results |
| Create or update eligibility verification | Whether this prototype supports this operation; if so, who is authorized to record verification |
| Update availability opt-in or withdrawal | Whether this prototype supports this operation; if so, who is authorized to record the update |
| Initialize or reset synthetic data | Development-only operator permission; never a public production-facing operation |

Rules:

- A role name alone does not grant access to another organization's records.
- The server obtains the caller's organization membership from trusted server-side state.
- Client-supplied `organization_id`, role, verifier identity, or equivalent authorization claims must not grant or broaden access.
- Cross-organization donor discovery is not authorized by this contract. If the team later approves a shared donor directory, it must define an explicit access model and update this contract before implementation.
- Return `404` for an out-of-scope resource where revealing its existence would disclose information. Use `403` for a known, in-scope action the authenticated user is not permitted to perform, where that distinction does not leak restricted information.
- Test authorization at both route and data-access boundaries, including list endpoints, aggregates, nested resources, and recommendation retrieval.

## Overview and emergency requests

| Method and path | Access | Contract |
|---|---|---|
| `GET /api/v1/overview` | Authenticated staff | Return organization-scoped synthetic-data counts and prototype/synthetic disclaimers; no donor contact or precise location |
| `POST /api/v1/emergency-requests` | Authorized authenticated staff, CSRF-protected | Validate and create an RBC-only request |
| `GET /api/v1/emergency-requests` | Authenticated staff | List only requests in the caller's organization, with pagination |
| `GET /api/v1/emergency-requests/{request_id}` | Authenticated staff | Return one organization-scoped request or `404` |
| `GET /api/v1/emergency-requests/{request_id}/recommendations` | Authenticated staff | Evaluate gates and return structured results as defined below |

Request creation accepts:

```json
{
  "component": "RBC",
  "requested_rbc_type": "<value supported by the configured compatibility source>",
  "urgency": "<value from the approved urgency configuration>"
}
```

The server supplies `created_by_user_id`, `organization_id`, and `created_at`. These values must not be accepted as authoritative client input.

Validation has two distinct layers:

1. **Structural validation:** Validate required fields, JSON types, `component == "RBC"`, permitted field names, and basic input shape. Reject malformed input with `400`.
2. **Configured policy validation:** Validate requested RBC-type and urgency values against the explicitly approved configuration. Reject unsupported values with `400` when the corresponding validator is configured and can evaluate them.

Missing compatibility configuration must not be replaced with invented blood-type rules. If structural validation succeeds but compatibility configuration is absent, request creation may store a syntactically valid request. Recommendation evaluation must then return `evaluation_status: "not_configured"` with no passing candidates.

If a required policy is unavailable, do not treat the value as validated merely because its string format appears valid. The response must make the unavailable policy explicit.

## Recommendation response

```json
{
  "request_id": "<opaque-id>",
  "evaluation_status": "not_configured",
  "disclaimer": "Synthetic-data student prototype; not clinically validated; not for real emergency medical decisions.",
  "gates": {
    "compatibility": {"status": "not_configured", "reason_code": "compatibility_not_configured"},
    "eligibility": {"status": "not_evaluated", "reason_code": "blocked_by_compatibility"},
    "availability": {"status": "not_evaluated", "reason_code": "blocked_by_compatibility"}
  },
  "candidates": [],
  "evaluated_at": "<UTC timestamp>"
}
```

`evaluation_status` describes the overall evaluation, not an individual gate:

- `not_configured`: a required evaluator or policy is missing or unverified. Return no candidates and identify the blocking gate.
- `completed`: every required gate policy was available and evaluated according to the recommendation contract. An empty `candidates` array means the evaluation completed but found no candidates passing every required gate.
- `failed`: an operational failure prevented a safe evaluation. Do not return candidates or fabricate defaults; use the common error envelope and an appropriate HTTP error.

Do not return `completed` when a required gate was skipped because its configuration was missing or unverified. Do not represent a database or unexpected operational failure as a valid empty result.

A completed evaluation may include aggregate gate counts and reason codes, but it must not disclose restricted donor fields. Candidate entries contain only the fields and explanations defined by the recommendations contract.

`evaluated_at` records when the evaluation attempt or result was recorded; it does not imply that every gate was evaluated.

Possible gate statuses are `pass`, `fail`, `not_configured`, and `not_evaluated`. For any request that cannot establish every required gate, return an empty candidate list. The recommendation contract defines reason codes and gate ordering.

## List pagination

List endpoints use a documented pagination format with a server-enforced maximum page size. The default page size, maximum page size, page parameter names, and out-of-range behavior must be agreed before implementation.

Pagination must not weaken organization filtering. Counts, next-page results, and any pagination metadata must describe only records the caller is authorized to access.

## Errors

All errors use:

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request contains invalid fields.",
    "field_errors": {"component": "Must be RBC."},
    "request_id": "<correlation-id>"
  }
}
```

`field_errors` is optional and contains safe, field-specific messages. Use consistent machine-readable codes, including `authentication_required`, `forbidden`, `not_found`, `validation_error`, `csrf_failed`, `compatibility_not_configured`, and `service_unavailable`. Do not return stack traces, database details, secrets, or success-shaped fallback data. Database failures produce a safe `503` response and a server-side diagnostic without credentials or sensitive record contents.

### Error response rules

- `400`: malformed JSON, structurally invalid input, or a configured validator rejecting an unsupported value.
- `401`: authentication is missing, invalid, or expired.
- `403`: the authenticated caller lacks permission for an in-scope action, where returning `403` does not reveal restricted information.
- `404`: the resource does not exist or is outside the caller's permitted scope.
- `500`: an unexpected server error, with no stack trace or internal detail exposed to the client.
- `503`: a required service or persistence dependency is unavailable and the request cannot be safely completed.

The team must agree on the status code for any additional domain-specific errors before implementation. A missing compatibility configuration during recommendation evaluation is a structured `not_configured` evaluation outcome, not a successful compatibility check.

`request_id` is a correlation identifier for the request. It must not contain a user ID, session token, credential, or other sensitive value. Include it in server diagnostics where practical, without logging sensitive record contents.

## Dependencies and unresolved decisions

- Depends on reviewed data-model and recommendation contracts.
- Member 1 owns route, session, CSRF, error, and organization enforcement implementation.
- Members 2 and 3 review data and recommendation payloads before implementation.
| Decision | Proposed owner(s) | Required before |
|---|---|---|
| Role-to-action matrix and overview permissions | Member 1; all affected members review | Protected-route implementation |
| Login identifier and account lookup behavior | Member 1; Member 2 reviews persistence constraints | Authentication implementation |
| Session lifetime, cookie attributes, and login-CSRF mitigation | Member 1 | Session implementation |
| Pagination parameter names, default, and maximum | Member 1; Members 2 and 3 review use cases | List endpoint implementation |
| Approved urgency values and validation behavior | Member 3 proposes; Member 1 reviews API validation | Request validation and ranking |
| Compatibility-source configuration interface and reviewer identity | Member 3 proposes; all members review; named reviewer must be confirmed | Any compatibility pass |
| Eligibility verification authority and expiry policy | Team decision; Member 3 implements gate behavior | Any eligibility pass |
| Availability freshness and withdrawal policy | Team decision; Members 2 and 3 review | Any availability pass |
| Cross-organization donor discovery | Administrator and all affected members | Any shared-directory access |

### API acceptance checks

- The recommendation example and prose use consistent evaluation statuses.
- Missing or unverified required gate configuration cannot produce a passing candidate.
- A completed evaluation with zero passing candidates is distinguishable from an evaluation that could not run.
- All protected operations derive identity, role, and organization context from trusted server-side state.
- Authorization tests cover single records, lists, aggregates, nested resources, and recommendation results.
- Session, CSRF, error-envelope, and pagination behavior are sufficiently specified to test.
- Responses expose no credentials, session tokens, precise location, restricted contact references, or unrelated organization records.
- No endpoint automatically contacts donors, dispatches staff, or authorizes transfusion.
