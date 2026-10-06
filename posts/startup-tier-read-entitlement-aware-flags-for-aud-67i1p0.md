# Startup Tier Read: Entitlement-Aware Flags for Auditable Production Key Rotation

Treat the startup tier read as a versioned authorization snapshot, then derive immutable feature flags and quota limits from that snapshot. Keep API credentials in a separate, overlapping key ring. This separation lets a media service rotate a production key without silently changing what an account may do, and it gives every access decision a stable audit identity.

**TL;DR:** load and validate one entitlement snapshot before accepting traffic; expose its derived flags through a read-only process-wide object; record the snapshot version on consequential operations; and rotate credentials through an explicit prepare, switch, verify, and retire sequence. Never put raw keys, hashes of keys, or bearer values in entitlement records or audit events.

## How should an entitlement-aware service read a tier and expose feature flags?

A production media worker answers two different questions. The credential asks, “may this process authenticate to the upstream API?” The entitlement snapshot asks, “may this account transcode 4K media, run a bulk export, or consume another unit of quota?” Combining those answers in one configuration value makes an emergency credential rotation look like a plan change. It also leaves an ambiguous audit trail: did access change because an account moved tiers, or because an operator replaced a secret?

The clean data flow is small. A deployment mechanism supplies a current and next credential by opaque key ID. At process startup, an entitlement reader fetches or receives a versioned account snapshot, validates its shape, and maps tier capabilities into local flags and quota limits. Request handlers read that frozen object. Audit events carry the account ID, entitlement version, decision, reason, and credential ID, but never the credential value.

Keep those clocks separate.

This boundary matters more than the flag library. A remotely evaluated flag can be useful, but it should not become a second, undocumented entitlement authority. For paid capabilities, one owned mapping from entitlement data to runtime policy is easier to test and review. The mapping should be boring enough to inspect in one sitting: capability names enter, known booleans and numeric limits leave, and any unfamiliar input stops the deployment before it can reinterpret access. That deliberately gives up live, per-request tier changes. In return, every request handled by one process sees the same entitlement version, which is the more useful property during a credential rotation and its audit review.

## Build the snapshot before serving traffic

The following Python example is deliberately compact enough for a notebook, yet its seams match a production service: input parsing, policy derivation, request-time checks, key selection, and audit output are separate. The sample data represents a media account allowed to publish standard video while 4K transcodes remain disabled.

```python
from __future__ import annotations

from dataclasses import dataclass
from types import MappingProxyType
from typing import Callable, Mapping
import json


@dataclass(frozen=True)
class EntitlementSnapshot:
    account_id: str
    tier: str
    version: str
    capabilities: frozenset[str]
    quotas: Mapping[str, int]


@dataclass(frozen=True)
class RuntimeAccess:
    account_id: str
    entitlement_version: str
    flags: Mapping[str, bool]
    quotas: Mapping[str, int]


@dataclass(frozen=True)
class ApiCredential:
    key_id: str
    secret: str


def load_snapshot(raw: str) -> EntitlementSnapshot:
    document = json.loads(raw)
    required = {"account_id", "tier", "version", "capabilities", "quotas"}
    missing = required.difference(document)
    if missing:
        raise ValueError(f"entitlement snapshot missing: {sorted(missing)}")

    quotas = {name: int(value) for name, value in document["quotas"].items()}
    if any(value < 0 for value in quotas.values()):
        raise ValueError("quota values must be non-negative")

    return EntitlementSnapshot(
        account_id=str(document["account_id"]),
        tier=str(document["tier"]),
        version=str(document["version"]),
        capabilities=frozenset(map(str, document["capabilities"])),
        quotas=MappingProxyType(quotas),
    )


def derive_access(snapshot: EntitlementSnapshot) -> RuntimeAccess:
    known = {"publish_video", "transcode_4k", "bulk_export"}
    unknown = snapshot.capabilities.difference(known)
    if unknown:
        raise ValueError(f"unknown capabilities: {sorted(unknown)}")

    flags = MappingProxyType(
        {name: name in snapshot.capabilities for name in sorted(known)}
    )
    return RuntimeAccess(
        account_id=snapshot.account_id,
        entitlement_version=snapshot.version,
        flags=flags,
        quotas=snapshot.quotas,
    )


def authorize(access: RuntimeAccess, capability: str, used: int) -> dict[str, object]:
    enabled = access.flags.get(capability, False)
    quota_name = f"{capability}_per_day"
    limit = access.quotas.get(quota_name)
    allowed = enabled and (limit is None or used < limit)
    return {
        "account_id": access.account_id,
        "entitlement_version": access.entitlement_version,
        "capability": capability,
        "allowed": allowed,
        "reason": "allowed" if allowed else "disabled_or_exhausted",
    }


def call_with_rotation(
    credentials: tuple[ApiCredential, ...],
    operation: Callable[[str], str],
) -> tuple[str, str]:
    for credential in credentials:
        try:
            return operation(credential.secret), credential.key_id
        except PermissionError:
            continue
    raise PermissionError("no accepted production credential")


raw_snapshot = json.dumps(
    {
        "account_id": "publisher-042",
        "tier": "studio",
        "version": "ent-1842",
        "capabilities": ["publish_video"],
        "quotas": {"publish_video_per_day": 250},
    }
)

ACCESS = derive_access(load_snapshot(raw_snapshot))
decision = authorize(ACCESS, "publish_video", used=73)
print(json.dumps(decision, sort_keys=True))
```

