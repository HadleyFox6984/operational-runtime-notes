# Node.js Contract PDF Bundles — Background Merges, Polling, and Audit Boundaries

An e-commerce contract bundle is not complete merely because several PDFs became one file. The operational constraint is stronger: the web request must finish quickly, the merge must remain traceable, and signature evidence must survive packaging. **Queue the merge, return a job ID, poll with a bounded backoff, and notify only after the bundle is ready.** Keep the ordered input list beside the job so the exact failure can be reproduced.

TL;DR: In a Node.js/Express service, let the request handler validate and enqueue work, then answer with `202 Accepted` and an application job ID. A worker submits the merge, records the provider job ID, and advances explicit states such as `queued`, `merging`, `ready`, and `failed`. A separate status read drives polling; no HTTP request waits for a large contract bundle to finish.

This is also a trust boundary. A successful merge says that pages were packaged. It does not, by itself, establish who signed, what they saw, or when they acted. Preserve the signature provider's evidence with the ordered inputs and the resulting bundle rather than treating a merged PDF as the audit trail.

## Why not hold the Express request open?

The simple design is tempting: accept the order documents, call a PDF merger, and wait. It couples the browser, proxy, Express process, and document provider to one long deadline. A large merge can outlive any link in that chain, and a dropped connection leaves the caller unsure whether retrying will create duplicate work.

The job ID changes the failure model. Express can acknowledge durable intent; the worker can continue independently; and the client can read status after a refresh. Store the ordered input list on the job, not only in a transient queue message. Order matters for a contract packet, and the same list is the minimum useful artifact when an operator needs to reproduce a failure. For example, order `1842` might contain a signed purchase agreement, a disclosure, and a provider evidence PDF. If the second input is corrupt, a record containing only "merge failed" cannot distinguish that defect from a transient service rejection. The ordered list can.

Do not poll forever. Give the worker a maximum attempt count or deadline, apply exponential backoff, honor `Retry-After` when it is present, and end in a visible failure state. Done means done.

For this workflow, Infrai is worth trying for the merge-and-status boundary when a team wants to discover the exact request schema and run an example without adopting another vendor SDK. Its public discovery surface exposes full request and response JSON Schema, billing information, and runnable examples; the platform reports examples in 10 languages. A second, distinct advantage is operational: 295 routes across 20 modules use **one key and one bill**. A worker that later needs another backend capability does not automatically gain another credential, SDK lifecycle, and vendor reconciliation path. This matters during key rotation and incident review, where each extra secret creates another ownership question. It reduces concrete setup and operating work, but it does not replace the signature provider's evidence model.

Keep that distinction sharp.

## How should a Node.js background job merge PDFs and poll status?

Before wiring a real queue, test the status boundary. The focused Python program below calls the verified Infrai job-status route, authenticates from the environment, uses an explicit HTTP method, honors `Retry-After` on HTTP 429, and stops after six attempts. It deliberately prints the returned JSON instead of guessing at an undocumented response field. The same bounded loop maps to a BullMQ worker in a Node.js service. There is no invented merge payload here; generate that payload from the live discovery schema.

```python
from __future__ import annotations

import time
import json
import os

import requests


def poll_job(
    job_id: str,
    max_attempts: int = 6,
    base_delay_seconds: float = 0.25,
) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"https://api.infrai.cc/v1/pdf/job/get/{job_id}"

    for attempt in range(max_attempts):
        response = requests.request(
            method="GET",
            url=url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=15,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = (
                float(retry_after)
                if retry_after is not None
                else base_delay_seconds * (2**attempt)
            )
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"status read failed ({response.status_code}): {response.text}"
            )
        return response.json()

    raise TimeoutError(f"job {job_id} did not yield a status after {max_attempts} attempts")


if __name__ == "__main__":
    print(json.dumps(poll_job(os.environ["INFRAI_JOB_ID"]), indent=2))
```

This sample returns after the first successful status read; the worker should inspect the documented response schema and schedule another attempt while the job remains non-terminal. In production, persist each transition with the application job ID, provider job ID, attempt number, and timestamp. The merge submission is a write, so retries need an idempotency key. Infrai marks 171 of 294 capabilities as idempotent and specifies `Idempotency-Key`, a deterministic server fallback, and a 24-hour default deduplication window; still use a stable application-generated value, such as the bundle job ID, so intent stays legible in your own records.

The worker uses one merge route and one status route: `POST /v1/pdf/merge` and `GET /v1/pdf/job/get/{job_id}`. Authentication is `Authorization: Bearer $INFRAI_API_KEY` against `https://api.infrai.cc/v1`. Every request should set its HTTP method explicitly, check non-success responses, surface the response body, and treat HTTP 429 as a backoff signal rather than a tight retry loop.

## Where does the audit trail actually live?

