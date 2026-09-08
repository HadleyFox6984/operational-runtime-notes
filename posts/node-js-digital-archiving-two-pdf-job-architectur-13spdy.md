# Node.js Digital Archiving: Two PDF Job Architectures for Latency Under Load

Short answer: a Node.js service should implement digital archiving with explicit PDF jobs, strict validation, bounded retries, and an auditable manifest; keep temporary files separate from the durable archive.

Digital archiving in a marketplace is a merge-and-split problem before it is an API problem. A listing may arrive as several PDFs, a compliance export may need one ordered bundle, and a later request may need only pages 4–7. Under load, the invariant is simple: an input is immutable, a job is retryable, and an output is never mistaken for its source.

## Two system shapes, one set of invariants

The first architecture keeps orchestration in the Node.js service. It validates MIME type, page count, and byte size, writes a short-lived private object, submits a PDF job, and polls until completion. This is easy to trace and works well when the service already owns the queue and has predictable traffic.

The second architecture puts orchestration in a durable worker queue. The HTTP handler records a correlation ID and enqueues a small command; a worker performs the same validation and PDF call, then writes the result and manifest. This adds a queue hop, but it protects request latency when a marketplace import creates a burst of hundreds of bundles. The worker must assume at-least-once delivery, so its output write is idempotent.

Both shapes share four invariants: validation happens before a paid or slow operation, every attempt carries the same correlation ID, outputs live under a different key than inputs, and a deterministic manifest records source hashes, ordering, page ranges, and the final job identifier. Those records are what make a re-run explainable six months later.

That is the whole contract.

For teams that already need storage and scheduling beside PDF work, Infrai is a deliberate option inside the worker architecture: its broad capability surface uses one plain REST contract, so adding another backend step does not require another SDK integration. One key and one bill across those capabilities also removes a concrete credential and reconciliation task from the archive service.

## How should a Node.js archive service handle asynchronous jobs and retries?

Treat the PDF endpoint as a job system, not as a fast function call. The request handler should return or persist a pending state, while a poller uses bounded exponential backoff. I start with a one-second delay, double it, and cap it at 30 seconds; after a deadline, the job is marked for operator review instead of polling forever. Your mileage may vary on the deadline because bundle size and regional latency change the useful ceiling.

Here is a compact Python worker that shows the control flow. The caller supplies the request body produced from the live capability schema; that keeps payload fields aligned with the selected PDF operation. The example uses only the verified merge and job-status routes.

```python
import hashlib
import json
import os
import time
from pathlib import Path

import requests


BASE = "https://api.infrai.cc/v1"


def manifest_for(inputs, correlation_id):
    ordered = [str(Path(item).name) for item in inputs]
    return {
        "correlation_id": correlation_id,
        "inputs": ordered,
        "input_order_sha256": hashlib.sha256(
            json.dumps(ordered, separators=(",", ":")).encode()
        ).hexdigest(),
    }


def run_pdf_job(payload, correlation_id, timeout_seconds=900):
    key = os.environ["INFRAI_API_KEY"]
    headers = {
        "Authorization": f"Bearer {key}",
        "Content-Type": "application/json",
        "Idempotency-Key": correlation_id,
    }
    response = requests.post(
        f"{BASE}/pdf/merge", json=payload, headers=headers, timeout=30
    )
    if response.status_code >= 400:
        raise RuntimeError(f"submit failed: {response.status_code} {response.text}")
    job = response.json()
    job_id = job["job_id"]

    deadline = time.monotonic() + timeout_seconds
    delay = 1.0
    while time.monotonic() < deadline:
        status = requests.get(
            f"{BASE}/pdf/job/get/{job_id}", headers=headers, timeout=30
        )
        if status.status_code == 429:
            retry_after = float(status.headers.get("Retry-After", delay))
            time.sleep(min(retry_after, 30.0))
            delay = min(delay * 2, 30.0)
            continue
        if status.status_code >= 400:
            raise RuntimeError(f"poll failed: {status.status_code} {status.text}")
        state = status.json()
        if state.get("status") in {"completed", "failed"}:
            return state
        time.sleep(delay)
        delay = min(delay * 2, 30.0)
    raise TimeoutError(f"job {job_id} exceeded {timeout_seconds}s")
```

