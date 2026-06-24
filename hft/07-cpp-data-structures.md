# Lesson 7 — C++ Data Structures: Knowing the Containers Cold

> [Lesson 1](01-computer-architecture.md) taught the deepest performance truth of the whole course: **computing is cheap; moving data is expensive**, memory moves in 64-byte cache lines, and contiguous, predictable access is rewarded while scattered pointer-chasing is punished. This lesson takes that truth and applies it, concretely, to the tools you reach for every single day as a C++ programmer: the **standard containers** — `array`, `vector`, `list`, `map`, `set`, and their `unordered_` cousins.
>
> The goal is for you to know these containers *cold*: what each one **is** underneath, what operations it makes **fast** and which it makes **slow** (or **impossible**), **what to use when**, the **typical use cases**, and **how to speed them up** when the standard version stalls. This is foundational C++ knowledge — useful in *any* domain — but we'll read it through the HFT lens, because HFT is where choosing the wrong container, or not knowing how to make the right one fast, costs measurable nanoseconds. Lesson 8 (C++ for HFT) builds directly on this; you can't write fast code until you know what these containers actually do.
>
> No assumption of prior STL depth — we build each container up from what it physically is.

---

## 0. The one question that governs every container choice

Every time you store a collection of things, you make a choice — `vector`? `map`? `unordered_set`? — and that choice silently decides two things at once:

1. **How your data is laid out in memory** (which, per Lesson 1, decides your cache behavior — often the *dominant* cost).
2. **Which operations are cheap and which are expensive** (the algorithmic complexity).

> **The mental model: a container is a contract.** It promises that *certain* operations will be fast, in exchange for accepting that *others* will be slow or impossible. There is no container that's fast at everything — if there were, there'd only be one. Choosing a container is choosing *which operations you want to be cheap*, and accepting the price elsewhere. Get the match right and your code is fast and clear; get it wrong and you're paying — in cache misses, in wasted cycles, in nanoseconds — on every operation, forever.

So the single question to ask before picking a container is: **"What operations will I actually perform on this data, and how often?"** Not "what's the most convenient," not "what do I usually use" — *what operations, how often.* Look it up by key? Iterate in order? Insert in the middle? Always append? Find the maximum? The answer points at exactly one or two right containers and rules out the rest. This lesson gives you the knowledge to answer that question correctly every time.

---

## 1. The cost vocabulary: Big-O, and why HFT distrusts it

Before the containers, we need the language for "fast" and "slow." That language is **Big-O notation** (asymptotic complexity), and then — crucially for HFT — the *caveat* that Big-O leaves out the thing Lesson 1 said matters most.

### What Big-O means

Big-O describes **how the cost of an operation grows as the number of elements (`n`) grows.** It's a statement about *scaling*, not absolute speed. The ones you'll meet:

| Notation | Name | Meaning | Everyday feel |
| --- | --- | --- | --- |
| **O(1)** | constant | Cost doesn't depend on `n` at all | Grabbing item #5 from a numbered shelf — same effort whether the shelf holds 10 or 10 million |
| **O(log n)** | logarithmic | Cost grows *very* slowly — doubling `n` adds one step | Finding a word in a dictionary by repeated halving — a million words is only ~20 steps |
| **O(n)** | linear | Cost grows in direct proportion to `n` | Reading every name on a guest list to find one — twice the list, twice the work |
| **O(n log n)** | linearithmic | The cost of a good sort | Sorting that guest list |
| **O(n²)** | quadratic | Cost grows with the *square* — doom for large `n` | Comparing every guest to every other guest |

One more vital term: **amortized.** When we say `vector::push_back` is "amortized O(1)," we mean: *almost every* call is O(1) (cheap), but *occasionally* one call is O(n) (expensive — it triggers a reallocation, §3), and *averaged over many calls* the cost is constant. Amortized O(1) is "usually instant, rarely a hiccup." **For HFT this distinction is critical**, because that rare O(n) hiccup is a latency *spike* (jitter, Module 0) — the average is fine but the *tail* is not. Hold that thought; it returns in §3 and §11.

### Why HFT engineers distrust Big-O — the constant-factor and cache trap

Here is the most important idea in this section, and it's a direct consequence of Lesson 1. **Big-O hides two things that often matter more than the asymptotic class itself:**

1. **The constant factor.** O(1) doesn't mean "one nanosecond" — it means "some constant number of operations, regardless of `n`." A hash-map lookup is O(1) but might involve computing a hash, a memory load to find the bucket, and a pointer chase to a scattered node — maybe 100 ns. A linear scan of a small `vector` is O(n) but each step is a single cache-friendly comparison — maybe 1 ns each. **For small `n`, the "slower" O(n) scan crushes the "faster" O(1) lookup.**

2. **Cache behavior.** This is the big one. Big-O counts *operations*, but Lesson 1 taught that not all operations cost the same: a cache hit is ~1 ns, a cache miss is ~100 ns. A data structure with great Big-O but terrible cache behavior (every step a cache-missing pointer chase) routinely loses to one with worse Big-O but perfect cache behavior (a tight contiguous scan the prefetcher loves).

