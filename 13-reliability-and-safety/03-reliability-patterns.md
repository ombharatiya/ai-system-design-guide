# Reliability Patterns

Production LLM systems need reliability patterns beyond basic retry logic. This chapter covers advanced patterns for building resilient AI applications.

## Table of Contents

- [Reliability Challenges](#reliability-challenges)
- [Retry Patterns](#retry-patterns)
- [Circuit Breaker](#circuit-breaker)
- [Bulkhead Pattern](#bulkhead-pattern)
- [Timeout Strategies](#timeout-strategies)
- [Graceful Degradation](#graceful-degradation)
- [Multi-Provider Failover](#multi-provider-failover)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Reliability Challenges

### LLM-Specific Failure Modes

| Failure Mode | Cause | Impact |
|--------------|-------|--------|
| Rate limiting | Quota exceeded, traffic ramping too fast | Request rejection |
| Spend cap | Monthly budget limit reached | A 429 that no retry clears until the cap resets or is raised |
| Timeouts | Long generation, network issues | Slow/failed responses |
| Provider outage | Infrastructure issues | Complete failure |
| Quality degradation | Model updates (sometimes under the same model ID), SDK default-model changes, load | Worse outputs with no error |
| Model retirement | Scheduled shutdown of a model ID (for example `claude-sonnet-4-5-20250929` on November 30, 2026) | Hard failures on a date announced in advance (Anthropic gives at least 60 days' notice; OpenAI six months for GA models, as little as two weeks for previews) |
| Safety refusal | Provider classifier declines the request | HTTP 200 with empty content that looks like success |
| Context overflow | Input too large | Request failure |
| Malformed output | Generation errors | Parsing failures |

Two of these rows are easy to miss because nothing throws. **Silent quality changes** happen under stable model IDs: GPT-6 Sol and GPT-6 Luna launched on September 22, 2026 with an image-encoding bug that degraded image understanding and computer use; OpenAI fixed it on September 25 without a new snapshot and told customers with image inputs to rerun their evals. Pinning a model ID pins the name, not the serving stack. Defaults move too: openai-agents 0.20.0 (August 11, 2026) made `gpt-5.6-luna` its implicit default model, so an agent that never named a model changed models on a routine dependency bump. Name the model explicitly everywhere. **Safety refusals** on the newest Claude models arrive as successful responses with `stop_reason: "refusal"`, so retry logic keyed on exceptions never sees them (see [Handling Safety Refusals](#handling-safety-refusals)).

### Reliability Targets

| Tier | Availability | Latency p99 | Examples |
|------|--------------|-------------|----------|
| Critical | 99.99% | < 3s | Payment processing |
| Standard | 99.9% | < 10s | Customer support |
| Best effort | 99% | < 30s | Background tasks |

---

## Retry Patterns

### Exponential Backoff with Jitter

```python
import random
import asyncio
from typing import TypeVar, Callable

T = TypeVar("T")

class RetryConfig:
    def __init__(
        self,
        max_retries: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 60.0,
        exponential_base: float = 2.0,
        jitter: float = 0.5
    ):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.exponential_base = exponential_base
        self.jitter = jitter
    
    def get_delay(self, attempt: int) -> float:
        delay = min(
            self.base_delay * (self.exponential_base ** attempt),
            self.max_delay
        )
        # Add jitter to prevent thundering herd
        jitter_range = delay * self.jitter
        delay += random.uniform(-jitter_range, jitter_range)
        return max(0, delay)


async def retry_with_backoff(
    func: Callable[[], T],
    config: RetryConfig,
    retryable_exceptions: tuple = (Exception,)
) -> T:
    last_exception = None
    
    for attempt in range(config.max_retries + 1):
        try:
            return await func()
        except retryable_exceptions as e:
            last_exception = e
            
            if attempt == config.max_retries:
                break
            
            delay = config.get_delay(attempt)
            await asyncio.sleep(delay)
    
    raise last_exception
```

### Retryable vs Non-Retryable Errors

Classify by status **and** error code. The same status now means different things: since September 2, 2026, OpenAI returns 429 with code `slow_down` when traffic ramps too fast and 503 on temporary model overload, both possibly with `Retry-After`. Anthropic returns 429 for acceleration limits, but its monthly spend cap is a 429 with `error_code: enforced_spend_limit_reached` and no `retry-after`, and nothing clears it until the cap resets or an admin raises it. A lower spend limit you set yourself is different again: it returns a 400 `invalid_request_error` whose message says you have reached your specified usage limits, so a classifier like the one below files an organization-wide stop as a bad request. Alert on a sudden jump in 400s, not only 429s and 5xx.

```python
from enum import Enum

class RetryAction(Enum):
    BACKOFF = "backoff"                      # same provider, exponential backoff + jitter
    FAILOVER = "failover"                    # next provider in the chain
    FAILOVER_AND_PAGE = "failover_and_page"  # will not clear on its own; a human must act
    DO_NOT_RETRY = "do_not_retry"            # fix the request

class LLMRetryPolicy:
    @staticmethod
    def classify(status: int, code: str | None) -> RetryAction:
        if status == 429 and code == "enforced_spend_limit_reached":
            return RetryAction.FAILOVER_AND_PAGE   # Anthropic spend cap
        if status == 429:
            return RetryAction.BACKOFF             # includes OpenAI "slow_down": ramp gently
        if status in (500, 502, 503, 504, 529):
            return RetryAction.BACKOFF             # overload or server error; fail over if it persists
        if status == 404:
            return RetryAction.FAILOVER            # retired or wrong model ID; then fix the registry
        return RetryAction.DO_NOT_RETRY            # auth, bad request, context too long

    @staticmethod
    def delay(retry_after_header: str | None, computed_backoff: float) -> float:
        # Never retry sooner than the provider asked
        if retry_after_header:
            return max(float(retry_after_header), computed_backoff)
        return computed_backoff
```

Timeouts and connection errors map to `BACKOFF`. Ignoring `Retry-After` is the most common backoff bug; treating every 429 the same is the second.

### Handling Safety Refusals

A refusal is not a transport error, so none of the retry machinery above sees it. On Claude Fable 5.1 and 5, Opus 5.5 and 5, and Sonnet 5.5, a safety-classifier decline returns HTTP 200 with `stop_reason: "refusal"`, empty content, and a `stop_details.category` (`cyber`, `bio`, `frontier_llm`, `reasoning_extraction`, `general_harms`, or `null`). Retrying the same request on the same model is wasted spend; route it instead:

- **Server-side fallback** (beta, Claude API): `fallbacks: "default"` with the `server-side-fallback-2026-07-01` header retries on the model Anthropic recommends for that category, or you name up to three targets from the model's allowed list (Fable 5.1's permitted targets are Opus 4.8 and Opus 5). It fires only on classifier declines; rate limits and overloads come back as-is. It is not offered on Bedrock, Google Cloud, or Foundry, where the Anthropic SDK's refusal-fallback middleware does the retry client-side, and a Message Batches item that sets `fallbacks` comes back as an errored result, so retry refused batch items yourself.
- **Size the fallback model's rate limits.** If the fallback model is rate limited or overloaded, the API skips the attempt and returns the original refusal, with `stop_details.recommended_model` as a hint for a direct retry. Each attempt counts against its own model's limits, so refusal volume lands on the fallback model's quota.
- **Expect sticky routing.** Once a conversation has fallen back, later requests for it that include `fallbacks` go straight to the fallback model (the routing is kept for about an hour and is best-effort, so the requested model can be tried again at any time). Per-model traffic and quality dashboards shift accordingly, so read the response's `model` field and `usage.iterations` rather than assuming the requested model answered.
- **`reasoning_extraction` has no recommended fallback.** It fires when a prompt asks the model to write out its reasoning (a `<thinking>` section, a `reasoning` field in JSON). Change the prompt; do not retry.
- **Refusals cost money and quota.** Since September 24, 2026, pre-output refusals in `bio`, `frontier_llm`, and `reasoning_extraction` are billed, and every refusal counts against rate limits.

Refusals say nothing about provider health, so they should not trip a circuit breaker. Track them as their own metric by category; a jump usually means a prompt, traffic-mix, or provider safeguard change, not an outage. [Guardrails](01-guardrails.md#provider-safeguards-and-refusals) covers the guardrail side, and the [gateway chapter](../11-infrastructure-and-mlops/03-ai-gateways-and-model-routing.md#fallback-and-reliability) has the full response-classification flow.

---

## Circuit Breaker

### Implementation

```python
from enum import Enum
from dataclasses import dataclass
from datetime import datetime, timedelta

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing recovery

@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5
    recovery_timeout: timedelta = timedelta(seconds=30)
    half_open_max_calls: int = 3
    success_threshold: int = 2  # Successes needed to close

class CircuitBreaker:
    def __init__(self, name: str, config: CircuitBreakerConfig):
        self.name = name
        self.config = config
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time: datetime | None = None
        self.half_open_calls = 0
    
    def can_execute(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        
        if self.state == CircuitState.OPEN:
            # Check if recovery timeout has passed
            if self._recovery_timeout_elapsed():
                self._transition_to_half_open()
                self.half_open_calls += 1
                return True
            return False
        
        if self.state == CircuitState.HALF_OPEN:
            # Allow a limited number of probe calls in half-open state
            if self.half_open_calls < self.config.half_open_max_calls:
                self.half_open_calls += 1
                return True
            return False
        
        return False
    
    def record_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.config.success_threshold:
                self._transition_to_closed()
        else:
            self.failure_count = 0
    
    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        
        if self.state == CircuitState.HALF_OPEN:
            self._transition_to_open()
        elif self.failure_count >= self.config.failure_threshold:
            self._transition_to_open()
    
    def _transition_to_open(self):
        self.state = CircuitState.OPEN
        self.success_count = 0
    
    def _transition_to_half_open(self):
        self.state = CircuitState.HALF_OPEN
        self.half_open_calls = 0
        self.success_count = 0
    
    def _transition_to_closed(self):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
    
    def _recovery_timeout_elapsed(self) -> bool:
        if self.last_failure_time is None:
            return True
        return datetime.now() - self.last_failure_time >= self.config.recovery_timeout
```

### Usage with LLM Client

```python
class ResilientLLMClient:
    def __init__(self):
        self.circuit_breakers = {
            "openai": CircuitBreaker("openai", CircuitBreakerConfig()),
            "anthropic": CircuitBreaker("anthropic", CircuitBreakerConfig()),
        }
    
    async def generate(self, prompt: str, provider: str = "openai") -> str:
        cb = self.circuit_breakers[provider]
        
        if not cb.can_execute():
            raise CircuitOpenError(f"Circuit breaker open for {provider}")
        
        try:
            result = await self._call_provider(provider, prompt)
            cb.record_success()
            return result
        except RetryableError as e:
            cb.record_failure()
            raise
```

**Decide what counts as a failure.** Count 5xx, overloads, timeouts, and 429s that persist through backoff. Do not count 400-class errors or safety refusals: they describe the request, not the provider's health, and counting them lets one bad prompt template open the breaker for everyone. A spend-cap 429 is the exception in the other direction: force the breaker open until the cap resets or is raised rather than probing it every 30 seconds.

---

## Bulkhead Pattern

### Isolating Resources

```python
import asyncio
from contextlib import asynccontextmanager

class Bulkhead:
    """
    Isolate resources to prevent cascade failures.
    """
    
    def __init__(
        self,
        name: str,
        max_concurrent: int,
        max_queued: int = 100
    ):
        self.name = name
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.queue_semaphore = asyncio.Semaphore(max_queued)
    
    @asynccontextmanager
    async def acquire(self, timeout: float = 30.0):
        # Check queue capacity
        if not self.queue_semaphore.locked():
            await self.queue_semaphore.acquire()
        else:
            raise BulkheadFullError(f"Bulkhead {self.name} queue full")
        
        try:
            # Wait for execution slot
            acquired = await asyncio.wait_for(
                self.semaphore.acquire(),
                timeout=timeout
            )
            self.queue_semaphore.release()
            
            try:
                yield
            finally:
                self.semaphore.release()
        except asyncio.TimeoutError:
            self.queue_semaphore.release()
            raise BulkheadTimeoutError(f"Bulkhead {self.name} timeout")


class BulkheadedLLMClient:
    def __init__(self):
        # Separate bulkheads for different workloads
        self.bulkheads = {
            "realtime": Bulkhead("realtime", max_concurrent=50),
            "batch": Bulkhead("batch", max_concurrent=200),
            "critical": Bulkhead("critical", max_concurrent=10)
        }
    
    async def generate(
        self,
        prompt: str,
        priority: str = "realtime"
    ) -> str:
        bulkhead = self.bulkheads[priority]
        
        async with bulkhead.acquire():
            return await self._call_llm(prompt)
```

---

## Timeout Strategies

### Layered Timeouts

```python
class TimeoutConfig:
    def __init__(
        self,
        connection_timeout: float = 5.0,
        first_token_timeout: float = 15.0,  # set per model and effort level
        read_timeout: float = 30.0,
        total_timeout: float = 60.0
    ):
        self.connection_timeout = connection_timeout
        self.first_token_timeout = first_token_timeout
        self.read_timeout = read_timeout
        self.total_timeout = total_timeout


class TimeoutManager:
    def __init__(self, config: TimeoutConfig):
        self.config = config
    
    async def execute_with_timeout(self, func, *args, **kwargs):
        try:
            return await asyncio.wait_for(
                func(*args, **kwargs),
                timeout=self.config.total_timeout
            )
        except asyncio.TimeoutError:
            raise LLMTimeoutError(
                f"Request timed out after {self.config.total_timeout}s"
            )
```

**Time to first token is its own timeout.** For streaming, the first-token deadline is the one that matters for failover: before the first token you can move to another backend cleanly, after it you must replay from scratch or surface the error. Gateways now expose this directly (Envoy AI Gateway 1.1 added `streamIdleTimeout`, which can fail over to the next backend when a stream stalls before its first token and returns a 504 when it stalls mid-stream). Set it per model and effort level: a high-effort reasoning request can legitimately stream nothing visible for a long stretch, and a TTFT deadline tuned for a fast model will fail those over for no reason.

### Adaptive Timeouts

```python
class AdaptiveTimeout:
    """
    Adjust timeouts based on observed latency.
    """
    
    def __init__(
        self,
        initial_timeout: float = 30.0,
        min_timeout: float = 10.0,
        max_timeout: float = 120.0,
        percentile: float = 0.99
    ):
        self.min_timeout = min_timeout
        self.max_timeout = max_timeout
        self.percentile = percentile
        self.latencies: list[float] = []
        self.current_timeout = initial_timeout
    
    def record_latency(self, latency: float):
        self.latencies.append(latency)
        
        # Keep last 1000 observations
        if len(self.latencies) > 1000:
            self.latencies = self.latencies[-1000:]
        
        # Update timeout to percentile + buffer
        if len(self.latencies) >= 10:
            sorted_latencies = sorted(self.latencies)
            idx = int(len(sorted_latencies) * self.percentile)
            p99_latency = sorted_latencies[idx]
            
            # Add 20% buffer
            new_timeout = p99_latency * 1.2
            self.current_timeout = max(
                self.min_timeout,
                min(self.max_timeout, new_timeout)
            )
    
    def get_timeout(self) -> float:
        return self.current_timeout
```

---

## Graceful Degradation

### Degradation Levels

```python
class DegradationLevel(Enum):
    FULL = "full"           # All features
    REDUCED = "reduced"     # Fewer features
    MINIMAL = "minimal"     # Core only
    CACHED = "cached"       # Cached responses only
    OFFLINE = "offline"     # Error message

class GracefulDegrader:
    def __init__(self):
        self.current_level = DegradationLevel.FULL
        self.health_checker = HealthChecker()
    
    async def get_response(self, query: str) -> str:
        level = await self.health_checker.get_degradation_level()
        
        if level == DegradationLevel.FULL:
            return await self.full_pipeline(query)
        
        elif level == DegradationLevel.REDUCED:
            # Skip expensive operations
            return await self.reduced_pipeline(query)
        
        elif level == DegradationLevel.MINIMAL:
            # Simpler model, no retrieval
            return await self.minimal_pipeline(query)
        
        elif level == DegradationLevel.CACHED:
            # Only return cached responses
            cached = await self.cache.get_similar(query)
            if cached:
                return cached
            return "I'm experiencing issues. Please try again later."
        
        else:
            return "Service temporarily unavailable."
    
    async def full_pipeline(self, query: str) -> str:
        # RAG + primary model + ensemble verification
        context = await self.retrieve(query)
        response = await self.generate(query, context, model="gpt-6.1-sol")
        verified = await self.verify(response)
        return verified
    
    async def reduced_pipeline(self, query: str) -> str:
        # RAG + smaller model, no verification
        context = await self.retrieve(query)
        return await self.generate(query, context, model="gpt-6-luna")
    
    async def minimal_pipeline(self, query: str) -> str:
        # Direct generation with smallest model, no retrieval
        return await self.generate(query, None, model="gpt-6-luna")
```

---

## Multi-Provider Failover

**Fail over across vendors, not across sibling models.** Outages take out a vendor's whole surface at once. Anthropic's status feed records at least 12 major or critical incidents between August 16 and September 29, 2026, several spanning the API, Claude Code, and claude.ai together. OpenAI's September 29 incident degraded the API (including the Agents API), ChatGPT, and Codex for about 5 hours 20 minutes. Clouds fail too: a Google Cloud us-west1 incident on August 20 had 2 hours 22 minutes of customer impact. A chain that steps from one model to another on the same provider survives none of these.

**Failover is not drop-in.** The fallback leg has to accept the request, carry the conversation, meet the same data terms, and do the job:

| What breaks | Example (November 2026) | Fix |
|-------------|------------------------|-----|
| Request shape | Claude Fable 5.1, Opus 5.5, and Sonnet 5.5 return 400 on forced `tool_choice`; Claude API version `2026-10-21` rejects `temperature`, `top_p`, and `top_k` at any value on every model; GPT-6 Astra rejects custom `temperature`, `top_p`, and `logprobs` | Translate per target from a capability profile; never pass parameters through blindly |
| Conversation state | The newest Claude models bind thinking blocks to the model and conversation | Fail over at turn boundaries; send the visible transcript and tool results |
| Data terms | Fable 5.1 requires 30-day retention unless authorized for ZDR; OpenAI's Agents API beta has no ZDR | Store retention and residency per model in the registry and enforce them at routing time |
| Output quality | The fallback model was never evaluated on your task | Run the eval suite on every fallback leg, on a schedule |

The [gateway chapter](../11-infrastructure-and-mlops/03-ai-gateways-and-model-routing.md#fallback-and-reliability) covers the routing mechanics; the code below shows the core loop.

### Provider Manager

```python
class ProviderManager:
    def __init__(self):
        self.providers = {
            "primary": OpenAIProvider(),
            "secondary": AnthropicProvider(),
            "tertiary": GoogleProvider()
        }
        self.health = {name: True for name in self.providers}
        self.priority_order = ["primary", "secondary", "tertiary"]
    
    async def generate(self, request: dict) -> str:
        for provider_name in self.priority_order:
            if not self.health[provider_name]:
                continue
            
            provider = self.providers[provider_name]
            
            try:
                result = await provider.generate(request)
                return result
            except RetryableError as e:
                # Mark unhealthy but continue to next provider
                self.health[provider_name] = False
                asyncio.create_task(
                    self._health_check_later(provider_name)
                )
                continue
        
        raise AllProvidersUnavailableError()
    
    async def _health_check_later(self, provider_name: str):
        await asyncio.sleep(30)  # Wait before retrying
        try:
            await self.providers[provider_name].health_check()
            self.health[provider_name] = True
        except:
            # Schedule another check
            asyncio.create_task(self._health_check_later(provider_name))
```

### Request Hedging

```python
class HedgedRequest:
    """
    Start the primary; if it is slow or fails, race the other providers
    against it and keep the first successful response.
    """
    
    def __init__(self, providers: list, hedge_delay: float = 2.0):
        self.providers = providers
        self.hedge_delay = hedge_delay
    
    async def generate(self, request: dict) -> str:
        primary = asyncio.create_task(self.providers[0].generate(request))
        
        # asyncio.wait, unlike wait_for, does not cancel the primary on timeout
        done, _ = await asyncio.wait({primary}, timeout=self.hedge_delay)
        if primary in done and primary.exception() is None:
            return primary.result()
        
        # Primary is slow or failed: hedge to the others, keeping a slow primary in the race
        pending = set() if primary in done else {primary}
        pending |= {
            asyncio.create_task(provider.generate(request))
            for provider in self.providers[1:]
        }
        
        try:
            while pending:
                done, pending = await asyncio.wait(
                    pending, return_when=asyncio.FIRST_COMPLETED
                )
                for task in done:
                    if task.exception() is None:
                        return task.result()  # first success wins, not first finisher
            raise AllProvidersFailedError()
        finally:
            for task in pending:
                task.cancel()  # stop paying for the losers
```

Hedge only idempotent requests (no tool calls with side effects), set `hedge_delay` near the primary's p95 latency for that model and effort level (time to first token if you hedge streams), and budget for it: a hedged request also pays for the losing legs, at least their input tokens and often more, since canceling on the client does not guarantee the provider stops generating.

---

## Interview Questions

### Q: How do you design for high availability in LLM systems?

**Strong answer:**

"I use multiple layers of reliability:

**Retry with backoff:** Exponential backoff with jitter for transient failures, honoring `Retry-After`. Important to distinguish retryable (most rate limits, overloads, timeouts) from non-retryable (auth, bad request) errors, and to branch on the error code rather than the status alone: a spend-cap 429 looks like a rate limit but needs failover and a human, not backoff.

**Circuit breaker:** If a provider fails repeatedly, stop trying for a cooldown period. This prevents wasting latency on a dead provider and gives it time to recover.

**Multi-provider failover:** Never depend on a single provider. I configure primary/secondary/tertiary across different vendors, because a vendor outage usually takes down every model it serves. Each provider has its own circuit breaker, and every fallback leg is evaluated on our task, since a fallback nobody tested is a second outage waiting to happen.

**Refusal handling:** Safety refusals from the newest Claude models come back as HTTP 200 with `stop_reason: "refusal"`, so I route them to an approved fallback model instead of counting them as errors or as successes.

**Graceful degradation:** Define what happens when no providers are available. Better to return a degraded response (simpler model, cached result) than to fail completely.

**Bulkheading:** Isolate different workloads. A batch processing surge should not take down real-time queries.

The key insight is assuming failure. LLM APIs are less reliable than traditional APIs. Design as if the provider will go down, because it will."

### Q: What is the difference between circuit breaker and retry?

**Strong answer:**

"They solve different problems:

**Retry** handles transient failures. If a single request fails, try again. It assumes failures are independent and the next attempt may succeed.

**Circuit breaker** handles systemic failures. If many requests are failing, stop trying entirely. It assumes the downstream system is unhealthy and repeated attempts waste resources and slow recovery.

**How they work together:**
1. Request fails → retry with backoff (attempt 1, 2, 3)
2. If all retries fail → circuit breaker records failure
3. After N failures → circuit opens, rejects requests immediately
4. After timeout → circuit half-opens, allows limited test requests
5. If tests succeed → circuit closes, normal operation resumes

Without circuit breaker: during an outage, every request waits through all retries before failing. Latency spikes, resources exhausted.

With circuit breaker: after detecting the outage, requests fail fast. System remains responsive, can fail over to alternatives."

### Q: Your provider shipped a fix under the same model ID and your quality metrics moved. How do you detect and handle silent model changes?

**Strong answer:**

"I assume the model behind an ID can change, because it does. GPT-6 Sol and Luna launched on September 22, 2026 with an image-encoding bug; OpenAI fixed it on September 25 under the same ID and told customers with image inputs to rerun their evals. Anyone who finalized an image or computer-use migration on launch-week numbers may have measured the broken path.

**Detection:**
- A small canary eval suite running against the production endpoint on a schedule (hourly for critical paths), not only in CI.
- Distribution monitors on cheap signals: output length, refusal rate by category, tool-call rate, schema-validation failures, judge scores on a traffic sample.
- Provider changelogs and status pages wired into the same alert channel as our own deploys.

**Handling:**
- Pin dated snapshots where the provider offers them, knowing that pins the name, not the serving stack.
- Gate migrations on a second eval run a week or two after launch, and keep the previous model warm until the new one is stable.
- When a provider announces a fix, rerun the affected eval slice and compare against the last known-good baseline before trusting either number.

The principle: model behavior is a dependency that changes without a version bump, so it needs the same continuous verification as any external service."

---

## References

- Microsoft Resilience Patterns: https://learn.microsoft.com/en-us/azure/architecture/patterns/
- Netflix Hystrix: https://github.com/Netflix/Hystrix
- OpenAI API changelog (September 2, 2026 `slow_down` 429 and 503 overload; September 25, 2026 GPT-6 Sol/Luna image-encoding fix): https://developers.openai.com/api/docs/changelog
- Anthropic, refusals and fallback: https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback
- Anthropic, rate limits: https://platform.claude.com/docs/en/api/rate-limits
- Anthropic status history: https://status.claude.com/history

---

*Previous: [Ensemble Methods](02-ensemble-methods.md) · Next: [AI Governance and Compliance](04-ai-governance-and-compliance.md)*