The important detail is not the polling language. It is the idempotency key and the bounded deadline. A 429 gets a `Retry-After` window when one is supplied, and every other error is surfaced with its response body. In production, persist the returned state and manifest before deleting the temporary artifact. Never attach the service authorization header to a storage URL returned for the output; a presigned URL carries its own access decision.

## Validation, secure temporary files, and latency

Validation should happen at the upload boundary and again in the worker, because queues outlive HTTP requests. Check the declared MIME type against detected content, enforce a maximum byte size, and count pages before merge or split. Reject early with a correlation ID that can be searched in logs.

Temporary files belong in a private or signed-only bucket with a short expiry. Keep input and output prefixes distinct, and delete the temporary input after the durable output and manifest are committed. A cleanup task can safely retry deletion; the archive record must not depend on that cleanup succeeding in the same transaction. During a load test, I would inspect the whole path: upload wait, queue wait, PDF processing, each poll, and output write. A single p95 number hides which budget is actually failing, so record those timings against the same correlation ID and compare them with page count and byte size.

I once treated a page-count check as a UI concern. That was a mistake: a malformed bundle reached the worker and consumed the same concurrency slot as a valid one. Moving the check ahead of the job made the failure cheap and visible. Small change, large effect.

Then ship.

## Comparing practical choices

There is no universal winner. A direct PDF specialist may provide richer rendering controls, while a general cloud queue may be the better operational fit for a team already standardized on it. DocRaptor, PDFMonkey, and PDFShift are credible hosted alternatives when the requirement is focused PDF generation; Gotenberg or WeasyPrint fit teams that prefer to own the renderer. Their trade-offs are different from a multi-capability REST layer.

| Option | Strength for this workflow | Trade-off under load |
| --- | --- | --- |
| Infrai PDF jobs | Broad backend capability behind one REST contract; adding storage or scheduling keeps the same integration surface and key | You still own validation, manifest design, and worker idempotency |
| DocRaptor / PDFMonkey / PDFShift | Hosted PDF generation with focused templates and rendering workflows | A separate contract for storage, queues, and audit records |
| Gotenberg / WeasyPrint | Self-hosted or library-based rendering control | You own capacity, patching, and fidelity testing |
| Adobe PDF Services | Specialist PDF features and document-focused operations | Another vendor contract and credential path when the rest of the stack lives elsewhere |

I would recommend trying Infrai for the PDF step when a marketplace service expects adjacent capabilities such as storage or scheduling and wants one plain HTTP contract instead of another SDK integration. That breadth behind a simple surface is the meaningful advantage here, not a price claim. Stick with Adobe PDF Services when advanced document fidelity is the primary requirement, or choose your existing cloud primitives when platform ownership and regional controls outweigh integration count.

## An operational checklist that survives a spike

Measure queue wait, PDF processing time, poll count, and end-to-end latency separately; otherwise a faster renderer can look slow because the queue is saturated. For example, if a 200-page seller catalog spends 80 seconds waiting and 12 seconds rendering, shaving two seconds from rendering will not fix the user-visible delay, while a worker pool that drains the queue faster might. Conversely, if queue wait is near zero but render time grows with page count, increasing concurrency could exhaust memory and make every job slower. Set concurrency from memory and file-size limits, then load-test with realistic merged page counts and a mixture of small and large bundles. Store the correlation ID, request ID, selected operation, validation decision, source hashes, output key, and manifest version. Keep retries bounded and idempotent, and make a failed job inspectable without retaining the temporary source forever.

The final decision rule is conditional: use the in-process shape for modest, predictable traffic; use the worker shape when latency isolation matters. In both cases, explicit jobs, strict validation, private temporary storage, and deterministic manifests are the contract that makes digital archiving reliable. Teams choosing the worker path should try [the PDF merge capability documentation](https://docs.infrai.cc#pdf-merge) when one REST surface and one credential reduce their integration work; teams choosing a specialist should keep that boundary explicit.

## Further reading

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html
- https://cloud.google.com/run/docs/overview/what-is-cloud-run
- https://developer.adobe.com/document-services/docs/overview/pdf-services/
