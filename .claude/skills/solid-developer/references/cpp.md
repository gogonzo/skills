# SOLID in C++

Modern C++ has three toolboxes for abstraction: virtual interfaces, templates/concepts, and `std::function`. Pick the one with the least coupling cost.

## S — Single Responsibility

```cpp
// Violation: OrderService does math, I/O, and formatting.
class OrderService {
public:
    double total(const Order&) const;
    void   save(const Order&);
    std::string toJson(const Order&) const;
};

// Fix: split by reason to change.
class OrderTotals   { public: double total(const Order&) const; };
class OrderRepo     { public: void   save(const Order&);       };
class OrderJson     { public: std::string render(const Order&) const; };
```

## O — Open/Closed

```cpp
// Violation: every new shape edits this function.
double area(const Shape& s) {
    switch (s.kind) {
        case Shape::Circle: return M_PI * s.r * s.r;
        case Shape::Square: return s.side * s.side;
    }
}

// Fix: virtual interface — adding a shape adds a class, not a case.
struct Shape { virtual ~Shape() = default; virtual double area() const = 0; };
struct Circle : Shape { double r; double area() const override { return M_PI * r * r; } };
struct Square : Shape { double side; double area() const override { return side * side; } };

// Or, if you want zero runtime overhead and know the set at compile time:
template <class S> concept HasArea = requires(const S& s) { { s.area() } -> std::convertible_to<double>; };
double area(HasArea auto const& s) { return s.area(); }
```

## L — Liskov Substitution

```cpp
// Violation: Square inherits Rectangle and silently couples width/height.
class Rectangle {
public:
    virtual void setWidth(double w)  { w_ = w; }
    virtual void setHeight(double h) { h_ = h; }
    double area() const { return w_ * h_; }
protected: double w_{}, h_{};
};
class Square : public Rectangle {
public:
    void setWidth(double w)  override { w_ = h_ = w; } // breaks callers
    void setHeight(double h) override { w_ = h_ = h; }
};

// Fix: siblings under an interface, not parent/child.
struct Shape { virtual ~Shape() = default; virtual double area() const = 0; };
struct Rectangle : Shape { double w, h; double area() const override { return w * h; } };
struct Square    : Shape { double side; double area() const override { return side * side; } };
```

## I — Interface Segregation

```cpp
// Violation: callers link against a fat IDevice with read/write/seek/flash/reset.

// Fix: small role interfaces; compose as needed.
struct Readable { virtual ~Readable() = default; virtual std::size_t read(std::span<std::byte>) = 0; };
struct Writable { virtual ~Writable() = default; virtual std::size_t write(std::span<const std::byte>) = 0; };

void dump(Readable& src, Writable& dst) { /* uses only the two methods it needs */ }
```

## D — Dependency Inversion

```cpp
// Violation: high-level policy `new`s a concrete logger and calls std::chrono directly.
class PaymentService {
    FileLogger logger_;                           // concrete dependency
public:
    void charge(int cents) {
        logger_.info("charging");
        auto now = std::chrono::system_clock::now();
        // ...
    }
};

// Fix: depend on abstractions; inject concretes at the composition root.
struct ILogger { virtual ~ILogger() = default; virtual void info(std::string_view) = 0; };
using Clock = std::function<std::chrono::system_clock::time_point()>;

class PaymentService {
    ILogger& logger_;
    Clock    now_;
public:
    PaymentService(ILogger& l, Clock now) : logger_(l), now_(std::move(now)) {}
    void charge(int cents) { logger_.info("charging"); auto t = now_(); /* ... */ }
};

// main.cpp
FileLogger log;
PaymentService svc(log, []{ return std::chrono::system_clock::now(); });
```

Notes for C++:
- Prefer `std::unique_ptr<Interface>` over raw `new`/`delete` when injecting.
- If the set of variants is closed and known at compile time, `std::variant` +
  `std::visit` often beats virtual dispatch on both clarity and performance
  while still satisfying OCP at the visitor level.
- Do not pay for a virtual when a concept/template gives you the same extension
  point statically.
