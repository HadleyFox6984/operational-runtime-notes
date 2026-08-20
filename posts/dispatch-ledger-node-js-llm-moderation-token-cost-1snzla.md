# Dispatch Ledger: Node.js LLM Moderation Token Cost for Text, Images, and JSON

Short answer: put a per-tenant admission ledger in front of classification. Build the exact moderation request, estimate its text-token budget, record the image cohort separately, and require a small JSON Schema verdict before a human reviewer sees the case. A cheap average is not a cost control; a visible, enforceable budget is.

In logistics, the queue is rarely uniform. A driver may upload one short note, while a warehouse account sends a long incident description with two photographs and structured shipment metadata. The moderation job is to classify that report before human review, so the useful unit is not “cost per request.” It is tenant, content shape, decision, and eventual review route.

This is the implementation I would take from a notebook to production: normalize an event, construct the final prompt, ask the selected runtime for a count, apply the tenant's ceiling, classify only admitted cases, validate the response, and append both estimate and actual usage to an accounting record. The classifier can change later. The ledger and evaluation contract should not.

## How can Node.js estimate LLM moderation token cost for text, images, and JSON Schema?

The estimator must see the payload that will actually be sent. Counting characters before adding the policy instructions, tenant label, JSON metadata, or image parts produces a number that looks precise and answers the wrong question. Tokenization also varies with language, punctuation, emoji, and serialized JSON, so a character divisor belongs in an early capacity sketch, not in the admission decision.

The Node.js edge can hand the normalized object to a small worker, or it can perform the same HTTP preflight itself. The boundary is language-neutral. The important ordering is stable:

1. Keep the raw moderation evidence and a redacted accounting copy separate.
2. Render the exact system and user messages from a versioned policy.
3. Count that rendered text with the chosen model and record the estimator version.
4. Apply a per-tenant token and image-item limit before classification.
5. Validate the returned decision against JSON Schema, then route `allow`, `review`, or `block`.

Images deserve their own lane. A text-token estimate does not establish how a vision-capable model accounts for pixels, dimensions, or multiple image parts. I would cap image count and upload dimensions at the application boundary, then measure representative image fixtures with the selected runtime. I'm not sure a universal image multiplier can be honest across models; the model's accounting documentation and a fixture run are what would settle it.

The ledger can therefore hold `estimated_text_tokens`, `image_count`, `actual_input_tokens`, `actual_output_tokens`, `decision`, and `review_route`. It should also hold `tenant_id` and `policy_version`. Never merge tenants into one daily average before checking the tail: one large account can hide an otherwise healthy queue.

## A Python worker that keeps the estimate beside the verdict

The following worker uses generic HTTPS endpoints and a provider-neutral request shape. It is intentionally boring. A Node.js service can call the same boundary, while the Python version is convenient for an eval harness. Set `TOKEN_COUNT_URL` and `CLASSIFIER_URL` to endpoints whose request and response contracts you have verified; the example does not invent a vendor-specific route.

```python
"""Preflight and classify a logistics report with per-tenant accounting."""

import json
import os
from typing import Any

import requests


TOKEN_COUNT_URL = os.environ["TOKEN_COUNT_URL"]
CLASSIFIER_URL = os.environ["CLASSIFIER_URL"]
API_KEY = os.environ["AI_API_KEY"]
MODEL = os.environ["AI_MODEL"]

POLICY_VERSION = "logistics-moderation-v3"
SYSTEM_PROMPT = (
    "Classify a logistics moderation report before human review. "
    "Return only a JSON object with decision and reason."
)
VERDICT_SCHEMA = {
    "type": "object",
    "properties": {
        "decision": {"type": "string", "enum": ["allow", "review", "block"]},
        "reason": {"type": "string", "enum": ["none", "spam", "abuse", "privacy", "unsafe"]},
    },
    "required": ["decision", "reason"],
    "additionalProperties": False,
}


def messages_for(report: dict[str, Any]) -> list[dict[str, Any]]:
    return [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": json.dumps(report, ensure_ascii=False)},
    ]


def post_json(url: str, payload: dict[str, Any]) -> dict[str, Any]:
    response = requests.post(
        url,
        headers={"Authorization": f"Bearer {API_KEY}"},
        json=payload,
        timeout=30,
    )
    response.raise_for_status()
    return response.json()


def classify(report: dict[str, Any], tenant_budget: int) -> dict[str, Any]:
    messages = messages_for(report)
    counted = post_json(
        TOKEN_COUNT_URL,
        {"model": MODEL, "messages": messages},
    )
    estimated = int(counted["tokens"])
    image_count = len(report.get("image_urls", []))
    ledger = {
        "tenant_id": report["tenant_id"],
        "policy_version": POLICY_VERSION,
        "estimated_text_tokens": estimated,
        "image_count": image_count,
    }
    if estimated > tenant_budget:
        ledger.update({"decision": "review", "review_route": "budget"})
        return ledger

    result = post_json(
        CLASSIFIER_URL,
        {
            "model": MODEL,
            "messages": messages,
            "response_format": {"type": "json_schema", "schema": VERDICT_SCHEMA},
        },
    )
    verdict = result["verdict"]
    if verdict["decision"] not in {"allow", "review", "block"}:
        raise ValueError("invalid moderation decision")
    ledger.update(
        {
            "decision": verdict["decision"],
            "reason": verdict["reason"],
            "actual_input_tokens": result.get("usage", {}).get("input_tokens"),
            "actual_output_tokens": result.get("usage", {}).get("output_tokens"),
            "review_route": "human" if verdict["decision"] == "review" else "none",
        }
    )
    return ledger


if __name__ == "__main__":
    sample_report = {
        "tenant_id": "warehouse-17",
        "text": "Package arrived with a threatening note.",
        "image_urls": ["https://example.invalid/photo-1.jpg"],
        "shipment": {"region": "north", "status": "exception"},
    }
    print(json.dumps(classify(sample_report, tenant_budget=1800), indent=2))
```

