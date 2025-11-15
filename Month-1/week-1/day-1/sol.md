## 25 short async-prediction examples (logic only — no code)

For each example I:

1. Describe the lines in order (what each line does).
2. **Prediction:** exact runtime order of visible effects (e.g., logs).
3. **Why:** explanation of the ordering — one or two crisp reasons.

### Example 1

1. Synchronous: print "A".
2. Schedule macrotask: setTimeout to print "C".
3. Schedule microtask: promise then to print "B".
4. Synchronous: print "D".
   **Prediction:** A, D, B, C.
   **Why:** Synchronous lines run immediately. Microtasks run after current sync completes but before macrotasks, so promise then prints B before the macrotask C.

### Example 2

1. Synchronous: print "start".
2. Schedule two microtasks: promise then to print "m1", another promise then to print "m2".
3. Synchronous: print "end".
   **Prediction:** start, end, m1, m2.
   **Why:** Microtasks queue FIFO and run after sync finishes; order of scheduling determines order.

### Example 3

1. Synchronous: print "0".
2. Schedule macrotask A (setTimeout) to print "1".
3. Schedule macrotask B to print "2".
4. Synchronous: print "3".
   **Prediction:** 0, 3, 1, 2.
   **Why:** Macrotasks run after microtasks and next tick of the loop; macrotasks maintain scheduling order.

### Example 4

1. Synchronous: print "alpha".
2. Schedule microtask that prints "µ" and inside it schedules macrotask to print "M".
3. Synchronous: print "omega".
   **Prediction:** alpha, omega, µ, M.
   **Why:** Microtask runs before macrotasks. A macrotask scheduled inside microtask is queued to next macrotask turn.

### Example 5

1. Synchronous: print "S".
2. Async function called (returns a promise); inside it, immediately print "in-async", then await a resolved promise, then print "after-await".
3. Synchronous: print "E".
   **Prediction:** S, in-async, E, after-await.
   **Why:** The body until the first await runs synchronously. After await resumes as a microtask, so it runs after the current sync completes.

### Example 6

1. Synchronous: print "1".
2. Microtask: promise then prints "2", and inside that then schedules another microtask that prints "2.1".
3. Synchronous: print "3".
   **Prediction:** 1, 3, 2, 2.1.
   **Why:** Microtasks run after sync; newly scheduled microtasks during microtask processing are appended and run before moving to macrotasks.

### Example 7

1. Synchronous: print "A".
2. Schedule macrotask via setTimeout to print "B".
3. Schedule a microtask that prints "C" and schedules another microtask "C2".
4. Synchronous: print "D".
   **Prediction:** A, D, C, C2, B.
   **Why:** All microtasks (including those created inside other microtasks) finish before macrotasks.

### Example 8

1. Synchronous: print "X".
2. Schedule macrotask to do heavy I/O callback print "I/O".
3. Schedule microtask print "µ1".
4. Synchronous: print "Y".
   **Prediction:** X, Y, µ1, I/O.
   **Why:** Microtasks finish before macrotasks/I/O callbacks.

### Example 9

1. Synchronous: print "start".
2. Schedule requestAnimationFrame callback that prints "raf".
3. Schedule setTimeout callback print "timeout".
4. Schedule microtask print "micro".
   **Prediction (browser):** start, micro, raf, timeout (raf runs before next paint, typically before setTimeout macrotask).
   **Why:** Microtasks run immediately after sync. requestAnimationFrame callbacks align with the next paint step which usually occurs before delayed macrotasks like setTimeout.

### Example 10

1. Synchronous: print "a".
2. Microtask that prints "b" and schedules a macrotask "c".
3. macrotask that prints "d".
4. Synchronous: print "e".
   **Prediction:** a, e, b, c, d (assuming 'macrotask that prints d' is scheduled earlier than c but both are macrotasks — ordering between c and d depends on scheduling time; if d was scheduled earlier, order is c after b then d).
   **Why:** Microtasks execute first; macrotasks retain the order they were queued.

### Example 11

1. Synchronous: print "0".
2. queueMicrotask schedules print "1".
3. setTimeout schedules print "2".
4. Synchronous: print "3".
   **Prediction:** 0, 3, 1, 2.
   **Why:** queueMicrotask is a microtask; it runs before macrotasks.

