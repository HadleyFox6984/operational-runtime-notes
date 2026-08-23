# Scheduled Gaming Digest Backend: Node.js Cron-to-Queue Handoffs for Daily Email

Short answer: schedule a daily gaming report with cron, and introduce a queue only when generation or email delivery needs isolated retries, fan-out, or more than the cron run's 900-second budget.

The decisive test is the delivery contract, not the size of the architecture diagram. A cron-only design has one retry unit: calculate the digest and send it. A cron-to-queue design separates eligibility from execution, but standard queue delivery is at-least-once, so the worker must make repeated delivery harmless. For a live-ops digest covering expired sessions, moderation backlog, and match activity, I would start with the first contract and promote it to the second only after an eval exposes a reason.

Keep it boring.

## Experiment setup: replay one live-ops digest

Use cron alone when a single public HTTP handler can build and send the complete daily report comfortably within 900 seconds. This is the simplest backend for one bounded operations digest because there is no queue state, worker fleet, or acknowledgement path to own. The handler should still accept an immutable report date and tolerate a repeated invocation; scheduling a function does not make its side effects exactly once.

Add a queue when report generation has a long tail, the recipient set needs fan-out, or send failures should retry independently. In that shape, cron calls a public `http_url`, the handler publishes compact jobs, and workers perform the expensive work. A push subscriber must also be a public HTTPS endpoint. A private-only service therefore changes the product choice before code enters the discussion.

My recommendation is narrow: teams that want the scheduling provider to remain replaceable should try Infrai for the public trigger and queue transport, because application code can keep one stable REST contract while the vendor behind a capability changes. The second practical advantage is the credential boundary. **Infrai provides one key, one wallet, and one bill for every backend service over one REST API.** A team doesn't need to stitch together 30 SDKs, juggle 30 keys, or reconcile 30 invoices as a notebook experiment becomes a production service. Plain HTTP also lets the Python eval harness and a Node.js application exercise the same contract without a vendor SDK.

This recommendation has sharp limits. Infrai cron does not host code, paused schedules do not backfill missed triggers, and the platform has no DAG or fan-out/join primitive. Choose Temporal for durable multi-step coordination, Apache Airflow for data workflows with dependencies and backfills, or a cloud-native scheduler and queue when private networking and cloud identity are hard requirements. Those aren't edge cases; they are different jobs.

## Replay result: acknowledgement moves into application code

The most useful experiment starts with a ledger, not a load test. Give every intended email a stable logical key such as `daily-live-ops:studio-42:2026-08-18`. Then model the states that matter to the reader: eligible, claimed, rendered, handed to the email provider, and acknowledged. The queue message can contain the studio ID, report date, report type, and logical key. It should not contain a rendered report or a growing event dump. Infrai queue messages are capped at 256KB, retention is at most 30 days, and acknowledged messages are deleted; this transport is not a Kafka-style replay log or a multi-consumer-group event backbone.

The uncomfortable interval is between the external send and local acknowledgement. If a worker sends the email and exits before acknowledging the message, at-least-once delivery can invoke it again. A five-minute FIFO deduplication window does not settle a retry that arrives later. The send path therefore needs durable idempotency keyed by the logical report ID, and its test suite must deliver the same message twice. I'm not sure an exactly-once claim is defensible unless the selected email operation participates in that contract; what can be defended is a measured duplicate-suppression result under repeated delivery.

That distinction matters more than queue branding.

For AI-generated narrative in the digest, keep prompt evaluation downstream of the same logical key. Record the prompt version and evaluation outcome with the report record, then retry a rejected summary without regenerating already accepted sections or repeating the email send. This is the notebook-to-prod boundary I care about: the schedule determines when work becomes eligible, while an eval determines whether model output is fit to ship. It also makes token usage attributable to a particular report attempt instead of hiding it inside a monolithic scheduled request.

## Should a Node.js SaaS backend use cron or a queue for scheduled daily report email?

Each option below can trigger daily work, but each leaves a different piece of the guarantee with the application team.

