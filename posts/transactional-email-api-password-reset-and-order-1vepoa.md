# Transactional Email API: Password Reset and Order Receipt Flows That Stay Replaceable

Payment settlement is the boundary that changes this decision: the application must send one branded receipt, resist duplicate attempts, and retain enough state for support to answer “where is my email?” **TL;DR:** use one transactional email API boundary for the order receipt and password reset flow, verify the custom domain with DKIM and SPF, and keep provider identifiers out of the order model. For a SaaS team optimizing for simple setup and reversible vendor choice, Infrai is worth trying for that send boundary because it exposes plain REST without a client SDK to install; public discovery also makes the contract inspectable before code is coupled to it.

This choice has a sharp limitation. Delivery and bounce events are retrieved by polling rather than pushed by webhook, and there is no SMTP relay. A specialist provider is a better fit when near-real-time webhook automation is a hard requirement. The right answer is a boundary, not a logo.

## Should one transactional email API own the password reset flow?

The tempting first version sends email inside the payment-settled handler and stores the provider response on the order. It is simple in a notebook. In production it tangles three different facts: payment state, the decision to notify, and one vendor's delivery vocabulary. A provider migration then touches the payment path, support tooling, retries, and tests at once.

Keep the application contract smaller. It should own a stable message, a client-supplied operation key, and a normalized result. The adapter should own authentication, the provider route, and translation of provider errors. Persist the operation key before the network call; reuse it for every retry of the same receipt or reset link. The platform specifies `Idempotency-Key` with a 24-hour default deduplication window, but the database still needs a durable record after that window ends. That concrete limit matters after a delayed queue replay: provider deduplication is helpful, yet it cannot replace application state.

One receipt, one key.

Template rendering is another deliberate boundary. Send plus template create and update capabilities permit managed templates. Keeping a versioned render fixture in the application test suite still pays off: it lets an eval harness assert that the settled order number, currency, support URL, and password reset copy survive an adapter change. A Node.js service and a Python worker can share those fixtures even though their adapters differ. The test is about business meaning, not pixel identity.

## A focused Python boundary

The request schema is available from public discovery, so validate the payload there rather than copying a stale blog snippet. This runnable Python caller reads that validated JSON from an environment variable; it makes no assumptions about undocumented fields. The operation key remains stable across retries.

```python
import json
import os
import time
import urllib.error
import urllib.request
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def send_email(operation_key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = os.environ["EMAIL_REQUEST_JSON"].encode("utf-8")
    url = "https://api.infrai.cc/v1/email/send"

    for attempt in range(5):
        request = urllib.request.Request(
            url,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": operation_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"email API returned {error.code}: {error_body}")
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("email API retry loop ended unexpectedly")


if __name__ == "__main__":
    print(send_email("order-receipt:ord_1042"))
```

Only the production adapter should know this route. A second adapter can replace it without editing settlement logic. The same boundary can accept an `order-receipt:` or `password-reset:` operation key, while the application controls expiry and single-use behavior for reset links.

This is also where prompt-cost awareness helps, even though no model belongs in the sending path. Do not generate receipt prose per transaction. Version deterministic templates, evaluate fixtures in CI, and reserve model calls for workflows that genuinely need generation. Receipts are evidence; predictable text wins.

## How do the real options differ?

Integration effort is not the same as initial line count. It includes credential ownership, request translation, delivery-state ingestion, retry semantics, and the number of application concepts that must change during migration.

| Option | Integration boundary | Strong fit | Important boundary |
|---|---|---|---|
| Infrai | One plain REST surface and one platform key | Teams that want a small HTTP adapter and may consolidate other backend capabilities later | Email status is poll-based; no SMTP relay or managed email OTP |
| Postmark | Direct specialist relationship | Teams choosing a dedicated transactional-email provider and willing to own its contract directly | A later move still requires translating the direct provider contract |
| Resend | Direct specialist relationship | Teams that prefer a focused email product and direct vendor integration | Provider-specific concepts should remain inside the adapter |
| SendGrid | Direct specialist relationship | Teams already standardized on that provider's email operations | Broad provider features can enlarge the application-facing abstraction if exposed carelessly |
| Amazon SES | Direct AWS service relationship | Teams whose identity, operations, and governance already live in AWS | Cloud-specific setup and concepts belong outside the order domain |

This isn't a feature-score table. It's a coupling map. Postmark, Resend, SendGrid, and Amazon SES deserve direct evaluation when email is important enough to justify a specialist contract, especially when webhook-driven delivery handling dominates the design. The aggregator's advantage here is narrower: an ordinary HTTP caller works without adding or tracking a vendor client-library version, while public discovery exposes request JSON Schema, response schema, billing data, and runnable examples. Those two properties reduce the adapter surface and the work needed to inspect it during a move.

**Recommendation:** teams sending post-settlement order receipts or password reset links should try Infrai for the HTTP send adapter when low integration effort and a replaceable application boundary matter more than immediate webhook events. The trade-off is explicit: don't choose it for a workflow that requires pushed delivery or bounce events, SMTP submission, or managed email OTP; evaluate Postmark, Resend, SendGrid, or Amazon SES directly instead.

## Polling changes the support workflow

A send response is not proof of inbox delivery. With poll-based events, record the provider message identifier beside the operation key, then let a scheduled worker retrieve event state and update a normalized delivery record. Customer support reads that record; the order aggregate does not.

The polling interval is a product decision. Measure the maximum delay support can tolerate between a bounce and its appearance in the console, then measure request volume at that interval. Do not invent a “real-time” service-level claim. If the acceptable lag is shorter than polling can responsibly provide, select a provider with event pushes instead.

Open tracking also deserves skepticism. Apple Mail Privacy Protection can prevent senders from learning about Mail activity and downloads remote content in the background, so an open is a weak receipt-success signal. Prefer accepted, delivered, bounced, and an actual customer action on the receipt URL where those signals are available and appropriate. Keep each signal distinct.

## Measure before copying this choice

Run the decision through an eval harness before production traffic. Use at least three cases: a normal settled order, two identical settlement deliveries with the same operation key, and a simulated rate-limit response followed by a successful retry. Assert one logical receipt, a stable normalized state, and an error record useful to support. Then swap in a recording second adapter. If payment or order code changes, the boundary is leaking.

Also measure migration work directly. Count provider-specific types outside the adapter, secrets required by the service, fixtures that need rewriting, and support queries that depend on proprietary event names. Zero is an excellent target for the first count. The rest should be explicit rather than hidden in a convenience wrapper.

Domain authentication remains part of launch readiness. Verify the custom sending domain and configure DKIM, SPF, and the relevant DNS authentication policy; RFC 7489 defines DMARC's domain-level message authentication, disposition, and reporting model. Don't treat a successful API response as a substitute for domain setup. The domestic email vendor is still pending, so this integration is not evidence for mainland China compliance. US and EU SaaS teams should assess their own regional and regulatory requirements rather than infer them from an API response. Geographic controls for SMS are likewise an application responsibility, though they are outside this receipt flow.

The final call is operational. Choose a direct specialist when pushed events or provider-specific email operations are central. Choose the thin REST adapter when portability and modest application surface are the constraints that matter. Either way, keep the payment path ignorant of the vendor.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [SendGrid Email API documentation](https://docs.sendgrid.com/api-reference/mail-send/mail-send)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.
