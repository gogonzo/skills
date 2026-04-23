# SOLID in Python

Python leans on duck typing — prefer `Protocol` (PEP 544) over ABCs for ISP and DIP.

## S — Single Responsibility

```python
# Violation: Report both computes and renders and writes to disk.
class Report:
    def __init__(self, rows): self.rows = rows
    def totals(self): ...
    def render_html(self): ...
    def save(self, path): ...

# Fix: split by reason to change.
@dataclass
class Report: rows: list

class ReportTotals:
    def of(self, r: Report) -> dict: ...

class ReportHtml:
    def render(self, r: Report) -> str: ...

class ReportWriter:
    def save(self, html: str, path: str) -> None: ...
```

## O — Open/Closed

```python
# Violation: new exporters edit this function.
def export(fmt, data):
    if fmt == "csv":  return to_csv(data)
    if fmt == "json": return to_json(data)

# Fix: registry.
EXPORTERS = {"csv": to_csv, "json": to_json}
def export(fmt, data): return EXPORTERS[fmt](data)

# Registering a new format does not touch export().
EXPORTERS["parquet"] = to_parquet
```

## L — Liskov Substitution

```python
# Violation: Penguin("a Bird") breaks Bird.fly callers.
class Bird:
    def fly(self): ...
class Penguin(Bird):
    def fly(self): raise NotImplementedError

# Fix: hoist the capability into its own type.
class Bird: ...
class FlyingBird(Bird):
    def fly(self): ...
class Penguin(Bird): ...
```

## I — Interface Segregation

```python
from typing import Protocol

# Violation: every consumer depends on a 10-method Storage.

# Fix: small protocols; consumers ask for exactly what they use.
class Reader(Protocol):
    def read(self, key: str) -> bytes: ...

class Writer(Protocol):
    def write(self, key: str, blob: bytes) -> None: ...

def load_config(src: Reader) -> dict: ...
def persist_result(dst: Writer, blob: bytes) -> None: ...
```

## D — Dependency Inversion

```python
# Violation: domain code imports requests and reads the wall clock.
import requests, time
def renew_token(user_id):
    if time.time() > expiry(user_id):
        requests.post("https://auth/...", json={"id": user_id})

# Fix: inject abstractions. Tests pass fakes without monkey-patching.
from typing import Protocol, Callable

class AuthClient(Protocol):
    def renew(self, user_id: str) -> None: ...

def renew_token(user_id: str, *, auth: AuthClient, now: Callable[[], float]) -> None:
    if now() > expiry(user_id):
        auth.renew(user_id)

# main.py — composition root wires concretes.
renew_token(uid, auth=HttpAuthClient(), now=time.time)
```

## Encapsulation — minimise the public surface

- Prefix every name that isn't public API with a single leading underscore
  (`_helper`, `_InternalCache`). This is enforced socially but respected by
  tooling (`from module import *` skips them; IDEs flag cross-module use).
- Declare the public API explicitly with `__all__` in every public module:
  ```python
  __all__ = ["Signup", "UserRepo"]    # everything else is private
  ```
- Double-underscore `__name` invokes name mangling — reserve it for class
  internals that must not collide with subclasses, not general "privacy".
- Put internal modules under an `_internal/` subpackage. Cross-package
  imports from `_internal` are a review-time red flag.
- `@dataclass(frozen=True)` for immutable value objects — immutability is
  encapsulation for data.
- Never move a name from `_private` to public solely to test it. Test through
  the public function, or extract a new module whose public API is that
  function.
