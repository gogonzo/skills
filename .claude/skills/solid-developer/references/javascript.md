# SOLID in JavaScript

Concrete violation → fix pairs. Keep fixes minimal; do not introduce classes where a function suffices.

## S — Single Responsibility

```js
// Violation: parses, validates, persists, and emails.
async function registerUser(body) {
  const data = JSON.parse(body);
  if (!data.email.includes("@")) throw new Error("bad email");
  await db.query("INSERT INTO users ...", data);
  await sendgrid.send({ to: data.email, subject: "Welcome" });
}

// Fix: each collaborator has one reason to change.
const parseUser    = (body) => JSON.parse(body);
const validateUser = (u) => { if (!u.email.includes("@")) throw new Error("bad email"); return u; };
const saveUser     = (u) => db.query("INSERT INTO users ...", u);
const welcome      = (mailer) => (u) => mailer.send({ to: u.email, subject: "Welcome" });

async function registerUser(body, mailer) {
  const user = validateUser(parseUser(body));
  await saveUser(user);
  await welcome(mailer)(user);
}
```

## O — Open/Closed

```js
// Violation: every new shape edits this function.
function area(shape) {
  if (shape.kind === "circle") return Math.PI * shape.r ** 2;
  if (shape.kind === "square") return shape.side ** 2;
}

// Fix: dispatch table; new shapes register themselves.
const areas = {
  circle: (s) => Math.PI * s.r ** 2,
  square: (s) => s.side ** 2,
};
const area = (s) => areas[s.kind](s);
```

## L — Liskov Substitution

```js
// Violation: ReadOnlyList claims to be a List but breaks `push`.
class List        { push(x) { this.items.push(x); } }
class ReadOnlyList extends List { push()  { throw new Error("nope"); } }

// Fix: split the contract.
class ReadableList { get(i) { /* ... */ } }
class WritableList extends ReadableList { push(x) { /* ... */ } }
```

## I — Interface Segregation

```js
// Violation: callers depend on a fat "repo" shape.
// repo = { find, findAll, create, update, delete, export, import, backup }

// Fix: pass only the role the caller needs.
function listUsers({ findAll }) { return findAll(); }
function createUser({ create }, u) { return create(u); }
```

## D — Dependency Inversion

```js
// Violation: business logic imports a concrete client and reads the clock directly.
import { stripe } from "./stripe-client.js";
async function chargeLateFee(userId) {
  if (Date.now() > dueDate(userId)) await stripe.charge(userId, 500);
}

// Fix: inject abstractions; wire concretes at the composition root.
export const chargeLateFee = ({ payments, now }) => async (userId) => {
  if (now() > dueDate(userId)) await payments.charge(userId, 500);
};

// main.js — only here do we know about Stripe and the real clock.
chargeLateFee({ payments: stripe, now: () => Date.now() });
```

## Encapsulation — minimise the public surface

- Only `export` what external callers need. Module-scoped helpers stay
  un-exported — they are private by being unreachable.
- Use `#private` class fields (not the `_name` convention) for true privacy;
  `#` is enforced by the runtime, `_` is a wink.
- Prefer closures for state that must never leak:
  ```js
  export function makeCounter() {
    let n = 0;                          // truly private
    return { inc: () => ++n, read: () => n };
  }
  ```
- An `index.js` barrel should re-export only the public API — never `export *`
  from a module that contains internal helpers.
- Do not export a symbol "just for tests". Test through the public API, or
  move the logic to its own small module whose public API *is* that function.
