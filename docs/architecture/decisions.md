# Architecture decisions and integration plan
- Status: Draft for team review
- Primary owner: Member 1 — Backend, Security and Integration Coordinator
- Reviewers: Members 2 and 3 for decisions affecting their owned areas
This document records constraints and proposals for the first architecture review. It is not approval to begin implementation. All clinical and operational questions marked unresolved must be resolved or explicitly bounded before dependent behavior is implemented.
Decision status and change control
Use these labels consistently:
- Confirmed constraint: Explicitly established by PROJECT_SCOPE.md or the approved project instructions.
- Proposed: Recommended for team review; not yet approved.
- Unresolved: Requires a named decision or an explicit deferral.
- Approved: Explicitly accepted by the administrator and affected reviewers, with the decision recorded here.
The existence of a proposal in this document is not authorization to implement it. No member may silently change a shared contract, authorization boundary, safety gate, or integration interface. Contract changes must be reviewed by the affected owners before implementation.
The project administrator coordinates decisions and integration, but a clinical or operational assumption must not be treated as approved merely because it is convenient to implement.
Confirmed project constraints
- Intended use: student prototype for authorized hospital or blood-bank staff to review synthetic RBC donor recommendations.
- Approved stack: Python, Flask REST API, MongoDB/PyMongo, Pandas, NumPy, scikit-learn Random Forest for optional historical response prediction, Flask templates/HTML/CSS/JavaScript, and pytest.
- Synthetic records exclusively. Never import real donor, patient, health, contact, location, or historical response data.
- The prototype is not clinically validated and must not be represented as a real emergency decision system.
- No non-RBC products, automatic donor contact, dispatch, or transfusion authorization.
- Compatibility, verified eligibility, and fresh affirmative availability are separate hard gates. Unknown, unverified, stale, expired, or unconfigured required checks fail closed.
- Prediction describes historical response behavior only and cannot affect gate outcomes or establish current willingness.
Architecture decision register
ADR-001 — Approved technology stack
- Status: Confirmed constraint.
- Decision: Python, Flask REST API, MongoDB through PyMongo, Pandas, NumPy, optional scikit-learn Random Forest, Flask templates with HTML/CSS/JavaScript, and pytest.
- Constraints: Do not replace the stack without explicit team approval. Verify dependency and runtime compatibility before selecting versions or adding a dependency manifest.
- Owner: Member 1 coordinates backend compatibility; Members 2 and 3 verify their owned dependency needs.
ADR-002 — Application and persistence boundaries
- Status: Proposed.
- Proposal: Keep HTTP routing, authentication/authorization, persistence, and recommendation orchestration as distinct responsibilities with explicit interfaces. Member 1 owns Flask/API/security foundations; Member 2 owns MongoDB client/repositories and synthetic-data initialization; Member 3 owns recommendation orchestration and dashboard behavior.
- Dependency: Approved data-model, API, and recommendation contracts.
- Approval needed: Agree on concrete interfaces and file ownership before application files are created.
ADR-003 — Authentication and organization authorization
- Status: Unresolved.
- Required behavior: Use server-side authentication, secure password hashing, CSRF protection for cookie-authenticated state changes, server-derived role and organization context, and organization-scoped data access.
- Unresolved: Login identifier, session lifetime, cookie settings, exact role-to-action matrix, and whether a shared donor directory is permitted.
- Owner: Member 1 proposes the security design; Members 2 and 3 review persistence and feature implications.
- Acceptance: Tests demonstrate that client-supplied role or organization values cannot grant access to protected data.
ADR-004 — Recommendation safety gates
- Status: Confirmed constraint; implementation policy details unresolved.
- Decision: Compatibility, verified eligibility, and fresh affirmative availability are independent hard gates. Ranking occurs only after every required gate passes.
- Fail-closed behavior: Missing, unverified, unknown, expired, withdrawn, stale, or otherwise invalid required information cannot pass its gate.
- Prohibited behavior: Do not invent compatibility rules or use model predictions or donation history as evidence of compatibility, eligibility, or current willingness.
- Unresolved: Authoritative RBC compatibility source and reviewer, eligibility verification authority and expiry, and availability freshness policy.
- Owner: Member 3 owns gate orchestration; Member 1 reviews API and error semantics; Member 2 reviews data representation.
ADR-005 — Optional historical-response model
- Status: Confirmed constraint for model boundaries; inclusion remains unresolved.
- Decision: If included, the Random Forest predicts historical response behavior using synthetic records only. It does not determine compatibility, eligibility, current availability, or guaranteed donation.
- Fallback: Recommendation behavior must remain safe if the model is disabled, unavailable, or untrained.
- Unresolved: Whether the model is included in the first prototype, the approved synthetic feature/target definitions, evaluation split, baseline, and acceptance metrics.
- Owner: Member 2 owns training/evaluation; Member 3 reviews how predictions are presented; Member 1 reviews integration behavior.
ADR-006 — Configuration and secrets
- Status: Confirmed constraint.
- Decision: Configuration is environment-based. A future .env.example contains placeholders only. Real secrets and local environment files must not be committed.
- Unresolved: Exact configuration keys and validation behavior will be specified with the relevant implementation interfaces.
- Owner: Member 1 coordinates shared configuration; affected members review required settings.
ADR-007 — Git collaboration and integration
- Status: Confirmed workflow constraint.
- Decision: Use feature/backend-api, feature/database-data, and feature/ml-recommendation for the respective member work. Use isolated working directories or separate Codespaces and reviewed pull requests. Do not commit directly to main during parallel development.
- Safety: No unapproved merges, destructive Git commands, or overwriting another member's work.
- Owner: Project administrator coordinates integration; each member owns and verifies their own branch changes.
Runtime and dependency compatibility
The inspected runtime is Python 3.14.2. No dependency manifest or application code exists in the inspected repository at this point. Before selecting dependency versions or adding a manifest, verify that the chosen releases of Flask, PyMongo, Pandas, NumPy, scikit-learn, and pytest support Python 3.14.2 and the project's supported platform. If a required library lacks compatible releases, report the constraint and request a team decision; do not silently substitute the approved stack or claim compatibility.
Configuration is environment-based. A future .env.example may contain placeholders only; actual secrets and local environment files must remain out of source control. No packages are installed as part of this contract batch.
Proposed boundaries and ownership
Area	Primary owner	Contract / dependency
Flask factory, shared config, API/errors, authentication, CSRF, roles, organization isolation, integration	Member 1	API and data-model contracts
MongoDB client/repositories, representations, synthetic-data generation/initialization, ML experiment	Member 2	Data-model contract; recommendation read contract
Compatibility/eligibility/availability orchestration, ranking/explanations, templates/static assets, recommendation QA	Member 3	Recommendation contract and API response contract


