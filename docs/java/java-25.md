# Java 25

Reach for this to confirm you're using the modern form of the language, not a 2015 habit.
Java 25 is the current LTS - target it unless a project forces otherwise.

## Records (over POJOs for immutable data)

```java
public record Point(int x, int y) {}          // ctor, accessors, equals, hashCode, toString

// Compact constructor for validation/normalization
public record Money(BigDecimal amount, String currency) {
    public Money {
        if (amount.signum() < 0) throw new IllegalArgumentException("negative");
        currency = currency.toUpperCase();     // normalize (no `this.`)
    }
}

// Static factory + extra behaviour still fine
public record Range(int lo, int hi) {
    public static Range of(int lo, int hi) { return new Range(lo, hi); }
    public boolean contains(int n) { return n >= lo && n <= hi; }
}
```

Accessors are `x()` / `y()`, **not** `getX()`. Records are implicitly final, can't extend, fields are final. Perfect for DTOs (your convention: DTOs are records, no Lombok).

## Sealed classes (closed hierarchies)

```java
public sealed interface Shape permits Circle, Square, Rectangle {}
public record Circle(double radius) implements Shape {}
public record Square(double side) implements Shape {}
public record Rectangle(double w, double h) implements Shape {}
```

`permits` lists the only allowed subtypes. Subtypes must be `final`, `sealed`, or `non-sealed`.
Pairs with pattern-matching `switch` for **exhaustive** handling - no `default` needed.

## Pattern matching - `instanceof`

```java
// Old
if (obj instanceof String) { String s = (String) obj; use(s); }

// Modern - bind in the check
if (obj instanceof String s && !s.isBlank()) use(s);
```

## Pattern matching - `switch` (with record deconstruction)

```java
double area = switch (shape) {                 // exhaustive over a sealed type
    case Circle(double r)        -> Math.PI * r * r;
    case Square(double s)        -> s * s;
    case Rectangle(double w, double h) -> w * h;
};

// Guards with `when`, plus null handling
String describe(Object o) {
    return switch (o) {
        case null            -> "nothing";
        case Integer i when i < 0 -> "negative int";
        case Integer i       -> "int " + i;
        case String s        -> "string " + s;
        default              -> "other";
    };
}
```

Sealed + record patterns = the compiler enforces you handled every case. This replaces
the visitor pattern and cast-and-if chains.

## Switch expressions

```java
int days = switch (month) {
    case FEB -> 28;
    case APR, JUN, SEP, NOV -> 30;
    default -> 31;
};

// Multi-statement arm uses yield
String label = switch (score) {
    case 1, 2, 3 -> "low";
    default -> {
        var normalized = score / 10;
        yield "band-" + normalized;
    }
};
```

`->` arms don't fall through (no `break`). Switch *expressions* must be exhaustive and return a value.

## Text blocks

```java
String json = """
    {
      "name": "%s",
      "role": "backend"
    }
    """.formatted(name);

String sql = """
    SELECT id, email
    FROM users
    WHERE active = true
    """;
```

Incidental leading whitespace is stripped to the least-indented line. `\` at line end = no newline; `\s` = keep trailing space.

## Virtual threads (IO-bound work)

```java
// One virtual thread per task - cheap, millions are fine. For blocking IO.
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = urls.stream()
        .map(url -> executor.submit(() -> fetch(url)))    // each blocks on IO, no pool starvation
        .toList();
}
```

In Spring Boot: `spring.threads.virtual.enabled=true` and the web server + `@Async` use virtual threads. Use them for IO-bound work (HTTP, DB); **not** for CPU-bound work.

## `var` (where the type is obvious)

```java
var users = new ArrayList<User>();        // obvious
var count = users.size();                 // obvious
for (var entry : map.entrySet()) { }      // less noise

// Don't: hides the type
var result = service.process();           // process() returns... what?
```

Local variables only. Use it when the RHS makes the type clear; skip it when it hurts readability.

## Streams & collectors (the ones you actually use)

```java
List<String> names = users.stream()
    .filter(User::isActive)
    .map(User::name)
    .sorted()
    .toList();                                   // immutable list (Java 16+)

Map<Role, List<User>> byRole = users.stream()
    .collect(Collectors.groupingBy(User::role));

Map<Role, Long> counts = users.stream()
    .collect(Collectors.groupingBy(User::role, Collectors.counting()));

Optional<User> first = users.stream()
    .filter(u -> u.age() > 30)
    .findFirst();

double avg = users.stream().mapToInt(User::age).average().orElse(0);
```

`Stream.toList()` over `collect(Collectors.toList())` - shorter and returns an unmodifiable list.

## Optional (do / don't)

```java
Optional<User> found = repository.findByEmail(email);

// Do
User u = found.orElseThrow(() -> new UserNotFoundException(email));
String name = found.map(User::name).orElse("anonymous");
found.ifPresent(this::notify);

// Don't
if (found.isPresent()) { User u = found.get(); }   // defeats the purpose
Optional<User> field;                               // never as a field or parameter
```

Use `Optional` as a **return type** to signal "maybe absent". Never as a field, parameter, or in collections.

## Gotchas / things I always forget

- Record accessors are `name()`, not `getName()` - surprises Jackson/Bean code expecting JavaBean naming (Boot handles records fine).
- Records can't have instance fields beyond the components, and can't extend a class (they extend `Record`).
- Sealed hierarchies need every permitted subtype to declare `final` / `sealed` / `non-sealed` - the compiler nags otherwise.
- Pattern `switch` is exhaustive over sealed types **without** `default`; adding `default` disables the exhaustiveness check (a new subtype won't error).
- Virtual threads don't help CPU-bound work - they shine only when threads block on IO.
- Pinning: a virtual thread inside a `synchronized` block or native call pins to its carrier - prefer `ReentrantLock` in hot paths.
- `var` is compile-time inference, not `Object` - the type is fixed, just not written.
- Text blocks: the closing `"""` position controls indentation stripping; misplacing it changes the output.

## Quick reference

| Old way | Modern way |
|---|---|
| POJO + getters/setters | `record` |
| `(String) obj` after `instanceof` | `instanceof String s` |
| cast + if-chain on type | pattern-matching `switch` |
| `switch` with `break` | `switch` expression `->` + `yield` |
| `"a\n" + "b\n"` | text block `"""` |
| enum-tree / visitor | sealed interface + record patterns |
| thread pool for blocking IO | virtual threads |
| `collect(toList())` | `.toList()` |
| explicit generic type | `var` (when obvious) |
