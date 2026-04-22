# SOLID in TypeScript

TypeScript's structural typing makes ISP and DIP especially cheap — use it.

## S — Single Responsibility

```ts
// Violation: Invoice owns tax math, formatting, and persistence.
class Invoice {
  constructor(private items: Item[]) {}
  total() { /* ... */ }
  tax()   { /* ... */ }
  toPdf() { /* ... */ }
  save()  { /* db ... */ }
}

// Fix: one reason to change each.
class Invoice       { constructor(readonly items: Item[]) {} }
class InvoiceTotals { of(inv: Invoice): number { /* ... */ return 0; } }
class InvoiceTax    { of(inv: Invoice): number { /* ... */ return 0; } }
class InvoicePdf    { render(inv: Invoice): Buffer { /* ... */ return Buffer.alloc(0); } }
class InvoiceRepo   { save(inv: Invoice): Promise<void> { /* ... */ return Promise.resolve(); } }
```

## O — Open/Closed

```ts
// Violation: every new channel edits Notifier.
class Notifier {
  send(kind: "email" | "sms" | "push", to: string, msg: string) {
    if (kind === "email") { /* ... */ }
    else if (kind === "sms") { /* ... */ }
    else if (kind === "push") { /* ... */ }
  }
}

// Fix: strategy with a small interface.
interface Channel { send(to: string, msg: string): Promise<void>; }
class Notifier {
  constructor(private channels: Record<string, Channel>) {}
  send(kind: string, to: string, msg: string) { return this.channels[kind].send(to, msg); }
}
```

## L — Liskov Substitution

```ts
// Violation: Square narrows Rectangle's contract.
class Rectangle { constructor(public w: number, public h: number) {} setW(w: number) { this.w = w; } }
class Square extends Rectangle { setW(w: number) { this.w = w; this.h = w; } } // breaks callers

// Fix: make them siblings, not parent/child.
interface Shape { area(): number; }
class Rectangle implements Shape { constructor(readonly w: number, readonly h: number) {} area() { return this.w * this.h; } }
class Square    implements Shape { constructor(readonly side: number) {}              area() { return this.side ** 2; } }
```

## I — Interface Segregation

```ts
// Violation: one fat interface forces every consumer to know everything.
interface UserService {
  get(id: string): Promise<User>;
  list(): Promise<User[]>;
  create(u: User): Promise<void>;
  update(u: User): Promise<void>;
  delete(id: string): Promise<void>;
  exportCsv(): Promise<string>;
}

// Fix: role interfaces; consumers depend on the slice they need.
interface ReadUsers   { get(id: string): Promise<User>; list(): Promise<User[]>; }
interface WriteUsers  { create(u: User): Promise<void>; update(u: User): Promise<void>; delete(id: string): Promise<void>; }
interface ExportUsers { exportCsv(): Promise<string>; }

function showProfile(svc: ReadUsers, id: string) { return svc.get(id); }
```

## D — Dependency Inversion

```ts
// Violation: high-level policy constructs its own low-level dependency.
import { PostgresUserRepo } from "./pg-user-repo";
class Signup {
  private repo = new PostgresUserRepo();
  register(u: User) { return this.repo.create(u); }
}

// Fix: depend on an abstraction; inject the concrete at the composition root.
interface UserRepo { create(u: User): Promise<void>; }
class Signup {
  constructor(private repo: UserRepo) {}
  register(u: User) { return this.repo.create(u); }
}
// main.ts
new Signup(new PostgresUserRepo());
```
