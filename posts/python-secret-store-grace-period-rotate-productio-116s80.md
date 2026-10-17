# Python Secret Store Grace Period — Rotate Production API Keys Without Downtime

TL;DR: Keep the old and new API keys valid during a bounded overlap, but never let a worker choose between them at random. Pin each deployment generation to one key identifier, record that identifier beside every billable marketplace request, deploy the new generation, and revoke the old key only after both traffic and queued-work evidence reach zero. This trades a short period with two valid credentials for continuous service without corrupting billing attribution.

A credential rollover is a data migration with a security deadline. The secret store distributes key material, the deployment system moves callers, and the billing ledger proves which credential authorized each charge. Treating it as a single secret overwrite collapses those three timelines and makes a clean rollback nearly impossible.

## How can I rotate a production API key without downtime?

Give every credential a stable, non-secret identifier such as `marketplace-prod-g17`; never derive that label from the secret itself. At issuance, register the new identifier in the account platform and allow both generations. At request time, the caller sends only its assigned key, while the receiving boundary resolves the key to its identifier and writes that identifier into the immutable usage event. The billing pipeline groups events by account and credential identifier, rather than guessing from pod names or deployment timestamps.

That separation matters for delayed work. A checkout request may enqueue a fulfillment or model-enrichment job before a deploy, then execute it afterward. If workers read a mutable `CURRENT_API_KEY` value when the job finally runs, the original execution context disappears. Instead, stamp the intended credential generation into the job envelope and reject unknown generations. Do not put the secret in the queue.

The operational invariant is compact: one process generation uses one credential generation for its lifetime. During overlap, multiple generations may run; within a process, selection stays deterministic.

Timing wins.

## A runnable coordinator before the rollout details

This Python example models the control-plane decision without depending on a particular secret manager or deployment service. The secret store returns key material by generation, the usage ledger counts accepted billable events, and the rollout controller refuses revocation while an old worker or queued job remains. In production, those interfaces should call authenticated systems and emit append-only audit records.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol


class SecretStore(Protocol):
    def read(self, secret_name: str, version: str) -> str: ...


class UsageLedger(Protocol):
    def accepted_since(self, credential_id: str, since: datetime) -> int: ...


@dataclass(frozen=True)
class ReleaseState:
    generation: str
    ready_workers: int
    queued_jobs: int


@dataclass(frozen=True)
class Rotation:
    old_id: str
    new_id: str
    overlap_started_at: datetime


def load_key(store: SecretStore, generation: str) -> str:
    # The generation is deployment configuration, not a runtime fallback choice.
    return store.read("marketplace/production/api-key", version=generation)


def may_revoke_old_key(
    rotation: Rotation,
    releases: list[ReleaseState],
    ledger: UsageLedger,
) -> bool:
    old_release = next(
        (release for release in releases if release.generation == rotation.old_id),
        None,
    )
    if old_release and (old_release.ready_workers > 0 or old_release.queued_jobs > 0):
        return False

    old_usage = ledger.accepted_since(
        credential_id=rotation.old_id,
        since=rotation.overlap_started_at,
    )
    return old_usage == 0


