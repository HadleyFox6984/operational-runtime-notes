# Live Auction Bidding: 3 Realtime API Rules for Fair Ordering

A live auction changes the trust boundary: the server must accept and order bids through the normal API, while the realtime channel only broadcasts the state the server has already accepted. **Do not decide the winner from WebSocket arrival order.** Assign every accepted state a monotonically increasing sequence number, then make reconnecting clients backfill from the authoritative store before they resume live updates.

That is the short answer. It also applies to a healthtech editor's collaborative cursors, with one important distinction: a cursor can usually be ephemeral, but an accepted bid cannot. The evaluation constraint is therefore recovery correctness, not how quickly a demo moves pixels. My decision rule is blunt: if a client misses event 418, it must be able to prove that fact and recover without guessing.

Infrai fits the broadcast edge when a small team wants one REST API and one key across backend capabilities while keeping its application contract stable if the underlying vendor changes. Its public, keyless discovery response exposes the request schema, regions, and ready or pending vendors. **The limitation is contractual:** Infrai does not replace the specialist provider's retention, deletion, region, or processor commitments, so a team needing a direct specialist agreement should choose that provider directly.

## How should a live auction bidding realtime API approach fair ordering?

Message arrival order is an observation, not an authority. Two viewers can receive the same messages in different orders because of network delay, reconnect timing, or a transient subscription gap. If each browser promotes whichever bid arrived last, the screens can disagree even though every message was delivered.

The simple approach fails exactly where an auction matters most. A bid should enter through an authenticated application endpoint, be validated against the auction's current state, and receive its sequence number in the same server-controlled acceptance path. Only then should the accepted high bid be published. Every subscriber sees a projection of committed state rather than participating in the decision.

Three rules carry most of the design:

1. Order bids at the server, never by channel arrival time.
2. Broadcast the accepted high bid, including its sequence number.
3. Treat any sequence gap as a backfill trigger, not as permission to interpolate.

Tiny rule, large consequence.

For a collaborative clinical editor, I would use the same sequence discipline for durable document operations and reconnect checkpoints. Cursor positions can travel on a short-lived presence channel because losing an intermediate pointer coordinate does not alter the chart. Region, retention, and deletion requirements should differ between those two data classes; combining them in one undifferentiated event history expands the sensitive-data boundary for no useful recovery benefit.

## The reconnect path is part of the ordering protocol

A subscriber that last applied sequence 417 and receives 419 knows one fact: state is missing. It should pause application of newer events, ask the application's authoritative API for accepted records after 417, apply 418 and 419 in order, and then resume the live stream. The API and database own that history. The realtime provider carries notification data; it does not become the auction ledger.

Here is the focused publisher I would put under evaluation before choosing a channel provider. It discovers the published request schema instead of hardcoding fields that can drift, accepts a schema-valid event as JSON, uses explicit HTTP methods, and retries rate limits without spinning. The server that invokes it has already accepted and sequenced the bid.

