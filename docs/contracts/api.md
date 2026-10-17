# API contract

- **Status:** Draft for team review
- **Primary owner:** Member 1 — Backend, Security and Integration Coordinator
- **Reviewers:** Member 2 for persistence and validation behavior; Member 3 for gate and recommendation response semantics

## Conventions

- The API is served by Flask under `/api/v1`. JSON request and response bodies use UTF-8.
- All data is synthetic. Responses must include an explicit prototype/synthetic-data disclaimer where results or overview data are presented.
- Authentication is server-side and session-based. Cookie-authenticated state-changing requests require CSRF protection. The server derives organization context from the authenticated session; clients cannot select another organization.
- Protected data requires an active authenticated staff account. Enforce role checks and organization isolation on every request. Return `404` for an out-of-scope object to avoid disclosing its existence.
- Do not include passwords, password hashes, session secrets, restricted contact references, precise location, or unrelated organization data in API responses or logs.
- No route automatically contacts donors, dispatches staff, or authorizes a transfusion. Demo initialization/reset is not a public API operation and must be restricted to development mode.
- A configured compatibility source and reviewer are prerequisites for any compatibility pass. Missing or unverified configuration returns `not_configured`; it must not be replaced with locally invented blood-type rules.

## Authentication and staff context

| Method and path | Access | Contract |
|---|---|---|
| `POST /api/v1/session` | Public | Accept synthetic demo credentials; establish an authenticated session on success |
| `DELETE /api/v1/session` | Authenticated, CSRF-protected | Invalidate the session and clear its cookie |
| `GET /api/v1/session` | Authenticated | Return the current user's opaque ID, approved role, and organization display context; never return credential data |

Login and logout errors use the common error envelope. Public self-registration is not provided.

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

The server supplies `created_by_user_id`, `organization_id`, and `created_at`. Unsupported values are rejected with `400` when a configured validator can evaluate them. If compatibility configuration is absent, request creation may store a syntactically valid request, but recommendation evaluation must explicitly report `not_configured` and return no passing donors.

## Recommendation response

```json
{
  "request_id": "<opaque-id>",
  "evaluation_status": "completed",
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

`evaluation_status` distinguishes an evaluation that could not run because required configuration is missing (`not_configured`) from a completed evaluation with zero passing candidates (`completed` and an empty `candidates` array). A completed evaluation may include aggregate gate counts and reason codes; it must not disclose restricted donor fields. Candidate entries contain only the fields and explanations defined by the recommendations contract.

Possible gate statuses are `pass`, `fail`, `not_configured`, and `not_evaluated`. For any request that cannot establish every required gate, return an empty candidate list. The recommendation contract defines reason codes and gate ordering.

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

## Dependencies and unresolved decisions

- Depends on reviewed data-model and recommendation contracts.
- Member 1 owns route, session, CSRF, error, and organization enforcement implementation.
- Members 2 and 3 review data and recommendation payloads before implementation.
- Unresolved: exact role-to-action matrix, session lifetime and cookie settings, pagination bounds, urgency values, compatibility-source configuration interface, and whether overview counts need more restrictive role access.
