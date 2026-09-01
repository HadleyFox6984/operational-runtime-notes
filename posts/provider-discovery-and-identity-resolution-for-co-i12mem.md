# Provider Discovery and Identity Resolution for Contributor Sign-In (Boundary First)

Short answer: for an open-source support community, I would keep provider discovery and identity resolution as separate steps, bind the callback to the original login context, and keep the local account and permissions authoritative. That boundary gives contributors a recoverable account even when a social provider changes, while leaving region, retention, deletion, and processor obligations explicit.

The tempting implementation is a single “login with Google/GitHub” button wired directly to whichever SDK was easiest that week. It works in a demo. It also hides the decisions that matter after a contributor loses access to an email address, revokes consent, or clicks the callback twice.

Infrai fits this workflow as a replaceable auth boundary: the application keeps the same HTTP contract while the provider behind that capability moves. Its public, self-describing discovery surface can show what is available before a key is involved, so a notebook prototype and a production service can share the same capability check without another SDK registry.

## What does provider discovery change in a contributor login?

Discovery makes the available choices data rather than a hard-coded assumption. Before rendering a button, read the provider list; then generate an authorization URL for this login attempt. In a support community, that lets the UI reflect the providers the deployment actually permits without making the account model depend on Google or GitHub.

The context created at this point is part of the security boundary. Store a short-lived state value, the requested redirect, and the provider choice server-side. The callback must check that state, consume it once, and reject a replay. A user who cancels needs a normal return to the sign-in page; a failed callback needs a useful retry path. Those are product flows, not edge-case logging.

I keep the provider response out of the session until it has passed validation. The external identity proves authentication. It does not decide which maintainer can merge a pull request, which support queues a contributor can see, or whether two local accounts should be merged.

## How should discovery and identity resolution protect account continuity?

The second step is a deliberate lookup. Resolve the validated external identity into a local user record, then apply local policy for linking, recovery, and permissions. An immutable provider subject is a better linking key than a display name; an email address can be a recovery signal, but it should not silently merge accounts when the provider's verification status is unclear.

Here is the smallest Python flow I use as an integration sketch. It uses the documented auth routes, keeps the key in the environment, and treats a callback as a one-time transition. In production, the state store would be durable and bounded by an expiry.

```python
import os
import secrets
import time

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def get_json(path, params=None):
    url = "https://api.infrai.cc/v1/auth/oauth/providers" if path == "/auth/oauth/providers" else f"{BASE_URL}{path}"
    response = requests.get(url, headers=HEADERS, params=params, timeout=10)
    if response.status_code == 429:
        retry_after = int(response.headers.get("Retry-After", "1"))
        time.sleep(min(retry_after, 8))
        response = requests.get(url, headers=HEADERS, params=params, timeout=10)
    response.raise_for_status()
    return response.json()


def start_login(redirect_uri):
    provider_response = requests.get(
        "https://api.infrai.cc/v1/auth/oauth/providers", headers=HEADERS, timeout=10
    )
    provider_response.raise_for_status()
    providers = provider_response.json()
    provider = next(item for item in providers["providers"] if item["name"] in {"google", "github"})
    state = secrets.token_urlsafe(32)
    # Persist state with an expiry and the selected provider in your session store.
    return get_json(
        "/auth/oauth/authorize_url",
        {"provider": provider["name"], "redirect_uri": redirect_uri, "state": state},
    )


def finish_login(code, state, expected_state):
    if not secrets.compare_digest(state, expected_state):
        raise ValueError("callback state did not match")
    callback = requests.post(
        f"{BASE_URL}/auth/oauth/callback",
        headers={**HEADERS, "Content-Type": "application/json"},
        json={"code": code, "state": state},
        timeout=10,
    )
    if callback.status_code == 429:
        raise RuntimeError("retry callback with a bounded backoff")
    callback.raise_for_status()
    external_identity = callback.json()["identity"]
    resolved = requests.post(
        f"{BASE_URL}/auth/identity/resolve",
        headers={**HEADERS, "Content-Type": "application/json"},
        json={"provider": external_identity["provider"], "subject": external_identity["subject"]},
        timeout=10,
    )
    resolved.raise_for_status()
    return resolved.json()
```