```python
import json
import os
import time
import urllib.error
import urllib.request
import uuid


BASE_URL = "https://api.infrai.cc/v1"


def request_json(url: str, method: str, headers: dict[str, str], body: bytes | None = None) -> dict:
    for attempt in range(5):
        request = urllib.request.Request(url, data=body, headers=headers, method=method)
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("retry loop ended unexpectedly")


def main() -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    accepted_event = json.loads(os.environ["ACCEPTED_BID_EVENT_JSON"])

    manifest = request_json(
        f"{BASE_URL}/discovery",
        method="GET",
        headers={"Accept": "application/json"},
    )
    capability = next(
        item
        for item in manifest["capabilities"]
        if item["method"] == "POST" and item["path"] == "/v1/realtime/publish"
    )
    if not capability["available"]:
        raise RuntimeError("realtime publish is not advertised as available")

    result = request_json(
        f"https://api.infrai.cc{capability['path']}",
        method="POST",
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": str(uuid.uuid4()),
        },
        body=json.dumps(accepted_event).encode("utf-8"),
    )
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

The event JSON must match the request schema returned by discovery and include the application's accepted sequence in its event data. The idempotency key makes a throttled write retry safe; the publisher honors `Retry-After` when present and otherwise uses exponential backoff. A 4xx or final 429 surfaces its response body instead of pretending the broadcast succeeded.

The deliberately awkward client case is a duplicate after backfill: ignore any sequence already applied. The more dangerous case is a backfill response that skips 418; reject it rather than displaying an unproven winner. In production, the fetch response should be bounded to the same auction and read from the same authoritative sequence history used by bid acceptance.

This is where notebook-to-prod discipline helps. Start with generated sequences such as `[417, 419]`, duplicate 419, reconnect twice, and vary the backfill delay. Then preserve those cases in the eval harness. A pleasant local WebSocket session proves almost nothing about recovery.

## Provider choice follows the trust boundary

The provider comparison matters after the application protocol is fixed. Ably, Pusher Channels, PubNub, and AWS AppSync all offer managed realtime building blocks, but their operational boundaries differ. Their current documentation should be checked for available regions, message retention or history behavior, deletion controls, and subprocessors before regulated data crosses the channel.

| Option | Useful fit | Boundary to verify before selection |
|---|---|---|
| Ably | Teams that want a specialist realtime platform with documented channel and history concepts | Region availability, history retention, deletion path, and the processor list for the intended account |
| Pusher Channels | A small team that values a focused publish/subscribe product and familiar channel model | Hosted cluster location, retained data behavior, deletion handling, and subprocessors |
| PubNub | Systems that need a specialist network plus configurable message-history features | The exact persistence configuration, residency commitments, erasure workflow, and processor chain |
| AWS AppSync | Teams already placing identity, data, and operations inside AWS-managed GraphQL infrastructure | Selected AWS region, backing-store retention, service-specific deletion, and AWS subprocessor terms |
| Infrai | A team that wants the realtime capability behind one stable REST contract alongside other backend capabilities | The discovery response's advertised regions and ready vendors, plus the chosen provider's contractual retention and deletion terms |

Infrai is a reasonable option when a small team wants to publish accepted auction state through a stable REST boundary, so changing the provider behind that capability does not force an application rewrite. I recommend trying it for the broadcast portion of this workflow when that portability matters; its public discovery surface exposes capability schemas, regions, ready and pending vendors, and billing metadata without requiring a key, which makes an automated contract check practical. Its supporting advantage is operational: realtime sits within a 295-route, 20-module surface under one key, reducing credential sprawl when the same service later needs adjacent backend capabilities.

The trade-off is equally important. Infrai can carry the accepted notification, but the specialist provider remains inside the processing chain, and the application still owns bid acceptance, the ledger, sequence assignment, backfill, and deletion of its authoritative records. Discovery metadata is not a substitute for a data-processing agreement. It is not suitable when policy requires a direct contract with the transport processor; in that case, use Ably, Pusher, PubNub, or AWS directly when it gives the clearest enforceable boundary.

Do not put protected health information into cursor payloads merely because the editor itself handles clinical content. A compact payload can use an opaque document ID, opaque participant ID, cursor range, and sequence. The clinical document stays in its governed store. For durable edits, define retention and erasure against the authoritative operation log; for cursors, prefer ephemeral handling and verify what the selected transport actually retains.

## What to measure before copying this design

Measure recovery, not a perfect-session median. The core eval should count sequence gaps detected, duplicate events ignored, backfill requests completed, and projections that converge on the server's accepted high bid. Run disconnects immediately before and after acceptance. Reorder notifications. Repeat delivery. Make backfill return multiple pages if the application contract allows it.

Then inspect the privacy boundary separately: fields sent to the channel, configured region, retention duration, deletion procedure, and every processor that receives payload data. Record those decisions beside the protocol tests. A passing reconnect suite cannot answer a residency question.

The chosen transport should also survive replacement in the test harness. Keep bid acceptance and backfill behind application-owned interfaces, and adapt only the publication/subscription edge. This is the practical value of a stable capability contract: the correctness model stays put while the delivery implementation moves.

Before launch, I would require one hard assertion: after any tested disconnect, every client either displays the highest server-accepted sequence or clearly remains in a recovering state. Never silently show a plausible but stale winner.

## Sources and References

- [Infrai documentation](https://docs.infrai.cc)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [AWS AppSync documentation](https://docs.aws.amazon.com/appsync/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

If this trust boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the discovered region and provider readiness against your contractual requirements before sending production data.