> **The HFT mantra: complexity classes are a starting point, not the answer.** A `std::map` lookup (O(log n), but a chain of cache-missing pointer hops through a tree) is frequently *slower* than a linear scan of a sorted `std::vector` (O(n), but a contiguous, prefetcher-friendly march) for the collection sizes HFT actually uses (often dozens to a few thousand elements). **You reason about the asymptotic class *and* the constant factor *and* the cache behavior — and then you measure.** This single insight explains most of the "surprising" container choices in HFT code, and we'll see it again and again below.

---

## 2. The great divide: contiguous vs. node-based

Before the individual containers, internalize the one distinction that explains the cache behavior of all of them — it's Lesson 1 §5 ("the linked-list trap") turned into a classification.

Every standard container is built one of two ways:

**Contiguous** — the elements live **packed together in one solid block of memory**, back to back. `std::array` and `std::vector` are the contiguous containers.

> **The analogy: a row of numbered lockers along one wall.** To visit them all, you walk straight down the row — every locker is right next to the last. To grab locker #500, you walk directly to it (you know exactly where it is: start + 500 × locker-width). The cache line (Lesson 1 §3) loves this: fetching one locker drags its neighbors along for free, and the prefetcher (Lesson 1 §4) sees your steady march and runs ahead. **Contiguous = cache-friendly = the machine's favorite.**

**Node-based** — each element lives in its **own separately-allocated chunk of memory (a "node")**, scattered wherever the allocator happened to put it, and the nodes are linked together by **pointers** (memory addresses saying "the next one is over there"). `std::list`, `std::map`, `std::set`, and (partly) the `unordered_` containers are node-based.

> **The analogy: a city-wide treasure hunt.** Each clue (node) is at a different scattered address, and holds a slip of paper telling you where the *next* clue is. To traverse the collection you drive to clue 1, read where clue 2 is, drive across town to it, read where clue 3 is, and so on. Each hop is a fresh, unpredictable trip — a **cache miss** (Lesson 1: ~100 ns, "a drive across town"), and the prefetcher can't help because it can't guess the next address until you've read the current clue. **Node-based = cache-hostile = the machine's least favorite.**

This divide is the single most useful lens for the whole lesson. As we go through each container, the *first* thing to ask is "contiguous or node-based?" — because that, more than the Big-O, predicts how it behaves on real hardware. HFT's strong default preference for contiguous containers falls straight out of it.

A subtle but important consequence: node-based containers also do a **heap allocation per element** (every `insert` calls the allocator to make a node), which is itself slow and — critically for HFT — *unpredictable* (the allocator can lock, can fault; Lesson 8). Contiguous containers allocate in bulk, rarely. This is a second strike against node-based structures in the hot path, on top of the cache misses.

---

## 3. `std::vector` — the default, and usually the right answer

We start with `std::vector` because it should be your **default container** — the one you reach for unless you have a specific reason not to. Understanding it well covers most of your needs.

### What it is

A `std::vector` is a **dynamic contiguous array**: a single block of memory holding your elements back-to-back (contiguous — the machine's favorite), that can **grow** as you add elements. It keeps three things internally: a pointer to the block, the **size** (how many elements you're using), and the **capacity** (how many the block can hold before it must grow).

> **The analogy: an apartment you can outgrow.** Your elements are furniture in a contiguous apartment. As long as there's room (capacity > size), adding furniture is instant — drop it in the next empty spot. But when the apartment is *full* and you add one more item, you must **move to a bigger apartment**: rent a larger place (allocate a bigger block, typically 2× the size), carry *every* piece of furniture over (copy/move all elements), and hand back the old keys (free the old block). That move is the expensive, occasional event behind "amortized O(1)."

### Operations and their complexity

| Operation | Complexity | Why |
| --- | --- | --- |
| **Random access** `v[i]` | **O(1)** | Address = start + i × element-size; one calculation, one cache-friendly load. The killer feature. |
| **Append** `push_back` / `emplace_back` | **Amortized O(1)** | Usually instant; occasionally triggers the "move to a bigger apartment" reallocation (O(n)). |
| **Insert/erase at the end** | Amortized O(1) | Same as append. |
| **Insert/erase in the middle or front** | **O(n)** | Everything after the insertion point must shift over to make room (or close the gap). Furniture-shuffling. |
| **Search (unsorted)** | **O(n)** | Linear scan — but a *cache-friendly* one the prefetcher loves, so the constant factor is tiny. |
| **Search (sorted, via `binary_search`/`lower_bound`)** | **O(log n)** | If you keep it sorted, you can binary-search — fast *and* cache-resident. |
| **Iterate all elements** | O(n) | The single best traversal on the machine — contiguous, prefetched. |

### Possible and impossible

