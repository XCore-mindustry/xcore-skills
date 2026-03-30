# Arc Collections

Arc replaces Java standard collections with its own GC-friendly implementations. These are used everywhere in Mindustry — never use `ArrayList`, `HashMap`, or `HashSet` in Mindustry plugin code.

**Package**: `arc.struct`

## Table of Contents

- [Seq\<T\>](#seqt--the-primary-list)
- [ObjectMap\<K,V\>](#objectmapkv--hash-map)
- [ObjectSet\<T\>](#objectsett--hash-set)
- [IntMap\<V\>](#intmapv--int-keyed-map)
- [IntSet](#intset)
- [OrderedMap\<K,V\>](#orderedmapkv)
- [Queue\<T\>](#queuet--ring-buffer)
- [Other collections](#other-collections)
- [Functional interfaces](#functional-interfaces)

---

## `Seq<T>` — The Primary List

Arc's replacement for `ArrayList`. Backed by a raw `T[] items` array with a public `int size` field.

### Construction

```java
new Seq<>()                    // ordered, default capacity 16
new Seq<>(capacity)            // ordered with initial capacity
new Seq<>(false, capacity)     // unordered (faster removal via swap)
Seq.with(a, b, c)              // from varargs
Seq.with(iterable)             // from Iterable
```

### Core operations

```java
seq.add(item)                  // append (returns this for chaining)
seq.add(a, b), seq.add(a, b, c) // multi-add
seq.addAll(otherSeq)           // bulk add
seq.addUnique(item)            // add only if not present (linear scan)
seq.insert(index, item)        // insert at index
seq.get(index)                 // indexed access (bounds-checked)
seq.set(index, item)           // indexed set
seq.first()                    // first element (throws if empty)
seq.firstOpt()                 // first element or null
seq.peek()                     // last element
seq.pop()                      // remove and return last
seq.remove(item)               // remove first occurrence
seq.remove(index)              // remove by index
seq.remove(Boolf)              // remove first matching
seq.removeAll(Boolf)           // remove all matching
seq.retainAll(Boolf)           // keep only matching
seq.clear()                    // clear (nulls references for GC)
seq.isEmpty()                  // is empty
seq.any()                      // has elements (opposite of isEmpty)
seq.size                       // element count (public field)
```

### Search

```java
seq.contains(item)             // linear equality search
seq.contains(Boolf)            // predicate search
seq.indexOf(item)              // index or -1
seq.indexOf(Boolf)             // predicate index or -1
seq.find(Boolf)                // first match or null
seq.count(Boolf)               // count matches
seq.allMatch(Boolf)            // all match predicate
```

### Functional-style (allocate new sequences)

```java
seq.map(Func)                  // transform elements → new Seq
seq.flatMap(Func)              // flatten → new Seq
seq.select(Boolf)              // filter → new Seq
seq.each(Cons)                 // iterate (no allocation)
seq.each(Boolf, Cons)          // filtered iterate
seq.reduce(init, Func2)        // fold
seq.min(Floatf)                // element with minimum float key
seq.max(Floatf)                // element with maximum float key
seq.min(Comparator)            // minimum by comparator
seq.sumf(Floatf)               // sum of float projections
seq.sum(Intf)                  // sum of int projections
```

### Sorting

```java
seq.sort()                     // natural order (mutating)
seq.sort(Comparator)           // custom comparator
seq.sort(Floatf)               // sort by float key
seq.sortComparing(Func)        // sort by Comparable key
seq.reverse()
seq.shuffle()
seq.distinct()                 // remove duplicates (preserves order)
```

### Conversion

```java
seq.list()                     // → ArrayList<T>
seq.toArray()                  // → Object[]
seq.toArray(Class)             // → T[]
seq.asMap(Func)                // → ObjectMap (key extractor, value=element)
seq.asMap(Func, Func)          // → ObjectMap (key, value extractors)
seq.asSet()                    // → ObjectSet
seq.toString(", ")             // join with separator
seq.toString(", ", Func)       // join with transformer + separator
```

### Iterator gotcha

`Seq.iterator()` **reuses** iterator instances (2 cached). Simple and one-level-nested for-each loops work. Breaking early or 3+ nesting levels causes issues. **Prefer `.each()` or indexed loops for server code.**

```java
// Preferred
seq.each(item -> process(item));

// Also fine
for (int i = 0; i < seq.size; i++) {
    process(seq.get(i));
}

// Works but fragile with early break
for (var item : seq) { ... }
```

---

## `ObjectMap<K,V>` — Hash Map

Cuckoo hash map. **Null keys are NOT allowed** (throws). Null values are fine.

### Construction & access

```java
new ObjectMap<>()
ObjectMap.of(key1, val1, key2, val2)  // varargs factory

map.put(key, value)                   // returns old value
map.get(key)                          // value or null
map.get(key, defaultValue)            // value or default
map.get(key, Prov<V>)                 // compute-if-absent pattern
map.getThrow(key, Prov<RuntimeException>) // get or throw
map.containsKey(key)
map.containsValue(value, identity)    // identity=true → ==, false → equals
map.remove(key)                       // returns removed value
map.each(Cons2<K,V>)                  // iterate
map.size                              // element count
map.isEmpty()
map.clear()
map.merge(otherMap)                   // putAll + return this
map.copy()                            // shallow copy
```

### Iterators

```java
// ⚠️ Reused (2 cached). Don't nest iterations on same map.
for (var entry : map.entries()) {
    entry.key; entry.value;
}
for (var key : map.keys()) { ... }
for (var val : map.values()) { ... }

// For nested iteration, create explicit iterator:
new ObjectMap.Entries<>(map);
```

---

## `ObjectSet<T>` — Hash Set

Same cuckoo hash internals. **`add(null)` is silently ignored** (unlike ObjectMap which throws for null keys).

```java
new ObjectSet<>()
ObjectSet.with(a, b, c)
ObjectSet.with(seq)

set.add(item)                  // true if newly added
set.contains(item)
set.get(item)                  // canonical instance (by equality)
set.remove(item)               // true if removed
set.addAll(seq)
set.removeAll(seq)
set.first()                    // arbitrary element
set.toSeq()
set.each(Cons)
set.select(Boolf)              // → new ObjectSet
```

---

## `IntMap<V>` — int-keyed Map

Cuckoo hash map with primitive `int` keys. Zero key handled specially.

```java
map.put(intKey, value)
map.get(intKey)
map.get(intKey, defaultValue)
map.get(intKey, Prov<V>)       // compute-if-absent
map.remove(intKey)
map.containsKey(intKey)
map.each(IntMapConsumer)       // iterate with int key
```

---

## `IntSet`

Primitive `int` hash set.

```java
IntSet.with(1, 2, 3)
set.add(intValue)
set.contains(intValue)
set.remove(intValue)
set.each(Intc)                 // iterate with primitive consumer
```

---

## `OrderedMap<K,V>`

Extends `ObjectMap` with insertion-order iteration via an internal `Seq<K>`.

```java
map.orderedKeys()              // Seq<K> in insertion order
map.removeIndex(int)           // remove by position in ordered keys
```

---

## `Queue<T>` — Ring Buffer

Circular buffer / deque. O(1) at both ends.

```java
queue.addLast(item)            // enqueue (same as add)
queue.addFirst(item)           // push to front
queue.removeFirst()            // dequeue
queue.removeLast()             // pop from back
queue.first()                  // peek front
queue.last()                   // peek back
queue.get(index)               // indexed access
queue.size                     // current size
queue.each(Cons)               // iterate
queue.clear()
```

---

## Other collections

| Class | Description |
|-------|-------------|
| `StringMap` | `ObjectMap<String,String>` with `getInt()`, `getFloat()`, `getBool()` parsing |
| `SnapshotSeq<T>` | Safe iteration while allowing concurrent modification (`begin()`/`end()`) |
| `DelayedRemovalSeq<T>` | Queues removals between `begin()`/`end()` |
| `IntSeq`, `FloatSeq`, `LongSeq`, `ByteSeq`, `ShortSeq` | Primitive-typed sequences |
| `IntIntMap`, `IntFloatMap`, `ObjectIntMap`, `ObjectFloatMap` | Primitive-valued maps |
| `OrderedSet<T>` | Insertion-ordered set |
| `EnumSet` | Lightweight enum bitflag set |
| `PQueue<T>` | Priority queue (binary heap) |
| `GridBits`, `GridMap` | 2D grid-indexed structures |
| `Bits` | Unlimited-size bitset (`get`, `set`, `clear`, `and`, `or`, `xor`) |

---

## Functional interfaces (`arc.func`)

Arc defines its own functional interfaces instead of `java.util.function`:

| Arc | Java Equivalent | Signature |
|-----|----------------|-----------|
| `Cons<T>` | `Consumer<T>` | `void get(T)` |
| `Cons2<T,N>` | `BiConsumer` | `void get(T, N)` |
| `Cons3<A,B,C>` | — | `void get(A, B, C)` |
| `Func<P,R>` | `Function<P,R>` | `R get(P)` |
| `Func2<A,B,R>` | `BiFunction` | `R get(A, B)` |
| `Prov<T>` | `Supplier<T>` | `T get()` |
| `Boolf<T>` | `Predicate<T>` | `boolean get(T)` |
| `Boolf2<A,B>` | `BiPredicate` | `boolean get(A, B)` |
| `Boolp` | `BooleanSupplier` | `boolean get()` |
| `Floatf<T>` | `ToDoubleFunction` | `float get(T)` |
| `Floatp` | — | `float get()` |
| `Floatc` | — | `void get(float)` |
| `Floatc2` | — | `void get(float, float)` |
| `FloatFloatf` | — | `float get(float)` |
| `Intf<T>` | `ToIntFunction` | `int get(T)` |
| `Intp` | `IntSupplier` | `int get()` |
| `Intc` | `IntConsumer` | `void get(int)` |
| `Intc2` | — | `void get(int, int)` |

The method is always called `get()` on all Arc functional interfaces.
