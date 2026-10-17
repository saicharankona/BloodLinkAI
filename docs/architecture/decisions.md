# Architecture decisions and integration plan

- **Status:** Draft for team review
- **Primary owner:** Member 1 — Backend, Security and Integration Coordinator
- **Reviewers:** Members 2 and 3 for decisions affecting their owned areas

This document records constraints and proposals for the first architecture review. It is not approval to begin implementation. All clinical and operational questions marked unresolved must be resolved or explicitly bounded before dependent behavior is implemented.

## Confirmed project constraints

- Intended use: student prototype for authorized hospital or blood-bank staff to review synthetic RBC donor recommendations.
- Approved stack: Python, Flask REST API, MongoDB/PyMongo, Pandas, NumPy, scikit-learn Random Forest for optional historical response prediction, Flask templates/HTML/CSS/JavaScript, and pytest.
- Synthetic records exclusively. Never import real donor, patient, health, contact, location, or historical response data.
- The prototype is not clinically validated and must not be represented as a real emergency decision system.
- No non-RBC products, automatic donor contact, dispatch, or transfusion authorization.
- Compatibility, verified eligibility, and fresh affirmative availability are separate hard gates. Unknown, unverified, stale, expired, or unconfigured required checks fail closed.
- Prediction describes historical response behavior only and cannot affect gate outcomes or establish current willingness.

## Runtime and dependency compatibility

The inspected runtime is Python **3.14.2**. No dependency manifest or application code exists in the inspected repository at this point. Before selecting dependency versions or adding a manifest, verify that the chosen releases of Flask, PyMongo, Pandas, NumPy, scikit-learn, and pytest support Python 3.14.2 and the project's supported platform. If a required library lacks compatible releases, report the constraint and request a team decision; do not silently substitute the approved stack or claim compatibility.

Configuration is environment-based. A future `.env.example` may contain placeholders only; actual secrets and local environment files must remain out of source control. No packages are installed as part of this contract batch.

## Proposed boundaries and ownership

| Area | Primary owner | Contract / dependency |
|---|---|---|
| Flask factory, shared config, API/errors, authentication, CSRF, roles, organization isolation, integration | Member 1 | API and data-model contracts |
| MongoDB client/repositories, representations, synthetic-data generation/initialization, ML experiment | Member 2 | Data-model contract; recommendation read contract |
| Compatibility/eligibility/availability orchestration, ranking/explanations, templates/static assets, recommendation QA | Member 3 | Recommendation contract and API response contract |

Each implementation file has one primary owner. Shared contracts, authorization behavior, configuration, and integration changes require review by affected members. Exact file-level ownership will be added when implementation paths are approved; no application files are established in this batch.

## Proposed integration order

1. Review and approve all four shared contracts together; resolve disagreements and record deferred questions.
2. Implement parallel foundations on separate feature branches: Member 1's Flask/auth/API foundation, Member 2's persistence/synthetic data, and Member 3's recommendation interfaces/dashboard foundation. Each owner adds focused tests.
3. Integrate repositories, routes, and recommendation gates without changing the approved contracts silently.
4. Add the isolated synthetic-only Random Forest experiment; compare to a baseline, check leakage, and record reproducible metadata. Keep recommendation behavior safe when the model is unavailable.
5. Integrate the authenticated dashboard and end-to-end synthetic workflow.
6. Run unit, API, integration, isolation, recommendation, ML, privacy, and dashboard tests; complete README setup and limitations after the contracts and implementation are approved.

The GitHub collaboration proposal is separate feature branches and reviewed pull requests merged in dependency order. Do not commit directly to `main` during parallel work. Each pull request includes tests and a change summary. This document batch creates no branches, commits, or pull requests.

## Security, privacy, and failure behavior

- Use password hashing, server-side authentication and authorization, organization-scoped database access, and CSRF protection for cookie-authenticated state changes.
- Minimize persisted and returned fields. Restrict contact/location information. Never log credentials, secrets, or sensitive record contents.
- Validate inputs and handle database errors explicitly. Do not silently fall back to unconfigured compatibility, eligibility, or availability.
- Restrict synthetic demo initialization and reset to development mode. Keep environment secrets, local artifacts, caches, and generated files out of version control.
- Do not claim production security, clinical validation, model effectiveness, or guaranteed outcomes.

## Unresolved decisions requiring review

1. Authoritative RBC compatibility source and named reviewer; configuration/update ownership.
2. Eligibility verification authority, allowed verification process, and expiration policy.
3. Definition and freshness duration of availability/opt-in, including withdrawal handling.
4. Staff role matrix and permitted operations across hospital and blood-bank organizations.
5. Supported urgency categories and their ranking semantics.
6. Distance method, location precision, and how synthetic coordinates are generated.
7. Donation-history event categories and permitted ranking contribution.
8. Whether to include the optional model, its synthetic features/target assumptions, and baseline.
9. Dependency releases compatible with Python 3.14.2 and the supported deployment platform.
10. Whether fake contact references are necessary; they are not required for recommendations and must not imply that the system contacts donors.

## Review and verification criteria

Before implementation begins, affected members review the four contracts and confirm:

- Data, API, and recommendation field names and states agree.
- Missing/unverified clinical or availability configuration cannot yield passing donors.
- `not_configured` is distinguishable from a completed evaluation with zero passing candidates.
- No compatibility mapping, eligibility criteria, freshness duration, ranking weights, or ML claims have been invented.
- Organization isolation, CSRF, minimized responses, synthetic-only data, and error behavior are testable.
- Ownership and dependency order are accepted, and unresolved decisions have an owner or explicit deferral.