The example rejects an over-budget report into review rather than silently trimming its evidence. That is a policy choice, not a universal rule. Truncation may be acceptable for a low-risk queue if the omitted portion is retained and the decision is clearly marked; it is a poor default for a threat, privacy complaint, or shipment dispute where the last sentence can change the meaning.

The response contract also needs a real validator. Checking that `decision` exists is not JSON Schema validation: it does not prove that extra fields are rejected, that `reason` is from the policy vocabulary, or that the runtime returned an object rather than a JSON string inside a wrapper. Validate before incrementing a successful-classification counter. A malformed result belongs in an explicit review or retry state, with a reason that cannot be mistaken for an allowed report.

## What does a useful per-tenant cost experiment measure?

Start with a frozen fixture set from the logistics workflow, scrubbed of live personal data. Include short driver notes, long warehouse narratives, structured JSON with optional fields, one-image reports, two-image reports, and adversarial strings containing braces, emoji, and mixed languages. The point is not to create a pretty average. The point is to expose the shapes that change the bill and the policy result.

For every fixture, store the tenant, content cohort, policy version, model identifier, estimated text tokens, actual input tokens, actual output tokens, image count, latency, parse result, and final route. Aggregate by tenant and cohort. Report p50 and p95, not only the mean. A queue with mostly short notes can still breach a tenant limit when a single long exception report fans out to a second review pass.

There is a useful acceptance rule here: a candidate configuration passes only if its policy labels meet the team's error thresholds and its p95 usage fits the tenant budget. Token savings without a label-quality check are not savings; they are deferred review work. Prompt-cost awareness starts with removing repeated prose and unused fields, then stops before a safety rule becomes ambiguous.

Keep images separate in the report. If the chosen model exposes image usage, compare measured image cases with their text-only counterparts. If it does not, budget image cases conservatively and state that the estimate covers text only. Do not multiply text tokens by an attractive constant and present that as a forecast.

## The operational boundary: admit, classify, observe

The worker should be idempotent around an event ID. A queue retry must not create a second paid classification or a second human-review task without an explicit policy. Store the admission decision before the remote call, attach a correlation ID, and update the same ledger record with the result. Redact report text from ordinary cost dashboards; tenant-level visibility does not require making sensitive content searchable.

The catch is that a budget gate can be the wrong control for a high-severity queue. If every over-budget case must be reviewed immediately, a hard ceiling can move cost from the model line to the human line. Use a specialist moderation path when the policy requires fixed categories, use local inference when data residency dominates, and keep a general structured classifier for nuanced cases where the evaluation set supports it. The least expensive request is not automatically the least expensive workflow.

Here is the decision table I use when the queue grows beyond one model call:

| Path | Interface | Good fit | Main limitation |
| --- | --- | --- | --- |
| Hosted structured classifier | REST or SDK | Fast iteration and a small operations team | Usage and image accounting depend on the selected runtime |
| Local classifier | HTTP inside the cluster | Data residency and predictable capacity matter most | The team owns serving, upgrades, and evaluation |
| Specialist moderation service | Product-specific API | A fixed category taxonomy is the requirement | Its labels may not match a logistics policy without mapping |
| Human-first queue | Internal event contract | Severe or ambiguous cases need judgment | Reviewer time becomes the scarce budget |

Watch the tail.

From notebook to production, the checklist fits in prose: version the policy and schema together; test the exact serialized request; keep text-only and image cohorts distinct; enforce tenant ceilings before the remote call; validate the complete response; record estimates beside actual usage; watch p95 and review rate; and rerun the fixture set after any model, prompt, or image-policy change. In one realistic tenant mix, a short driver note might be admitted immediately, a long warehouse report might consume the full text ceiling, and a two-photo exception might be routed to human review because its image usage cannot be inferred from the text count. That three-way split is why the ledger needs both an estimate and a later usage record: collapsing those cases into one average makes the dashboard tidy while the queue remains expensive.

Keep it typed.

## References

- https://docs.cohere.com/docs/rerank-overview
- https://github.com/openai/whisper
- https://json-schema.org/specification
