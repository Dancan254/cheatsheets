# Java Contest Idioms

Reach for this for the Java-specific tricks that make LeetCode less painful.

!!! note "Stub - scaffolded, not yet filled"
    Starter snippets below; expand over time.

## Collections you actually want

- `ArrayDeque` for stack **and** queue (never `Stack`/`LinkedList`)
- `PriorityQueue` min-heap; max-heap via `Collections.reverseOrder()`
- `TreeMap`/`TreeSet` for sorted + `floorKey`/`ceilingKey`/`higherKey`

## Map accumulation

```java
map.merge(key, 1, Integer::sum);          // count
map.computeIfAbsent(key, k -> new ArrayList<>()).add(v);   // group
map.getOrDefault(key, 0);
```

## Arrays & sorting

```java
Arrays.sort(a, (x, y) -> x[0] - y[0]);    // custom comparator (objects only)
Arrays.fill(dp, -1);
int[] copy = Arrays.copyOf(a, a.length);
Comparator.comparingInt(...).thenComparing(...);
```

## Strings

```java
char[] cs = s.toCharArray();
StringBuilder sb = new StringBuilder();   // never += in a loop
int digit = c - '0';  int idx = c - 'a';
```

## Numeric limits & overflow

```java
Integer.MAX_VALUE / MIN_VALUE, Long.MAX_VALUE
long to avoid overflow; mid = lo + (hi - lo) / 2;
```

## Char/int/parse conversions

## 2D arrays & grids

## Gotchas / things I always forget

- Custom comparators only work on **object** arrays, not `int[]` - box to `Integer[]` or sort manually.
- `PriorityQueue` is a **min**-heap by default.
- `remove(int)` vs `remove(Object)` on a List - index vs value ambiguity.

## Quick reference

| Need | Idiom |
|---|---|
| Count | `map.merge(k,1,Integer::sum)` |
| Group | `computeIfAbsent(k, x->new ArrayList<>())` |
| Max-heap | `new PriorityQueue<>(reverseOrder())` |
| Build string | `StringBuilder` |
| Safe mid | `lo + (hi-lo)/2` |
