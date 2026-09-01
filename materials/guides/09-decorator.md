# Guide 09 — Decorator

**Lab:** [Requirements](../../phases/phase-09/requirements.md) · [Guided check](../../phases/phase-09/guided-check.md) · [Questions](../../phases/phase-09/questions.md)

## Chapter 9: "Wrapping the HTTP client at ProbeAPI"

**ProbeAPI** is an internal HTTP client. Teams keep subclassing into `LoggingHttpClient`, `RetryHttpClient`, `LoggingRetryHttpClient`… Inheritance explodes.

You need **Decorator**: wrap a component with the same interface, stacking behaviors without a class combinatorial nightmare.

*Head First Design Patterns* Chapter 3. Refactoring Guru: attach new behaviors with wrapper objects ([Decorator](https://refactoring.guru/design-patterns/decorator)).

---

## Learning Objectives

By the end of this guide you will be able to:

- State Decorator’s intent
- Stack cross-cutting behaviors without inheritance explosion
- Reason about wrapper order
- Bridge the idea to Phase 9 without copying ProbeAPI types

---

## Theory

### Intent

**Decorator** attaches additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality. Each decorator implements the same interface as the component it wraps, delegates to an inner object, and adds behavior before or after the call.

*Head First Design Patterns* covers Decorator in Chapter 3 (the classic coffee example). Refactoring Guru: stack wrappers instead of multiplying subclasses ([Decorator](https://refactoring.guru/design-patterns/decorator)).

### The problem in plain language

Cross-cutting concerns—logging, retries, timeouts, metrics, authorization—apply to many operations. Subclassing for every combination (`LoggingRetryHttpClient`, `RetryLoggingHttpClient`, …) leads to **combinatorial explosion**. Copy-paste between subclasses drifts. Removing one concern from a stack of inherited classes is painful.

You need to add behavior **without changing the core component** and **without a new subclass per combination**.

### Analogy — coffee shop add-ons

You order a simple espresso. Add-ons—extra shot, steamed milk, caramel—wrap the drink: each add-on presents the same “beverage” interface to the cashier and passes the order inward, adding cost and preparation steps. You can stack milk + caramel without inventing a `MilkCaramelEspresso` subclass for every permutation.

Decorators are those add-on wrappers for code: same `request()` interface, extra behavior around the inner client.

### Solution structure

| Role | Responsibility | In Phase 9 lab |
| ---- | -------------- | -------------- |
| **Component** | Core interface | `ActuatorPort` |
| **Concrete component** | Base behavior | Raw actuator adapter |
| **Decorator** | Implements component; holds inner reference | `LoggingActuatorDecorator` |
| **Concrete decorators** | Specific cross-cutting policy | max runtime, execution log |

Stack at composition root: `MaxRuntimeDecorator(LoggingDecorator(real_actuator))`. **Order matters**—logging outside retry sees every attempt; max runtime outside everything caps total wall time.

### How collaboration works

```mermaid
flowchart LR
  Client --> Outer[LoggingDecorator]
  Outer --> Middle[RetryDecorator]
  Middle --> Core[HttpClient]
```

Each layer calls `inner.operation(...)`, then returns (possibly modified) results.

### When to use / when to skip

**Use Decorator when:**

- Multiple optional behaviors combine around a stable interface.
- You want to add/remove policies at runtime or per environment.
- Subclassing would explode combinations.

**Skip or simplify when:**

- One behavior applies everywhere—a single well-placed function or middleware is simpler.
- Wrappers must change the method signature—that is Adapter territory.
- Debugging stacked wrappers becomes opaque—document order and keep stacks shallow.

### Related patterns

- **Adapter** — changes interface; Decorator **keeps** interface and adds behavior.
- **Chain of Responsibility** — passes along a chain until someone handles; Decorator **always** delegates through the full stack.
- **Proxy** — controls access (lazy load, remote); structurally similar wrapper, different intent.

---

# Part 2 — Follow along

> **Follow along:** Stdlib Python only. Run the problem sketch, then the complete decorator stack.

## 1. The sticky problem

### 1.1 Smell: subclass combinations

```python
# Smell: each combo of behaviors wants a new subclass
class LoggingRetryHttpClient(HttpClient):
    ...  # copy-paste from Logging + Retry
```

**Root cause:** Behavior combinations grow exponentially with inheritance.

### 1.2 Runnable problem demo

> **Follow along:** Save as `probeapi_before.py`, run `python probeapi_before.py`.

```python
"""Problem demo: inheritance cannot scale combinations cleanly."""


class HttpClient:
    def request(self, method: str, url: str) -> str:
        return f"{method} {url} -> 200 OK"


class LoggingHttpClient(HttpClient):
    def request(self, method: str, url: str) -> str:
        print("log", method, url)
        return super().request(method, url)


class RetryHttpClient(HttpClient):
    def request(self, method: str, url: str) -> str:
        print("retry-capable call")
        return super().request(method, url)


if __name__ == "__main__":
    print(LoggingHttpClient().request("GET", "/status"))
    print(RetryHttpClient().request("GET", "/status"))
    print("PAIN: need Logging+Retry+Auth? another subclass (or worse, copy-paste)")
```

**Expected output (problem):**

```text
log GET /status
GET /status -> 200 OK
retry-capable call
GET /status -> 200 OK
PAIN: need Logging+Retry+Auth? another subclass (or worse, copy-paste)
```

### What goes wrong when requirements change

Add auth headers and metrics: subclasses multiply. Order of logging vs retry becomes inconsistent across teams.

---

## 2. Pattern in practice

The ProbeAPI runnable example below stacks `LoggingDecorator` and `RetryDecorator` around a base HTTP client—same interface, no combinatorial subclasses.

---

## 3. Building the solution (step by step)

### Step A — Shared interface + base decorator

```python
class HttpClientDecorator(HttpClient):
    def __init__(self, inner: HttpClient) -> None:
        self._inner = inner  # wrap, don't subclass the concrete client tree

    def request(self, method: str, url: str) -> str:
        return self._inner.request(method, url)
```

### Step B — One behavior per wrapper

```python
class LoggingClient(HttpClientDecorator):
    def request(self, method: str, url: str) -> str:
        print(f"HTTP {method} {url}")
        return super().request(method, url)
```

### Step C — Compose at wiring time

```python
client: HttpClient = BasicHttpClient()
client = AuthHeaderClient(client, token="secret-token")
client = RetryClient(client, attempts=3)
client = LoggingClient(client)  # outermost sees the logical call
```

---

## 4. Complete worked solution (runnable)

> **Follow along:** Save as `probeapi_decorator.py`, run `python probeapi_decorator.py`.

```python
"""Decorator demo — ProbeAPI HTTP client stack (stdlib only)."""

from abc import ABC, abstractmethod
import time


class HttpClient(ABC):
    @abstractmethod
    def request(self, method: str, url: str) -> str:
        ...


class BasicHttpClient(HttpClient):
    def __init__(self, fail_times: int = 0) -> None:
        self._fail_times = fail_times
        self._attempts = 0

    def request(self, method: str, url: str) -> str:
        self._attempts += 1
        # Simulate transient failure for the first N attempts
        if self._attempts <= self._fail_times:
            raise OSError(f"transient failure #{self._attempts}")
        return f"{method} {url} -> 200 OK"


class HttpClientDecorator(HttpClient):
    def __init__(self, inner: HttpClient) -> None:
        self._inner = inner

    def request(self, method: str, url: str) -> str:
        return self._inner.request(method, url)


class LoggingClient(HttpClientDecorator):
    def request(self, method: str, url: str) -> str:
        print(f"HTTP {method} {url}")
        result = super().request(method, url)
        print(f"HTTP done: {result}")
        return result


class RetryClient(HttpClientDecorator):
    def __init__(self, inner: HttpClient, attempts: int = 3) -> None:
        super().__init__(inner)
        self.attempts = attempts

    def request(self, method: str, url: str) -> str:
        last_error: Exception | None = None
        for i in range(self.attempts):
            try:
                return super().request(method, url)
            except OSError as exc:
                last_error = exc
                print(f"retry after: {exc}")
                time.sleep(0.01)
        raise RuntimeError("retries exhausted") from last_error


class AuthHeaderClient(HttpClientDecorator):
    def __init__(self, inner: HttpClient, token: str) -> None:
        super().__init__(inner)
        self.token = token

    def request(self, method: str, url: str) -> str:
        print(f"auth Bearer {self.token[:4]}...")
        return super().request(method, url)


def build_client() -> HttpClient:
    client: HttpClient = BasicHttpClient(fail_times=1)
    client = AuthHeaderClient(client, token="secret-token")
    client = RetryClient(client, attempts=3)
    return LoggingClient(client)


if __name__ == "__main__":
    print(build_client().request("GET", "https://probe.example/status"))
```

**Expected output (solution):**

```text
HTTP GET https://probe.example/status
auth Bearer secr...
retry after: transient failure #1
auth Bearer secr...
HTTP done: GET https://probe.example/status -> 200 OK
GET https://probe.example/status -> 200 OK
```

Notice auth runs per attempt (inside retry), while logging wraps the whole logical call once at the outside—order is a design choice you can see by running.

---

## 5. Compare & contrast

| | Decorator | Adapter (Guide 05) |
|--|-----------|---------------------|
| Interface | Same as wrapped component | Different (target vs adaptee) |
| Goal | Add responsibilities | Make incompatible APIs work |

---

## 6. Watch out for these traps

- Decorators that change the interface “a little”
- Ignoring call order
- Copying ProbeAPI names into the actuator lab

---

## 7. Try this

Move `LoggingClient` *inside* `RetryClient` in `build_client` (log per attempt). Re-run and compare how many `HTTP GET` lines you see.

---

## Key Terms

| Term | Definition |
|------|------------|
| **Decorator** | Structural pattern that wraps a component with the same interface |
| **Composition** | Building behavior by wrapping rather than subclassing |

---

## Reading Assignments

**Required:**

- *Head First Design Patterns* — Chapter 3
- [Refactoring Guru — Decorator](https://refactoring.guru/design-patterns/decorator)
- Lab: [Requirements](../../phases/phase-09/requirements.md) · [Guided check](../../phases/phase-09/guided-check.md) · [Questions](../../phases/phase-09/questions.md)

---

## Bridge to your lab

Phase 9 stacks cross-cutting policies around an actuator port. Wire the stack in composition roots.

| Teaching (this guide) | Your lab (greenhouse) |
| --------------------- | --------------------- |
| `HttpClient` | `ActuatorPort` (`apply`) |
| `BasicHttpClient` | Simulation (or vendor) actuator adapter |
| `LoggingClient` / `RetryClient` | Logging decorator; max-run-time (or lab) policy decorator |
| Wrapper order | Logging outermost vs policy inner — document it |
| Console / in-memory client | `actuator_execution_log` rows |

**Do / don’t:** **Do** compose wrappers that share one interface; order matters. **Don’t** nest `if policy` in the router, and don’t give each decorator a different method name.

---

## Summary

- Inheritance combinations explode when cross-cutting behaviors multiply.
- Decorator stacks wrappers that share one interface.
- Mind wrapper order—it changes observable behavior.

**Next:** [Guide 10 — Command](./10-command.md)
