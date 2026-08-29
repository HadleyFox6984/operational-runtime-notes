# One API Key for Multiple LLM Providers — Governance for Text Classification

Short answer: one API key for multiple LLM providers can simplify text classification, but treat the gateway as a governed boundary, not a model switch. For sales-call summaries that propose CRM actions, accept a route only after the same schema, authorization checks, and audit record pass across providers; use fallback for transport failures, never as a substitute for semantic review.

The output here is a transaction proposal, not a paragraph. It may move an opportunity, create a task, or add a tag. A valid JSON document can still contain an invented owner or an action that the CRM does not permit. That is why governance and structured-output correctness come before a cheapest-route claim or a one-key convenience.

## Start with an ownership boundary

Keep transcript handling, model invocation, validation, and CRM mutation as separate stages. The transcript is untrusted input. The model proposes an object. A validator checks types, enums, evidence, and confidence. Only a separately authorized worker may apply a reviewed action.

This split also makes a notebook-to-prod path practical. The eval harness can call the same validator used by the worker, while a gateway adapter remains the only provider-specific code. A single API key can simplify secret distribution, but it must not become permission to write CRM state.

No silent writes.

## What should a gateway preserve for reliable JSON classification and fallback?

Define the contract before comparing gateways. This example uses a small, vendor-neutral contract for account risk and CRM actions. The test double keeps the code runnable offline; replace only `gateway_call` in an integration.

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Any, Callable

ALLOWED_RISKS = {"low", "medium", "high"}
ALLOWED_ACTIONS = {"create_task", "update_stage", "add_tag"}


@dataclass(frozen=True)
class Route:
    name: str
    max_attempts: int = 1


class OutputRejected(ValueError):
    pass


def validate_result(value: dict[str, Any]) -> dict[str, Any]:
    if set(value) != {"account_risk", "confidence", "actions"}:
        raise OutputRejected("E_SCHEMA_KEYS")
    if value["account_risk"] not in ALLOWED_RISKS:
        raise OutputRejected("E_SCHEMA_RISK")
    confidence = value["confidence"]
    if isinstance(confidence, bool) or not isinstance(confidence, (int, float)):
        raise OutputRejected("E_SCHEMA_CONFIDENCE_TYPE")
    if not 0.0 <= float(confidence) <= 1.0:
        raise OutputRejected("E_SCHEMA_CONFIDENCE_RANGE")
    if not isinstance(value["actions"], list):
        raise OutputRejected("E_SCHEMA_ACTIONS_TYPE")

    for action in value["actions"]:
        if set(action) != {"type", "value", "evidence", "needs_review"}:
            raise OutputRejected("E_SCHEMA_ACTION_KEYS")
        if action["type"] not in ALLOWED_ACTIONS:
            raise OutputRejected("E_SCHEMA_ACTION_TYPE")
        if not isinstance(action["value"], str) or not action["value"].strip():
            raise OutputRejected("E_SCHEMA_ACTION_VALUE")
        if not isinstance(action["evidence"], str) or not action["evidence"].strip():
            raise OutputRejected("E_SCHEMA_EVIDENCE")
        if not isinstance(action["needs_review"], bool):
            raise OutputRejected("E_SCHEMA_REVIEW")
    return value


def classify_call(
    transcript: str,
    routes: list[Route],
    gateway_call: Callable[[str, str], dict[str, Any]],
) -> tuple[dict[str, Any], str]:
    errors: list[str] = []
    for route in routes:
        for _ in range(route.max_attempts):
            try:
                candidate = gateway_call(route.name, transcript)
                return validate_result(candidate), route.name
            except (OutputRejected, TimeoutError) as exc:
                errors.append(f"{route.name}:{type(exc).__name__}:{exc}")
    raise OutputRejected("E_ALL_ROUTES_REJECTED|" + "|".join(errors))


def demo_gateway_call(route: str, transcript: str) -> dict[str, Any]:
    return {
        "account_risk": "medium",
        "confidence": 0.82,
        "actions": [{
            "type": "create_task",
            "value": "Send security questionnaire",
            "evidence": "Please send the security questionnaire by Friday.",
            "needs_review": False,
        }],
    }