Each implementation file has one primary owner. Shared contracts, authorization behavior, configuration, and integration changes require review by affected members. Exact file-level ownership will be added when implementation paths are approved; no application files are established in this batch.
Proposed integration order
1. Review and approve all four shared contracts together; resolve disagreements and record deferred questions.
2. Implement parallel foundations on separate feature branches: Member 1's Flask/auth/API foundation, Member 2's persistence/synthetic data, and Member 3's recommendation interfaces/dashboard foundation. Each owner adds focused tests.
3. Integrate repositories, routes, and recommendation gates without changing the approved contracts silently.
4. Add the isolated synthetic-only Random Forest experiment; compare to a baseline, check leakage, and record reproducible metadata. Keep recommendation behavior safe when the model is unavailable.
5. Integrate the authenticated dashboard and end-to-end synthetic workflow.
6. Run unit, API, integration, isolation, recommendation, ML, privacy, and dashboard tests; complete README setup and limitations after the contracts and implementation are approved.
The GitHub collaboration proposal is separate feature branches and reviewed pull requests merged in dependency order. Do not commit directly to main during parallel work. Each pull request includes tests and a change summary. This document batch creates no branches, commits, or pull requests.
Batch gates and integration acceptance
Batch 1 — Shared contracts
Do not begin application implementation until all four shared contracts have been reviewed and approved by the administrator and the affected members.
The Batch 1 review must confirm:
- The data model, API, and recommendation contracts use consistent field names, gate statuses, evaluation statuses, and reason codes.
- The API distinguishes an unconfigured evaluation from a completed evaluation with no passing candidates.
- The role/action matrix and organization-access policy are approved or explicitly deferred with dependent behavior disabled.
- Compatibility, eligibility, and availability policies are not invented or inferred from unrelated fields.
- Ranking inputs, missing-value behavior, and optional model boundaries are documented sufficiently for implementation.
- Every unresolved decision has a lead and a safe implementation boundary.
Batch 2 — Parallel foundations
- Member 1 implements the approved Flask, configuration, session, authorization, CSRF, and error foundations.
- Member 2 implements the approved persistence representation and synthetic-data foundation.
- Member 3 implements recommendation interfaces and dashboard foundations against the approved contracts.
- Each owner adds tests for their own files and reports actual test outcomes.
- Shared-contract changes require review by the affected owners; implementation must not silently redefine the contract.
Pull-request acceptance
Before a pull request is considered integration-ready, require:
1. A change summary and clear file ownership.
2. Tests actually executed, including failures and skipped tests.
3. Evidence that organization isolation and authorization are preserved where relevant.
4. Confirmation that no real donor or patient data, credentials, or local secrets were introduced.
5. A list of contract deviations and unresolved dependencies.
6. Review by the affected member(s) and administrator approval before merge.
No merge or release is implied by this document. The administrator retains control over integration into main.
Security, privacy, and failure behavior
- Use password hashing, server-side authentication and authorization, organization-scoped database access, and CSRF protection for cookie-authenticated state changes.
- Minimize persisted and returned fields. Restrict contact/location information. Never log credentials, secrets, or sensitive record contents.
- Validate inputs and handle database errors explicitly. Do not silently fall back to unconfigured compatibility, eligibility, or availability.
- Restrict synthetic demo initialization and reset to development mode. Keep environment secrets, local artifacts, caches, and generated files out of version control.
- Do not claim production security, clinical validation, model effectiveness, or guaranteed outcomes.
Decision backlog
The following items remain unresolved unless and until the decision register records an explicit approval or deferral.
ID	Decision required	Proposed lead	Blocks or constrains
D-01	Authoritative RBC compatibility source, named reviewer, configuration and update ownership	Administrator confirms reviewer; Member 3 proposes implementation interface	Any compatibility pass
D-02	Eligibility verification authority, accepted verification process, and expiry policy	Team decision; Member 3 owns gate implementation	Any eligibility pass
D-03	Availability opt-in freshness duration and withdrawal behavior	Team decision; Members 2 and 3 review	Any availability pass
D-04	Staff role/action matrix and organization boundaries	Member 1 proposes; all affected members review	Protected API and shared-data access
D-05	Approved urgency categories and ranking semantics	Member 3 proposes; Member 1 reviews request validation	Request validation and ranking
D-06	Distance method, precision, and synthetic-coordinate generation	Member 3 proposes; Member 2 reviews stored fields	Distance ranking and explanation
D-07	Donation-history event categories and permitted ranking contribution	Member 2 proposes data representation; Member 3 reviews ranking use	Data generation and ranking
D-08	Optional model inclusion, synthetic feature/target assumptions, baseline, and evaluation plan	Member 2 proposes; Members 1 and 3 review integration and output	Batch 4 model work
D-09	Runtime/dependency compatibility and supported deployment platform	Member 1 coordinates verification; each owner checks relevant dependencies	Dependency selection and reproducible setup
D-10	Whether any synthetic contact reference is needed and who may access it	Administrator and affected members	Contact-field storage or exposure


An item may be deferred only when the dependent feature can remain safely disabled or bounded. A deferred decision must not be implemented through an implicit default.
Review and verification criteria
Before implementation begins, affected members review the four contracts and confirm:
- Data, API, and recommendation field names and states agree.
- The API's example for missing compatibility configuration uses evaluation_status: "not_configured", not completed.
- Operational failures cannot be represented as successful evaluations with empty results.
- Missing/unverified clinical or availability configuration cannot yield passing donors.
- not_configured is distinguishable from a completed evaluation with zero passing candidates.
- No compatibility mapping, eligibility criteria, freshness duration, ranking weights, or ML claims have been invented.
- Organization isolation, CSRF, minimized responses, synthetic-only data, and error behavior are testable.
- Ownership and dependency order are accepted, and unresolved decisions have an owner or explicit deferral.
- The dependency/runtime verification records actual interpreter and package compatibility results; no compatibility claim is based only on an assumed version.
- The final contract review records approvals and deferred decisions before Batch 2 begins.