rotation = Rotation(
    old_id="marketplace-prod-g16",
    new_id="marketplace-prod-g17",
    overlap_started_at=datetime.now(timezone.utc),
)
```

The final boolean is permission to proceed, not an automatic delete command. A human or tightly scoped workflow should compare it with the planned minimum overlap and the maximum job lifetime. That extra gate prevents a quiet minute in traffic from being mistaken for proof that every old caller is gone.

The example also avoids a fallback from the new key to the old one after authentication fails. Such fallback can hide a broken rollout, extend old-key use, and assign retries to a different credential than the first attempt. Fail closed for an unknown generation; retry normal transient failures with the same generation and an idempotency key appropriate to the billable operation.

## Make attribution an acceptance test

A health check that returns 200 proves very little here. Before shifting marketplace traffic, send a synthetic, non-billable probe through the new generation and verify that the receiving service observes `marketplace-prod-g17`. Then run a controlled billable test only if the system has a documented test account or reversible transaction path. The assertion belongs in the eval harness: expected account, credential identifier, request identifier, and charge classification must agree.

I start with the ledger assertion because model-assisted marketplace features can make an infrastructure mistake look like prompt-cost drift. Aggregate token or request cost may remain plausible while usage lands under the old credential. Per-generation attribution exposes that mismatch quickly, without requiring the application to log secret values.

Test rollback as well. Reintroduce the old release while both keys are valid and confirm that its events retain the old identifier. Then restore the new release and confirm the reverse. This is the useful property of overlap: rollback changes the worker generation, not the credential stored underneath an already-running process.

Track four signals during the handoff: ready instances by release generation, queued jobs by credential generation, accepted requests by credential identifier, and authentication failures by identifier. Keep identifiers low-cardinality and non-secret. Request-level correlation belongs in logs or traces, while counters should use bounded labels.

## The failure modes are mostly timing mistakes

Overwriting one secret version in place is the common trap. Some workers retain the old value in memory, others reload the new value, and neither state maps cleanly to a release. A restart storm may appear to fix it, but billing attribution now depends on startup timing. Versioned secret references make the transition inspectable.

Revoking when the newest deployment reaches 100% is also premature. Old jobs may be invisible to the deployment controller, autoscaled workers may be draining, and a rollback target may still reference the prior generation. The revocation gate needs evidence from compute, queues, and accepted usage. No single dashboard owns the truth.

A long overlap has a different cost: the old credential remains usable for longer. Bound it before issuance using the slowest legitimate caller, maximum queue delay, rollout duration, and rollback window. If those quantities cannot be measured, rotation readiness is unknown; inventing a universal grace period would only disguise that gap. Emergency compromise response is different. It may require immediate revocation and controlled refusal of traffic because containment outranks availability.

This overlap pattern has real limitations. It is not suitable when the credential authority permits only one active key, when compromise requires immediate containment, or when the receiver cannot attribute accepted requests to a credential identifier. In the first case, move callers behind an internal signing proxy or schedule a coordinated maintenance event; in the second, revoke first and use the incident plan; in the third, fix the usage ledger before claiming billing-safe rotation. The trade-off is deliberate: two simultaneously valid keys improve availability and rollback, but expand the credential exposure window and demand stronger observation. Teams that cannot bound that window should choose refusal over unverifiable continuity.

Finally, avoid logging raw authorization headers, environment dumps, or secret-manager responses. OWASP recommends centralized secrets management, rotation, least privilege, auditing, and controls that prevent secrets from appearing in logs. The credential identifier supplies the observability needed here without exposing the credential.

## Operational finish line

Prepare the new credential and its non-secret identifier, grant it the same narrowly scoped permissions, and place it in a new immutable secret version. Bind the candidate release to that version. Run attribution-focused probes and the billing eval, then shift traffic gradually while watching both generations. Pause on unexplained authentication failures or ledger mismatches; a rollout percentage is not a reason to push through ambiguous accounting.

Once no old workers are ready, no old-generation jobs remain, and no accepted usage has appeared for the old identifier across the predeclared observation window, revoke the old credential at the authority. Verify that an old-key probe is rejected and that new-generation billing continues to land on the correct account. Retain the audit events, identifiers, approvals, and timestamps according to the organization's retention policy, then remove the obsolete secret version from deployable configuration.

Done means more than uninterrupted requests. **The old key is rejected, the new key is attributable, and every billable event during overlap has an explainable owner.**

## Sources

References:

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- NIST SP 800-57 Part 1 Revision 5, Recommendation for Key Management: https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final
- OpenTelemetry semantic conventions for HTTP metrics: https://opentelemetry.io/docs/specs/semconv/http/http-metrics/