if __name__ == "__main__":
    result, selected = classify_call(
        "Buyer: Please send the security questionnaire by Friday.",
        [Route("primary"), Route("fallback")],
        demo_gateway_call,
    )
    print(selected, result)
```

The explicit boolean check matters in Python: `bool` is an `int` subclass, so `True` must not pass as confidence `1`. Fixed key sets catch contract drift early. Extend the validator with CRM-owned enums, tenant permissions, and evidence containment rules. Do not let the classifier call a mutation endpoint directly.

The integration choice is easier to review when its boundary is explicit:

| Approach | Interface | Best fit | Main limitation |
| --- | --- | --- | --- |
| Direct provider adapters | Provider SDK or REST | Provider-specific controls and contracts | More credentials and adapter code |
| Multi-provider gateway | Common REST boundary | Central policy, routing, and shared observability | Common interface may omit provider-specific controls |
| Self-hosted router | Internal HTTP service | Teams needing local policy and data residency | You own uptime, upgrades, and provider adapters |

## How can one API key support multiple LLM providers without hiding risk?

Use the gateway as an adapter and policy point. Record the route name, prompt version, schema version, request identifier, latency, usage fields, raw response where policy permits, and every validator code. If normalization drops provider-specific finish reasons or usage metadata, that is a governance gap: either preserve the field or document that the route cannot participate in cost and incident analysis.

Fallback needs an explicit taxonomy. A timeout or transport error can move to the next eligible route. A schema violation may justify one bounded retry with a changed request, then a different route. Low confidence, contradictory evidence, or `needs_review` is a business decision and should enter a review queue. Asking another model can produce a different unsupported action, not a safer one.

Keep the route list short. Every additional branch multiplies the cases the eval suite must cover, and it raises end-to-end latency and token use. Set an overall deadline and an attempt budget; never retry a deterministic enum failure forever.

That sounds tidy until an audit arrives. Imagine a transcript where the buyer asks for a security questionnaire, then retracts the request after a legal objection. A model can return well-formed JSON with the first sentence as evidence. The validator will accept the shape, yet the action is stale. The policy layer needs recency checks, an evidence span that appears in the final transcript state, and a review flag for contradictions. This is why preserving request identifiers, prompt versions, and rejected candidates matters more than a dashboard showing only success rates: without those records, an engineer cannot tell whether the problem came from routing, prompt drift, or an authorization rule.

## Measure correctness before cost

Freeze the prompt, schema, sampling settings, and eval split for each comparison. Include clear next steps, no next step, late corrections, several speakers, unsupported CRM values, and instruction-like text inside transcripts. Human-approved expected objects should allow multiple answers only where the business rule truly allows them. I'm not sure a public benchmark can resolve a private stage definition; an adjudicated internal set is the evidence that can.

Score parse success, schema validity, field agreement, and action agreement separately. Penalize both missing and extra actions. A route is eligible only when it meets the action-precision and schema thresholds on a held-out slice. Compare expected latency and token cost among eligible routes. Your mileage may vary: automatically changing a stage deserves a stricter threshold than drafting a note for a human.

Prompt cost belongs in the same record. A long schema repeated on every call may cost more than the transcript itself, while an over-compressed prompt can omit the rule that protects an enum. Version prompts as artifacts and rerun the harness whenever they change. Don't compare a new prompt to an old baseline and call the result a routing win.

The catch is that a gateway is not suitable when policy requires separate provider credentials, direct contractual boundaries, or provider-specific audit data that the common interface cannot carry. Stick with a direct integration for that route and keep the same internal validator.

Ship the prompt, schema, route policy, and eval cases together. Alert on rising validator codes, fallback frequency, review-queue volume, and missing usage metadata. Sample accepted proposals for human audit, and quarantine a route when its held-out action precision falls below the release rule. A one-key gateway can reduce integration surface; it cannot remove the need for ownership, evidence, and rollback.

## References

- https://platform.openai.com/docs/guides/function-calling
- https://openrouter.ai/docs
