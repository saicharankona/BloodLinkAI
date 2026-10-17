# Recommendation contract

- **Status:** Draft for team review
- **Primary owner:** Member 3 — Recommendation Engine, Dashboard and QA
- **Reviewers:** Member 1 for API/auth/error behavior; Member 2 for available and validated repository fields

## Safety boundary

Recommendations are synthetic-data outputs of an unvalidated student prototype. They are not medical advice, clinical eligibility determinations, or confirmation that anyone can donate. The system must not automatically contact donors, dispatch staff, or authorize a transfusion.

Compatibility must use an authoritative source selected and reviewed for this prototype. Until both the source and reviewer are confirmed and explicitly configured, compatibility is `not_configured`; no donor can be returned as a passing candidate. Do not implement or imply an unreviewed compatibility table.

## Evaluation order

For each valid RBC request, evaluate in this order:

1. Validate request shape, RBC component, configured request-value validation, and organization context.
2. Evaluate RBC compatibility using the explicitly configured authoritative source.
3. Evaluate externally verified eligibility, independently of compatibility.
4. Evaluate fresh, affirmative availability/opt-in, independently of the other gates.
5. Exclude any donor that fails, is unknown for, or cannot pass any required gate.
6. Rank only candidates passing every required gate.
7. Return gate outcomes, freshness information, ranking factors, and explanations.

If a required gate is not configured or cannot be evaluated for the request, do not rank or return passing candidates. A valid evaluation with no passing candidates is different from an evaluation blocked by missing configuration.

## Gate result contract

Each gate result contains:

```json
{
  "status": "pass",
  "reason_code": "compatibility_pass",
  "evaluated_at": "<UTC timestamp>",
  "data_updated_at": "<UTC timestamp or null>",
  "valid_until": "<UTC timestamp or null>"
}
```

Allowed statuses are `pass`, `fail`, `not_configured`, and `not_evaluated`. Use stable reason codes:

| Gate | Reason code | Meaning |
|---|---|---|
| Compatibility | `compatibility_not_configured` | No approved compatibility source/reviewer configuration |
| Compatibility | `compatibility_pass` | Configured source evaluated and found compatible |
| Compatibility | `compatibility_fail` | Configured source evaluated and found incompatible |
| Eligibility | `eligibility_verified` | Explicit verification is current under the approved policy |
| Eligibility | `eligibility_unverified` | No qualifying verification |
| Eligibility | `eligibility_unknown` | Status cannot be established |
| Eligibility | `eligibility_expired` | Verification is outside its approved validity period |
| Availability | `availability_opted_in` | Affirmative opt-in is current under the approved freshness policy |
| Availability | `availability_withdrawn` | Donor withdrew availability |
| Availability | `availability_unknown` | Current opt-in cannot be established |
| Availability | `availability_stale` | Opt-in is outside its approved freshness window |
| Any later gate | `blocked_by_compatibility` | Not evaluated because compatibility was not configured |

When later gates are evaluated after a compatibility pass/fail, report their own outcomes. Do not treat a failed earlier gate as proof of any later gate outcome. Unknown, missing, stale, expired, and invalid timestamps never pass.

## Evaluation status and empty results

- `not_configured`: a required evaluator/policy is missing; return no candidates and identify the blocked gate.
- `completed`: all required gate policies were available and the request was evaluated. If no donor passes every gate, return an empty candidate list and gate-level counts/reasons.
- `failed`: an operational failure prevented a safe evaluation; return no candidates and a safe API error rather than fabricated defaults.

Never use an empty result alone to conceal missing configuration. Compatibility `not_configured` is not a valid compatibility check that found zero matching candidates.

## Ranking and explanations

Ranking applies only after all hard gates pass. Permitted potential factors from the approved scope are:

- Approximate distance/location.
- Urgency.
- Synthetic donation history.
- Optional model estimate of historical response behavior.

The team must agree on factor definitions, normalization, weights, missing-value behavior, tie-breaking, location precision, and urgency categories before implementation. No factor may rescue or override a failed or unevaluated gate. Keep the ranking deterministic for identical inputs and data.

Each returned candidate contains an opaque donor ID, a synthetic display label, final rank/score (if a score is used), passed gate results, freshness timestamps/indicators, and human-readable plus machine-readable ranking explanations. Do not include contact details or precise location. If a ranking input is missing, do not silently assign a success-shaped value; omit that factor with an explicit explanation or follow a separately approved policy.

## Optional response model

The Random Forest may contribute only an explicitly labeled estimate based on synthetic historical response records. Its feature definitions, target assumptions, train/test split, leakage checks, baseline, metrics, and metadata belong to the ML experiment contract/documentation. The estimate:

- does not determine or infer compatibility, eligibility, or current availability;
- does not turn an unknown gate into a pass;
- does not establish present willingness or guarantee a response or donation; and
- may be omitted safely when the model is unavailable or untrained, with an explicit explanation.

## Dependencies and unresolved decisions

- Depends on the data-model contract for verified eligibility, fresh opt-in, and synthetic ranking inputs.
- Defines result semantics consumed by the API contract and dashboard.
- Member 3 owns gate orchestration, ranking, explanation, and recommendation QA implementation.
- Unresolved: compatibility source/reviewer; eligibility verification authority/expiry; opt-in freshness; ranking factor formulas and weights; urgency and distance policy; tie-break; and optional ML inclusion.