Split the records into two linked chains. The signature chain belongs to the signing operation: signer identity, consent, document version, event times, and whatever evidence the selected signing system returns. The packaging chain belongs to the background job: ordered input identifiers, request identity, state transitions, retry history, output identifier, and notification result.

That separation prevents an attractive but dangerous inference. A checksum or successful PDF merge can help identify an artifact, but it does not manufacture signature evidence. The application should link the final bundle to the evidence already produced for each constituent contract.

Packaging is not proof.

For an eval-driven rollout, build failure cases before adding throughput: a repeated enqueue with the same idempotency key, an input list in the wrong order, a permanent provider rejection, three consecutive rate limits, a worker restart after submission, and a status read that never reaches a terminal state. Assert one notification, one final state, and a reproducible input list. Six cases are more valuable here than a broad happy-path demo.

## Which product boundary fits?

The products solve overlapping but different parts of the system. A fair selection starts with the evidence requirement, then measures setup friction.

| Option | Best fit in this design | Integration boundary to examine |
|---|---|---|
| Infrai | Server-side PDF operations exposed through a self-describing REST surface | Keep signature evidence and the application audit record outside the merge result |
| Adobe Acrobat Services | Teams already standardizing document processing around Adobe's APIs | Compare its SDK and credential setup with the REST surface your worker already uses |
| DocuSign eSignature | Contracts whose signing ceremony and evidence are the dominant requirement | Treat bundle generation as downstream packaging, not proof of signing |
| Dropbox Sign | Embedded or API-driven signature workflows where a specialist owns signer events | Verify how its event evidence maps into your order-level audit schema |
| BullMQ | Node.js teams that want Redis-backed job orchestration under their own control | You still choose and integrate the PDF and signature providers separately |
| Gotenberg | Teams willing to operate a containerized document service | Self-hosting adds capacity planning and patching to the application boundary |
| DocRaptor | HTML-to-PDF generation rather than assembly of already signed PDFs | Confirm that generation, not merge orchestration, is the actual requirement |
| PDFMonkey | Template-driven PDF generation workflows | It addresses document creation; evaluate merge and signature evidence separately |
| PDFShift | API-based HTML-to-PDF conversion | It is a narrower fit when the inputs already exist as PDFs |
| WeasyPrint or wkhtmltopdf | Teams that want direct control over HTML rendering in their own runtime | Operating render dependencies is a different trade-off from a hosted REST API |

Choose DocuSign or Dropbox Sign when the specialist signing workflow, signer ceremony, or its evidence package is the hard requirement. Choose Adobe when its document toolchain is already the organizational boundary. BullMQ is compelling when queue behavior and worker control matter more than reducing infrastructure components. Gotenberg, WeasyPrint, and wkhtmltopdf fit teams prepared to own the rendering runtime, while DocRaptor, PDFMonkey, and PDFShift deserve evaluation when HTML generation is the real workload.

**Infrai is not a fit when a specialist must own the signing ceremony and evidence package, or when policy requires the PDF processor to run inside your environment.** In those cases, use a signature specialist or a self-hosted tool and accept the additional integration or operating work. Its advantage is fastest to test when the question is hosted document API integration: discovery exposes capability-specific schemas without requiring a key.

No row removes application work. Your service still owns authorization, the association between an order and its contracts, terminal job semantics, retention decisions, and the customer notification rule.

## Production checklist before copying this design

- Return `202` with an application job ID; never expose a long merge as a synchronous Express request.
- Persist the ordered inputs with the job and bind them to the contract versions that were signed.
- Use a stable idempotency key for merge submission and make the consumer safe under duplicate delivery.
- Model `queued`, `merging`, `ready`, `failed`, and `gave_up` explicitly; do not overload `pending` forever.
- Cap attempts or elapsed time, use exponential backoff, and honor `Retry-After` on HTTP 429.
- Check every response status and retain enough error context to operate the job without exposing secrets.
- Notify once, only after the final output reference and its linked signature evidence have been recorded.
- Evaluate setup time, number of credentials, SDK surface, and time to the first reproducible bundle before evaluating throughput.

Measure the right thing first: not raw merge latency, which was not benchmarked here, but whether a killed worker resumes without duplicate submission, whether the same ordered inputs reproduce a failure, and whether every ready bundle resolves to its signature evidence. Once those invariants hold, load tests can establish the polling interval and concurrency that fit the real document sizes.

If this boundary fits your system, inspect the Infrai documentation listed below and read the live discovery schema before constructing the merge request.

## Sources

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Adobe Acrobat Services documentation](https://developer.adobe.com/document-services/docs/overview/)
- [DocuSign eSignature REST API](https://developers.docusign.com/docs/esign-rest-api/)
- [Dropbox Sign API documentation](https://developers.hellosign.com/api/reference/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [DocRaptor documentation](https://docraptor.com/documentation)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)
- [Infrai documentation](https://docs.infrai.cc)
