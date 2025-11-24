# **1) Draw prototype chains for 5 custom object setups (conceptual, no code)**

Draw these on paper as vertical chains (top = parent prototype).

---

### **Prototype Chain 1 — Simple Parent → Child**

- `childObj`
- → `parentObj`
- → `Object.prototype`
- → `null`

**Notes:** Child inherits all properties parent defines. If child accesses something it doesn’t have, lookup climbs the chain.

---

### **Prototype Chain 2 — Object created with `Object.create`**

- `user`
- → prototype: `{ role: "admin" }`
- → `Object.prototype`
- → `null`

**Notes:** `user` has no own properties. It inherits `role` from its prototype object.

---

### **Prototype Chain 3 — “Class-like” Constructor Setup**

- `instance`
- → `Constructor.prototype`
- → `Object.prototype`
- → `null`

**Notes:** Constructor’s prototype holds shared methods. Each instance points to that prototype.

---

### **Prototype Chain 4 — Multi-level “class” inheritance**

- `student`
- → `Person.prototype`
- → `BaseObject.prototype`
- → `Object.prototype`
- → `null`

**Notes:** JS simulates multi-level inheritance just by linking prototypes in a chain.

---

### **Prototype Chain 5 — Function object’s dual prototype chain**

For a **function** itself:

- `someFunction`
- → `Function.prototype`
- → `Object.prototype`
- → `null`

For instances created by that function:

- `instance`
- → `someFunction.prototype`
- → `Object.prototype`
- → `null`

Two different chains depending on who you’re talking about: **functions** have their own prototype chain separate from **instances**.

---

# **2) Step-by-step: how JS finds a method on an object**

Write this in your notes and explain it aloud:

1. **Start at the object itself.**
   JS checks if the property exists as an "own" property.

2. **If missing, go to the object’s prototype (`[[Prototype]]`).**
   JS now checks that prototype object.

3. **Continue walking up the chain.**
   Parent → grandparent → etc.

4. **Stop when you reach `null`.**
   If still not found, return `undefined`.

5. **When the found value is a function, JS binds `this` based on how you call it.**
   The lookup does **not** bind `this`. The **call site** does.

This is a **pure pointer crawl**, not magic.
If you don’t find the property before hitting `null` → it simply doesn’t exist.

---

# **3) Ten “this binding” scenarios — predict the value each time**

These are conceptual — you must reason through them.

---

### **Scenario 1 — Simple object method**

Call: `obj.method()`.
**this = obj**
Because it’s a method call: “base object” wins.

---

### **Scenario 2 — Method extracted**

`const fn = obj.method; fn();`
**this = undefined (strict mode) / global (non-strict)**
Losing the base object loses method context.

---

### **Scenario 3 — Callback in setTimeout**

`setTimeout(obj.method, 0)`
**this = undefined / global**
Timers don’t preserve base objects.

---

### **Scenario 4 — Call with `.call()`**

`obj.method.call(x)`
**this = x**
`.call()` manually sets `this`.

---

### **Scenario 5 — Arrow function inside method**

`obj.method()` contains an arrow callback using `this`.
**this = obj (outer scope) for arrow function**
Arrows don’t bind `this`; they _capture lexical this_.

---

### **Scenario 6 — Arrow function used as method**

`obj = { f: () => console.log(this) }`
Call `obj.f()`.
**this = outer scope this (NOT obj)**
Arrow ignores “base-object rules”.

---

### **Scenario 7 — Constructor call**

Called with `new Person()`
**this = newly-created instance**
Constructor mode always overrides.

---

### **Scenario 8 — Event listener**

`element.onclick = handler`
Call: event fired
**this = element**
DOM rules bind `this` to the receiving element.

---

### **Scenario 9 — Bound function**

`const g = obj.method.bind(y)`
Call g:
**this = y**
Binding is permanent; base object irrelevant.

---

### **Scenario 10 — Chained access**

`a.b.c.fn()`
**this = the object before `.fn` → c**
Only the last dot matters.

---

# **4) Compare function call styles and how they change behavior**

### **Direct call:** `fn()`

- `this` = undefined/global
- normal invocation

### **Method call:** `obj.fn()`

- `this = obj`
- determined by call site

### **Call/Apply:** `fn.call(x)` / `fn.apply(x)`

- explicitly set this
- call/apply differ only in argument format

### **Bind:** `fn.bind(x)`

- creates a _new_ function whose `this` is permanently locked
- cannot be overridden

### **Constructor call:** `new fn()`

- creates new object
- sets `this` to new instance
- completely ignores bound values of `this` for the constructor body (tiny nuance: bound args still apply, but bound this is ignored)

### **Arrow function call:**

- `this` comes from lexical scope, not call site
- so call style doesn’t matter for arrow functions

Use this list to force yourself to think:
**“What call style is being used?” → This tells you what `this` becomes.**

---

# **5) Conceptual model: how JS simulates “classes”**

JS does **not** have native class-based inheritance.
Even the `class` keyword is syntactic sugar over prototypes.

Here’s the reality:

1. **Constructor function defines instance initialization.**
   When you call with `new`, JS:

   - creates an empty object
   - links its prototype to `Constructor.prototype`
   - binds `this` inside constructor to the new object
   - returns the object (unless constructor returns its own object)

2. **Shared behavior lives on the prototype.**
   `Constructor.prototype` contains methods shared by all instances.

3. **Inheritance is prototype chaining.**
   If you want “class inheritance”, JS sets:

   - `Child.prototype` → an object that inherits from `Parent.prototype`
   - `Child.prototype.constructor = Child`

4. **Method lookup is prototype traversal.**
   When you call instance.method(), JS looks up the property walking the chain until found.

5. **Class fields are syntactic sugar for assigning properties to this.**
   `class A { x = 10 }`
   → inside constructor, JS sets `this.x = 10`.

6. **Super is syntactic sugar for calling parent constructor/prototype methods.**
   But underlying mechanism is still:

   - lexical reference to parent prototype
   - property lookup on prototype chain
   - `call` used to bind `this`

### **Core truth:**

JS “classes” are _just functions + prototype linkage_.
If you don’t understand prototypes, you don’t understand classes.

---
