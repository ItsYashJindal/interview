[Home](README.md) | Next: [02 JavaScript Coding →](02-javascript-coding.md)

# 01 – JavaScript Concepts

## Contents
- [Main topics](#main-topics)
  1. [var, let, const](#1-var-let-const)
  2. [Hoisting and Temporal Dead Zone](#2-hoisting-and-temporal-dead-zone-tdz)
  3. [Closures and lexical scope](#3-closures-and-lexical-scope)
  4. [Promises and async/await](#4-promises-and-asyncawait)
  5. [Event loop, call stack, microtasks](#5-event-loop-call-stack-microtasks)
  6. [map, filter, reduce](#6-map-filter-reduce)
  7. [Debouncing and throttling](#7-debouncing-and-throttling)
  8. [Shallow copy vs deep copy](#8-shallow-copy-vs-deep-copy)
  9. [this, call, apply, bind](#9-this-call-apply-bind)
  10. [Prototypes and inheritance](#10-prototypes-and-inheritance)
- [Extra topics often asked (added)](#extra-topics-often-asked)

---

## Main topics

### 1. var, let, const

**Q. What is the difference between var, let and const?**

> 🎤 **Interview answer:** "The main differences are scope, re-assignment and hoisting. `var` is function-scoped, so it ignores blocks like `if` or `for`. `let` and `const` are block-scoped, so they only live inside the nearest curly braces. `var` and `let` can be re-assigned, `const` cannot. All three are hoisted, but `var` starts as `undefined`, while `let` and `const` sit in the Temporal Dead Zone, so using them before the declaration throws a ReferenceError.
>
> In my code I use `const` by default, `let` only when the value really changes, like a counter or a retry variable, and I don't use `var` at all, because its function scope causes bugs like the classic setTimeout loop problem."

| | var | let | const |
|---|---|---|---|
| Scope | Function | Block `{}` | Block `{}` |
| Re-declare in same scope | Yes | No | No |
| Re-assign | Yes | Yes | No |
| Hoisted | Yes, set to `undefined` | Yes, but in TDZ | Yes, but in TDZ |
| Added to `window` (global) | Yes | No | No |

```js
if (true) {
  var a = 1;
  let b = 2;
}
console.log(a); // 1  (var ignores block)
console.log(b); // ReferenceError
```

**Q. Can we change a const object?**

> 🎤 **Interview answer:** "Yes, we can change its properties. `const` only means the variable can't point to a new value. The object it points to can still change. So `user.name = 'Rahul'` works, but `user = {}` throws a TypeError. If I really want the object to be read-only I use `Object.freeze`, but that is shallow, so nested objects can still change. In TypeScript I usually get the same safety at compile time with `readonly` or `as const`."

Yes, we can change its properties. `const` only stops re-assigning the variable, not changing the object inside.
```js
const user = { name: "Yash" };
user.name = "Rahul"; // OK
user = {};           // TypeError
```
To stop property changes use `Object.freeze(user)` (shallow only).

**Q. Classic loop question**

> 🎤 **Interview answer:** "With `var` it prints 3, 3, 3, and with `let` it prints 0, 1, 2. The reason is that `var` is function-scoped, so the whole loop shares one `i`. By the time the setTimeout callbacks run, the loop is done and `i` is 3. `let` is block-scoped and JavaScript creates a new binding of `i` for every iteration, so each callback closes over its own value. Before `let` existed, people fixed this with an IIFE or by passing `i` as the third argument to setTimeout."

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0); // 3 3 3
for (let i = 0; i < 3; i++) setTimeout(() => console.log(i), 0); // 0 1 2
```
`var` has one shared `i` for the whole loop. `let` creates a new `i` for each loop iteration.

**What to use:** `const` by default, `let` when the value changes, avoid `var`.

---

### 2. Hoisting and Temporal Dead Zone (TDZ)

**Q. What is hoisting?**

> 🎤 **Interview answer:** "Hoisting means JavaScript processes all declarations before it runs the code, so they are registered at the top of their scope. Function declarations are hoisted completely, so I can call them before the line where they are written. `var` is hoisted and set to `undefined`, so reading it early gives `undefined` instead of an error. `let` and `const` are hoisted too, but not initialised. Accessing them early throws a ReferenceError because of the Temporal Dead Zone. Function expressions and arrow functions follow the rules of the variable they are assigned to. That's why calling a `var` function expression early gives 'is not a function'."

Before running code, JavaScript reads all declarations and creates memory for them. So you can use some things before the line where they are written.

- `var` → hoisted and set to `undefined`.
- `let` / `const` → hoisted but **not initialised**. Using them before the line gives `ReferenceError`.
- Function declarations → hoisted fully (you can call them before they are written).
- Function expressions / arrow functions → follow the rule of the variable (`var`/`let`/`const`).

```js
console.log(x); // undefined
var x = 5;

console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 5;

greet();        // works
function greet() { console.log("hi"); }

sayHi();        // TypeError: sayHi is not a function (sayHi is undefined here)
var sayHi = function () {};
```

**Q. What is the Temporal Dead Zone?**

> 🎤 **Interview answer:** "The TDZ is the period from the start of a block until the line where a `let` or `const` variable is declared. The variable already exists in that scope, but it isn't initialised, so any access throws a ReferenceError like 'Cannot access x before initialization'. It's there on purpose, to catch bugs where we use a variable before giving it a value, which `var` silently allowed by returning `undefined`."

The time between the start of the block and the line where a `let`/`const` variable is declared. During this time the variable exists but cannot be used.
```js
{
  // TDZ for 'a' starts
  console.log(a); // ReferenceError
  let a = 10;     // TDZ ends
}
```
**Why it exists:** to catch bugs where you use a variable before giving it a value.

---

### 3. Closures and lexical scope

**Q. What is lexical scope?**

> 🎤 **Interview answer:** "Lexical scope means a function's access to variables depends on where the function is written in the code, not where it is called from. An inner function can read variables of its outer functions and the global scope, but the outer scope can't see variables inside the inner function. JavaScript decides this at write time, and it's the base on which closures work."

A function can access variables from the place where it is **written** (not where it is called). Inner functions can see outer variables, outer cannot see inner.

**Q. What is a closure?**

> 🎤 **Interview answer:** "A closure is when a function remembers the variables from the scope where it was created, even after that outer function has finished running. For example, a `counter` function returns an inner function that increments a `count` variable. `count` isn't accessible from outside, but the inner function keeps it alive.
>
> I use closures all the time without thinking about it. Debounce keeps the timer id in a closure. Memoize keeps its cache in a closure. In Express I write middleware factories like `authorize('admin')`, which return a middleware that remembers the role. The one thing to be careful about is memory: if a long-lived closure, like a global event listener, holds a reference to a big object, that object can never be garbage-collected."

A closure is a function that **remembers the variables of its outer function** even after the outer function has finished running.

```js
function counter() {
  let count = 0;               // private variable
  return function () {
    count++;
    return count;
  };
}
const inc = counter();
inc(); // 1
inc(); // 2   -> count is remembered
```

**Where closures are used in real code:**
- Private variables (data hiding) – like above.
- Debounce / throttle (timer id is kept in a closure).
- Memoization (cache object kept in a closure).
- Function factories: `const add5 = makeAdder(5)`.
- Event handlers and callbacks that use outer variables.
- Express middleware factories: `const auth = (role) => (req, res, next) => {...}`.

**Gotcha:** Closures can cause memory leaks if they hold big objects and the function is kept alive (e.g. stored in a global listener).

---

### 4. Promises and async/await

**Q. What is a Promise?**

> 🎤 **Interview answer:** "A Promise is an object that represents the result of an async operation that will finish in the future. It has three states: pending, fulfilled with a value, or rejected with an error. Once it settles, it can't change again. We consume it with `.then`, `.catch` and `.finally`, or with `async/await`. In Node almost everything async, like DB queries, HTTP calls and file reads, returns a promise today, so it's the base of all my backend code."

An object that represents a value that will be available in the future. It has 3 states:
- `pending` – still running
- `fulfilled` – done with a value (`resolve`)
- `rejected` – failed with an error (`reject`)

Once fulfilled or rejected it never changes again.

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve("done"), 1000);
});
p.then(v => console.log(v)).catch(e => console.error(e)).finally(() => console.log("end"));
```

**Q. Why Promises over callbacks?**

> 🎤 **Interview answer:** "Callbacks work, but when async steps depend on each other you end up with deeply nested code, which is callback hell. You also have to handle errors separately at every level. Promises let me chain steps in a flat way and handle all errors in one `.catch`. They also give us `Promise.all`, `allSettled` and `race` to combine operations. And `async/await` is built on promises, which makes async code read like normal synchronous code."

Callbacks inside callbacks become hard to read ("callback hell") and error handling is repeated in every level. Promises allow chaining and one `.catch` for all errors.

**Q. What is async/await?**

> 🎤 **Interview answer:** "async/await is syntax sugar over promises. An `async` function always returns a promise. Inside it, `await` pauses only that function until the promise settles, so the event loop keeps serving other requests. Errors from an awaited promise can be caught with a normal `try/catch`, which makes the code much easier to read. In Express I wrap async handlers with an asyncHandler, or use Express 5, so a rejected promise goes to the error middleware instead of becoming an unhandled rejection."

Cleaner syntax over promises. `async` function always returns a promise. `await` pauses that function (not the whole program) until the promise settles.
```js
async function getUser(id) {
  try {
    const user = await User.findById(id);
    return user;
  } catch (err) {
    console.error(err);
    throw err;
  }
}
```

**Q. Promise.all vs allSettled vs race vs any**

> 🎤 **Interview answer:** "`Promise.all` waits for all promises and gives an array of results, but it rejects as soon as any one fails. I use it when I need everything, like loading user, orders and wallet for a dashboard. `allSettled` never rejects. It waits for all of them and tells me the status of each one, so it's good for bulk jobs like sending notifications where some may fail. `race` settles with whichever promise finishes first, success or failure, and the common use is adding a timeout to a slow call. `any` resolves with the first success and only rejects with an AggregateError if all of them fail. It's useful when calling mirror servers."

| Method | Resolves when | Rejects when |
|---|---|---|
| `Promise.all` | All succeed (returns array of results) | Any one fails (fails fast) |
| `Promise.allSettled` | All finish (success or fail), returns status of each | Never rejects |
| `Promise.race` | First one to finish (success or fail) | If first one to finish fails |
| `Promise.any` | First one to succeed | All fail (`AggregateError`) |

Real use:
- `all` – load user, orders and wallet together for a dashboard.
- `allSettled` – send notifications to many users, some may fail, we still want the report.
- `race` – add a timeout to a slow API call.
- `any` – call 2 mirror servers and use whichever answers first.

**Q. Sequential vs parallel await**

> 🎤 **Interview answer:** "If I `await` two independent calls one after the other, the total time is the sum of both. If they don't depend on each other, I start them together with `Promise.all`, and the total time is only the slowest one. This is one of the easiest performance wins in APIs. For example, the list query and the count query for pagination can run in parallel. I only keep things sequential when the second call needs the result of the first, or when I deliberately want to limit load on a downstream service."

```js
// Sequential – slow (total = a + b)
const a = await getA();
const b = await getB();

// Parallel – fast (total = max(a, b))
const [a, b] = await Promise.all([getA(), getB()]);
```
Use parallel when the calls don't depend on each other. This is a common performance fix.

**Q. await inside forEach?**

> 🎤 **Interview answer:** "`forEach` doesn't wait for promises. It just fires the callbacks and returns immediately, so code after the loop runs before the async work is done, and errors become unhandled rejections. If I need the items one by one, I use `for...of` with `await`. If they can run in parallel, I use `Promise.all(items.map(async ...))`. For big lists I process them in batches so I don't open thousands of DB connections or API calls at once."

`forEach` does not wait for promises. Use `for...of` (sequential) or `Promise.all(arr.map(...))` (parallel).

---

### 5. Event loop, call stack, microtasks

**Q. Explain the event loop.**

> 🎤 **Interview answer:** "JavaScript runs on a single thread with one call stack. Slow operations like timers, network and file I/O are handed off to the browser or to Node's libuv, and when they finish, their callbacks are queued. There are two main queues. The microtask queue holds promise callbacks and `queueMicrotask`, and in Node `process.nextTick`, which runs even before them. The macrotask queue holds setTimeout, setInterval, I/O callbacks and setImmediate.
>
> The rule is: when the call stack is empty, run all microtasks, then take one macrotask, then again drain all microtasks, and repeat. That's why `console.log(1)`, a `setTimeout`, a `Promise.then` and `console.log(4)` print 1, 4, 3, 2. In practice this is why I never put heavy CPU work on the main thread in Node: while it runs, the loop is blocked and every other request waits."

JavaScript runs on **one thread**. It has:
1. **Call stack** – where functions run one by one.
2. **Web APIs / Node APIs** – timers, network, file system run outside the stack.
3. **Microtask queue** – promise callbacks (`.then`, `await` continuation), `queueMicrotask`. In Node also `process.nextTick` (runs even before other microtasks).
4. **Macrotask (callback) queue** – `setTimeout`, `setInterval`, I/O callbacks, `setImmediate`.

**The rule:** When the call stack is empty → run **all** microtasks → then take **one** macrotask → again run all microtasks → repeat.

```js
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");
// Output: 1 4 3 2
```
Why: 1 and 4 are synchronous. 3 is a microtask (runs as soon as stack is empty). 2 is a macrotask (runs after microtasks).

**Q. Why is setTimeout(fn, 0) not instant?**

> 🎤 **Interview answer:** "Zero is only the minimum delay, not a guarantee. The callback goes into the macrotask queue, and it can only run once the current call stack is empty and all pending microtasks are done. If there's heavy synchronous work or a long chain of promises, the timeout runs much later. Browsers also clamp nested timers to about 4 milliseconds."

0 ms is the minimum delay. The callback still has to wait until the stack is empty and all microtasks are finished.

**Q. Can microtasks block the page?**

> 🎤 **Interview answer:** "Yes. The event loop drains the entire microtask queue before it moves on. If a microtask keeps scheduling new microtasks, for example a recursive `Promise.resolve().then(...)`, macrotasks and rendering never get a turn, so the page freezes. In Node the same thing happens with recursive `process.nextTick`: it starves I/O. If I need to break up long work, I yield with `setTimeout` or `setImmediate` so other tasks can run."

Yes. If a microtask keeps adding new microtasks, the macrotask queue (and rendering) never gets a turn.

Node.js specific event loop phases are in [03 Node.js → Event loop](03-nodejs-express.md#3-event-loop-in-nodejs).

---

### 6. map, filter, reduce

> 🎤 **Interview answer:** "All three are higher-order array methods and none of them changes the original array. `map` transforms every item and returns a new array of the same length, like getting a list of emails from users. `filter` keeps only the items that pass a condition, like active users. `reduce` combines the array into one value, like a cart total, or grouping orders by status into an object. I often chain them, like `users.filter(u => u.active).map(u => u.email)`. For very large data, though, I'd rather let the database do the filtering than load everything into Node."

All three do **not** change the original array. They return new values.

```js
const nums = [1, 2, 3, 4];
nums.map(n => n * 2);             // [2, 4, 6, 8]   – transform each item
nums.filter(n => n % 2 === 0);    // [2, 4]         – keep matching items
nums.reduce((sum, n) => sum + n, 0); // 10          – combine into one value
```

**Real examples:**
```js
// Total cart price
const total = cart.reduce((sum, item) => sum + item.price * item.qty, 0);

// Group orders by status
const byStatus = orders.reduce((acc, o) => {
  (acc[o.status] ||= []).push(o);
  return acc;
}, {});

// Active user emails
const emails = users.filter(u => u.active).map(u => u.email);
```

**Q. map vs forEach?**

> 🎤 **Interview answer:** "`map` returns a new array built from the return value of each callback, so I use it when I'm transforming data. `forEach` returns `undefined` and is only for side effects like logging or pushing into something else. Neither one can be stopped early with `break`. For that I use `for...of`, `some` or `find`. And neither waits for async callbacks."

`map` returns a new array. `forEach` returns `undefined` and is used only for side effects (logging, pushing).

**Q. What if you don't give an initial value to reduce?**

> 🎤 **Interview answer:** "Then reduce uses the first element as the accumulator and starts from the second one. That works for simple sums, but on an empty array it throws a TypeError. If the accumulator type is different from the elements, like building an object from an array, the result is wrong. So I always pass an initial value, like `0`, `{}` or `[]`."

It uses the first element as the start value. On an empty array this throws a `TypeError`. Always pass an initial value.

**Other useful methods:** `find`, `findIndex`, `some`, `every`, `includes`, `flat`, `flatMap`, `sort` (changes original array!), `slice` (no change) vs `splice` (changes original).

Polyfills for these are in [02 Coding](02-javascript-coding.md#polyfills).

---

### 7. Debouncing and throttling

> 🎤 **Interview answer:** "Both control how often a function runs when an event fires many times. Debounce waits until the calls stop for a given time and then runs once. It's perfect for a search box, so we hit the search API only after the user stops typing for 300 milliseconds. Throttle runs the function at most once per interval, no matter how many times it's called. It's good for scroll or resize handlers, or stopping button spam.
>
> Both are built with closures: debounce keeps a timer id and clears it on every call, and throttle keeps a 'waiting' flag. On the backend the same idea is rate limiting, which I've implemented with Redis to protect login and OTP APIs."

**Debounce:** Run the function only after the user **stops** calling it for X ms.
Use: search box API call, auto-save, window resize end.

**Throttle:** Run the function **at most once** every X ms, no matter how many times it is called.
Use: scroll events, button spam, tracking mouse move, infinite scroll.

```js
function debounce(fn, delay) {
  let timer;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

function throttle(fn, limit) {
  let waiting = false;
  return function (...args) {
    if (waiting) return;
    fn.apply(this, args);
    waiting = true;
    setTimeout(() => (waiting = false), limit);
  };
}

const search = debounce((q) => fetch(`/api/search?q=${q}`), 300);
```

**Easy way to remember:**
- Debounce = lift door waits until people stop entering, then closes.
- Throttle = a tap that lets out one drop every second.

Backend version of throttling is **rate limiting** → see [03 Node → Rate limiting](03-nodejs-express.md#10-api-rate-limiting).

---

### 8. Shallow copy vs deep copy

> 🎤 **Interview answer:** "A shallow copy copies only the top level. Nested objects are still shared by reference, so changing `copy.address.city` also changes the original. Spread and `Object.assign` give shallow copies. A deep copy duplicates every level, so nothing is shared. Today I use `structuredClone`, which handles Dates, Maps, Sets and circular references. `JSON.parse(JSON.stringify())` is a common trick, but it loses functions and `undefined`, turns Dates into strings and fails on circular references. This matters a lot in Redux and React state, where accidentally mutating a shared nested object causes bugs that are hard to find."

**Shallow copy:** copies only the first level. Nested objects are still shared (same reference).
**Deep copy:** copies every level. Nothing is shared.

```js
const user = { name: "Yash", address: { city: "Gurugram" } };

const shallow = { ...user };        // or Object.assign({}, user)
shallow.address.city = "Delhi";
console.log(user.address.city);     // "Delhi"  -> original changed!

const deep = structuredClone(user); // modern way
deep.address.city = "Noida";
console.log(user.address.city);     // "Delhi"  -> original safe
```

**Ways to deep copy:**
| Method | Problem |
|---|---|
| `JSON.parse(JSON.stringify(obj))` | Loses functions, `undefined`, `Date` becomes string, fails on circular refs |
| `structuredClone(obj)` | Best built-in; handles Date, Map, Set, circular. Cannot copy functions |
| `lodash.cloneDeep` | Works well, extra library |
| Custom recursive function | See [02 Coding → deep clone](02-javascript-coding.md#q12-deep-clone-an-object) |

**Q. Primitive vs reference types?**

> 🎤 **Interview answer:** "Primitives, which are string, number, boolean, null, undefined, symbol and bigint, are stored and copied by value, so assigning one to another variable gives an independent copy. Objects, arrays and functions are reference types: the variable holds a reference, so two variables can point to the same object, and changing it through one is visible through the other. That's also why `{} === {}` is false: it compares references, not content."

Primitives (`string, number, boolean, null, undefined, symbol, bigint`) are copied by value. Objects, arrays and functions are copied by reference (both variables point to the same object).

---

### 9. this, call, apply, bind

**Q. What is `this`?**

> 🎤 **Interview answer:** "`this` is decided by how a function is called, not where it is written. When it's called as `obj.method()`, `this` is `obj`. As a plain function call, it's `undefined` in strict mode, or the global object otherwise. With `new` it's the newly created object, and with `call`, `apply` or `bind` it's whatever we pass. Arrow functions are the exception: they don't have their own `this` and take it from the surrounding scope. A common bug is passing a method as a callback, like `setTimeout(obj.method)`: it loses `this`, and the fix is `bind` or an arrow function."

`this` is the object that is calling the function. Its value depends on **how** the function is called:

| How it is called | `this` is |
|---|---|
| `obj.method()` | `obj` |
| Plain `fn()` | `undefined` in strict mode, global object otherwise |
| `new Fn()` | the new object |
| `fn.call(x)` / `apply(x)` / `bind(x)` | `x` |
| Arrow function | `this` of the outer scope (arrow has no own `this`) |
| Event listener (normal function) | the element |

```js
const user = {
  name: "Yash",
  normal() { console.log(this.name); },
  arrow: () => console.log(this.name),
};
user.normal(); // "Yash"
user.arrow();  // undefined – arrow takes 'this' from outside the object

const fn = user.normal;
fn();          // undefined – lost 'this' because called without the object
```

**Q. call vs apply vs bind**

> 🎤 **Interview answer:** "All three let me set `this` manually. `call` invokes the function immediately and takes the arguments one by one. `apply` also invokes immediately but takes the arguments as an array. `bind` doesn't invoke at all: it returns a new function with `this`, and optionally some arguments, fixed for later. I remember them as Call is Comma, Apply is Array, Bind is Back later. The most practical use of `bind` is passing a class method as a callback without losing `this`. I've also written a bind polyfill, which is a common interview question."

All three set `this` manually.
```js
function intro(city, country) {
  console.log(`${this.name} from ${city}, ${country}`);
}
const me = { name: "Yash" };

intro.call(me, "Gurugram", "India");     // runs now, args one by one
intro.apply(me, ["Gurugram", "India"]);  // runs now, args as array
const later = intro.bind(me, "Gurugram"); // returns new function, runs later
later("India");
```
Trick to remember: **C**all = **C**omma, **A**pply = **A**rray, **B**ind = **B**ack later.

Use of bind: passing a class method as a callback (`setTimeout(this.save.bind(this))`) so it does not lose `this`.

---

### 10. Prototypes and inheritance

**Q. What is a prototype?**

> 🎤 **Interview answer:** "Every object in JavaScript has a hidden link to another object, called its prototype. When I access a property that isn't on the object itself, JavaScript looks it up on the prototype, then that prototype's prototype, and so on until it reaches `null`. That chain is the prototype chain. That's how an array can call `map`: the method lives on `Array.prototype`, not on each array."

Every JavaScript object has a hidden link to another object called its **prototype**. If a property is not found on the object, JS looks at the prototype, then the prototype's prototype, and so on. This chain is called the **prototype chain**. It ends at `null`.

```js
const arr = [1, 2];
// arr -> Array.prototype (has map, filter...) -> Object.prototype (has toString...) -> null
```

**Q. How does inheritance work in JS?**

> 🎤 **Interview answer:** "JavaScript uses prototypal inheritance: objects inherit directly from other objects through the prototype chain. The ES6 `class` and `extends` syntax is mostly syntax sugar over this. When `Dog extends Animal`, `Dog.prototype` is linked to `Animal.prototype`, so a dog instance can use Animal's methods unless Dog overrides them. `super` calls the parent constructor or method. In NestJS and TypeScript I use classes daily, but underneath it's still prototypes."

Objects inherit from other objects through the prototype chain. `class` is just cleaner syntax over prototypes.

```js
// Old way
function Animal(name) { this.name = name; }
Animal.prototype.speak = function () { return `${this.name} makes a sound`; };

// Class way (same thing underneath)
class Dog extends Animal {
  speak() { return `${this.name} barks`; }
}
new Dog("Tommy").speak(); // "Tommy barks"
```

**Q. `__proto__` vs `prototype`?**

> 🎤 **Interview answer:** "`prototype` is a property that exists on constructor functions and classes. It's the object that will become the parent of instances created with `new`. `__proto__` is the actual link on an instance that points to its parent. So `new Dog().__proto__ === Dog.prototype` is true. In real code I'd use `Object.getPrototypeOf` instead of `__proto__`, because `__proto__` is a legacy accessor."

- `prototype` is a property on **functions/classes**. It is the object that will become the parent of instances created with `new`.
- `__proto__` (or `Object.getPrototypeOf(obj)`) is the actual link on an **object** to its parent.
- `new Dog().__proto__ === Dog.prototype` → `true`.

**Q. Why put methods on prototype instead of inside the constructor?**

> 🎤 **Interview answer:** "If I define a method inside the constructor, every instance gets its own copy of that function, which wastes memory when there are many objects. Methods on the prototype are created once and shared by all instances through the chain. Class methods automatically go on the prototype, which is one reason classes are efficient."

Methods on the prototype are shared by all instances (one copy in memory). Methods inside the constructor are created again for each object.

---

## Extra topics often asked

### == vs ===

> 🎤 **Interview answer:** "`==` is loose equality: it converts both values to a common type before comparing, so `'5' == 5` is true and even `0 == ''` is true. `===` is strict equality and checks both type and value, so there are no surprises. I always use `===`, and ESLint enforces it in our projects. The only place I've seen `== null` used on purpose is as a shortcut to check null or undefined together."

`==` converts types before comparing (`"5" == 5` → true). `===` checks value and type (`"5" === 5` → false). Always use `===`.

### null vs undefined

> 🎤 **Interview answer:** "`undefined` means a value was never assigned. JavaScript gives it to uninitialised variables, missing object properties and functions that return nothing. `null` is an intentional 'no value' that the developer sets. A known quirk is that `typeof null` returns `'object'`, which is an old bug. In APIs I use `null` explicitly for fields that exist but are empty, and I use `??` to handle both cases."

- `undefined` – variable declared but no value given (set by JS).
- `null` – "no value" set on purpose by the developer.
- `typeof null` is `"object"` (old bug in JS).

### Truthy and falsy

> 🎤 **Interview answer:** "Falsy values are the ones that become false in a boolean context: `false`, `0`, `-0`, `0n`, empty string, `null`, `undefined` and `NaN`. Everything else is truthy, including empty arrays, empty objects and the string `'0'`. This matters with defaults: `limit || 10` would replace a valid `0` with 10, so I use `??` when 0 or an empty string are valid values."

Falsy values: `false, 0, -0, 0n, "", null, undefined, NaN`. Everything else is truthy, including `[]`, `{}`, `"0"`, `"false"`.

### Data types

> 🎤 **Interview answer:** "JavaScript has seven primitive types: string, number, bigint, boolean, undefined, null and symbol. Everything else is an object, including arrays, functions, dates and maps. `typeof` helps tell them apart, but it has quirks: `typeof null` is `'object'`, and arrays are also `'object'`, so I use `Array.isArray`. For money I avoid plain numbers because of floating-point errors. I store the smallest unit, like paise, as an integer, or use a decimal type."

7 primitives: `string, number, bigint, boolean, undefined, null, symbol`. Everything else is an `object` (arrays, functions, dates).

### Arrow function vs normal function

> 🎤 **Interview answer:** "The biggest difference is `this`. Arrow functions don't have their own `this`, they take it from the surrounding scope. A normal function's `this` depends on how it's called. Arrow functions also have no `arguments` object and can't be used as constructors with `new`. So I use arrows for callbacks, like in map, filter and promise chains, where inheriting `this` is what I want. I use normal functions or class methods for object methods and constructors."

| | Normal | Arrow |
|---|---|---|
| Own `this` | Yes | No (takes from outside) |
| `arguments` object | Yes | No (use `...args`) |
| Can use with `new` | Yes | No |
| Good for | Object methods, constructors | Callbacks, short functions |

### Spread vs rest

> 🎤 **Interview answer:** "It's the same `...` syntax doing opposite things. Spread expands an array or object into individual items, like merging arrays, cloning an object, or passing an array as function arguments. Rest collects multiple items into one: in function parameters, like `sum(...nums)`, or in destructuring. A practical example from my APIs is `const { password, ...safeUser } = user` to remove the password before sending the user in a response."

Same `...` syntax.
- **Spread** expands: `[...arr1, ...arr2]`, `{...obj, age: 30}`, `fn(...args)`.
- **Rest** collects: `function sum(...nums) {}`, `const { password, ...safeUser } = user;`

### Destructuring

> 🎤 **Interview answer:** "Destructuring lets me pull values out of objects or arrays into variables in one line. With objects I can rename, set defaults and go into nested fields, like `const { page = 1, limit = 10 } = req.query`. With arrays it goes by position, and it's very handy with `Promise.all`: `const [users, count] = await Promise.all([...])`. It keeps code short and readable, but I'm careful with deep nesting on data that might be missing, because destructuring `undefined` throws."

```js
const { name, address: { city }, role = "user" } = user;
const [first, , third] = arr;
```

### Optional chaining and nullish coalescing

> 🎤 **Interview answer:** "Optional chaining, `?.`, safely accesses nested properties. If anything in the chain is null or undefined, it returns undefined instead of throwing 'cannot read property of undefined'. Nullish coalescing, `??`, gives a fallback only when the value is null or undefined. That's different from `||`, which also replaces 0, empty string and false. So for something like a pagination limit where 0 could be valid, `??` is the correct choice."

```js
user?.address?.city          // undefined instead of crash
const limit = req.query.limit ?? 10; // only null/undefined fall back
const limit2 = req.query.limit || 10; // also 0 and "" fall back
```

### Higher-order functions

> 🎤 **Interview answer:** "A higher-order function either takes a function as an argument or returns a function. `map`, `filter` and `reduce` take functions. Debounce, throttle and memoize take a function and return a new, wrapped one. In Express, middleware factories like `authorize('admin')` or an `asyncHandler` wrapper are higher-order functions I write regularly. They let me reuse behaviour without repeating code."

A function that takes a function as argument or returns a function. Examples: `map`, `filter`, `debounce`, Express middleware factories.

### Pure functions

> 🎤 **Interview answer:** "A pure function always gives the same output for the same input and has no side effects: no API calls, no DB writes, no changes to outside variables. That makes them easy to test and reason about, and safe to memoize. Redux reducers must be pure. In my backend code I try to keep business calculations, like price, discount or reward calculation, as pure functions and keep the side effects in the service layer, so unit tests don't need mocks."

Same input → same output, and no side effects (no API call, no change of outside variables). Easy to test. Reducers in Redux must be pure.

### Currying

> 🎤 **Interview answer:** "Currying turns a function that takes several arguments into a chain of functions that each take one, so `f(a, b, c)` becomes `f(a)(b)(c)`. The benefit is creating pre-configured functions. For example, `discount(10)` returns a function that applies 10% to any price. Express middleware factories like `validate(schema)` or `authorize(role)` follow the same idea. A curry utility is also a common coding question."

Turning `f(a, b, c)` into `f(a)(b)(c)`. Useful to create pre-filled functions.
```js
const discount = (pct) => (price) => price - price * pct / 100;
const tenOff = discount(10);
tenOff(500); // 450
```

### IIFE (Immediately Invoked Function Expression)

> 🎤 **Interview answer:** "An IIFE is a function that's defined and executed immediately, by wrapping it in parentheses and calling it. Before ES modules and block scope existed, it was the main way to create a private scope and avoid polluting global variables. Today I mostly see it for running top-level async code in older Node scripts, like `(async () => { await main(); })()`."

```js
(function () { /* runs once, private scope */ })();
```
Used before modules existed to avoid polluting global scope.

### Callback hell

> 🎤 **Interview answer:** "Callback hell is when async operations depend on each other and each one is nested inside the previous callback. The code drifts to the right, becomes hard to read, and error handling has to be repeated at every level. The fix is promises and async/await, which flatten the flow and allow one try/catch, plus splitting the logic into small named functions."

Many nested callbacks → hard to read and handle errors. Solution: promises, async/await, small named functions.

### Event bubbling, capturing and delegation (browser)

> 🎤 **Interview answer:** "When an event fires on an element, it first travels down from the window to the target, which is the capturing phase, and then bubbles back up through the parents, which is bubbling. By default, listeners run in the bubbling phase. Event delegation uses bubbling: instead of adding a listener to every item in a long list, I add one to the parent and check `e.target`. That's more efficient and also works for items added later. `stopPropagation` stops the event from going further, and `preventDefault` stops the browser's default action, like a form submit."

- **Bubbling:** event goes from the clicked element up to its parents.
- **Capturing:** event goes from the top down to the element (`addEventListener(type, fn, true)`).
- **Delegation:** attach one listener on the parent and check `e.target` – good for long lists.
- `e.stopPropagation()` stops bubbling. `e.preventDefault()` stops default action (like form submit).

### Map vs Object, Set, WeakMap

> 🎤 **Interview answer:** "A `Map` is a key-value collection where keys can be any type, even objects. It keeps insertion order, has a `size`, and performs better with frequent adds and deletes. A plain object is better for fixed, structured data and JSON. A `Set` stores unique values, so `[...new Set(arr)]` is the quickest way to remove duplicates. A `WeakMap` only takes objects as keys and holds them weakly, so if the object isn't referenced anywhere else, it gets garbage-collected. That makes it good for caching data about objects without causing memory leaks."

- `Map` – keys can be any type, keeps insertion order, has `.size`, better for frequent add/remove.
- `Set` – unique values. `[...new Set(arr)]` removes duplicates.
- `WeakMap` – keys must be objects; if the object is not used anywhere else, it is garbage-collected. Good for caching data about objects without memory leaks.

### Object.freeze vs Object.seal

> 🎤 **Interview answer:** "`Object.freeze` makes an object fully read-only: you can't add, delete or change properties. `Object.seal` stops adding and deleting properties, but you can still change the existing values. Both are shallow, so nested objects stay mutable unless you freeze them recursively. I've used freeze for config or constant objects, like enums in plain JavaScript."

- `freeze` – cannot add, delete or change properties.
- `seal` – cannot add or delete, but **can** change existing values.
- Both are shallow.

### CommonJS vs ES Modules

> 🎤 **Interview answer:** "CommonJS is Node's original module system with `require` and `module.exports`. It loads synchronously at runtime, so you can require conditionally. ES Modules use `import` and `export`. They're static, so they're analysed before the code runs, which allows tree shaking and top-level await, and they're the standard in browsers. In Node you enable ESM with `"type": "module"` or `.mjs` files. Most of my TypeScript code is written with import syntax and compiled based on the tsconfig module setting."

| CommonJS | ES Modules |
|---|---|
| `require` / `module.exports` | `import` / `export` |
| Loads synchronously at runtime | Static, analysed before running (allows tree shaking) |
| Default in older Node | Use `"type": "module"` or `.mjs` |

### Generators

> 🎤 **Interview answer:** "A generator is a function declared with `function*` that can pause at each `yield` and resume later when `next()` is called. It returns an iterator, so it's great for producing values lazily, like an infinite ID sequence or processing a large dataset in chunks. Async generators with `for await...of` are useful for streaming paginated API results or reading streams. Conceptually, async/await is built on the same pause-and-resume idea."

Functions that can pause with `yield` and continue later. `function* gen() { yield 1; yield 2; }`. Used for lazy sequences; async/await is built on a similar idea.

### Garbage collection and memory leaks

> 🎤 **Interview answer:** "JavaScript manages memory automatically with a mark-and-sweep garbage collector: anything that isn't reachable from the roots, like global variables and the call stack, gets freed. A memory leak happens when we keep references to things we no longer need. Common causes in Node are an in-memory cache or array that keeps growing with no limit, timers or intervals that are never cleared, event listeners that are never removed, and closures holding large objects. To find one, I watch memory growth with `process.memoryUsage()` or PM2 metrics, take heap snapshots with Chrome DevTools and compare them. The fixes are usually an LRU limit or TTL, cleaning up listeners, or moving the cache to Redis."

JS frees memory of objects that are no longer reachable (mark-and-sweep). Common leaks:
- Global variables that keep growing (e.g. an in-memory cache with no limit).
- Timers/intervals not cleared.
- Event listeners not removed.
- Closures holding large objects.

### Strict mode

> 🎤 **Interview answer:** "Strict mode, enabled with `'use strict'`, makes JavaScript throw errors for mistakes it would otherwise ignore silently, like assigning to an undeclared variable or writing to a read-only property. It also makes `this` `undefined` in plain function calls instead of the global object. ES modules and classes are strict by default, so in modern TypeScript code we're always in strict mode."

`"use strict"` – throws errors for silent mistakes (e.g. assigning to undeclared variable), `this` is `undefined` in plain functions. Classes and ES modules are strict by default.

### Error handling in JS

> 🎤 **Interview answer:** "For synchronous code and `await` I use `try/catch/finally`. For promise chains I use `.catch`. An async function that throws doesn't crash immediately: it returns a rejected promise, so someone has to await or catch it, otherwise it's an unhandled rejection. In my backend I create custom error classes like `AppError` or `NotFoundError` with a status code, throw them from services, and have one central Express error middleware that formats the response and logs the error. I also add `process.on('unhandledRejection')` and `uncaughtException` handlers that log and shut down gracefully."

- `try/catch/finally` for sync code and `await`.
- `.catch()` for promises.
- Custom errors: `class NotFoundError extends Error {}`.
- An async function that throws returns a **rejected promise** – see [11 Scenarios → Q11](11-scenario-questions.md#11-what-happens-when-an-async-function-throws-an-error).

### Memoization

> 🎤 **Interview answer:** "Memoization means caching a function's result for a given input, so the next call with the same input returns the cached value instead of recomputing. It's built with a closure holding a Map or object keyed by the arguments. It only makes sense for pure, expensive functions. In React, `useMemo` and `React.memo` use the same idea. On the backend, the same concept at a larger scale is caching results in Redis."

Cache results of a function for the same input. Example in [02 Coding](02-javascript-coding.md#q15-memoize-function).

---

[Home](README.md) | Next: [02 JavaScript Coding →](02-javascript-coding.md)
