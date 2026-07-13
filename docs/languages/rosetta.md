# Rosetta - Go / Python / Node → Java

Reach for this when learning a new backend language by mapping to Java/Spring you already know.

!!! note "Stub - scaffolded, not yet filled"
    Starter table below; expand each concept over time.

## Concept map

| Java / Spring | Go | Python | Node |
|---|---|---|---|
| Package/module | package + module | package/module | ES module / CommonJS |
| Maven/Gradle | `go mod` | pip + `pyproject.toml` | npm/pnpm + `package.json` |
| `interface` | `interface` (implicit) | duck typing / `Protocol` | duck typing / TS `interface` |
| Constructor DI | struct + explicit wiring | `__init__` / DI libs | constructor / DI libs |
| `@RestController` | `net/http` / gin / echo | FastAPI / Flask | Express / Nest |
| Checked-ish errors | `error` return values | exceptions | exceptions / rejected promises |
| Streams | slices + loops / `range` | comprehensions / generators | array methods / iterators |
| `Optional<T>` | `(val, ok)` / pointer nil | `Optional`/`None` | `undefined`/`null` |
| Virtual threads | goroutines | asyncio | event loop / async-await |
| `@Transactional` | explicit tx / `defer` | context managers / ORM tx | ORM tx |
| Testcontainers | testcontainers-go | testcontainers-python | testcontainers-node |

## Concurrency models (side by side)

- Java: virtual threads / `ExecutorService`
- Go: goroutines + channels
- Python: asyncio / GIL caveats
- Node: single-threaded event loop

## Error handling philosophies

## Dependency management & project layout

## HTTP server "hello world" in each

## Gotchas per language (stub)

## Quick reference

| Idea | Best analogy to Java |
|---|---|
| Go `defer` | try-with-resources / `finally` |
| Python context manager | try-with-resources |
| Go channel | `BlockingQueue` |
| Node middleware | Servlet `Filter` |
