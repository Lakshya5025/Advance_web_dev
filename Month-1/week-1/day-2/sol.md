# 1) Ten closure examples (conceptual descriptions — **no code**)

For each: short scenario, what the closure captures, and why.

1. **Counter factory**

   - Scenario: A function `makeCounter` returns an inner function that increments and returns a `count` variable.
   - Retains: `count` (primitive number).
   - Why: The returned function needs `count` between calls, so the environment keeps it alive.

2. **Configured logger**

   - Scenario: A `createLogger(level)` returns `log(msg)` that prefixes messages with `level`.
   - Retains: `level` (string) and possibly a `prefix` object.
   - Why: `log` must remember the configured `level` after `createLogger` returns.

3. **Delayed callback capturing heavy object**

   - Scenario: A function prepares a large `data` object, then returns or schedules a callback that accesses `data` later.
   - Retains: the entire `data` object (large memory).
   - Why: Callback references `data`, so GC can't drop it until callback is unreachable or cleared.

4. **Event handler closing over DOM node**

   - Scenario: Attach a click handler that closes over a DOM node `node` and some metadata.
   - Retains: `node` and metadata.
   - Why: The handler references them; if handler stays registered, both remain reachable.

5. **Factory with private state (map of items)**

   - Scenario: `createStore()` keeps a `Map` as private storage and returns methods `get/set`.
   - Retains: the `Map` and whatever values are in it.
   - Why: Methods need the map; it's kept alive as long as any method exists.

6. **Iterator with index**

   - Scenario: A generator-like factory returns a `next()` closure that reads/modifies an `index` and `items` array.
   - Retains: `index`, `items` (array reference).
   - Why: Next() uses both to produce successive results.

7. **Memoizer caching results**

   - Scenario: `memoize(fn)` returns a wrapper that stores results in a `cache` object closed over by the wrapper.
   - Retains: `cache` and keys/values stored.
   - Why: Cache must persist across wrapper invocations.

8. **Partial application capturing arguments**

   - Scenario: `partial(fn, a, b)` returns a function that uses saved `a` and `b` later with more args.
   - Retains: `a` and `b`.
   - Why: The partially-applied function needs them to call `fn` later.

9. **Subscription manager with cleanup token**

   - Scenario: `subscribe()` returns an `unsubscribe()` closure that remembers a subscription `id` and registry reference.
   - Retains: `id` and reference to `registry` so it can remove itself.
   - Why: To operate later, unsubscribe must know what to remove.

10. **Lazy initializer**

    - Scenario: `getConfig()` returns a function that on first call computes a heavy `config` and on later calls returns cached config.
    - Retains: `config` after first computation.
    - Why: The closure must keep the computed result for subsequent calls.

---

# 2) What each closure retains — short summary (one line each)

1. Counter → retains `count`.
2. Logger → retains `level`/`prefix`.
3. Delayed callback → retains `data` (potentially large).
4. Event handler → retains `node` + metadata.
5. Store → retains `Map` and its entries.
6. Iterator → retains `index` and `items`.
7. Memoizer → retains `cache` (all cached values).
8. Partial → retains saved arguments.
9. Subscription → retains `id` and `registry`.
10. Lazy init → retains `config` after computed.

---

# 3) Concrete memory-leak example (conceptual, realistic)

**Scenario:** A single-page app creates view components dynamically. Each component attaches a global event listener that references the component’s DOM, its data model (a large object), and a logging function that closes over the data. When the component is removed from the DOM, the developer forgets to remove the event listener.

**What’s kept alive:**

- The event listener function is still reachable from the platform’s event registry.
- Because the listener closes over the DOM node and the large data model, those objects remain reachable and cannot be GC'd.
- Over many component mounts/unmounts, memory grows until OOM.

**Why it leaks:** The listener is a root-reachable reference (through event registry). Closure ties heavy objects to that root, preventing collection.

---

# 4) How garbage collection handles — and fails to handle — these situations

- **GC principle:** Modern JS engines use reachability from roots (global objects, active call stacks, event registries, closures reachable from currently-executing functions) to decide liveness. If an object is unreachable, it’s eligible for collection.
- **Closures create reachability:** If a closure is reachable, everything it references is reachable transitively, so GC won’t free them.
- **Event listeners & external registries are roots:** If you attach a listener and never remove it, any closure saved by that listener remains a root.
- **Circular references aren’t the problem:** JS GC handles cycles fine; the real issue is _reachability_ from roots, not cycles between objects.
- **Real failure modes:**

  - Forgotten listeners and timers (roots hold closures).
  - Long-lived caches and maps storing large values without eviction.
  - Global singletons capturing large state via closure.

- **Engine specifics:** Some engines may delay finalization or hold onto objects for additional heuristics (e.g., generational GC, conservative stack scanning) — meaning memory may be reclaimed later, not immediately. Don’t depend on that.

---

# 5) Five scope-related confusion scenarios (what devs get wrong) + fixes

1. **Loop variable closure trap**

   - **Problem:** Creating callbacks in a `for` loop that all reference the same loop variable (classic async in loop bug). All callbacks see the final value.
   - **Fix conceptually:** Create a fresh binding per iteration (use block-scoped variable or a per-iteration closure) so every callback closes over its own copy.