### Example 12

1. Synchronous: log "sync".
2. Promise.then that throws an error (uncaught) and then catches it in another microtask chain printing "caught".
3. Synchronous: log "done".
   **Prediction:** sync, done, caught, then error handling side-effects.
   **Why:** Promise rejections and catches are processed in microtasks; sync completes first.

### Example 13

1. Synchronous: increment counter and print "c1".
2. setImmediate (Node) scheduled to print "immediate".
3. process.nextTick scheduled to print "next".
4. Synchronous: print "c2".
   **Prediction (Node):** c1, c2, next, immediate.
   **Why:** In Node, process.nextTick runs before microtasks and before other microtasks queue; setImmediate is a macrotask stage that runs later.

### Example 14

1. Synchronous: print "main".
2. schedule microtask that prints "µ" and inside it schedules nextTick to print "nt".
3. Synchronous: print "end".
   **Prediction (Node):** main, end, µ, nt.
   **Why:** nextTick queued inside microtask typically runs before returning to event loop macrotask stages (Node's nextTick runs before other microtasks that were scheduled earlier? Practically nextTick runs immediately after current operation but before the next microtask tick — treat it as highest priority in Node).

### Example 15

1. Synchronous: register two event handlers for clicks; handler A does synchronous log "A"; handler B schedules a microtask to log "B".
2. User triggers click.
   **Prediction:** A runs, then B microtask runs. Visible order: A, B.
   **Why:** Event handlers run as part of a macrotask; synchronous parts run immediately, microtasks scheduled within handler run after that handler completes but before next macrotask.

### Example 16

1. Synchronous: print "start".
2. Schedule Promise.resolve().then(() => schedule setTimeout to print "timeout from micro").
3. Schedule setTimeout to print "timeout outer".
4. Synchronous: print "end".
   **Prediction:** start, end, micro runs (schedules timeout from micro), timeout outer, timeout from micro (order of timeouts depends on scheduling timing; but generally the outer timeout was scheduled earlier, so it fires first).
   **Why:** Microtask runs and queues a macrotask; macrotasks run in queue order they were scheduled.

### Example 17

1. Synchronous: print "begin".
2. Async function A awaits a promise that resolves later; async function B is called and immediately awaits; both are pending.
3. Synchronous: print "middle".
4. Promise resolves and microtasks run to resume awaits.
   **Prediction:** begin, middle, (then resumes in the order the promises resolved — if both resolve same tick, the one scheduled first resumes first).
   **Why:** await resumes in microtasks; order depends on which promise settles first and scheduling order.

### Example 18

1. Synchronous: for loop that schedules 3 microtasks printing 1,2,3.
2. Synchronous: print "after loop".
   **Prediction:** after loop, 1,2,3.
   **Why:** Microtasks are processed after synchronous code finishes; they preserve the scheduling order.

### Example 19

1. Synchronous: print "A".
2. Microtask that prints "B" and then schedules another microtask to print "C".
3. Synchronous: print "D".
   **Prediction:** A, D, B, C.
   **Why:** Nested microtasks queue up and are processed before macrotasks.

### Example 20

1. Synchronous: print "sync1".
2. Schedule macrotask that prints "mac1" and inside it schedules a microtask printing "insideMicro".
3. Synchronous: print "sync2".
   **Prediction:** sync1, sync2, mac1, insideMicro.
   **Why:** Microtask scheduled inside a macrotask runs after that macrotask's synchronous part finishes but before the next macrotask.

### Example 21

1. Synchronous: print "A".
2. Microtask schedules microtask2 and macrotask M.
3. microtask2 schedules microtask3.
4. Synchronous: print "Z".
   **Prediction:** A, Z, microtask, microtask2, microtask3, M.
   **Why:** Microtasks are processed fully; any microtask-scheduled microtasks are appended and processed within the microtask queue before macrotasks.

### Example 22

1. Synchronous: print "top".
2. Promise.then that prints "p1" and in same then schedules a promise that resolves and its then prints "p2".
3. Synchronous: print "bottom".
   **Prediction:** top, bottom, p1, p2.
   **Why:** then callbacks are microtasks; p2 is scheduled within p1 and runs after p1 finishes.

### Example 23

1. Synchronous: print "start".
2. setTimeout callback schedules a microtask to print "m-from-timeout".
3. Another setTimeout scheduled earlier prints "early-timeout".
   **Prediction:** start, early-timeout, then m-from-timeout (the microtask scheduled inside the macrotask runs immediately after that macrotask finishes).
   **Why:** Microtasks scheduled within a macrotask run before the next macrotask, but only after that macrotask's sync finishes.

### Example 24

1. Synchronous: print "s1".
2. Microtask A prints "mA" and throws error not caught; next microtask B prints "mB".
3. Synchronous: print "s2".
   **Prediction:** s1, s2, mA (error handling may interrupt stack), mB (depending on whether error crashed the microtask queue; typically unhandled rejection reported but other microtasks still run in many environments).
   **Why:** Microtasks still run in queue order; unhandled errors cause side effects but don’t necessarily stop queue progress in all engines — don’t rely on engine-specific behavior.

### Example 25

1. Synchronous: print "go".
2. schedule multiple macrotasks and microtasks interleaved: micro, mac, micro, mac.
3. Synchronous: print "done".
   **Prediction:** go, done, micro1, micro2, mac1, mac2 (microtasks grouped and executed before any macrotasks).
   **Why:** Microtasks are fully drained before any macrotask stage proceeds.

---

## 10 microtask vs macrotask conflict scenarios (concentrated)

Each contains the scenario, **prediction first**, then short explanation.

### Scenario 1 — Microtask scheduled inside macrotask

- Macrotask M runs; during M it schedules microtask µ.
  **Prediction:** M runs, then µ runs (before next macrotask).
  **Why:** Microtasks scheduled during a macrotask execute immediately after that macrotask completes, before the event loop moves on.

### Scenario 2 — Macrotask scheduled inside microtask

- Microtask µ runs and schedules macrotask M.
  **Prediction:** µ finishes (and any new microtasks it schedules), then M runs later in macrotask phase.
  **Why:** Microtasks are drained first; macrotasks execute afterward.

### Scenario 3 — Microtask scheduling another microtask

- µ1 schedules µ2.
  **Prediction:** µ1, µ2.
  **Why:** µ2 is appended to microtask queue and runs in same microtask draining sequence.

### Scenario 4 — Many macrotasks vs one microtask

- Several macrotasks are queued; a microtask appears before they execute.
  **Prediction:** microtask executes first, then macrotasks in order.
  **Why:** Microtasks have priority immediately after the current sync execution.

### Scenario 5 — NextTick (Node) vs Promise microtasks

- process.nextTick scheduled and a promise.then scheduled.
  **Prediction (Node):** nextTick runs before promise microtasks.
  **Why:** nextTick has higher priority in Node's scheduling model.

### Scenario 6 — requestAnimationFrame vs setTimeout

- RAF and setTimeout are both scheduled after sync.
  **Prediction (browser):** microtasks run first, then RAF at paint, then setTimeout macrotasks (setTimeout may run after RAF).
  **Why:** RAF aligns with paint cycle and often runs before delayed macrotasks.

### Scenario 7 — I/O callback vs microtask

- An I/O completion callback (macrotask) and microtask are both ready.
  **Prediction:** microtask runs before the I/O callback.
  **Why:** Microtasks are drained between macrotask turns, before entering I/O callback handling.

### Scenario 8 — setImmediate vs setTimeout (Node)

- Both scheduled; which runs first depends on environment and timing.
  **Prediction (Node typical):** setTimeout(0) often runs before setImmediate when called from top-level, but order is not guaranteed — don’t rely on it.
  **Why:** Node’s timers and check phases are separate; order varies by when the events were queued.

### Scenario 9 — microtask queue starvation risk

- Microtasks keep scheduling new microtasks in their callbacks, making macrotasks never execute.
  **Prediction:** microtasks keep running; macrotasks are starved until microtask queue is empty (which may never happen if code keeps appending).
  **Why:** Microtask draining continues until empty — don’t create infinite microtask loops.

### Scenario 10 — nested macrotask scheduling

- Macrotask M1 schedules M2; M2 schedules µ.
  **Prediction:** M1, M2, µ (µ runs after M2 finishes).
  **Why:** µ scheduled inside M2 won't run until that macrotask's synchronous part completes; then microtasks run.

---

## Rules summarizing event loop behavior (concise)

1. **Synchronous code runs first** — always.
2. **Microtasks (Promise.then, queueMicrotask, MutationObserver)** run immediately after the current synchronous code finishes and before any macrotask.
3. **Microtasks are FIFO** and newly queued microtasks during draining are appended and also executed in the same draining loop.
4. **Macrotasks (setTimeout, setInterval, setImmediate, I/O callbacks, UI events)** execute after microtasks are drained.
5. **Node has extra phases** (process.nextTick, setImmediate) with `nextTick` running at very high priority (before promise microtasks).
6. **requestAnimationFrame** runs tied to paint frames; it’s not interchangeable with setTimeout.
7. **Errors inside async microtasks** may be surfaced as unhandled rejections — don’t assume they stop the queue uniformly across engines.
8. **Scheduling inside microtasks vs macrotasks affects timing:** microtask → runs before macrotasks; macrotask → microtasks scheduled inside it run before next macrotask.
9. **Avoid microtask starvation** — if microtasks keep scheduling microtasks, macrotasks won’t run.
10. **Don’t rely on timer ordering** (e.g., setTimeout vs setImmediate) — behavior can vary by environment and timing.

---

## Common mistakes developers make with async (and how that bites them)

1. **Copy-pasting async code without understanding microtasks vs macrotasks.** Result: tests pass locally but production timing breaks.
2. **Using `await` inside loops without thinking about concurrency.** Result: serializes work unnecessarily.
3. **Assuming setTimeout(0) runs immediately.** It’s a macrotask — it runs later than microtasks.
4. **Creating infinite microtask chains** (scheduling microtasks from microtasks endlessly). Result: UI freeze or starvation.
5. **Not handling promise rejections.** Unhandled rejections cause unpredictable behavior and noisy errors.
6. **Relying on order of timers across environments (setImmediate vs setTimeout).** Non-portable assumptions.
7. **Expecting `async` function bodies to be fully synchronous.** First part runs sync until first await; resume is async (microtask).
8. **Mixing heavy synchronous work with async callbacks** — blocks the event loop and negates benefits of async.
9. **Using Promises for sequencing without catching errors** — chain breaks silently.
10. **Debugging by sprinkling logs without understanding event loop phases** — makes race conditions harder to see.

---

## Journaling exercise — explain the event loop to a junior

> Think of JavaScript like a single cook in a kitchen. The cook has an order pad (call it the _call stack_) and two trays for delayed tasks: the **microtask tray** (fast lane) and the **macrotask tray** (regular lane).
>
> 1. The cook handles immediate orders first — anything you call directly (function calls, synchronous code).
> 2. While cooking, the cook can jot tasks onto the microtask tray or the macrotask tray. Microtasks are tiny cleanup tasks (like finishing a sauce). Macrotasks are larger jobs (like baking something in the oven).
> 3. When the cook finishes the current immediate work, they always check and clear the **microtask tray** fully before touching the macrotask tray. If new microtasks are added while cleaning microtasks, the cook keeps cleaning until it's empty.
> 4. Only after the microtask tray is empty does the cook pull the next macrotask and start it. Some environments (Node) have special secret shortcut trays (like `nextTick`) that get serviced even before microtasks. Browser paint/animation frames are scheduled differently and happen around the painting step.
>
> **Practical tips to teach a junior:**
>
> - Always run the simple experiments: add logs for sync, microtask, macrotask and predict the order. Then run and compare.
> - When debugging weird timing, ask: “Is this scheduled as a microtask or a macrotask?” That question often points to the bug.
> - Avoid long synchronous code; it blocks everything else. Break it into chunks if needed.
> - Don’t chain microtasks indefinitely. If you need repeated work, use a macrotask or a controlled loop that yields.
> - Treat `await` as “pause this function and resume later in a microtask” — not as an immediate synchronous pause.
>
> **One-line summary for a junior:** Sync first; microtasks next (finish them all); then macrotasks; repeat. If behavior looks wrong, you probably scheduled the job on the wrong tray.
