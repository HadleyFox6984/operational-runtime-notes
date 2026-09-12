# Per-Tenant Agent Budgets: Enforce Spend Caps at the Credential Boundary

Use one scoped credential per paying customer and hang the spend ceiling on that credential, because a counter living inside your autonomous agent loop can't bound what one runaway run costs every other tenant sharing the key. That is the short version for anyone billing a metered product. The rest of this is why the in-loop guard keeps losing, and how a small reservation ledger — the same design in Python or Node.js — turns an agent budget into an invoice line you can defend in front of a customer.

The system I keep coming back to is a property-management back office. An agent pulls utility statements and meter photos for each managed building, reconciles the readings against unit occupancy, decides when a reading looks wrong enough to re-check, and writes one metered usage line per landlord. Every iteration costs tokens, often an OCR call, sometimes a second opinion from a larger model when two readings disagree. Those costs belong to a named customer, and thirty days later they appear on that customer's invoice.

So the meter is the product. Getting it wrong is a billing dispute, not a graph that looks odd.

## Where should an autonomous agent enforce a spend limit — inside the loop or at the credential?

There are three honest places to put the limit, and they are not interchangeable.

The first is a counter in the loop: `spent += price(response)` after each step, break when it crosses the ceiling. It's free, it's one line, and it holds exactly as long as your process does. Crash mid-run and the counter resets to zero. Start a second worker for the same customer — say a retry of a stuck nightly job — and you now have two counters, each convinced it's the only one spending.

The second is a shared ledger behind a service boundary that every worker calls before it calls a model. State survives restarts, parallel workers see the same balance, and the numbers are queryable at month end. This is the design most teams land on, and it's the one worth building. It has a hole, though: if all of those workers authenticate to the provider with the same fleet-wide key, the ledger is advice. Anything that reaches the key directly — a debugging script, a new code path that forgot the wrapper, a tool the agent talks into calling in a strange order — spends without asking.

The third closes that hole by moving the ceiling onto the credential itself: a short-lived, per-tenant credential minted for a run, carrying its own hard limit, useless for anything else. Now the worst case for one poisoned document is one customer's daily allowance, and the ledger and the credential agree on the number instead of racing each other.

| Enforcement point | Survives a crash | Correct with 8 parallel workers | Blast radius of one bad run |
| --- | --- | --- | --- |
| Counter inside the agent loop | No | No | Every tenant that worker touches |
| Shared reservation ledger | Yes | Yes, if reservations are atomic | Whatever the shared key can reach |
| Per-tenant scoped credential | Yes | Yes | One tenant, one window |

My rule for picking: if the same credential is reachable by code running on behalf of more than one paying customer, an in-loop counter cannot answer the only question that matters at month end, which is whose money got spent. Move the ceiling outward until the answer is unambiguous.

## One key, eleven buildings, and a blast radius nobody sized

The failure that actually bites is boring. A shared key, a loop with no external ceiling, and an input the agent trusted more than it should have.

Scanned utility statements are user-supplied content. A PDF that contains instructions — re-read every page, compare against every prior month, produce a full audit — is a prompt-injection vector that pays out in tokens rather than data. OWASP catalogs this as unbounded consumption in its LLM application risk list, and the reason it earns its own entry is that the loop does not look broken while it happens. It looks busy.

Three more mechanics decide whether your meter survives contact with production:

Concurrency. Two workers read `spent_cents`, both see room under the ceiling, both proceed, both write. Classic lost update. The fix isn't a mutex in your Python process — it's making the check and the debit a single atomic statement in whatever store already holds your invoice lines.

Retries. An HTTP client that retries a timed-out request will happily bill the same unit twice if your meter counts on the way out. Every reservation needs an idempotency key derived from run plus unit, so the second attempt reuses the first hold instead of opening a new one.

Ordering. Reserve before the call, settle after it. If the process dies in between, you've over-held, and over-holding is recoverable — a sweeper releases stale holds a few minutes later. Count after the call and a crash silently under-bills, which nobody notices until the numbers are already on an invoice.

## A reservation ledger, in about sixty lines of Python

The flow is plain: estimate what a step could cost, ask the ledger to hold that amount against the customer, call the model with that customer's credential, then settle the hold with what it actually cost. Two writes per model call, both cheap, both in the same database as the billing lines.

