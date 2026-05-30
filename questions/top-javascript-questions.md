<p align="center">
  <img src="../assets/javascript-logo.png" alt="JavaScript logo" width="120" />
</p>

# Top 12 Most Difficult JavaScript Questions

## Contents

1. [Stale closure & primitive capture](#1-stale-closure--primitive-capture)
2. [JavaScript context trap: losing `this`](#2-javascript-context-trap-losing-this)
3. [Illusion of an "own" property on mutation](#3-illusion-of-an-own-property-on-mutation)
4. [JavaScript coercion & precedence trap](#4-javascript-coercion--precedence-trap)
5. [JavaScript event loop & queue priority trap](#5-javascript-event-loop--queue-priority-trap)
6. [Microtask order: `async/await` vs promise chains](#6-microtask-order-asyncawait-vs-promise-chains)
7. [JavaScript string length trap: UTF-16 & surrogate pairs](#7-javascript-string-length-trap-utf-16--surrogate-pairs)
8. [JavaScript `super` trap: arrow functions & lexical binding](#8-javascript-super-trap-arrow-functions--lexical-binding)
9. [JavaScript prototype trap: `prototype` vs `__proto__`](#9-javascript-prototype-trap-prototype-vs-__proto__)
10. [JavaScript `.call.call()` trap: the double call trick](#10-javascript-callcall-trap-the-double-call-trick)
11. [JavaScript private fields trap: `#fields` + `Proxy` incompatibility](#11-javascript-private-fields-trap-fields--proxy-incompatibility)
12. [JavaScript memory leak trap: shared closure contexts](#12-javascript-memory-leak-trap-shared-closure-contexts)

---

## 1. Stale closure & primitive capture

What is the output of the following code?

```javascript
function createIncrement() {
  let count = 0;
  const message = `Count is ${count}`;

  function increment() {
    count++;
  }

  function log() {
    console.log(message);
  }

  return { increment, log };
}

const { increment, log } = createIncrement();
increment();
increment();
log();
```

> Test your understanding of closures, lexical scope, and primitive value capture.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
Count is 0
```

### 🧠 Explanation

This is a classic stale closure trap — but not in the way most developers expect.

**Step-by-step execution:**

1. `createIncrement()` is invoked → new lexical environment created:
   - `count = 0` (mutable binding)
   - `message = "Count is 0"` (primitive string, interpolated immediately at assignment)
2. Inner functions `increment` and `log` are defined. Both close over the same lexical environment.
3. `increment()` is called twice:
   - `count` mutates: `0 → 1 → 2` ✓
   - This works as expected.
4. `log()` is called:
   - It references the variable `message`
   - `message` still holds the original string value `"Count is 0"`
   - The template literal was evaluated once, at the moment of assignment — not re-evaluated when `log()` runs.

### 🔑 Core Concept

> Closures capture **variables**, not **expressions**.
> But if a variable holds a primitive value (string, number, boolean), that value is fixed at assignment time.

`message` is not a live reference to `count`. It is a **snapshot**.

### 🛠 How to fix it (if dynamic output is desired)

Re-evaluate the template literal inside `log()`:

```javascript
function log() {
  console.log(`Count is ${count}`);
}
```

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| Template literal evaluation timing | They run at assignment, not at access |
| Primitive vs reference types | Primitives are copied by value; objects/arrays are referenced |
| Closure capture semantics | Closures close over bindings, but the value of a primitive is immutable once assigned |
| Mental model of "live" variables | Not all variables in a closure are "live views" — only the bindings themselves are |

### 📚 Further reading

- [MDN: Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures)
- [ECMAScript Spec: Lexical Environments](https://tc39.es/ecma262/#sec-lexical-environments)

</details>

---

## 2. JavaScript context trap: losing `this`

What will this code output when `user.greet()` is called?

```javascript
const user = {
  name: "Alex",
  greet() {
    console.log(`Hello, ${this.name}!`);

    const innerNormal = function () {
      console.log(`Normal: ${this.name}`);
    };

    const innerArrow = () => {
      console.log(`Arrow: ${this.name}`);
    };

    innerNormal();
    innerArrow();
  },
};

user.greet();
```

> Test your understanding of execution context, function invocation patterns, and arrow function lexical binding.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
Hello, Alex!
Normal: undefined
Arrow: Alex
```

### 🧠 Explanation

This question tests three different ways `this` is resolved in JavaScript:

#### 1. Method call: `greet()`

- Called as `user.greet()`
- `this` is bound to the object preceding the dot → `user`
- Output: `Hello, Alex!`

#### 2. Regular function: `innerNormal()`

- Called as a standalone function: `innerNormal()`
- No object precedes the call. In strict mode (default in modern JS/modules), `this` is `undefined`. In non-strict browser environments, it falls back to `window`.
- `undefined.name` evaluates to `undefined` (or `window.name` which is typically `""`)
- Output: `Normal: undefined`

#### 3. Arrow function: `innerArrow()`

- Arrow functions **do not have their own `this` binding**
- They capture `this` lexically from the enclosing execution context at definition time
- The enclosing context is `greet()`, where `this === user`
- Output: `Arrow: Alex`

### 🔑 Core Concept

> Regular functions resolve `this` **at call time** (dynamic binding).
> Arrow functions capture `this` **at definition time** (lexical binding).

### 🛠 How to fix `innerNormal` if you want it to see `user`

```javascript
// Option 1: Explicit binding at call time
innerNormal.call(this);

// Option 2: Explicit binding at definition time
const innerNormal = function () {
  console.log(this.name);
}.bind(this);

// Option 3: Lexical capture via closure (older pattern)
const self = this;
const innerNormal = function () {
  console.log(self.name);
};
```

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| Implicit `this` binding | Regular functions lose context when called without an explicit receiver |
| Lexical `this` capture | Arrow functions inherit `this` from their parent scope |
| Strict vs non-strict mode | Changes fallback behavior (`undefined` vs global object) |
| Mental model of execution context | `this` is dynamic for regular functions, static for arrows |

### 📚 Further reading

- [MDN: this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)
- [MDN: Arrow functions (no own this)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)

</details>

---

## 3. Illusion of an "own" property on mutation

What will the code log to the console at the end?

```javascript
const grandparent = {
  heritage: ["gold", "land"],
  coins: 100,
};

const parent = Object.create(grandparent);
const child = Object.create(parent);

child.coins += 50;
child.heritage.push("debts");

console.log(grandparent.coins); // (1) ?
console.log(grandparent.heritage); // (2) ?
console.log(child.coins); // (3) ?
console.log(child.heritage); // (4) ?
```

> Test your understanding of the difference between reading a property from the prototype chain and writing a property on an object.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
100
["gold", "land", "debts"]
150
["gold", "land", "debts"]
```

### 🧠 Explanation

This is the mental trap:

#### For `coins`

The expression `child.coins += 50` desugars to `child.coins = child.coins + 50`.

1. JavaScript **reads** `child.coins`. It is not found on `child`, so the engine walks the prototype chain to `grandparent` and reads `100`.
2. It adds `50` → `150`.
3. It **writes** the result to `child.coins`.

Writes always land on the object that received the assignment. `child` gets its own `coins: 150`, while `grandparent.coins` stays `100`.

#### For `heritage`

The expression `child.heritage.push("debts")` does **not** assign to `heritage`.

1. JavaScript **reads** `child.heritage`, finds the array on `grandparent`, and keeps a reference to that same array in the heap.
2. `.push("debts")` **mutates** that shared array.

`child` never gets an own `heritage` property — it still resolves to the grandparent's array, which now includes `"debts"`.

### 🔑 Core Concept

> **Read** vs **write** behave differently on the prototype chain.
> Assignment creates (or updates) an **own** property on the target object.
> Mutating a referenced object affects the **shared** value visible to every object in the chain that points to it.

### 🛠 How to avoid accidental prototype mutation

```javascript
// Option 1: Assign a new array instead of mutating the inherited one
child.heritage = [...child.heritage, "debts"];

// Option 2: Copy mutable values when creating the child
const child = Object.create(parent);
Object.assign(child, {
  coins: grandparent.coins,
  heritage: [...grandparent.heritage],
});
child.coins += 50;
child.heritage.push("debts"); // now only child's copy is affected
```

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| Prototype property lookup | Reads fall through the chain until a property is found |
| Assignment vs mutation | `+=` writes locally; `.push()` mutates a shared reference |
| Own vs inherited properties | `hasOwnProperty` and debugging surprises depend on this distinction |
| Reference types on prototypes | Shared mutable state on a prototype affects all descendants |

### 📚 Further reading

- [MDN: Prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)
- [MDN: Object.create()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create)
- [ECMAScript Spec: OrdinarySet](https://tc39.es/ecma262/#sec-ordinaryset)

</details>

---

## 4. JavaScript coercion & precedence trap

What will this code output?

```javascript
console.log(+"5" + [1] + !"0");
```

> Test your understanding of operator precedence, implicit type conversion, and left-to-right evaluation.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
"51false"
```

### 🧠 Explanation

This expression combines unary operators, object-to-primitive coercion, and left-to-right associativity. Here's the exact execution flow:

#### Step 1: Unary operators (highest precedence)

Unary `+` and `!` are evaluated before binary `+`.

- `+"5"` → `ToNumber` conversion → `5`
- `!"0"` → `"0"` is truthy → `!true` → `false`
- Expression becomes: `5 + [1] + false`

#### Step 2: Left-to-right evaluation & first `+`

Binary `+` is left-associative. It evaluates `5 + [1]` first.

- One operand is a `Number`, the other is an `Object`.
- JS invokes `ToPrimitive([1])` → calls `Array.toString()` → `"1"`
- Since one operand is now a string, `+` switches to **string concatenation**
- `"5" + "1"` → `"51"`
- Expression becomes: `"51" + false`

#### Step 3: Second `+` & final coercion

- `"51"` (string) + `false` (boolean)
- Boolean `false` is coerced to string → `"false"`
- Concatenation: `"51" + "false"` → `"51false"`

### 🔑 Core Concept

> **Precedence** decides what runs first.
> **Associativity** decides direction (left → right for `+`).
> **Coercion** decides how types interact when mismatched.

The `+` operator is unique in JS: it performs both addition and concatenation. If **either** operand becomes a string during `ToPrimitive`, the entire operation switches to string concatenation.

### 🛠 Common pitfalls to avoid

| Mistake | Why it's wrong |
|---|---|
| Thinking `5 + [1]` equals `6` | `[1]` is not a number; it's coerced to `"1"` |
| Thinking `!"0"` equals `true` | Only `""`, `0`, `null`, `undefined`, `NaN` are falsy. `"0"` is a non-empty string → truthy |
| Assuming right-to-left evaluation | `+` is strictly left-associative in JS |

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| Operator precedence table | Determines evaluation order before execution begins |
| `ToPrimitive` & `ToString` algorithms | How objects/arrays convert in mixed-type expressions |
| Left-to-right associativity | Critical for chaining `+` operations with side effects or type shifts |
| Falsy/truthy rules | Essential for `!`, `&&`, `||`, and conditional logic |

### 📚 Further reading

- [MDN: Operator Precedence](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence)
- [MDN: Type Coercion](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#type_conversion)
- [ECMAScript Spec: Addition operator `+`](https://tc39.es/ecma262/#sec-addition-operator-plus)

</details>

---

## 5. JavaScript event loop & queue priority trap

In what order will the numbers be printed to the console?

```javascript
console.log("1");

setTimeout(() => {
  console.log("2");
  Promise.resolve().then(() => {
    console.log("3");
  });
}, 0);

new Promise((resolve) => {
  console.log("4");
  resolve();
}).then(() => {
  console.log("5");
  setTimeout(() => {
    console.log("6");
  }, 0);
});

console.log("7");
```

> Test your understanding of the call stack, microtask queue, and macrotask queue execution order.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
1
4
7
5
2
3
6
```

### 🧠 Explanation

This question tests the exact execution order defined by the JavaScript event loop.

#### Step 1: Synchronous execution (call stack)

- `console.log("1")` runs immediately → outputs `1`
- `setTimeout` schedules its callback as a **macrotask** and exits
- `new Promise` executor runs **synchronously** → `console.log("4")` outputs `4`. `resolve()` is called
- `.then()` schedules its callback as a **microtask**
- `console.log("7")` runs immediately → outputs `7`
- Call stack is now empty

#### Step 2: Microtask queue (priority over macrotasks)

- The event loop checks the microtask queue before picking the next macrotask
- Microtask 1 runs: `console.log("5")` → outputs `5`
- Inside it, `setTimeout` schedules a new callback as **macrotask 2** (appended to the macrotask queue)
- Microtask queue is now empty

#### Step 3: Macrotask queue (first cycle)

- Event loop picks **macrotask 1** (the first `setTimeout`)
- `console.log("2")` → outputs `2`
- `Promise.resolve().then(...)` schedules **microtask 2**

#### Step 4: Microtask queue (again)

- Before moving to the next macrotask, the event loop **must completely drain** the microtask queue
- Microtask 2 runs: `console.log("3")` → outputs `3`
- Microtask queue is empty

#### Step 5: Macrotask queue (second cycle)

- Event loop picks **macrotask 2** (the second `setTimeout`)
- `console.log("6")` → outputs `6`

### 🔑 Core Concept

> **Execution order:** synchronous code → microtasks (`Promise`, `queueMicrotask`) → macrotasks (`setTimeout`, `setInterval`, I/O) → repeat.
>
> Every time a macrotask finishes, the event loop **must completely drain the microtask queue** before proceeding to the next macrotask. Nested microtasks created during a macrotask will delay the next macrotask.

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| Promise executor timing | Runs synchronously during construction, not asynchronously |
| Microtask vs macrotask priority | Microtasks always clear before the next macrotask runs |
| Nested async scheduling | New tasks are appended to the back of their respective queues |
| Event loop phases | Critical for debugging race conditions, UI freezes, and API batching |

### 📚 Further reading

- [MDN: Event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop)
- [MDN: Using Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)
- [HTML Spec: Event loops](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)

</details>

---

## 6. Microtask order: `async/await` vs promise chains

In what exact order will the logs be printed to the console?

```javascript
async function asyncFunc() {
  console.log("2");
  await Promise.resolve();
  console.log("3");
}

console.log("1");
asyncFunc();

Promise.resolve()
  .then(() => {
    console.log("4");
  })
  .then(() => {
    console.log("5");
  });

console.log("6");
```

> Test your understanding of how `await` is compiled under the hood and its exact scheduling priority compared to `.then()`.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
1
2
6
3
4
5
```

### 🧠 Explanation

This question reveals exactly how `async/await` is translated into promise chains and how the event loop schedules continuations.

#### Step 1: Synchronous phase

- `console.log("1")` executes → outputs `1`
- `asyncFunc()` is invoked. Code runs synchronously until the first `await`
- `console.log("2")` executes → outputs `2`
- `await Promise.resolve()` is encountered. The expression on the right evaluates to an already-resolved promise. The engine wraps the remaining function body (`console.log("3")`) in a **microtask** and suspends execution. This microtask is queued
- Control returns to the global scope
- `Promise.resolve().then(...)` registers a callback. This creates **microtask 2** (logs `4`). The chained `.then` (logs `5`) is _not_ queued yet; it waits for microtask 2 to resolve
- `console.log("6")` executes → outputs `6`
- Call stack is empty. Event loop switches to the microtask queue

#### Step 2: Microtask queue processing (FIFO order)

- **Microtask 1** (from `await` continuation): runs `console.log("3")` → outputs `3`
- **Microtask 2** (from first `.then`): runs `console.log("4")` → outputs `4`. Its resolution immediately queues the next `.then` callback as **microtask 3**
- **Microtask 3** (from chained `.then`): runs `console.log("5")` → outputs `5`
- Microtask queue is empty

### 🔑 Core Concept

> `await` is syntactic sugar for `.then()`.
> The code after `await` is compiled into a `.then()` callback and scheduled as a **microtask**.
> Microtasks are processed in strict FIFO order based on when they were registered during synchronous execution.

### 🛠 Common pitfalls

| Mistake | Why it's wrong |
|---|---|
| Assuming `await` blocks the thread | It yields to the event loop instead |
| Thinking `await` has higher priority than `.then()` | They share the exact same microtask queue |
| Expecting all `.then()` callbacks to queue at once | Chained `.then()` blocks are scheduled only after the previous one resolves |

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| `await` compilation model | `await` desugars to `.then()` under the hood |
| Microtask FIFO scheduling | Execution order depends on registration time, not declaration order |
| Promise chain evaluation | Subsequent `.then()` blocks queue only after the previous one resolves |
| Async function suspension | How JS pauses and resumes execution without blocking the main thread |

### 📚 Further reading

- [MDN: async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
- [MDN: Using promises (Chaining)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises#chaining)
- [ECMAScript Spec: Await](https://tc39.es/ecma262/#sec-await)
- [HTML Spec: Microtask queues](https://html.spec.whatwg.org/multipage/webappapis.html#microtask-queue)

</details>

---

## 7. JavaScript string length trap: UTF-16 & surrogate pairs

What values will `.length` return for these three strings?

```javascript
const str1 = "A";
const str2 = "𠮷"; // Rare CJK character (U+20BB7)
const str3 = "👨‍👩‍👧‍👦"; // Emoji: Family (Man + Woman + Girl + Boy, ZWJ sequence)

console.log(str1.length); // (1)
console.log(str2.length); // (2)
console.log(str3.length); // (3)
```

> Test your understanding of JavaScript's internal string encoding, grapheme clusters, and the difference between code units and visual characters.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
1
2
11
```

### 🧠 Explanation

JavaScript strings are sequences of **16-bit code units** in UTF-16 encoding — not visual characters (graphemes). The `.length` property counts code units, not what you see on screen.

#### Breakdown by string

| String | Visual characters | Code points | Code units (UTF-16) | `.length` |
|---|---|---|---|---|
| `"A"` | 1 | `U+0041` (BMP) | 1 | `1` |
| `"𠮷"` | 1 | `U+20BB7` (supplementary) | 2 (surrogate pair: `0xD842 0xDFB7`) | `2` |
| `"👨‍👩‍👧‍👦"` | 1 (family emoji) | 7 code points: `👨` `ZWJ` `👩` `ZWJ` `👧` `ZWJ` `👦` | 11 code units (4 emoji × 2 units each + 3 ZWJ × 1 unit each) | `11` |

### 🔑 Core concepts

1. **BMP vs supplementary planes**  
   Characters in the Basic Multilingual Plane (U+0000 to U+FFFF) fit in one 16-bit code unit. Characters outside (like `"𠮷"`) require a **surrogate pair** — two code units.

2. **Emoji are often sequences**  
   Complex emoji like `👨‍👩‍👧‍👦` are built from multiple code points joined by **Zero Width Joiners (ZWJ, U+200D)**. Each emoji component outside BMP uses 2 code units; ZWJ uses 1.

3. **`.length` counts code units, not graphemes**  
   This is by design. It is fast, deterministic, and encoding-agnostic — but not human-readable.

### 🛠 How to count visual characters (graphemes)

```javascript
// Method 1: Spread operator (uses iterator, handles surrogate pairs)
[..."𠮷"].length; // 1
[..."👨‍👩‍👧‍👦"].length; // 1 (but may vary by engine/Unicode version)

// Method 2: Array.from (same as spread for this purpose)
Array.from("𠮷").length; // 1

// Method 3: Intl.Segmenter (modern, grapheme-aware, best for production)
const segmenter = new Intl.Segmenter("en", { granularity: "grapheme" });
[...segmenter.segment("👨‍👩‍👧‍👦")].length; // 1
```

> Note: Even `[...str]` may not perfectly handle all ZWJ sequences across all environments. For production-grade grapheme counting, use `Intl.Segmenter` (ES2022+) or a dedicated library like `grapheme-splitter`.

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| UTF-16 internal encoding | JavaScript's string model is based on code units, not code points or graphemes |
| Surrogate pairs | Essential for correctly handling characters outside BMP (many CJK, historic scripts, emoji) |
| Grapheme vs code point vs code unit | Critical for input validation, text truncation, analytics, and i18n |
| ZWJ emoji sequences | Modern emoji are often composite; naive length checks break UI/UX |

### 📚 Further reading

- [MDN: String.prototype.length](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/length)
- [MDN: JavaScript string internals (UTF-16)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#string_type)
- [Unicode Standard: Surrogate pairs](https://unicode.org/faq/utf_bom.html#utf16-4)
- [ECMAScript Spec: String type](https://tc39.es/ecma262/#sec-ecmascript-language-types-string-type)
- [MDN: Intl.Segmenter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter)

</details>

---

## 8. JavaScript `super` trap: arrow functions & lexical binding

What will this code output when `child.test()` is called?

```javascript
class Parent {
  say() {
    return "Parent";
  }
}

class Child extends Parent {
  say() {
    return "Child";
  }

  test() {
    const arrow = () => super.say();
    return arrow();
  }
}

const child = new Child();
console.log(child.test());
```

> Test your understanding of class inheritance, `super` resolution, and how arrow functions interact with class internals.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
Parent
```

### 🧠 Explanation

This question reveals a subtle but critical detail: **arrow functions do not have their own `super` binding**. They resolve `super` lexically from their enclosing scope.

#### Step-by-step resolution

1. **Class setup**
   - `Parent` has a method `say()` returning `"Parent"`
   - `Child` extends `Parent` and overrides `say()` to return `"Child"`

2. **Method `test()` in `Child`**
   - Defined with its own `[[HomeObject]]` internal slot pointing to `Child.prototype`
   - This slot is used by the engine to resolve `super` at definition time, not call time

3. **Arrow function `arrow = () => super.say()`**
   - Arrow functions have **no own `this`, `arguments`, `new.target`, or `super`**
   - When `super.say()` is encountered inside the arrow, JS looks up the lexical environment chain
   - The enclosing scope is `test()`, which _does_ have a valid `[[HomeObject]]` (`Child.prototype`)
   - Therefore, `super` resolves to `Parent.prototype`, and `super.say()` calls `Parent.prototype.say()`

4. **Execution**
   - `child.test()` invokes `test()` with `this === child`
   - The arrow function is created and immediately called
   - `super.say()` resolves to `Parent.prototype.say()` → returns `"Parent"`

### 🔑 Core Concept

> `super` is **lexically bound** at function definition time via the internal `[[HomeObject]]` slot.
> Arrow functions inherit `super` from their enclosing scope — they cannot rebind it.
> This is consistent with how they inherit `this`, but applies to class inheritance mechanics.

### 🛠 Common pitfalls

| Mistake | Why it's wrong |
|---|---|
| Assuming arrow functions can rebind `super` | They inherit `super` statically from the enclosing scope |
| Confusing `super.method()` with `this.method()` | `super` uses static resolution via `[[HomeObject]]`; `this` uses dynamic prototype lookup |
| Expecting `super` inside an arrow to refer to the caller | It refers to the enclosing method's home object |

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| `[[HomeObject]]` internal slot | The engine uses this to statically resolve `super` — critical for correct inheritance |
| Arrow function lexical scoping | Arrows inherit bindings (`this`, `super`, `arguments`) from their definition context |
| Static vs dynamic dispatch | `super` is static (compile-time), `this` is dynamic (runtime) |
| Class method vs arrow semantics | Understanding which bindings are own vs inherited in class bodies |

### 📚 Further reading

- [MDN: super](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/super)
- [MDN: Arrow functions (no own bindings)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)
- [ECMAScript Spec: Method definitions & `[[HomeObject]]`](https://tc39.es/ecma262/#sec-method-definitions-runtime-semantics-propertydefinitionevaluation)
- [ECMAScript Spec: Arrow function definitions](https://tc39.es/ecma262/#sec-arrow-function-definitions)

</details>

---

## 9. JavaScript prototype trap: `prototype` vs `__proto__`

What will these three comparisons return?

```javascript
function Fake() {}

console.log(Fake.__proto__ === Fake.prototype); // (1)
console.log(Fake.__proto__ === Function.prototype); // (2)
console.log(new Fake().__proto__ === Fake.prototype); // (3)
```

> Test your understanding of the prototype chain, constructor functions, and the difference between `[[Prototype]]` and `prototype` properties.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
false
true
true
```

### 🧠 Explanation

This question tests a fundamental but often misunderstood distinction: **`prototype` is a property of functions used for inheritance, while `__proto__` (or `[[Prototype]]`) is the internal link that forms the prototype chain**.

#### (1) `Fake.__proto__ === Fake.prototype` → `false`

| Expression | What it refers to | Value |
|---|---|---|
| `Fake.prototype` | The object that will become the `[[Prototype]]` of instances created via `new Fake()` | `{}` (empty object by default) |
| `Fake.__proto__` | The `[[Prototype]]` of the function object `Fake` itself | `Function.prototype` |

These are two completely different objects. `Fake.prototype` is for _instance inheritance_. `Fake.__proto__` is for _function inheritance_.

#### (2) `Fake.__proto__ === Function.prototype` → `true`

- `Fake` is a function object
- All functions in JavaScript are instances of the built-in `Function` constructor
- Therefore, the `[[Prototype]]` of any function (including `Fake`) is `Function.prototype`
- `__proto__` is an accessor for the internal `[[Prototype]]` slot (legacy, but standardized for web compatibility)

#### (3) `new Fake().__proto__ === Fake.prototype` → `true`

- `new Fake()` creates a new object
- The internal `[[Prototype]]` of this new object is set to the value of `Fake.prototype` at the time of construction (per the `[[Construct]]` algorithm)
- `__proto__` exposes this `[[Prototype]]` link
- Therefore, the instance's prototype chain points directly to `Fake.prototype`

### 🔑 Core Concept

> **`prototype`**: a property **of constructor functions**. Used to define methods and properties shared by all instances.
>
> **`__proto__` / `[[Prototype]]`**: an internal link **on every object** (including functions) that points to its prototype for property lookup.
>
> **Rule:** `new Constructor().__proto__ === Constructor.prototype` (always true by spec).

### 🛠 Visual diagram

```
Fake (function)
 ├─ [[Prototype]] (__proto__) ──► Function.prototype
 ├─ prototype ──► {} (object used for instances)
      │
      └─ [[Prototype]] of new Fake() ──► (this same {})

new Fake() (instance)
 ├─ [[Prototype]] (__proto__) ──► Fake.prototype
```

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| `prototype` vs `[[Prototype]]` | Confusing these breaks inheritance modeling and debugging |
| Function objects are objects too | Functions have their own prototype chain (`Function.prototype`) |
| `new` operator mechanics | Understanding how `[[Construct]]` links instances to `prototype` |
| `__proto__` as legacy accessor | Modern code should use `Object.getPrototypeOf()` / `Object.setPrototypeOf()` |

### 📚 Further reading

- [MDN: `__proto__` vs `prototype`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain#the_difference_between_prototype_and__proto__)
- [MDN: `Object.prototype.__proto__`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/proto)
- [MDN: `Function.prototype`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/prototype)
- [ECMAScript Spec: `[[Construct]]` algorithm](https://tc39.es/ecma262/#sec-ordinaryconstruct)
- [ECMAScript Spec: `prototype` property of function objects](https://tc39.es/ecma262/#sec-function-instances)

</details>

---

## 10. JavaScript `.call.call()` trap: the double call trick

What will this code output? Which function actually executes?

```javascript
function funcA() {
  console.log("Function A called");
}

function funcB() {
  console.log("Function B called");
}

funcA.call.call(funcB, { name: "Test" });
```

> Test your understanding of `Function.prototype.call`, method borrowing, and how `this` binding works at the meta level.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
Function B called
```

### 🧠 Explanation

This is an advanced metaprogramming trick that exploits how `Function.prototype.call` resolves its own `this` context.

#### Step-by-step resolution

1. **Expression parsing**
   - `funcA.call` resolves to `Function.prototype.call` (a reference to the generic `call` method)
   - The expression becomes: `Function.prototype.call.call(funcB, { name: "Test" })`

2. **Outer `.call` execution**
   - The outer `.call` is invoked with:
     - `this` = `Function.prototype.call` (the function object itself)
     - arguments = `[funcB, { name: "Test" }]`

3. **How `call` works internally (simplified spec logic)**

   ```javascript
   // Pseudocode of Function.prototype.call
   function call(target, ...args) {
     // `this` here is the function to invoke
     // `target` is the context to invoke it with
     return this.apply(target, args);
   }
   ```

   - In our case:
     - `this` = `Function.prototype.call` (the generic `call` method)
     - `target` = `funcB`
     - `args` = `[{ name: "Test" }]`

4. **Final dispatch**
   - The generic `call` method now executes: `funcB.apply({ name: "Test" }, [])`
   - `funcB` runs with `this = { name: "Test" }`
   - Since `funcB` only logs a static string (no `this` access), the context is irrelevant
   - Output: `"Function B called"`

### 🔑 Core Concept

> `Function.prototype.call` is itself a function.
> When you do `X.call.call(Y, ctx)`, you're borrowing the `call` method to invoke `Y` with `ctx`.
> `X` is irrelevant — only its `.call` property (which is always `Function.prototype.call`) matters.

### 🛠 Equivalent readable forms

```javascript
// All of these do the same thing:
funcA.call.call(funcB, ctx);
Function.prototype.call.call(funcB, ctx);

// The straightforward version:
funcB.call(ctx);
```

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| Methods are values | `func.call` can be detached and reused on other functions |
| `this` in method borrowing | `call` uses its own `this` as the target function |
| Spec-level behavior | Critical for polyfills, proxies, and advanced functional patterns |
| Argument forwarding | How `call`/`apply`/`bind` manipulate execution context and arguments |

### 📚 Further reading

- [MDN: Function.prototype.call](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call)
- [MDN: Function.prototype.apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply)
- [ECMAScript Spec: Function.prototype.call](https://tc39.es/ecma262/#sec-function.prototype.call)
- [ECMAScript Spec: OrdinaryCallBindThis](https://tc39.es/ecma262/#sec-ordinarycallbindthis)

</details>

---

## 11. JavaScript private fields trap: `#fields` + `Proxy` incompatibility

What happens when `proxy.secret` is accessed?

```javascript
class Vault {
  #code = "1234-SUPER-SECRET";

  get secret() {
    return this.#code;
  }
}

const vault = new Vault();
const proxy = new Proxy(vault, {});

console.log(proxy.secret);
```

> Test your understanding of ECMAScript private class fields, `Proxy` traps, and why `this` binding breaks encapsulation.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Output

```
TypeError: Cannot read private member #code from an object whose class did not declare it
```

### 🧠 Explanation

This is a critical edge case when combining **private class fields** (`#field`) with **`Proxy`** wrappers. The error occurs because private fields are **lexically scoped to the class that declared them**, not dynamically resolved via `this`.

#### Step-by-step breakdown

1. **Class definition**
   - `Vault` declares a private field `#code`
   - The getter `secret` accesses `this.#code`

2. **Proxy creation**
   - `proxy = new Proxy(vault, {})` creates a wrapper around `vault`
   - The proxy has no own properties and no internal slots from `Vault`

3. **Property access `proxy.secret`**
   - The `get` trap (default behavior via `Reflect.get`) returns the `secret` getter from `Vault.prototype`
   - The getter is invoked with `this === proxy` (the receiver)

4. **Private field resolution**
   - Per ECMAScript spec, accessing `#code` requires:
     - The `this` value must be an instance of the **exact class** that declared `#code` (`Vault`)
     - The engine checks an internal **PrivateName** brand check, not just the prototype chain
   - `proxy` is not a `Vault` instance — it's a `Proxy` exotic object
   - Brand check fails → `TypeError` is thrown

### 🔑 Core Concept

> Private fields (`#field`) are **not** looked up via prototype chain or dynamic `this`.
> They use a **static brand check**: the object must be an instance of the class that lexically declared the field.
> `Proxy` objects do not inherit internal slots or private name tables from their target.

### 🛠 How to fix it

**Option 1: Bind methods to the original target**

```javascript
const proxy = new Proxy(vault, {
  get(target, prop, receiver) {
    const value = Reflect.get(target, prop, receiver);
    if (typeof value === "function") {
      return value.bind(target);
    }
    return value;
  },
});
// Note: getters accessing #fields still fail — the brand check happens before bind helps
```

**Option 2: Avoid `Proxy` for classes with private fields**

```javascript
class SafeVault {
  #code = "1234-SUPER-SECRET";

  getSecret() {
    return this.#code;
  }
}

const vault = new SafeVault();
const api = {
  get secret() {
    return vault.getSecret(); // always calls with correct `this`
  },
};
```

**Option 3: Use `WeakMap` for true privacy (pre-`#fields` pattern)**

```javascript
const _code = new WeakMap();

class Vault {
  constructor() {
    _code.set(this, "1234-SUPER-SECRET");
  }

  get secret() {
    return _code.get(this); // works with Proxy — WeakMap checks by reference
  }
}
```

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| Private field brand checks | `#field` access is lexically scoped, not dynamically resolved |
| `Proxy` exotic object behavior | Proxies don't inherit internal slots or private name tables |
| `this` binding in getters | Getters receive the receiver (proxy), not the original target |
| Encapsulation boundaries | Understanding where JS privacy guarantees end and metaprogramming begins |

### 📚 Further reading

- [MDN: Private class fields](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_class_fields)
- [MDN: Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- [ECMAScript Spec: Private field resolution](https://tc39.es/ecma262/#sec-privatefieldget)
- [ECMAScript Spec: Proxy exotic objects](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots)

</details>

---

## 12. JavaScript memory leak trap: shared closure contexts

Each time `replaceThing()` is called, the previous `theThing` object should become eligible for garbage collection since it's no longer referenced in the global scope. Will this code cause a memory leak?

```javascript
let theThing = null;

function replaceThing() {
  const originalThing = theThing; // reference to previous state

  // Unused function that closes over originalThing
  function unused() {
    if (originalThing) console.log("hi");
  }

  // Overwrite global variable with new heavy object
  theThing = {
    longStr: new Array(1000000).join("*"), // ~1MB of memory
    someMethod: function () {
      // empty method
    },
  };
}

// Run in interval loop
setInterval(replaceThing, 10);
```

> Test your understanding of lexical environment sharing, closure retention, and how V8 manages function contexts.

<details>
<summary>💡 Click to reveal answer & explanation</summary>

### ✅ Answer

**Yes — this code causes a catastrophic memory leak and will quickly crash with OOM (Out of Memory).**

### 🧠 Explanation

This is the classic **"Meteor closure leak"** pattern. The leak occurs because of how V8 shares lexical environments between functions defined in the same scope.

#### Step-by-step retention chain

1. **First call to `replaceThing()`**
   - `originalThing = null` (initial global value)
   - `unused` function is created, closing over the lexical environment containing `originalThing`
   - `theThing` is assigned a new object with `longStr` (~1MB) and `someMethod`

2. **Second call to `replaceThing()`**
   - `originalThing = theThing` (now references the object from call #1)
   - A **new** `unused` function is created, closing over the **new** lexical environment
   - A **new** `theThing` object is created (call #2)
   - Critically: `someMethod` (from call #2) and `unused` (from call #2) are defined in the **same execution context**

3. **The shared context problem**
   - In V8, all functions created within the same function invocation **share a single `LexicalEnvironment` record**
   - Even though `unused` is never called or exported, its mere existence forces V8 to retain the entire context object
   - `theThing.someMethod` (from call #2) holds a reference to this shared context
   - The shared context holds `originalThing` → which points to `theThing` from call #1
   - `theThing` from call #1 has its own `someMethod`, which holds its own shared context, which holds _its_ `originalThing` → call #0

4. **Result: a linked list in memory**

   ```
   global.theThing (call N)
     └─ someMethod
          └─ [[Environment]] (shared context)
               └─ originalThing → theThing (call N-1)
                    └─ someMethod
                         └─ [[Environment]]
                              └─ originalThing → theThing (call N-2)
                                   └─ ... (infinite chain)
   ```

   - Every new object retains the previous one via the shared closure context
   - Garbage collector cannot free any node in this chain because all are reachable from `global.theThing`
   - Memory grows linearly with each interval tick → OOM crash

### 🔑 Core Concept

> Functions defined in the same scope **share a single lexical environment object** in V8.
> If **any** function in that scope is retained (even unused), the **entire context** — including all captured variables — is retained.
> This creates implicit retention chains that bypass obvious reference analysis.

### 🛠 How to fix it

**Option 1: Break the shared context by isolating closures**

```javascript
function replaceThing() {
  const originalThing = theThing;

  const unused = (() => {
    const captured = originalThing;
    return function () {
      if (captured) console.log("hi");
    };
  })();

  theThing = {
    longStr: new Array(1000000).join("*"),
    someMethod: function () {},
  };
}
```

**Option 2: Avoid capturing large objects in unused closures**

```javascript
function replaceThing() {
  function unused() {
    console.log("hi");
  }

  theThing = {
    longStr: new Array(1000000).join("*"),
    someMethod: function () {},
  };
}
```

**Option 3: Nullify captured references explicitly**

```javascript
function replaceThing() {
  let originalThing = theThing;

  function unused() {
    if (originalThing) console.log("hi");
  }

  theThing = {
    longStr: new Array(1000000).join("*"),
    someMethod: function () {},
  };

  originalThing = null;
}
```

### 🎯 What this question tests

| Concept | Why it matters |
|---|---|
| Lexical environment sharing | Functions in same scope share context — retaining one retains all captured vars |
| Closure retention semantics | Unused closures still prevent GC if their context is reachable |
| Implicit reference chains | Memory leaks can hide in closure graphs, not just explicit object graphs |
| V8 implementation details | Understanding engine behavior is critical for high-performance/long-running JS apps |

### 📚 Further reading

- [MDN: Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures)
- [MDN: Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)
- [ECMAScript Spec: Lexical Environments](https://tc39.es/ecma262/#sec-lexical-environments)

</details>

---

## Want more?

Solve more JavaScript interview questions at [SkillHacker.io](https://skillhacker.io).
