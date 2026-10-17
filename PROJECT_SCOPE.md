# BloodLink AI — Project Scope Proposal

## Objective and intended users

BloodLink AI is a student prototype for recommending potential red blood cell (RBC) donors in response to emergency requests. It will provide an API and a simple workflow for **authorized hospital or blood-bank staff** to review candidate recommendations.

The prototype is **not clinically validated** and must not be used to make real emergency medical decisions or to confirm that a donor can donate.

## Included features

- Python and Flask REST API backed by MongoDB.
- A simple interface for authorized staff to submit and review synthetic emergency requests and recommendations.
- Synthetic donor, request, availability, eligibility-status, donation-history, location, and historical response records.
- RBC-only compatibility checks using an authoritative source that remains to be selected and confirmed.
- Recommendation ranking that considers donation history, location/distance, urgency, and—optionally—predicted response probability, after hard gates are applied.
- Optional donor-response prediction using Pandas, NumPy, and scikit-learn. A Random Forest may predict **historical response behavior only**; it does not determine clinical eligibility, compatibility, or current willingness.
- Recommendation explanations that identify applied gates and ranking factors.

## Explicitly excluded

- Any use of real donor, patient, or historical response data.
- Clinical eligibility decisions, medical advice, or clinical validation.
- Support for blood products other than RBCs.
- Guarantees that a recommended donor is reachable, willing, eligible to donate, or will donate.
- Automated donor contact, dispatch, or emergency-care decisions.
- Claims of model accuracy or effectiveness before evaluation on an appropriately designed dataset.

## Synthetic-data and privacy rules

- Use synthetic records only, including in development, tests, demonstrations, and model experiments.
- Do not import, copy, or log real personal, health, contact, or location information.
- Use fake identifiers and contact details. Use approximate or generated locations where distance is demonstrated.
- Limit stored and returned fields to those needed for the prototype; restrict recommendation access to authorized staff.
- Clearly label synthetic records and prototype recommendations in the interface and API.

## Recommendation safeguards

**Hard gates — must pass before ranking:**

- RBC compatibility, according to the authoritative source selected for the prototype.
- An explicitly verified eligibility status under a verification process that remains to be defined.
- Current availability under a freshness rule that remains to be defined.

A donor who fails a gate, has an unknown status, or has stale required information must not be presented as a passing candidate. Optional response prediction must never override a failed or unknown gate.

**Ranking factors — used only among candidates that pass all gates:**

- Donation history.
- Location or estimated distance.
- Emergency urgency.
- Optional predicted probability of historical response.

The ranking method and factor contributions should be documented and explainable. Predicted response is not a commitment to respond or donate.

## Assumptions and unresolved requirements

The initial scope is a student demonstration for authorized institutional users, limited to RBC requests and synthetic data.

The following require confirmation before implementation:

- **Authoritative RBC compatibility source** and who will review its use in the prototype.
- **Eligibility verification process:** who verifies eligibility, what “explicitly verified” means, and when that status expires. The prototype must not infer clinical eligibility.
- Definition of “current” availability and how it becomes stale.
- Urgency categories and how they affect ranking.
- Distance calculation method and location precision.
- Staff roles, organization boundaries, and access rules.
- Whether the optional response model is included, and which synthetic features and evaluation approach are appropriate.

## Testable acceptance criteria

Using synthetic data only:

- The API accepts valid RBC requests and rejects malformed or unsupported request values with clear errors.
- Compatibility, eligibility, and availability are applied as separate hard gates before ranking.
- Candidates with incompatible, unverified, unknown, or stale required gate information are not returned as passing recommendations.
- Changing donation history, distance, or urgency can affect ranking among otherwise gate-passing candidates, with documented reasons visible in the result.
- If enabled, the Random Forest produces a response-probability estimate from synthetic historical response records only; it cannot change hard-gate outcomes.
- API tests verify that unauthorized users cannot access protected recommendation data.
- Tests verify that responses contain only the intended synthetic data fields and no real personal or health data.
- Documentation and UI state that the prototype is not clinically validated and must not be used for real emergency medical decisions.