```python
import os
import uuid
from dataclasses import dataclass

IN_RATE, OUT_RATE = 3, 15      # integer cents per 1k tokens; never use floats for money
MAX_OUTPUT_TOKENS = 700
OCR_CENTS = 2


class BudgetExhausted(Exception):
    """Raised at the loop boundary, before any provider call is made."""


@dataclass(frozen=True)
class Hold:
    hold_id: str
    tenant: str
    run_id: str
    cents: int


def estimate_cents(unit: dict) -> int:
    # Pessimistic on purpose: assume the model writes its full output allowance.
    prompt_tokens = 900 + 40 * len(unit["meters"])
    tokens = (prompt_tokens * IN_RATE + MAX_OUTPUT_TOKENS * OUT_RATE) / 1000
    return round(tokens) + OCR_CENTS


class BudgetLedger:
    """Per-tenant ledger. reserve() is the gate, settle() records what happened."""

    def __init__(self, conn):
        self.conn = conn

    def reserve(self, tenant: str, run_id: str, cents: int, idem_key: str) -> Hold | None:
        with self.conn.transaction():
            existing = self.conn.execute(
                "SELECT hold_id, cents FROM holds WHERE idem_key = %s", (idem_key,)
            ).fetchone()
            if existing:                       # a retry of the same step, not new spend
                return Hold(existing[0], tenant, run_id, existing[1])

            # Check and debit in one statement: a second worker cannot read a stale balance.
            row = self.conn.execute(
                "UPDATE tenant_budget SET held_cents = held_cents + %s "
                " WHERE tenant = %s "
                "   AND spent_cents + held_cents + %s <= ceiling_cents "
                "RETURNING ceiling_cents - spent_cents - held_cents",
                (cents, tenant, cents),
            ).fetchone()
            if row is None:
                return None                    # ceiling reached; the caller stops the loop

            hold = Hold(str(uuid.uuid4()), tenant, run_id, cents)
            self.conn.execute(
                "INSERT INTO holds (hold_id, tenant, run_id, cents, idem_key) "
                "VALUES (%s, %s, %s, %s, %s)",
                (hold.hold_id, tenant, run_id, cents, idem_key),
            )
            return hold

    def settle(self, hold: Hold, actual_cents: int) -> None:
        with self.conn.transaction():
            self.conn.execute(
                "UPDATE tenant_budget "
                "   SET held_cents = held_cents - %s, spent_cents = spent_cents + %s "
                " WHERE tenant = %s",
                (hold.cents, actual_cents, hold.tenant),
            )
            self.conn.execute("DELETE FROM holds WHERE hold_id = %s", (hold.hold_id,))


def meter_units(ledger: BudgetLedger, provider, tenant: str, units: list[dict]) -> list[dict]:
    run_id = str(uuid.uuid4())
    credential = mint_run_credential(tenant, run_id)   # scoped, short-lived, per tenant
    lines = []
    for unit in units:
        hold = ledger.reserve(tenant, run_id, estimate_cents(unit), f"{run_id}:{unit['id']}")
        if hold is None:
            raise BudgetExhausted(tenant)              # a partial invoice beats a surprise one
        reading = provider.read_meters(unit, credential=credential)
        ledger.settle(hold, reading.usage_cents)
        lines.append({"unit_id": unit["id"], "kwh": reading.kwh, "cents": reading.usage_cents})
    return lines
```

`mint_run_credential` is where the credential boundary lives. It asks your secret store for a token scoped to this tenant and this run, with a short expiry, and it never hands back the long-lived key that issued it. OWASP's secrets management guidance is blunt about the two properties that matter here — short lifetime and narrow scope — and both do more for your blast radius than another layer of application checks.

Operationally, a handful of habits keep this boring. Run the sweeper that releases holds older than the longest plausible step, or a crashed worker quietly shrinks a customer's ceiling until someone investigates. Alert on ceiling hits per tenant rather than on aggregate spend, since aggregate hides the one account that went sideways. Keep the ledger in the same transaction domain as the invoice lines so the number you bill and the number you enforced can't drift. And give support staff a way to raise one tenant's ceiling without a deploy, because they will need it during a month-end close and you don't want that request arriving as a hotfix.

## What the eval harness and the traces have to show

A spend cap is a behavior, so test it like one. I keep a fake provider that returns fixed token counts and replay a week of real statements through the loop, asserting two things: the ledger never lets total holds exceed the ceiling, and the same statement replayed twice produces one billable line. Eight threads hammering `reserve` against a 4,000-cent ceiling with 800-cent steps should yield exactly five holds and three refusals. That test catches the lost-update bug that code review usually doesn't.

For traces, the OpenTelemetry semantic conventions for generative AI already define the attribute names for token usage and model identity, which means your spans can carry cost data without inventing a private schema — worth adopting even if you only ever look at the data in one dashboard.

Refusals deserve a real error shape too. RFC 9457 problem details give you a typed body for "budget exhausted for this tenant", so an upstream scheduler can tell that apart from a transport error and stop retrying something that will never succeed. Pair it with 429 handling on the provider side and your loop stops guessing what a failure means.

## Where this stops being worth the round trips

The catch is latency and moving parts. Two ledger writes per model call, plus credential minting per run, on a loop that might do forty steps — that's real overhead for an interactive experience, and it's a second stateful dependency in the path of a job that used to only need the provider.

For a single-tenant internal tool, stick with the in-process counter and a hard step limit. If one crashed run costs you lunch money and nobody gets invoiced, the ledger is over-engineering.

Reservations also fit badly where cost is unknowable before the call. Long agent turns with tool fan-out, or streaming output with no ceiling on length, force you to reserve pessimistically, and a pessimistic hold refuses work that would have fit. I'm honestly not sure there's a clean answer there; the options I've seen are reserving in small increments mid-stream or accepting slack in the estimate, and neither is free.

One more boundary: this design caps spend, it doesn't make a halted run useful. A loop that stops at unit fourteen of twenty leaves a partial invoice, so the run state has to be resumable and the partial result has to be visible to whoever signs off on billing. Enforce the ceiling in the ledger, put the credential where the loop can't outrun it, and let the invoice be the thing you can prove.

## Further reading

- OWASP Secrets Management Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- OWASP Top 10 for LLM Applications — https://genai.owasp.org/llm-top-10/
- OpenTelemetry semantic conventions for generative AI — https://opentelemetry.io/docs/specs/semconv/gen-ai/
- RFC 9457, Problem Details for HTTP APIs — https://www.rfc-editor.org/rfc/rfc9457.html
- RFC 6585, Additional HTTP Status Codes (429 Too Many Requests) — https://www.rfc-editor.org/rfc/rfc6585
- PostgreSQL documentation, Explicit Locking — https://www.postgresql.org/docs/current/explicit-locking.html
