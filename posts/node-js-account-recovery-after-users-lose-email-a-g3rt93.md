# Node.js Account Recovery After Users Lose Email Access (Social Sign-In)

Account recovery gets dangerous when a user has lost email access: a developer tool cannot treat the inbox, or a matching address from another social login, as permanent proof of identity. Someone who signed in with Google or GitHub may also lose one provider while retaining the other. Recover the existing internal account only after independent proof, and prevent an operator from casually overriding identity.

**TL;DR:** give every user an immutable internal ID, and attach social identities to it as separate records. A matching email is a routing hint, never proof of ownership. Accept a still-working linked provider, a previously enrolled recovery factor, or a tightly controlled review path. If none succeeds, preserve the old account and create a new one rather than guessing.

## How should account recovery work after a user loses email access?

The simple implementation looks attractive: accept a successful social login, read its email, and select the user row with that value. It collapses three different things into one field: a contact address, a provider assertion, and the application's durable identity key. Recovery exposes the mistake.

The safer model makes the distinction visible. A local `user_id` owns projects, API keys, audit history, and organization membership. A linked identity is keyed by a provider namespace plus that provider's subject identifier. Email remains mutable contact data with its own verification state. Google and GitHub sign-in can coexist without either email becoming the primary key.

Do not auto-link two identities merely because their email strings match. The person should prove control of the current account and then authenticate with the identity being added. If access to the current account is already gone, that operation belongs to recovery and must satisfy its policy.

No proof, no link.

This boundary matters for developer tools. A recovered account may control source integrations, deployment credentials, or organization settings. A wrong merge can move authority across trust boundaries.

## Model proof paths before endpoints

Start with evidence, not screens. The recovery service should receive a request tied to a candidate internal account, collect one allowed proof path, and issue a narrowly scoped decision. It should not reveal whether an account exists through different public messages or conspicuously different behavior. OWASP recommends generic responses for authentication and recovery-related account flows to reduce user enumeration.

| Available evidence | Recovery decision | Main trade-off |
| --- | --- | --- |
| Active session or working linked provider, followed by authentication of a new provider | Permit identity linking | Strong continuity, unavailable after total credential loss |
| Previously enrolled independent recovery factor | Permit constrained recovery | Useful self-service, with factor-lifecycle burden |
| Human review under a documented policy | Delay, log, and separately approve high-impact accounts | Covers rare cases, but adds social-engineering and operator risk |
| Only a matching email, profile name, repository name, or public activity | Reject as insufficient | More false negatives, fewer speculative takeovers |

The last row is deliberate. Public account details can help support locate a record, but locating it is not authenticating its owner. The decision logic and operator interface should preserve that distinction. Recovery also should not silently become account merging. If one person created two local accounts through separate providers, an authenticated merge needs rules for project ownership, organization membership, API credentials, and audit history. Picture the concrete case: the Google-created account owns a private project and an API key, while the GitHub-created account holds an organization role. Joining those rows based on one matching email does not answer which sessions survive, where the key belongs, or whether the role may move. Each of those is an authorization decision. Keeping merge outside an urgent recovery attempt reduces the number of irreversible actions performed under pressure and makes the recovery evaluation much easier to read.

## A focused policy check

Even when the production service is Node.js, the policy can be specified independently and exercised in the same Python notebooks used for eval work. The important line is the one absent from the decision: `email_match` never satisfies a proof requirement.

```python
from dataclasses import dataclass
from enum import Enum


class Decision(str, Enum):
    ALLOW_LINK = "allow_link"
    REQUIRE_REVIEW = "require_review"
    DENY = "deny"


@dataclass(frozen=True)
class RecoveryEvidence:
    authenticated_linked_identity: bool
    verified_recovery_factor: bool
    review_eligible: bool
    email_match: bool


def decide_recovery(evidence: RecoveryEvidence) -> Decision:
    if evidence.authenticated_linked_identity or evidence.verified_recovery_factor:
        return Decision.ALLOW_LINK
    if evidence.review_eligible:
        return Decision.REQUIRE_REVIEW
    return Decision.DENY
```

That example is intentionally small. In production, `review_eligible` must come from a documented rule rather than an operator's intuition, and `ALLOW_LINK` should authorize only the next recovery step. It should not become a reusable login credential or grant every sensitive action immediately.

Build the eval set around distinct failures: Google is lost but an authenticated GitHub identity remains; GitHub is lost but a recovery factor remains; both providers are unavailable and the email matches; an active session tries to link an identity already attached elsewhere; support can locate the account but cannot establish proof. Give each case an expected decision and expected side effects.

One trap is evaluating only the happy path. Denied cases carry most of the security meaning. Test that denial leaves identity links, project ownership, sessions, and contact data unchanged. Test concurrent attempts too: two individually valid requests must not attach the same external identity to different users.

## Recovery is a controlled state transition

A practical implementation records a pending attempt, its purpose, expiry, consumed state, and the proof class that satisfied policy. The final write verifies that the attempt remains valid, attaches the new identity, updates contact data only when separately verified, and emits an audit event as one controlled transition. A uniqueness constraint should prevent one external identity from being attached twice. Keep the blast radius narrow. After recovery, rotate or revoke sessions according to policy, notify an already established contact channel where that does not disclose sensitive data, and require fresh authentication before high-impact changes. OWASP calls for reauthentication after risk events such as account recovery and session invalidation or token rotation after reauthentication. Support is part of this security model, so an operator should see which proof class passed, which action is permitted, and which evidence is merely contextual; direct identity-record editing turns ticket urgency into authority. For privileged accounts, separate review from execution and preserve the audit trail. There is a real trade-off: a strict policy will strand some legitimate users who enrolled no independent factor and lost every linked provider, while a permissive policy transfers that pain into takeover risk. The honest design names this constraint during enrollment, offers a second path early, and refuses to manufacture certainty later.

That trade-off cannot be automated away.

## What should the experiment measure?

Before copying this choice, run it against the application's actual authority graph. Measure completion by proof path, abandonment before proof, review volume, review disagreement, denied attempts, link conflicts, and time from initiation to completion. Break results down by account privilege without exposing user enumeration in the public flow.

Security metrics alone are incomplete. Track duplicate-account creation after failed sign-in because it shows where users cannot distinguish login, linking, and recovery. Track how often recovery is followed by sensitive changes. Watch support overrides separately. Zero overrides may mean the policy is clear, or it may mean legitimate users have no viable escalation path; sampled case review resolves that ambiguity better than a dashboard total.

Cost belongs in the experiment, but it is not the decision. Human review time, notification delivery, audit retention, and another recovery factor consume resources. Prompt or model calls should not decide identity ownership: probabilistic output is difficult to audit, and public profile evidence is easy to imitate. If an AI system summarizes a support case, keep its output advisory and exclude evidence it does not need.

Ship the policy with a replayable evaluation corpus. Every rule change should rerun allowed and denied cases, with attention to side effects and ambiguous evidence. This notebook-to-production path keeps the recovery contract inspectable while Node.js handlers, the database transaction, and the interface evolve around it.

The decision rule is short: recover through a previously established, independently verifiable path; treat email as contact data; preserve the account when proof is insufficient. That answer can feel unfriendly in the hardest ticket. It is safer than converting familiarity, urgency, or a matching string into ownership.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