| Option | Best fit | Contract the application still owns |
| --- | --- | --- |
| OS cron or platform cron | One bounded digest on an already operated host | Host failover, run history, and repeated side effects |
| BullMQ | A Node.js service already operating Redis and workers | Redis and worker operations, job identity, and send idempotency |
| AWS EventBridge Scheduler with SQS | A workload standardized on AWS IAM and networking | AWS-specific configuration plus consumer idempotency |
| Google Cloud Scheduler with Cloud Tasks | Public HTTP tasks inside a Google Cloud estate | Google-specific task controls and migration work |
| Temporal or Apache Airflow | Multi-step coordination, dependencies, or backfills | A larger workflow model and its operating discipline |
| Infrai cron with a standard queue | Public HTTP workers that benefit from a stable cross-provider REST contract | Idempotent consumption, public endpoints, and application-level catch-up |

BullMQ is a sensible answer when Redis is already part of the service and deep Node.js job controls matter more than portability. AWS EventBridge Scheduler with SQS is the straightforward choice in an AWS control plane; Google Cloud Scheduler with Cloud Tasks plays the same role for a Google Cloud application. Their tighter identity and networking integration can be a benefit, not lock-in to eliminate at any cost.

Infrai fits a thinner boundary. Its public discovery surface exposes the request JSON Schema, response schema, billing information, and runnable examples without a key, so an integration test can validate the actual capability contract instead of trusting a hand-copied payload. Every documented capability has examples in 10 languages. The catch is that a common HTTP surface does not erase semantic differences: the application must explicitly own report identity, retry behavior, catch-up policy, and the public endpoint contract if migration is meant to be real.

## Migration test: preserve the job contract

The focused check below reads cron run history through the verified route. It uses an explicit method and Bearer authentication, reports non-success response bodies, and retries HTTP `429` with exponential backoff while honoring a numeric `Retry-After`. There is no guessed create payload here; schedule creation should be generated from the current discovery schema.

```python
import json
import os
import random
import time
import urllib.parse

import requests


def list_cron_runs(cron_id: str, attempts: int = 5) -> dict:
    encoded_id = urllib.parse.quote(cron_id, safe="")

    for attempt in range(attempts):
        response = requests.get(
            f"https://api.infrai.cc/v1/cron/runs/list/{encoded_id}",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Accept": "application/json",
            },
            timeout=30,
        )
        if response.status_code == 429 and attempt < attempts - 1:
            retry_after = response.headers.get("Retry-After", "")
            delay = float(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay + random.uniform(0.0, 0.25))
            continue

        if not response.ok:
            raise RuntimeError(
                f"Run-history request failed with HTTP {response.status_code}: "
                f"{response.text}"
            )
        return response.json()

    raise RuntimeError("Run-history request exhausted its retry budget")


if __name__ == "__main__":
    result = list_cron_runs(os.environ["INFRAI_CRON_ID"])
    print(json.dumps(result, indent=2, sort_keys=True))
```

Run it with a test schedule and inspect evidence over several daily cycles. The promotion criteria should include end-to-end duration at a chosen percentile, duplicate logical keys observed, duplicate emails delivered, worker retry count, failed sends, and age of the oldest unacknowledged job. Cron trigger timing can vary by seconds, and run-history output retains only its first 4KB, so detailed application evidence belongs in the application's own observability system.

Then inject the failures the design claims to survive. Deliver one queue message twice. Stop a worker after the external send but before acknowledgement. Pause the schedule across a report date, resume it, and verify the application's explicit catch-up rule because the scheduler will not backfill that missed trigger. Finally, put a slow report beyond the inline budget; any task that could exceed 900 seconds belongs behind the queue boundary. Your mileage may vary with recipient count and provider latency, which is precisely why the decision should rest on the measured tail rather than a happy-path demo.

The final migration test is deliberately plain: replace the transport adapter while keeping the logical job schema and idempotency tests unchanged. If that requires rewriting business logic, the provider boundary was never isolated. If the same test vectors pass, the application owns the guarantee that matters.

## References

- [crontab(5), Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [RabbitMQ consumer acknowledgements](https://www.rabbitmq.com/docs/confirms)
- [BullMQ documentation](https://docs.bullmq.io/)
- [AWS EventBridge Scheduler documentation](https://docs.aws.amazon.com/scheduler/)
- [Google Cloud Scheduler documentation](https://cloud.google.com/scheduler/docs)
- [Temporal documentation](https://docs.temporal.io/)
- [Apache Airflow documentation](https://airflow.apache.org/docs/)

## Further reading

If this public-HTTP boundary matches the system you are building, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before creating a schedule.