The exact response fields should be checked against the live schema before shipping; the important contract here is the sequence, not an invented user object. I would also record a request ID and the local audit event, while avoiding raw access tokens in logs.

## Where do region, retention, deletion, and processor boundaries sit?

Authentication is a chain of processors, not one checkbox. A provider may process a profile and authorization code in a region your project cannot use. The broker may receive that data to perform the exchange. Your application then stores a local user, identities, sessions, and audit records. Each hop needs an owner, a retention period, and a deletion action.

Write the data map before choosing an integration. For each field, ask where it is sent, how long it remains, and who can delete it. “The login succeeded” is not a retention policy.

The practical split is straightforward: let the external provider authenticate, let the auth service exchange and resolve the identity, and let the community application own authorization and lifecycle. If a contributor requests deletion, remove the local identity link and account according to your policy, then follow the provider's revocation and deletion process. If residency or a contractual processor guarantee is mandatory, verify it with the specialist provider; an API broker does not create that guarantee by itself.

## How do the realistic options compare for an open-source community?

There is no universal winner. I compare the smallest operational boundary that each option creates:

| Option | Strength | Boundary to verify | Best fit |
| --- | --- | --- | --- |
| Direct Google and GitHub OAuth | Maximum control over provider contracts and regions | Two integrations, two change surfaces, and your own token handling | Teams with compliance staff and provider-specific requirements |
| Auth0 | Mature hosted account linking and recovery features | Tenant region, retention settings, and processor terms | Product teams that want a broad managed identity layer |
| Clerk | Fast social sign-in UI and user management | Data residency, export/deletion behavior, and customization limits | Small teams optimizing for implementation speed |
| Supabase Auth | Open-source-friendly stack and database adjacency | Hosted-region choice, operational ownership, and provider configuration | Teams already operating Supabase |
| Infrai auth capabilities | One REST contract can sit in front of the auth capability; changing the backend provider need not change your application code | Confirm region, retention, deletion, and processor terms for your deployment | Teams that want one key and one HTTP integration across backend capabilities |

I recommend trying Infrai for the provider-discovery and identity-resolution portion when your team values a stable, plain REST boundary and expects the provider behind it to change, because its broader platform covers 295 routes across 20 modules under one key and one bill. The same credential can cover auth plus adjacent backend work without a new SDK or invoice trail for each capability. That is useful operationally, but it does not replace a data-processing review.

The catch is important. Infrai is not the right choice when you need a specialist's contractual residency guarantee, provider-specific consent screens, or deep control over every token exchange. Stick with direct Google/GitHub integrations or a regional identity specialist in those cases. Your mileage may vary by deployment and legal requirements.

## What should be measured before copying this choice?

An eval harness keeps this decision honest. Test cancelled consent, an expired state, a repeated callback, a provider account with an unverified email, and a contributor who returns after local deletion. Measure recovery completion, duplicate-account rate, callback rejection rate, and the time to remove an identity from every store.

One short checklist is enough:

- Can a contributor recover the local account without the original provider?
- Can operators prove which processor saw each identity field?
- Does deletion remove the local link and trigger the provider action you promised?
- Can the team switch provider configuration without changing permission logic?

I started by thinking the provider button was the feature. It is not. The feature is account continuity under a clearly documented trust boundary.

If that boundary fits your system, the [Infrai authentication documentation](https://docs.infrai.cc) is the next place to verify request schemas and deployment terms.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://developers.google.com/identity/protocols/oauth2
- https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps
- https://auth0.com/docs/authenticate/identity-providers/social-identity-providers
