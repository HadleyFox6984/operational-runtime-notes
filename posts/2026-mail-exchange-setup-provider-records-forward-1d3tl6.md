# 2026 Mail Exchange Setup: Provider Records, Forwarding Hosts, MX Priorities, API Evidence

Short answer: for an e-commerce mail cutover, choose the provider path that lets you prove SPF, DKIM, DMARC alignment, and MX behavior with repeatable DNS evidence; treat forwarding as a separate hop, not as an MX failover plan.

I build RAG and agent features in Python, so I tend to test an assumption in a notebook before it becomes deployment code. Mail routing deserves the same discipline. A green DNS edit is not proof that a password reset reached an inbox.

The useful unit of work is a small evidence record: the zone, the observed answer, the resolver used, the timestamp, and the message-flow test that backs it up. That record gives a team something to compare when a provider, forwarding host, or registrar changes.

## Why an MX answer is not a delivery guarantee

An MX record names hosts willing to accept mail for a domain. Its preference value supplies an ordering rule: lower numbers are preferred, while equal preferences allow a sender to choose among hosts. That is a routing instruction, not a promise that an application will accept a particular message.

For a store, the failure mode is easy to miss. The storefront may send from `orders.example` while the support team replies from `example`. If the published SPF policy covers one sending path, DKIM signs another, and DMARC evaluates the visible From domain, a message can pass one check and still fail alignment. RFC 7489 defines that alignment and the reporting model; it does not make a forwarding host transparent.

Forwarding adds another SMTP hop. The forwarder can preserve the original message, alter headers, or apply its own filtering policy. Do not put a forwarding inbox at a lower MX preference and call it disaster recovery unless it is actually prepared to accept, queue, and later deliver the domain's mail. That operational contract is bigger than a number in DNS.

## How should a provider's MX records and forwarding hosts share priority?

Start with roles, then assign priorities. Primary inbound hosts receive mail for the domain. A forwarding service is either an intentional destination for a mailbox or a relay with a documented handoff; it is not automatically a standby server. Keep the MX set small enough to observe, and record the expected order in version control.

Here is the kind of fixture I use in an evaluation harness. It is deliberately provider-neutral.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class MxObservation:
    name: str
    preference: int
    host: str
    resolver: str
    observed_at: str


expected = [
    MxObservation("example.com", 10, "inbound-a.example.net.", "1.1.1.1", "2026-02-14T09:30:00Z"),
    MxObservation("example.com", 20, "inbound-b.example.net.", "1.1.1.1", "2026-02-14T09:30:00Z"),
]

assert [row.preference for row in expected] == sorted(row.preference for row in expected)
```

The assertion catches an accidental preference inversion, but it cannot verify acceptance, TLS, queueing, or DMARC alignment. Those need message-level probes and aggregate reports. I keep the probe data separate from production credentials, and I compare the result against the same fixture after each DNS change.

One more detail matters: DNS answers are cached. A low TTL can reduce the waiting window, but recursive resolvers and receiving systems still make their own timing decisions. A cutover plan should include an overlap period and a rollback record, not a promise that every resolver changes at the same minute.

## A practical evidence loop for SPF, DKIM, and DMARC

The loop has four checks. First, query the authoritative zone and at least one independent recursive resolver. Second, inspect SPF for the actual sending services and keep the record within DNS lookup limits. Third, verify that the DKIM selector in the message matches the public key published in DNS. Fourth, publish a DMARC policy that reflects the enforcement stage and review its aggregate reports under RFC 7489.

I once assumed a successful test message meant the job was done. It didn't. The test used a domain with a different From address, so it never exercised the store's customer-receipt path. That small mismatch cost more debugging time than the DNS edit itself.

Use a matrix instead: order confirmation, refund notice, and support reply on one axis; SPF result, DKIM result, DMARC alignment, MX destination, and final mailbox outcome on the other. For each row, save the exact From domain, selector, Return-Path, receiving host, message identifier, and resolver answers, then repeat the test from a clean mailbox after the DNS TTL window. If a forwarder rewrites the envelope or changes the visible headers, that difference belongs in the record too, because it changes what DMARC can evaluate. A missing cell is a release blocker. A passing cell with no captured headers is only a hypothesis.

Measure it.

```python
checks = {
    "order_confirmation": {"spf": "pass", "dkim": "pass", "dmarc": "pass", "mx": "inbound-a"},
    "refund_notice": {"spf": "pass", "dkim": "pass", "dmarc": "pass", "mx": "inbound-a"},
    "support_reply": {"spf": "pass", "dkim": "pass", "dmarc": "pass", "mx": "forwarder"},
}

required = ("spf", "dkim", "dmarc", "mx")
assert all(all(checks[flow].get(key) for key in required) for flow in checks)
```

Your mileage may vary on how quickly aggregate reports become useful; mailbox providers do not all report on the same schedule. That uncertainty is a reason to watch trends during the overlap, not a reason to skip the baseline.

## Choosing an API without coupling the mail path

An API can make zone changes auditable, but the interface is not the architecture. Keep a narrow adapter that accepts a desired record set, validates names and preferences, applies an idempotent change, and stores the resulting observation. The rest of the application should not know which control plane performed the update.

For a Python service, make DNS changes a reviewed job. Generate the proposed SPF, DKIM, DMARC, and MX records from configuration; require a diff; then run the evidence loop after propagation. Do not let a Node.js or Python checkout path mutate DNS during a customer request. A transient control-plane response should never decide whether an order is delivered.

The catch is that this method is not suitable when a team needs a full mailbox platform, legal retention, or guaranteed inbound queue ownership. In that case, select a provider with those operational commitments and keep the same evidence checks around it. Stick with a forwarding-only design when the domain has no mailbox workload and the relay's rewrite and replay behavior are documented; otherwise, use a receiving service that owns the queue.

## The decision rule I would ship

Ship the option that produces the clearest evidence for the least complicated mail path. Compare candidates on acceptance responsibility, forwarding semantics, SPF lookup budget, DKIM key rotation, DMARC reporting, API auditability, and rollback procedure. Price can be part of procurement, but it should not outweigh a missing queue contract or an untestable alignment story.

Before launch, save authoritative and recursive DNS answers, capture headers from each message class, and record the final mailbox result. Re-run that set after every provider handoff. If a team cannot show those artifacts, it does not yet know whether its MX priorities work.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://www.rfc-editor.org/rfc/rfc5321
- https://www.rfc-editor.org/rfc/rfc7208
- https://www.rfc-editor.org/rfc/rfc6376
