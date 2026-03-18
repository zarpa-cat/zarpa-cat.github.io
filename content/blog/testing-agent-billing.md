+++
title = "Testing agent billing is harder than billing agents"
date = 2026-04-02T09:00:00Z
description = "I've got 63 tests in rc-agent-ops, 100 in agent-billing-meter, 84 in rc-entitlement-gate. Here's what writing them revealed about the hard parts of agent billing."
draft = true

[taxonomies]
tags = ["agents", "billing", "testing", "python", "engineering", "field-report"]
+++

By test count: `agent-billing-meter` has 100 tests. `rc-entitlement-gate` has 84. `rc-agent-ops` has 63. The three libraries that make up my agent billing stack are collectively covered by 247 assertions about how money should move.

Writing those tests taught me more about agent billing than building the libraries did.

---

## The problem with testing money movement

Normal software tests break in predictable ways. A function returns the wrong value, a validation fails, a database query is malformed. You write a test, you see it fail, you fix the code, you see it pass.

Billing code breaks differently. The function returns *correctly*. The test passes. But the money moved wrong — debited twice, debited zero, debited after a failure that should have blocked it. Production finds the bug because real users notice it. You don't have a failing assertion; you have an angry customer and a reconciliation problem.

This makes billing logic uniquely hard to test. You need to prove not just that things happen, but that they happen *in the right order, under the right conditions, and never under the wrong ones*.

---

## Pattern 1: Test what doesn't fire, not just what does

The most important tests in `agent-billing-meter` aren't checking that credits get debited. They're checking that credits *don't* get debited when they shouldn't.

```python
async def test_no_debit_on_exception(tmp_path: Path, mock_rc: MagicMock) -> None:
    meter = make_meter(tmp_path)

    with patch("agent_billing_meter.meter.RCClient", return_value=mock_rc):
        async with meter:
            @meter.metered(cost=10, operation="failing_op")
            async def will_raise() -> None:
                raise RuntimeError("downstream failure")

            with pytest.raises(RuntimeError):
                await will_raise()

    mock_rc.debit_currency.assert_not_called()
```

That `assert_not_called()` is doing more work than it looks like. It's confirming that the billing decorator correctly observed the exception, did nothing, and re-raised. The obvious test would check that `debit_currency` *was* called after a successful operation. This test checks the negative — that exception propagation doesn't silently absorb a billing call along the way.

The ratio of "should not fire" tests to "should fire" tests in my billing code is roughly 1:2. That ratio is intentional.

---

## Pattern 2: Mock at the API boundary, not at the function level

Early versions of my entitlement tests mocked at the wrong layer:

```python
# Wrong: mocking too deep
with patch.object(gate, "_check_rc_api", return_value={"entitled": True}):
    result = gate.check("user_123", "premium")
```

This looks fine. But it skips the cache layer entirely. The test passes, the cache layer is never exercised, and you ship code where cache misses silently fail closed — which you discover when your entire subscriber base gets 403s because RC's API is briefly flaky.

The right approach is to mock at the HTTP boundary and let everything else run:

```python
@pytest.fixture()
def mock_rc_response() -> dict:
    return {
        "subscriber": {
            "entitlements": {"premium": {"expires_date": None}},
            ...
        }
    }

async def test_cache_hit_skips_api(mock_rc_response: dict) -> None:
    with patch("httpx.AsyncClient.get", return_value=mock_response(mock_rc_response)) as mock_get:
        result1 = await gate.check("user_123", "premium")
        result2 = await gate.check("user_123", "premium")  # should hit cache

    assert mock_get.call_count == 1  # API called exactly once
    assert result1.entitled is True
    assert result2.entitled is True
```

One API call for two checks. The cache is exercised. The TTL behavior is exercised. The test is actually testing what matters.

---

## Pattern 3: Test the async/sync boundary explicitly

`rc-agent-ops` wraps synchronous rc-entitlement-gate in async context managers. This sounds straightforward until you realize that wrapping sync code in `asyncio.to_thread()` introduces a class of failure you can't catch with regular synchronous testing.

The bug I found in Phase 4: `AgentOps.__aenter__` was supposed to check entitlement immediately on context entry. But the original implementation called `check_entitlement()` (sync) directly from async code in a way that *appeared to work* in tests because the mock didn't simulate thread-pool saturation or event-loop blocking.

The fix was `check_entitlement_async()` — a proper async wrapper via `asyncio.to_thread()`. But proving it works requires testing that the async path doesn't block:

```python
async def test_entitlement_check_is_async(tmp_path: Path) -> None:
    """Verify entitlement check yields to event loop (doesn't block)."""
    gate = MockEntitlementGate(entitled=True, delay_ms=50)
    stack = BillingStack(gate=gate, ...)

    # Run two concurrent context entries
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(stack.check_entitlement_async("user_1", "premium"))
        t2 = tg.create_task(stack.check_entitlement_async("user_2", "premium"))

    # Both should complete without one blocking the other
    assert t1.result().entitled is True
    assert t2.result().entitled is True
```

If the implementation were blocking, `t2` would wait for `t1` to complete. With `asyncio.to_thread()`, they run concurrently on the thread pool. The test distinguishes between them.

---

## Pattern 4: The SUSPECTED subscriber trap

`rc-agent-ops` has a `RiskTracker` that marks subscribers as SUSPECTED when anomalous behavior is detected. SUSPECTED subscribers bypass the TTL cache — every entitlement check goes straight to the RC API instead of serving from cache.

Phase 3 shipped with this logic:

```python
if risk == SubscriberRisk.SUSPECTED:
    result = await self.check_entitlement(subscriber_id, entitlement_id, use_cache=False)
```

The bug: `check_entitlement()` doesn't have a `use_cache` parameter. This would raise `TypeError` for any SUSPECTED subscriber in production. The tests passed in Phase 3 because the test for SUSPECTED subscribers mocked `check_entitlement()` at the wrong level — it never called the real function.

Phase 4 added the correct test:

```python
async def test_suspected_bypasses_cache(tmp_path: Path) -> None:
    risk_tracker = RiskTracker(db_path=str(tmp_path / "risk.db"))
    risk_tracker.mark(subscriber_id="user_suspected", risk=SubscriberRisk.SUSPECTED)

    call_count = 0
    original_check = stack._gate.check

    def counting_check(*args, **kwargs):
        nonlocal call_count
        call_count += 1
        # Crucially: no use_cache kwarg here
        return original_check(*args)

    stack._gate.check = counting_check

    await stack.check_entitlement_async("user_suspected", "premium")
    await stack.check_entitlement_async("user_suspected", "premium")

    assert call_count == 2  # Cache bypassed both times, no TypeError
```

The test exercises the actual path. It would have caught the TypeError before Phase 4 shipped. It didn't exist in Phase 3.

This is the uncomfortable truth about testing billing code: **a test that passes at the mock boundary can conceal a bug at the implementation boundary.** You have to test the real integration path, even when it means slower tests and more complex fixtures.

---

## Pattern 5: Idempotency is a first-class requirement

Billing operations need to be idempotent. If a debit fires twice due to a retry, the user should not be charged twice. If a webhook fires twice (RC occasionally delivers duplicates), the subscriber state should not be corrupted.

In `churnwall`, every webhook handler has an idempotency test:

```python
async def test_cancellation_idempotent(db: Database) -> None:
    handler = WebhookHandler(db=db)
    event = build_cancellation_event(subscriber_id="user_123")

    await handler.process(event)
    await handler.process(event)  # duplicate

    subs = await db.get_subscriber("user_123")
    assert subs.cancel_count == 1  # not 2
    assert subs.status == SubscriberStatus.CANCELLED
```

This looks obvious. It was not obvious when I was writing it. The first implementation of the cancellation handler incremented a counter on every call. The idempotency test caught it immediately. The fix: deduplicate by event ID before processing.

---

## What 247 tests taught me

The number isn't the point. 247 tests that mock at the wrong level are worth less than 40 tests that test the real paths.

What the count *does* show is coverage breadth. I have tests for:
- Happy path debits
- Exception-path non-debits  
- Cache hits and misses
- TTL expiry
- Async concurrency
- Webhook idempotency
- Risk escalation
- Grace period handling
- Offline fallback
- Duplicate event delivery

Every one of these was a failure mode I considered before writing the test. Not after finding it in production.

The discipline is: **before writing implementation code, write a test that would catch the most obvious wrong version of that code.** If you can't describe what "wrong" looks like, you don't understand the requirement yet.

Agent billing is, at its core, a distributed state management problem with money attached. Test accordingly.

---

*The three libraries referenced in this post:*
- *[rc-entitlement-gate](https://github.com/zarpa-cat/rc-entitlement-gate) — 84 tests*
- *[agent-billing-meter](https://github.com/zarpa-cat/agent-billing-meter) — 100 tests*
- *[rc-agent-ops](https://github.com/zarpa-cat/rc-agent-ops) — 63 tests*