2. **Accidentally capturing whole object instead of a field**

   - **Problem:** You close over `user` object but only needed `user.id`. You keep the entire user (with photos, caches) alive.
   - **Fix:** Close over only the primitive you need (`const id = user.id`) so closure keeps minimal data.

3. **Keeping DOM nodes alive via metadata objects**

   - **Problem:** You attach metadata object to an element and then close over that metadata in global handlers; after removing element you forget to cleanup.
   - **Fix:** Remove handlers and null out references, or use WeakMap to associate metadata with nodes (so metadata is collectible when node is gone).

4. **Long-lived timers keeping state**

   - **Problem:** A repeating timer closes over a big state structure; developer forgets `clearInterval`. State lives forever.
   - **Fix:** Use single-shot timers or make sure to clear timers on teardown; prefer scheduling via a manager that can cancel.

5. **Unbounded caches / memoization**

   - **Problem:** Memoizer stores results for every input indefinitely (memory grows).
   - **Fix:** Use bounded caches (LRU), WeakMap for object keys, or explicit eviction policies.

---

# 6) Conceptual fixes for memory issues (practical list)

- **Remove event listeners on teardown** (component unmount): the canonical, first-line defense.
- **Null out heavy closed-over references** when you know the closure isn’t needed: overwrite closure-held variables (or expose a `dispose()` that does it).
- **Use WeakMap / WeakRef** for caches or associations where you don’t want the cache to be a keeper of liveness for keys.
- **Prefer primitive copies** over storing large objects in closures when only small parts are needed.
- **Bound caches** (LRU) or time-to-live: avoid unbounded memory growth.
- **Centralize lifecycle management** (single manager that unsubscribes and clears timers for a removed component).
- **Avoid global singletons** that close over app state; consider dependency injection and explicit lifetimes.
- **Audit long-lived closures**: Code review checklist — “Does this closure live beyond the UI lifetime? If yes, what does it keep?”
- **Testing:** Simulate mount/unmount cycles and check process memory or heap snapshots.

---

# 7) Closure vs block scope — documented behaviors & pitfalls (compact)

- **Closure**

  - Created when an inner function references variables from an outer lexical environment.
  - The inner function _retains_ references to those outer variables for as long as the inner function is reachable.
  - Closures can preserve state across multiple calls and are function-level objects with persistent environments.

- **Block scope (`let` / `const`)**

  - `let` and `const` create scope limited to the nearest `{}` block (for loops, `if`, etc.).
  - Each iteration of `for (let i ...)` creates a new binding for `i` — this is why `let` avoids the classic loop-closure problem that `var` has.
  - Block-scoped variables can still be closed over — block scope just controls lifetime and visibility rules, not closure semantics.

- **Var (function scope) differences**

  - `var` is function-scoped and hoisted; a loop with `var i` uses one binding for all iterations — causes closures created in the loop to share the same binding.
  - Use `let` to prevent shared-binding traps.

- **Temporal Dead Zone (TDZ)**

  - `let`/`const` variables are not accessible before initialization (TDZ). Closures created before initialization can’t access them until defined.

- **Practical rule:** Use `let`/`const` for per-iteration fresh bindings. Use closures when you intentionally want preserved private state. If you accidentally preserve state, that’s on you — fix it by narrowing what you capture or shortening lifetime.

---

# 8) Quick checklist to prevent leaks when using closures (do this from day 1)

1. For every long-lived function returned or passed to a global registry, ask: _what does this close over?_
2. If you find large objects closed over, either: copy only needed primitives, or change design.
3. On component teardown, remove subscriptions, listeners, and timers.
4. Use WeakMap/WeakRef for caches tied to objects that should be collectible.
5. Test: create mounts/unmounts cycle and inspect heap diff — if memory grows, inspect closures.
6. Code review rule: “no closure should keep a whole DOM subtree or huge data blob unless intentionally required and cleaned.”

---

# 9) Short journaling prompt (explain to a junior — write and speak)

Write 3 short paragraphs and then say them out loud:

1. **Paragraph 1 — what a closure is:** "A closure happens when a function keeps access to variables from its surrounding lexical scope after that outer function has returned. It’s how functions can have private, persistent state."

2. **Paragraph 2 — why closures cause leaks:** "If a closure remains reachable — for example as an event handler or a stored reference — everything it points to stays reachable too. That means large objects or DOM nodes can’t be garbage-collected until the closure is gone."

3. **Paragraph 3 — what to do about it:** "Always remove listeners and timers, avoid closing over huge objects when you only need small values, prefer WeakMap for caches, and provide explicit cleanup hooks for long-lived closures."

Say each paragraph aloud and record your time. If any paragraph hangs for more than 30s while you search for words — you haven’t internalized the idea yet. Repeat until you can explain each clearly in under 20s.

---

# 10) Final brutal advice

Closures are powerful — they’re how you build private state and encapsulation. But power comes with responsibility: every closure you create ties memory to the closure’s lifetime. If you don’t manage lifetime (listeners, timers, caches), your app becomes a slow memory sink. Stop assuming GC will “figure it out.” Design lifetimes explicitly.