- **Possible and excellent:** indexing, appending, iterating in insertion order, sorting (`std::sort`), binary search if sorted.
- **Slow (possible but avoid in hot paths):** inserting/erasing anywhere but the end (O(n) shifting).
- **The invalidation gotcha:** when a `vector` reallocates (grows past capacity), it moves its block — so **every pointer, reference, and iterator into the old block becomes dangling and invalid.** This is the classic `vector` bug: holding a pointer to `v[3]`, calling `push_back`, and now your pointer points at freed memory. Erasing/inserting in the middle also invalidates iterators from that point on. (Node-based containers, §5–6, don't have this problem — a key reason they sometimes earn their keep.)

### How to speed it up

- **`reserve(n)` up front.** If you know (even approximately) how many elements you'll add, call `reserve` once to allocate the capacity ahead of time. Now *every* `push_back` is genuinely O(1) — **no reallocations, no copying, no latency spikes.** In HFT this is essential: you `reserve` all your vectors at startup so the hot path never reallocates (and never touches the allocator). This converts "amortized O(1) with occasional spikes" into "O(1) with no spikes" — killing the jitter.
- **Use `emplace_back` instead of `push_back`** when constructing elements in place, to avoid building a temporary and copying it in.
- **Iterate by index or with iterators, contiguously** — you're already on the machine's happy path.
- **Keep it sorted and binary-search it** rather than reaching for a `map`, when lookups matter (this is the "flat map" idea — §12).
- **`shrink_to_fit`** if you've shed many elements and want the memory back (rare in HFT, where you keep capacity reserved).

### Typical use cases

The default for *any* list of things: market-data records to scan, a batch of orders, a buffer of recent prices, lookup tables indexed by a small integer ID. **If you're not sure what to use, use `vector`** — and only switch when a specific operation it's bad at (middle insertion, keyed lookup) turns out to dominate.

---

## 4. `std::array` — the fixed-size, zero-overhead sibling

### What it is

`std::array<T, N>` is a **fixed-size contiguous array** whose size `N` is known *at compile time* and **lives on the stack** (or inline inside another object), not the heap. It's a thin, zero-overhead wrapper around a raw C array `T[N]`, giving it the nice STL interface (`.size()`, iterators, bounds-checked `.at()`) without any cost.

> **The analogy: a row of lockers bolted to the wall — you can never add or remove a locker, but there's zero overhead and zero uncertainty about where anything is.** Because the size never changes, there's no capacity, no growing, no reallocation, no heap allocation at all.

### Operations and complexity

Same contiguous goodness as `vector` for access (O(1) indexing, O(n) cache-friendly iteration, O(log n) binary search if sorted) — but **you cannot change its size.** No `push_back`, no `insert`, no `erase`. It is exactly `N` elements, forever.

### Why HFT loves it

- **No heap allocation, ever** — it's on the stack or inline, so there's *nothing* for the allocator to do, and no pointer indirection to reach the data. This is the most predictable, lowest-latency storage possible for a known-size collection.
- **No reallocation, ever** — the size can't change, so there's no possibility of a latency spike or iterator invalidation.
- **Perfect cache behavior** and the compiler often knows the size, enabling more optimization (unrolling, vectorization — Lesson 2).

### Typical use cases

Anything with a **fixed, known-at-compile-time count**: a fixed number of price levels in an order book, a fixed set of trading venues, a small lookup table, a SIMD-friendly buffer. In HFT, prefer `std::array` over `std::vector` whenever the size is fixed and known — you get all the cache benefits with *none* of the dynamic-allocation uncertainty.

---

## 5. `std::list` — the node-based sequence you'll rarely want

### What it is

`std::list` is a **doubly-linked list**: the archetypal node-based container. Each element is its own heap-allocated node holding the value plus two pointers — one to the previous node, one to the next. `std::forward_list` is the singly-linked version (one pointer, slightly leaner).

This is the "treasure hunt" from §2 in its purest form: elements scattered across the heap, linked by pointers.

### Operations and complexity

| Operation | Complexity | Note |
| --- | --- | --- |
| **Insert/erase given an iterator to the spot** | **O(1)** | Just relink a few pointers — no shifting. Its one genuine strength. |
| **`splice` (move a range between lists)** | **O(1)** | Relink pointers; no copying. Unique to linked lists. |
| **Random access** `l[i]` | **Impossible** | There is no `operator[]`. You must walk from the start — O(n). |
| **Search** | **O(n)** | Walk the chain — and every hop is a **cache miss**. |
| **Iterate** | O(n) | But cache-hostile: a cache miss per element. |

### The honest verdict for HFT

On paper, "O(1) insert/erase anywhere" sounds better than `vector`'s O(n) middle-insert. **In practice it's almost always slower**, because: (a) every traversal and search is a cache-missing pointer chase (Lesson 1 — ~100 ns per hop vs. ~1 ns for a contiguous step); (b) every insert is a separate heap allocation (slow and unpredictable); and (c) to insert "in the middle" you first have to *find* the middle, which is an O(n) cache-missing walk anyway. The famous practical result (demonstrated by Bjarne Stroustrup himself): for the vast majority of workloads, **`std::vector` beats `std::list` even at tasks list is theoretically better at**, because cache behavior dominates the Big-O.

> **When does `list` actually win?** Rarely, but genuinely: when you (a) hold **stable references/iterators** that must *never* invalidate even as you insert/erase elsewhere (list nodes never move, unlike vector's reallocations), or (b) need **O(1) splicing** of large sublists between containers. Outside those, reach for `vector`. **In HFT hot paths, `std::list` is essentially banned** — it's the cache-hostile, allocation-per-element antithesis of everything Lesson 1 wants.

---

## 6. `std::map` and `std::set` — ordered, tree-based associative containers

Now the **associative** containers — the ones that store data by *key* for fast lookup, rather than by position. There are two flavors, *ordered* (this section) and *unordered* (§7), and understanding the difference is one of the most practically important things in this lesson.

### What they are

`std::map<Key, Value>` stores **key→value pairs, kept sorted by key**. `std::set<Key>` stores just **keys, kept sorted** (a `map` with no values — used to track membership: "is this in the set?"). Both are built as a **balanced binary search tree** (specifically a red-black tree): a node-based structure where each node has a key and pointers to a left child (smaller keys) and right child (larger keys), kept balanced so the tree never gets lopsided.

> **The analogy: a library card catalog kept in sorted drawers.** To find a card, you don't scan every drawer — you jump to the middle, see if your target is before or after, and repeat, halving the search each time (that's the O(log n) binary search). Because it's *sorted*, you can also walk the drawers *in order*, or ask "give me every card between G and M" (a range query). But it's still node-based — the drawers (nodes) are scattered in memory, so each step of the search is a cache-missing hop.

### Operations and complexity

| Operation | Complexity | Note |
| --- | --- | --- |
| **Lookup by key** `m.find(k)` / `m[k]` | **O(log n)** | Tree descent — but each step is a cache-missing pointer hop. |
| **Insert** | **O(log n)** | Find the spot, allocate a node, relink, rebalance. |
| **Erase by key** | **O(log n)** | Plus a heap deallocation. |
| **Ordered iteration** (smallest→largest) | **O(n)** | Walks the tree *in sorted order* — a key feature. |
| **Range queries** `lower_bound(k)` / `upper_bound(k)` | **O(log n)** | "First element ≥ k" / "first element > k" — find a price range, a time window. |
| **Min / max element** | O(log n) | Leftmost / rightmost node. |

### Possible and impossible — read this carefully

This is exactly the "possible and impossible operations" the curriculum asks about, and `map`/`set` have the most important *impossibilities* of all the containers:

- **✅ Possible:** keyed lookup, ordered iteration, range queries, finding min/max, "find the closest key ≥ X."
- **❌ Impossible: you cannot sort a `map` by its *values*.** A `std::map` is sorted by *key*, always, as an unbreakable invariant — that ordering *is* how the tree is built. If you want your data ordered by value (e.g., "show instruments sorted by volume" when volume is the *value*), the map cannot do it. **You must copy the pairs into a `std::vector` and `std::sort` that vector by value.** This is a constant source of beginner confusion ("how do I sort my map by value?" — you don't; you copy it out to a vector and sort that). The map's whole structure is committed to key-order; value-order is simply not a thing it can represent.
- **❌ Impossible: you cannot change a key in place.** The keys are `const` inside the map, because changing a key would break the sorted tree invariant (the node would be in the wrong place). To "change a key," you erase the element and re-insert it with the new key.
- **❌ Impossible: no random access by position.** There's no `m[3]` meaning "the 3rd element." `m[k]` means "the element with *key* k" (and for `map`, it even *inserts* a default element if key `k` is absent — another classic gotcha). To get "the 3rd in order" you iterate.
- **❌ No reordering / no manual positioning.** Order is dictated entirely by the key's sort order; you can't move elements around.

### How to speed it up

- **The biggest win: question whether you need it at all.** Because it's node-based and cache-hostile, `std::map` is one of the most *over-used* containers. For small/medium collections (the HFT norm), a **sorted `std::vector` searched with `lower_bound`** (a "flat map," §12) gives you the same O(log n) lookup *and* ordered iteration *and* range queries — but contiguous and cache-friendly, often several times faster. Reach for `std::map` only when you have *frequent insertions/erasures interleaved with lookups* on a *large* collection, where keeping a vector sorted would cost too much shifting.
- **Reserve nothing — it can't.** (Node-based; allocates per element. Another mark against it in the hot path.)
- **Provide a cheap comparator** and a cheap-to-compare key type.
- **Use a custom node allocator / memory pool** (Lesson 8) if you must use it, to tame the per-node allocation cost and scatter.

### Typical use cases

The textbook HFT example is the **order book**: prices mapped to the quantity available at each price, where you constantly need (a) lookup by price, (b) the best (min/max) price, and (c) iteration in price order. A `std::map<Price, Level>` expresses this perfectly *semantically* — which is why it's the classic teaching example — though high-performance order books often replace it with flat or array-based structures (§12, Lesson 9) precisely because of the cache cost. More broadly: anything needing **sorted-by-key access, ordered traversal, or range queries**.

---

## 7. `std::unordered_map` and `std::unordered_set` — hash tables

The unordered associative containers solve the same "look up by key" problem as `map`/`set`, but trade away *ordering* to get (usually) *faster* lookups.

### What they are

`std::unordered_map<Key, Value>` and `std::unordered_set<Key>` are **hash tables.** Instead of a sorted tree, they keep an array of "buckets." To store a key, they run it through a **hash function** — a function that turns the key into a big pseudo-random-looking number — then use that number to pick a bucket. To look a key up, they hash it again, go straight to that bucket, and check what's there. No searching, no tree descent — *jump directly to the right bucket.*

> **The analogy: a coat check.** You hand over your coat (key→value) and get a numbered ticket; the attendant hangs it on hook #47 (the hash of your ticket). To retrieve it, you give the ticket, they go *directly* to hook #47 — no searching through all the coats. Lightning fast on average. But the coats are in *hash order*, which is to say **no meaningful order at all** — you cannot ask "give me the coats alphabetically" or "the coats between G and M." That information is simply not preserved.

### Operations and complexity

| Operation | Complexity | Note |
| --- | --- | --- |
| **Lookup by key** | **O(1) average**, O(n) worst case | Average: hash and jump. Worst case (many keys colliding into one bucket, or a bad hash): degrades to a linear scan. |
| **Insert** | **O(1) average**, O(n) worst | Worst case includes an occasional **rehash** (see below). |
| **Erase by key** | O(1) average | |
| **Ordered iteration** | **Impossible** | Iteration order is hash order — arbitrary and unstable. |
| **Range queries** | **Impossible** | No ordering means no "between X and Y." |

### Possible and impossible

- **✅ Possible:** fast membership test and keyed lookup/insert/erase — *when order doesn't matter.*
- **❌ Impossible: any ordered operation.** No sorted iteration, no `lower_bound`/range queries, no min/max. If you need *order*, you need `map`/`set` (or a sorted vector), not the unordered version. This is the fundamental trade: **`unordered_` gives you (usually) faster lookup but throws away all ordering.**
- **The worst-case trap:** the lovely O(1) is an *average*. A bad hash function (or an adversary deliberately feeding colliding keys) can collapse everything into one bucket and turn every operation O(n). HFT cares because that worst case is a latency *spike*.

### The hidden cache cost (and rehashing)

Two things make `std::unordered_map` slower than its O(1) suggests, both from Lesson 1:

1. **It's still partly node-based.** The standard mandates an interface that forces most implementations to store each element in a *separately-allocated node* (chaining: each bucket is a linked list of nodes). So a lookup is: hash (cheap) → load the bucket (one cache access) → **chase a pointer to a scattered node** (a cache miss) → maybe more nodes if collisions. That pointer chase is why `std::unordered_map` underperforms its theoretical O(1) and why faster open-addressing alternatives exist (§12).
2. **Rehashing.** As you add elements, the table's "load factor" (elements per bucket) rises; when it crosses a threshold, the table **rehashes** — allocates a bigger bucket array and re-inserts everything. That's an O(n) spike — jitter, again.

### How to speed it up

- **`reserve(n)` at startup** to size the table for your expected count, eliminating rehash spikes (the hot-path jitter). Just like `vector::reserve`, this is essential HFT hygiene.
- **Provide a fast, well-distributed hash function** for your key type (the default for integers is fine; for custom keys, invest here).
- **For real speed, replace it** with an **open-addressing** hash map (no per-element nodes — elements live in a contiguous array, so far fewer cache misses): `abseil`'s `flat_hash_map`, `boost::unordered_flat_map`, or `robin_hood`/`ankerl::unordered_dense`. These are often **2–5× faster** than `std::unordered_map` precisely because they're contiguous (Lesson 1 again). In HFT, `std::unordered_map` is frequently swapped out for one of these.

### Typical use cases

Fast lookup by an unordered key where you never need ordering: mapping a symbol string or instrument ID to its record, deduplicating IDs, caching, session lookups. **Use `unordered_map` over `map` when you need keyed lookup and *don't* need order** — but reach for an open-addressing variant when it's hot.

---

## 8. The `multi` and other variants, briefly

For completeness, so the family is whole:

- **`std::multimap` / `std::multiset`** — like `map`/`set` but **allow duplicate keys** (several values under the same key). Same tree, same complexity. Use when one key legitimately maps to many values and you want them kept in key-order.
- **`std::unordered_multimap` / `unordered_multiset`** — the hash-table versions allowing duplicates.
- **Container adaptors** — `std::stack`, `std::queue`, and `std::priority_queue` aren't new structures; they're *interfaces* layered on top of the containers above (a `priority_queue` is a heap, usually over a `vector`, giving O(log n) insert and O(1) access-to-max — useful when you repeatedly need "the largest remaining"). `std::deque` is a double-ended queue (fast push/pop at *both* ends), built from chunks — more cache-friendly than `list` but less than `vector`.

These round out the toolbox, but the six core containers (§3–7) cover the overwhelming majority of decisions.

---

## 9. The complete comparison table

Here is the whole family on one page — the reference you'll come back to. "Cache" rates how friendly the layout is to the hardware (Lesson 1); for HFT it often matters as much as the complexity column.

| Container | Underlying structure | Layout / Cache | Lookup by key | Insert | Random access | Ordered? | Key superpower | Key limitation |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **`array`** | Fixed C array | Contiguous / ★★★ | O(n) scan, O(log n) if sorted | — (fixed size) | **O(1)** | insertion order | Zero overhead, no heap, no realloc | Size fixed at compile time |
| **`vector`** | Dynamic array | Contiguous / ★★★ | O(n) scan, O(log n) if sorted | **O(1)** at end, O(n) middle | **O(1)** | insertion order | The cache-friendly default | Middle insert shifts; realloc invalidates iterators |
| **`list`** | Doubly-linked list | Node-based / ★ | O(n) | **O(1)** with iterator | impossible | insertion order | O(1) splice; stable iterators | Cache-hostile; alloc per node |
| **`map` / `set`** | Red-black tree | Node-based / ★ | **O(log n)** | O(log n) | impossible | **sorted by key** | Ordered iteration + range queries | Can't sort by value; cache-hostile |
| **`unordered_map` / `set`** | Hash table (chaining) | Mostly node-based / ★★ | **O(1) avg**, O(n) worst | O(1) avg | impossible | none | Fastest average keyed lookup | No ordering; rehash spikes; pointer-chasing |

> **How to read this table for HFT:** the right two columns ("superpower" / "limitation") tell you *when* to use each; the "Layout / Cache" column tells you *how much it'll cost you per operation* on real hardware. The recurring HFT pattern: start from the semantically-correct container (this table), then — if it's hot — substitute the contiguous, cache-friendly equivalent (§12) that preserves the operations you need while fixing the layout.

---

## 10. The "impossible operations" gathered in one place

The curriculum specifically asks about what each container *can't* do. Scattered above, here they are together — the impossibilities are often what *forces* a container choice:

- **You cannot sort a `map`/`set` by value** — they're sorted by *key*, immutably. To order by value, copy into a `vector` and `std::sort`. (The single most-asked "how do I...?" with the answer "you can't, copy it out.")
- **You cannot change a key in place** in a `map`/`set` — keys are `const` (changing one would corrupt the sorted order). Erase and re-insert.
- **You cannot get ordered iteration or range queries from an `unordered_` container** — hashing destroys order. Use the ordered (tree) version or a sorted vector.
- **You cannot random-access (`[i]` by position) a `map`, `set`, or `list`** — no O(1) indexing; you must iterate. (For `map`, `m[k]` is keyed access, *not* positional — and on `map` it silently *inserts* if the key is missing.)
- **You cannot resize a `std::array`** — its size is fixed at compile time.
- **You cannot binary-search an *unsorted* `vector`** — binary search requires sorted data; on unsorted data you must linear-scan (O(n)).
- **You cannot safely hold a pointer/iterator into a `vector` across a `push_back`/reallocation** — it may move and invalidate them. (Node-based containers *can* — their elements never move; that stability is sometimes their only reason to exist.)

Each "impossible" is the flip side of a "possible": the map can't sort by value *because* it's busy keeping sorted by key; the hash map can't iterate in order *because* it's busy giving you O(1) lookup. **The limitations and the superpowers are the same design decision viewed from two sides** — which is exactly why "what operations do I need?" (§0) is the question that picks the container.

---

## 11. What to use when — the decision guide

Put it all together into a practical procedure. Ask these questions in order:

1. **Is the size fixed and known at compile time?** → **`std::array`.** (Stack, zero overhead, no allocation. The HFT favorite when it fits.)
2. **Do I mostly append and index/iterate, with few or no middle insertions?** → **`std::vector`.** (The default. Covers most cases. `reserve` it.)
3. **Do I need to look things up by a key?** Then:
   - **Do I need the keys kept in order** (ordered iteration, range queries, min/max, "closest key ≥ X")? → **`std::map`/`std::set`** *semantically* — but for small/medium or hot collections, a **sorted `std::vector` + `lower_bound`** (flat map, §12) is usually faster.
   - **Do I only need fast lookup, order irrelevant?** → **`std::unordered_map`/`unordered_set`** — and swap in an open-addressing map (§12) when it's hot.
4. **Do I need stable references that never invalidate, or O(1) splicing of sublists?** → the rare legitimate **`std::list`** case.
5. **Do I repeatedly need "the largest/smallest remaining"?** → **`std::priority_queue`** (a heap).
6. **Do I need fast push/pop at *both* ends?** → **`std::deque`.**

> **And then the HFT overlay on top of all of the above:** once you've picked the *semantically* correct container, ask "is this on the hot path?" If yes, prefer the **contiguous, pre-sized, allocation-free** version of whatever you chose (§12) — because on the hot path, cache behavior and the absence of latency spikes (Module 0's jitter) outrank everything else. The decision guide picks *correctness and clarity*; the HFT overlay picks *the cache-friendly realization of that choice.*

---

## 12. How to speed them up — the HFT toolbox

Gathering and extending the per-container tips into the techniques HFT actually uses. Every one of these is, at root, an application of Lesson 1 ("make it contiguous, make it predictable, never allocate on the hot path").

### Pre-size everything (`reserve`)

Call `reserve()` on every `vector` and `unordered_map` at startup, sized for the worst case. This eliminates the two big sources of latency *spikes* — `vector` reallocation and hash-map rehashing — converting "amortized O(1) with occasional O(n) hiccups" into "flat O(1) with no hiccups." Spikes are jitter; jitter loses races (Module 0). This is non-negotiable HFT hygiene.

### Prefer contiguous; replace node-based with flat equivalents

The headline technique. For each cache-hostile container, there's a contiguous replacement that keeps the operations you need:

- **`std::map`/`set` → "flat map/set": a sorted `std::vector` searched with `lower_bound`.** You keep O(log n) lookup, ordered iteration, and range queries — but contiguous and cache-friendly, often several times faster for the sizes HFT uses. The cost: insertion is O(n) (shifting), so this wins when lookups vastly outnumber insertions, or when you build-then-query. Available ready-made as `boost::flat_map`, and standardized as **`std::flat_map`/`std::flat_set` in C++23.**
- **`std::unordered_map`/`set` → open-addressing hash maps** (`absl::flat_hash_map`, `boost::unordered_flat_map`, `ankerl::unordered_dense`). Elements live in a contiguous array, not scattered nodes — far fewer cache misses, typically 2–5× faster.
- **`std::list` → `std::vector`** in nearly all cases (§5).

### Use a small integer key and index an array directly

The fastest "map" of all isn't a map. If your keys are dense small integers (e.g., instrument IDs 0…N), **just use a `std::array` or `std::vector` indexed by the key** — `table[instrument_id]`. That's a single O(1) cache-friendly load: no hashing, no tree, no pointer chase. HFT systems assign compact integer IDs to instruments precisely so they can do this. When you can turn a lookup into an array index, you've beaten every map.

### Control allocation: custom allocators and memory pools

When you *must* use a node-based container (or any container) in a latency-sensitive path, give it a **custom allocator** backed by a **pre-allocated memory pool** (Lesson 8) so that "allocate a node" becomes "grab the next slot from a pool I reserved at startup" — fast, predictable, and keeping the nodes close together in memory (recovering some cache locality). This tames the per-node allocation cost and scatter that make node-based containers slow.

### Avoid copies; construct in place

Use `emplace`/`emplace_back` to build elements directly inside the container rather than constructing a temporary and copying it in. Pass by reference, use move semantics (Lesson 8). Every avoided copy is data you didn't move (Lesson 1's golden rule).