The process should not announce readiness if required fields are missing, quota values are invalid, or an unknown capability appears. That is a deployment failure, not a reason to guess. A permissive default can accidentally grant a paid feature; a blanket restrictive default can interrupt publishing. Validation before readiness makes the choice explicit and observable.

The example freezes both flags and quotas. That prevents a handler from mutating shared policy halfway through a request. It also keeps prompt and model costs visible: a media-generation endpoint can check its numeric allowance before creating a large prompt, uploading source material, or calling a metered model. Boolean access and consumption accounting remain distinct; a flag says the workflow exists for the account, while a counter decides whether this particular use fits the allowance.

## Rotate through an overlap, not an instant replacement

A no-downtime rotation needs a bounded period in which old and new credentials are both usable. First provision the new secret under a fresh key ID while the current key remains accepted. Deploy readers that can hold the ordered pair. Switch the preferred credential to the new ID, verify successful authenticated operations by key ID, and only then revoke and remove the old one. Keep the fallback narrow: retry the alternate credential only after an authentication rejection, and cap the attempt at the two expected IDs. Timeouts, quota denials, malformed requests, and upstream server errors are not evidence that a credential is stale; replaying them with another secret can duplicate work or hide the real failure. For a publish operation, an idempotency key or an application-level operation ID should survive the credential retry. Fast rollback is the trade-off that justifies the overlap. The cost is temporary exposure to two valid secrets, so the overlap should have an owner and a retirement deadline. OWASP’s secrets-management guidance treats rotation, revocation, expiration, auditing, and least privilege as parts of the secret lifecycle. The practical implication here is simple: audit by stable key ID and action, restrict who can read secret material, and remove the retiring credential after verification rather than leaving it as a permanent fallback.

The overlap is temporary.

Do not log a transformed secret. Hashing a low-entropy or structured credential does not turn it into a safe correlation ID. Generate a non-secret key ID when the credential is issued and log that identifier instead.

## Test policy changes like an API contract

The highest-value tests form a small matrix across snapshot version, capability, current usage, and credential state. For `ent-1842`, verify that publishing at usage 73 is allowed under a limit of 250, that usage 250 is denied, and that an absent `transcode_4k` capability is denied. Then test the rotation states: old only, overlap with old preferred, overlap with new preferred, and new only. The entitlement decision must stay identical across all four.

Add two failure cases that happy-path demos often miss. An unknown capability must stop startup rather than disappear into a false value, because a spelling mismatch between the account system and application is a contract break. An authentication rejection may try the alternate key once, while a timeout must return through the normal retry or error policy without walking the key ring.

For eval-driven development, store the policy cases as fixtures and compare complete decision records, including `entitlement_version` and `reason`. That catches accidental changes to both behavior and audit evidence. It also makes a notebook experiment portable: the same fixture can run against the local mapping function in CI before a deployment changes production access.

## Operate the audit trail as part of the gate

Before rollout, confirm that every replica loads the intended entitlement version and that readiness depends on successful validation. During the credential overlap, watch authentication outcomes grouped by non-secret key ID; do not mix those metrics with feature denials. Sampled request logs are insufficient for consequential access decisions, so emit a structured decision event at the point of authorization with the account, capability, snapshot version, usage input, result, and reason.

Then rehearse retirement. Remove the old key from a canary replica, exercise a real but reversible media operation, and verify that its decision record still points to the same entitlement version. Expand the deployment, revoke the old credential at its authority, and confirm that no replica attempts its key ID. Finally, remove the old secret reference from deployment configuration and close the rotation record with the verifier and timestamp.

The decision rule stays compact: **credentials prove the workload’s identity; entitlement snapshots define the account’s allowed behavior**. Rotate the former without rewriting the latter. When both are versioned, immutable in process, and joined only by audit metadata, a key change becomes an observable operational procedure rather than an untracked feature change.

## References

- OWASP, “Secrets Management Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