### Choose cheap-to-compare, cheap-to-hash keys

Comparisons (tree containers) and hashing (hash containers) happen on every operation. A key that's expensive to compare or hash (a long string) makes every operation expensive; a small integer key makes them trivial. Prefer compact integer keys; intern strings to IDs.

> **The unifying principle:** every speed-up here is the same move — **push the data layout toward contiguous, push the work toward compile-time or startup, and push allocation out of the hot path entirely.** The standard containers are general-purpose; HFT specializes them (pre-sized, flat, pool-backed, integer-indexed) to fit the machine Lesson 1 described. You start from the clear, correct standard container and *earn* your way to the specialized one with measurement (Lesson 2's `perf` and benchmarking discipline) — never by premature guessing.

---

## 13. Recap

- A **container is a contract**: it makes some operations cheap by making others slow or impossible. Choose by asking **"what operations, how often?"** (§0)
- **Big-O** describes scaling, but **hides the constant factor and cache behavior** — which in HFT often matter more. A "worse" complexity with great cache behavior routinely beats a "better" one that chases pointers. Reason about class *and* cache, then **measure.** (§1)
- The great divide: **contiguous** (`array`, `vector` — cache-friendly, the machine's favorite) vs. **node-based** (`list`, `map`, `set`, hash chaining — a cache miss per hop, plus per-element allocation). This predicts real performance better than Big-O. (§2)
- **`vector`** is the default: O(1) index, amortized-O(1) append, O(n) middle-insert; `reserve` it to kill realloc spikes; beware iterator invalidation. (§3)
- **`array`**: fixed-size, stack, zero-overhead, no allocation ever — the HFT pick when the size is known. (§4)
- **`list`**: O(1) splice and stable iterators, but cache-hostile and allocation-heavy — **essentially banned in HFT hot paths**; `vector` usually wins even at list's "specialty." (§5)
- **`map`/`set`** (sorted tree): O(log n) keyed lookup, **ordered iteration and range queries** — but node-based and cache-hostile. **Can't sort by value, can't mutate keys, no random access.** The semantic order-book structure; often replaced by a flat map when hot. (§6)
- **`unordered_map`/`set`** (hash table): O(1)-average lookup but **no ordering at all**, worst-case O(n), rehash spikes, and hidden pointer-chasing — `reserve` it and prefer open-addressing variants when hot. (§7)
- The **impossibilities** (can't sort a map by value, can't order an unordered container, can't index a map/list, can't resize an array, can't binary-search unsorted data) are the flip side of each container's superpower, and often what *forces* the choice. (§10)
- **Decision guide:** fixed size → `array`; append/iterate → `vector`; keyed + ordered → `map`/flat-map; keyed + unordered → `unordered_map`/open-addressing; stable refs/splice → `list`. Then overlay the HFT rule: on the hot path, pick the **contiguous, pre-sized, allocation-free** realization. (§11)
- **Speed-ups** are all one idea — **contiguous layout, work moved to startup/compile-time, no hot-path allocation**: `reserve`, flat maps, open-addressing hash maps, integer-indexed arrays, pool allocators, `emplace`, cheap keys. (§12)

---

## 14. Glossary

| Term | Meaning |
| --- | --- |
| **Container** | A standard library type that stores a collection of elements (`vector`, `map`, ...). |
| **Big-O / complexity** | How an operation's cost grows with element count `n`: O(1) constant, O(log n) logarithmic, O(n) linear, O(n log n), O(n²). |
| **Amortized O(1)** | Almost every call is O(1), with rare O(n) exceptions, averaging to constant — but the rare case is a latency *spike*. |
| **Constant factor** | The actual per-operation cost Big-O hides; an O(1) hash lookup can be slower than an O(n) cache-friendly scan for small `n`. |
| **Contiguous container** | Elements packed in one memory block (`array`, `vector`) — cache-friendly, prefetcher-loved. |
| **Node-based container** | Each element in its own heap-allocated node linked by pointers (`list`, `map`, `set`) — a cache miss per hop, allocation per element. |
| **`std::array`** | Fixed-size, compile-time-sized, stack-allocated contiguous array; zero overhead, no heap, no resize. |
| **`std::vector`** | Dynamic contiguous array; O(1) index, amortized-O(1) append; the default container. |
| **Capacity vs. size** | Capacity = slots allocated; size = slots used. Appending past capacity triggers reallocation. |
| **`reserve`** | Pre-allocates capacity to avoid future reallocation/rehash spikes — essential HFT hygiene. |
| **Reallocation** | A `vector` outgrowing its block, allocating a bigger one and moving all elements (O(n)); **invalidates iterators/pointers**. |
| **Iterator invalidation** | When an operation makes existing iterators/pointers/references dangling (vector realloc, middle erase). |
| **`std::list`** | Doubly-linked list; O(1) splice and stable iterators, but cache-hostile — rarely the right choice. |
| **Associative container** | Stores data by key for lookup (`map`, `set`, `unordered_*`) rather than by position. |
| **`std::map` / `std::set`** | Sorted (red-black tree) key→value / key containers; O(log n) ops, ordered iteration, range queries. |
| **Red-black tree** | The self-balancing binary search tree behind `map`/`set`. |
| **`lower_bound` / `upper_bound`** | "First element ≥ k" / "first element > k" — O(log n) range-query primitives on sorted containers. |
| **`std::unordered_map` / `unordered_set`** | Hash-table key→value / key containers; O(1)-average lookup, no ordering. |
| **Hash function** | Turns a key into a bucket-selecting number; a good one spreads keys evenly to keep operations O(1). |
| **Load factor / rehash** | Elements-per-bucket ratio; crossing a threshold triggers an O(n) table rebuild (a spike). |
| **Open addressing** | A hash-table design storing elements in a contiguous array (no per-element nodes) — far more cache-friendly; e.g. `absl::flat_hash_map`. |
| **Flat map/set** | An ordered container implemented as a sorted `std::vector` (`boost::flat_map`, C++23 `std::flat_map`) — contiguous, cache-friendly. |
| **Memory pool / custom allocator** | Pre-allocated storage handed out cheaply, used to make container allocation fast and predictable (Lesson 8). |
| **`emplace` / `emplace_back`** | Construct an element directly inside the container, avoiding a temporary + copy. |

---

## 15. Where to go next

- **Lesson 8 — C++ for HFT.** The direct sequel: now that you know *what* the containers do, Lesson 8 shows how to wield them (and replace them) in genuinely fast code — data-oriented design (struct-of-arrays over array-of-structs), zero-allocation hot paths with the pool allocators mentioned here, `alignas` against false sharing, the lock-free SPSC ring buffer, and the templates/`constexpr` machinery that makes flat containers zero-overhead. This lesson is the *vocabulary*; Lesson 8 is the *composition*.
- **Lesson 9 — Industry Practices.** The **order book** — the central HFT data structure teased in §6 — gets built for real there, and you'll see exactly why production order books use flat/array-based structures over `std::map`, paying off this lesson's central lesson.
- **Back to Lesson 1.** If any of the "why is contiguous faster?" reasoning felt thin, re-read [Lesson 1 §§3–5](01-computer-architecture.md) (cache lines, the prefetcher, the linked-list trap) — this lesson is, fundamentally, that one applied to the STL.
- **Hands-on milestone (from the README):** benchmark a `std::map<int,int>` against a sorted `std::vector` (or `boost::flat_map`) for lookups, at sizes 10, 100, 1,000, 100,000. Plot the times. Find the crossover where the tree's O(log n) finally beats the vector's cache-friendliness — and watch the cache-miss counts in `perf stat` explain the whole curve. Feeling that crossover in your own measurements makes §1's "distrust Big-O" lesson permanent.

> The through-line: **the standard containers are the right place to start and the wrong place to stop.** Know them cold — their operations, their costs, their impossibilities — and you'll choose correctly by default; understand the cache reality beneath them (Lesson 1), and you'll know exactly when and how to specialize them for the nanosecond game.
