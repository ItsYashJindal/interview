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

> 🎤 **Interview answer:** "There are three differences: where the variable is visible, whether we can change it, and what happens if we use it before the line where it is declared.
>
> `var` is visible in the whole function. It ignores `if` and `for` blocks. `let` and `const` are visible only inside the nearest `{ }` block.
>
> `var` and `let` can be given a new value. `const` cannot.
>
> If I use `var` before its line, I get `undefined`. If I use `let` or `const` before their line, I get an error.
>
> In my code I use `const` by default, `let` only when the value changes, like a counter, and I never use `var`."

| | var | let | const |
|---|---|---|---|
| Visible in | Whole function | Only the `{}` block | Only the `{}` block |
| Declare again with same name | Yes | No | No |
| Give a new value | Yes | Yes | No |
| Used before its line | `undefined` | Error | Error |
| Added to `window` (global) | Yes | No | No |

```js
if (true) {
  var a = 1;
  let b = 2;
}
console.log(a); // 1  (var ignores the block)
console.log(b); // ReferenceError (b lives only inside the block)
```

**Q. Can we change a const object?**

> 🎤 **Interview answer:** "Yes, we can change what is inside the object. `const` only stops us from pointing the variable to a new value. So `user.name = 'Rahul'` works, but `user = {}` gives an error. If I want the object to be fully read-only, I use `Object.freeze`. But freeze only works on the top level. Objects inside it can still change."

```js
const user = { name: "Yash" };
user.name = "Rahul"; // OK – changing inside the object
user = {};           // TypeError – pointing to a new object is not allowed
```

**Q. Classic loop question**

> 🎤 **Interview answer:** "With `var` it prints 3, 3, 3. With `let` it prints 0, 1, 2. With `var`, there is only one `i` for the whole loop. The setTimeout code runs after the loop finishes, and by then `i` is 3. With `let`, every loop round gets its own fresh `i`, so each setTimeout remembers its own value."

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0); // 3 3 3
for (let i = 0; i < 3; i++) setTimeout(() => console.log(i), 0); // 0 1 2
```
`var` = one shared `i` for all rounds. `let` = a new `i` for every round.

**What to use:** `const` by default, `let` when the value changes, avoid `var`.

---

### 2. Hoisting and Temporal Dead Zone (TDZ)

**Q. What is hoisting?**

> 🎤 **Interview answer:** "Before running the code, JavaScript first scans it and notes down all the variables and functions. This is called hoisting. Because of this:
> - A normal function can be called even before the line where it is written.
> - A `var` variable can be read before its line, but its value is `undefined`.
> - `let` and `const` cannot be used before their line. JavaScript throws an error.
> - Arrow functions and functions stored in variables follow the same rule as the variable (`var`, `let` or `const`)."

```js
console.log(x); // undefined
var x = 5;

console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 5;

greet();        // works – normal functions are fully hoisted
function greet() { console.log("hi"); }

sayHi();        // TypeError: sayHi is not a function (it is still undefined here)
var sayHi = function () {};
```

**Q. What is the Temporal Dead Zone?**

> 🎤 **Interview answer:** "It is the part of the code from the start of the block up to the line where a `let` or `const` variable is declared. In that part, the variable exists but we are not allowed to use it. If we try, we get an error. This is good, because it catches the bug of using a variable before giving it a value. With `var`, we would silently get `undefined` instead."

```js
{
  // 'a' cannot be used here (TDZ)
  console.log(a); // ReferenceError
  let a = 10;     // from here, 'a' can be used
}
```

---

### 3. Closures and lexical scope

**Q. What is lexical scope?**

> 🎤 **Interview answer:** "It means a function can use variables from the place where it is **written** in the code. An inner function can see the variables of the outer function. But the outer function cannot see the variables inside the inner function. Closures work because of this rule."

**Q. What is a closure?**

> 🎤 **Interview answer:** "A closure is a function that remembers the variables around it, even after the outer function has finished running.
>
> Example: a `counter` function creates a `count` variable and returns an inner function that does `count++`. Nobody outside can touch `count`, but the inner function still remembers it and keeps increasing it.
>
> I use closures often. My debounce function remembers the timer in a closure. In Express, `authorize('admin')` returns a middleware that remembers the role 'admin'. One thing to be careful about: if a closure holds a big object and the closure lives forever, that memory is never freed."

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
- Private variables (nobody outside can change them).
- Debounce / throttle (the timer is remembered).
- Memoization (the cache object is remembered).
- Pre-filled functions: `const add5 = makeAdder(5)`.
- Express middleware: `const auth = (role) => (req, res, next) => {...}`.

**Be careful:** A closure that holds a big object can cause a memory leak if the function is kept alive forever (for example, a global event listener).

---

### 4. Promises and async/await

**Q. What is a Promise?**

> 🎤 **Interview answer:** "A Promise is an object that says: 'I don't have the result yet, but I will give it to you later.' For example, a database query or an API call.
>
> It has three states:
> - **pending** – still working,
> - **fulfilled** – finished successfully with a value,
> - **rejected** – failed with an error.
>
> Once it is fulfilled or rejected, it never changes again. We read the result with `.then` and `.catch`, or with `async/await`."

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve("done"), 1000);
});
p.then(v => console.log(v)).catch(e => console.error(e)).finally(() => console.log("end"));
```

**Q. Why Promises over callbacks?**

> 🎤 **Interview answer:** "With callbacks, when one step depends on another, we keep putting callbacks inside callbacks. The code goes deeper and deeper and becomes hard to read. This is called 'callback hell'. We also have to handle errors at every level.
>
> With promises, we write the steps one after another in a flat chain, and one `.catch` handles all errors. We also get helpers like `Promise.all` to run many things together."

**Q. What is async/await?**

> 🎤 **Interview answer:** "async/await is a simpler way to write promise code. It works on top of promises. It does not replace them.
>
> - When I write `async` before a function, that function always returns a promise.
> - When I write `await` before a promise, that function waits for the result, and then continues to the next line. Only this function waits. The rest of the app keeps running.
> - For errors I can use a normal `try/catch`.
>
> So the code looks like normal top-to-bottom code, which makes it much easier to read."

```js
// Same thing written in 2 ways

// With .then
function getUser(id) {
  return User.findById(id)
    .then(user => user)
    .catch(err => { console.error(err); throw err; });
}

// With async/await – reads top to bottom
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

> 🎤 **Interview answer:**
> - "`Promise.all` – waits for all promises. If even one fails, the whole thing fails. I use it when I need every result, like loading user, orders and wallet together.
> - `Promise.allSettled` – waits for all promises and tells me which passed and which failed. It never fails. Good for sending notifications to many users.
> - `Promise.race` – gives me whichever promise finishes first, pass or fail. I use it to add a timeout to a slow API call.
> - `Promise.any` – gives me the first one that **passes**. It fails only if all of them fail."

| Method | Gives result when | Fails when |
|---|---|---|
| `Promise.all` | All pass (array of results) | Any one fails |
| `Promise.allSettled` | All finish (pass or fail), with status of each | Never |
| `Promise.race` | First one finishes | The first one to finish failed |
| `Promise.any` | First one passes | All fail |

**Q. Sequential vs parallel await**

> 🎤 **Interview answer:** "If I `await` two calls one after another, the second one starts only after the first one finishes. So total time = time of A + time of B. If the two calls don't need each other's result, I start both together with `Promise.all`. Then total time = only the slower one. This is an easy way to make APIs faster."

```js
// One after another – slow (A + B)
const a = await getA();
const b = await getB();

// Together – fast (whichever is slower)
const [a, b] = await Promise.all([getA(), getB()]);
```

**Q. await inside forEach?**

> 🎤 **Interview answer:** "`forEach` does not wait for `await`. It starts all the callbacks and moves on immediately. So the code after the loop runs before the work is done. If I need one by one, I use `for...of` with `await`. If they can run together, I use `Promise.all(items.map(...))`."

```js
// Wrong – does not wait
items.forEach(async (item) => await save(item));

// One by one
for (const item of items) await save(item);

// All together
await Promise.all(items.map(item => save(item)));
```

---

### 5. Event loop, call stack, microtasks

**Q. Explain the event loop.**

> 🎤 **Interview answer:** "JavaScript can do only one thing at a time. It has one **call stack**, which is the list of functions running right now.
>
> Slow work like timers, API calls and file reading is not done on the call stack. It is handed over to the browser or Node. When that work is done, its callback is put in a waiting line (queue).
>
> There are two waiting lines:
> - **Microtask queue** (high priority) – promise `.then` and code after `await`. In Node, `process.nextTick` runs even before these.
> - **Macrotask queue** (normal priority) – `setTimeout`, `setInterval`, I/O callbacks, `setImmediate`.
>
> The event loop follows a simple rule: when the call stack is empty, first run **all** microtasks, then run **one** macrotask, then again all microtasks, and so on.
>
> That's why promise code runs before setTimeout code, even if the timeout is 0."

```js
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");
// Output: 1 4 3 2
```
Why: 1 and 4 are normal code, so they run first. 3 is a promise (high priority). 2 is setTimeout (normal priority), so it runs last.

**Q. Why is setTimeout(fn, 0) not instant?**

> 🎤 **Interview answer:** "0 ms only means 'run as soon as possible'. The callback still has to wait until the current code finishes and all promise callbacks are done. If something heavy is running, the timeout will run much later."

**Q. Can microtasks block the page?**

> 🎤 **Interview answer:** "Yes. The event loop finishes all microtasks before doing anything else. If a microtask keeps adding new microtasks forever, setTimeout callbacks and page updates never get a chance to run. The page freezes. In Node the same happens with endless `process.nextTick`."

Node.js specific event loop phases are in [03 Node.js → Event loop](03-nodejs-express.md#3-event-loop-in-nodejs).

---

### 6. map, filter, reduce

> 🎤 **Interview answer:** "None of these three change the original array. They give back a new result.
> - `map` changes every item and gives a new array of the same length. Example: get all emails from a list of users.
> - `filter` keeps only the items that pass a condition. Example: only active users.
> - `reduce` turns the whole array into one value. Example: total price of a cart.
>
> I often use them together, like `users.filter(u => u.active).map(u => u.email)`."

```js
const nums = [1, 2, 3, 4];
nums.map(n => n * 2);                // [2, 4, 6, 8]   – change each item
nums.filter(n => n % 2 === 0);       // [2, 4]         – keep matching items
nums.reduce((sum, n) => sum + n, 0); // 10             – combine into one value
```

**Real examples:**
```js
// Total cart price
const total = cart.reduce((sum, item) => sum + item.price * item.qty, 0);

// Group orders by status
const byStatus = orders.reduce((groups, order) => {
  if (!groups[order.status]) groups[order.status] = [];
  groups[order.status].push(order);
  return groups;
}, {});

// Active user emails
const emails = users.filter(u => u.active).map(u => u.email);
```

**Q. map vs forEach?**

> 🎤 **Interview answer:** "`map` gives back a new array. `forEach` gives back nothing (`undefined`). So I use `map` when I want a new array, and `forEach` when I just want to do something with each item, like logging."

**Q. What if you don't give a starting value to reduce?**

> 🎤 **Interview answer:** "Then reduce uses the first item as the starting value. On an empty array this throws an error. So I always give a starting value, like `0`, `{}` or `[]`."

**Other useful methods:** `find`, `findIndex`, `some`, `every`, `includes`, `flat`, `sort` (changes the original array!), `slice` (does not change original) vs `splice` (changes original).

Polyfills for these are in [02 Coding](02-javascript-coding.md#polyfills).

---

### 7. Debouncing and throttling

> 🎤 **Interview answer:** "Both are used when an event fires many times very quickly, like typing or scrolling.
>
> **Debounce** waits until the user **stops** for some time, then runs once. Example: a search box. We call the search API only after the user stops typing for 300 ms, not on every key press.
>
> **Throttle** runs the function **at most once** every X ms, no matter how many times the event fires. Example: scroll events, or stopping a user from clicking a button 10 times.
>
> On the backend, the same idea is rate limiting, which I built with Redis for login and OTP APIs."

```js
function debounce(fn, delay) {
  let timer;
  return function (...args) {
    clearTimeout(timer);                       // cancel the old timer
    timer = setTimeout(() => fn(...args), delay); // start a new one
  };
}

function throttle(fn, limit) {
  let waiting = false;
  return function (...args) {
    if (waiting) return;     // still in cool-down, ignore
    fn(...args);
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

> 🎤 **Interview answer:** "A **shallow copy** copies only the top level. Objects inside it are still shared with the original. So if I change `copy.address.city`, the original also changes. Spread `{...obj}` and `Object.assign` make shallow copies.
>
> A **deep copy** copies every level, so nothing is shared. Today I use `structuredClone` for this. The old trick `JSON.parse(JSON.stringify(obj))` also works, but it loses functions and `undefined`, and turns dates into strings."

```js
const user = { name: "Yash", address: { city: "Gurugram" } };

const shallow = { ...user };
shallow.address.city = "Delhi";
console.log(user.address.city);     // "Delhi"  -> original changed!

const deep = structuredClone(user);
deep.address.city = "Noida";
console.log(user.address.city);     // "Delhi"  -> original safe
```

**Ways to deep copy:**
| Method | Problem |
|---|---|
| `JSON.parse(JSON.stringify(obj))` | Loses functions and `undefined`, Date becomes a string |
| `structuredClone(obj)` | Best built-in way. Cannot copy functions |
| `lodash.cloneDeep` | Works well, but needs a library |
| Your own function | See [02 Coding → deep clone](02-javascript-coding.md#q12-deep-clone-an-object) |

**Q. Primitive vs reference types?**

> 🎤 **Interview answer:** "Simple values like string, number and boolean are **copied**. If I do `b = a` and change `b`, `a` does not change.
>
> Objects, arrays and functions are **shared**. If I do `b = a` and change `b`, `a` also changes, because both point to the same object. That's also why `{} === {}` is false: they are two different objects, even if they look the same."

---

### 9. this, call, apply, bind

**Q. What is `this`?**

> 🎤 **Interview answer:** "`this` is the object that called the function. It depends on **how** the function is called, not where it is written.
> - `user.greet()` → `this` is `user`.
> - Just `greet()` → `this` is `undefined` (in strict mode).
> - `new User()` → `this` is the new object.
> - Arrow functions don't have their own `this`. They use the `this` of the code around them.
>
> A common bug: `setTimeout(user.greet)` loses `this`, because the function is called without `user.` in front. Fix: use `bind` or an arrow function."

| How it is called | `this` is |
|---|---|
| `obj.method()` | `obj` |
| Plain `fn()` | `undefined` in strict mode |
| `new Fn()` | the new object |
| `fn.call(x)` / `apply(x)` / `bind(x)` | `x` |
| Arrow function | `this` of the outer code |

```js
const user = {
  name: "Yash",
  normal() { console.log(this.name); },
  arrow: () => console.log(this.name),
};
user.normal(); // "Yash"
user.arrow();  // undefined – arrow takes 'this' from outside the object

const fn = user.normal;
fn();          // undefined – called without 'user.' so 'this' is lost
```

**Q. call vs apply vs bind**

> 🎤 **Interview answer:** "All three let me choose what `this` should be.
> - `call` – runs the function now. Arguments are given one by one.
> - `apply` – runs the function now. Arguments are given as an array.
> - `bind` – does not run it. It gives back a new function with `this` fixed, to run later.
>
> Easy trick: **C**all = **C**omma, **A**pply = **A**rray, **B**ind = **B**ack later."

```js
function intro(city, country) {
  console.log(`${this.name} from ${city}, ${country}`);
}
const me = { name: "Yash" };

intro.call(me, "Gurugram", "India");      // runs now, args one by one
intro.apply(me, ["Gurugram", "India"]);   // runs now, args as array
const later = intro.bind(me, "Gurugram"); // returns a new function
later("India");                           // runs later
```

---

### 10. Prototypes and inheritance

**Q. What is a prototype?**

> 🎤 **Interview answer:** "Every object in JavaScript has a hidden link to a parent object, called its prototype. If I ask for a property that the object doesn't have, JavaScript checks the parent, then the parent's parent, and so on. This is called the prototype chain. That's why every array can use `map`: `map` is not stored in each array, it is stored once on `Array.prototype`, which is the parent of all arrays."

```js
const arr = [1, 2];
// arr -> Array.prototype (has map, filter...) -> Object.prototype (has toString...) -> null
```

**Q. How does inheritance work in JS?**

> 🎤 **Interview answer:** "In JavaScript, objects get methods from their parent object through the prototype chain. The `class` and `extends` keywords are just an easier way to write this. Under the hood, it still uses prototypes. When `Dog extends Animal`, a dog can use all of Animal's methods, unless Dog writes its own version. `super` calls the parent's constructor or method."

```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}

class Dog extends Animal {
  speak() { return `${this.name} barks`; } // own version
}
new Dog("Tommy").speak(); // "Tommy barks"
```

**Q. `__proto__` vs `prototype`?**

> 🎤 **Interview answer:** "`prototype` is found on classes and constructor functions. It is the object that will become the parent of every object created with `new`. `__proto__` is found on the object itself and points to its parent. So `new Dog().__proto__ === Dog.prototype` is true."

**Q. Why put methods on the prototype instead of inside the constructor?**

> 🎤 **Interview answer:** "If a method is written inside the constructor, every object gets its own copy, which wastes memory. If it is on the prototype, there is only one copy and all objects share it. Class methods go on the prototype automatically."

---

## Extra topics often asked

### == vs ===

> 🎤 **Interview answer:** "`==` changes the types before comparing, so `'5' == 5` is true. This causes surprises. `===` checks both type and value, so `'5' === 5` is false. I always use `===`."

### null vs undefined

> 🎤 **Interview answer:** "`undefined` means no value was given yet. JavaScript sets it by itself, for example for a variable with no value or a missing property. `null` means 'empty', and the developer sets it on purpose. A known quirk: `typeof null` is `'object'`. That is an old bug in JavaScript."

### Truthy and falsy

> 🎤 **Interview answer:** "Falsy values are the ones that act like `false` in an `if`: `false`, `0`, `''` (empty string), `null`, `undefined` and `NaN`. Everything else is truthy, even `[]`, `{}` and the string `'0'`.
>
> This matters for default values. `limit || 10` will replace a valid `0` with 10. So when 0 is a valid value, I use `limit ?? 10`."

### Data types

> 🎤 **Interview answer:** "JavaScript has 7 simple (primitive) types: string, number, bigint, boolean, undefined, null and symbol. Everything else is an object, including arrays, functions and dates. To check for an array I use `Array.isArray`, because `typeof []` is also `'object'`. For money I don't use decimal numbers like 10.25, because they give small rounding errors. I store paise as whole numbers instead."

### Arrow function vs normal function

> 🎤 **Interview answer:** "The main difference is `this`. An arrow function doesn't have its own `this`; it uses the `this` of the code around it. Arrow functions also can't be used with `new` and don't have `arguments`. I use arrow functions for callbacks like `map` and `filter`, and normal functions or class methods for object methods."

| | Normal | Arrow |
|---|---|---|
| Own `this` | Yes | No (takes from outside) |
| `arguments` object | Yes | No (use `...args`) |
| Can use with `new` | Yes | No |
| Good for | Object methods, constructors | Callbacks, short functions |

### Spread vs rest

> 🎤 **Interview answer:** "Both use `...` but do opposite jobs. **Spread** opens up an array or object into separate items, like `[...arr1, ...arr2]` to merge arrays. **Rest** collects many items into one, like `function sum(...nums)`. A real example from my APIs: `const { password, ...safeUser } = user` removes the password before sending the user in the response."

### Destructuring

> 🎤 **Interview answer:** "Destructuring lets me take values out of an object or array into variables in one line. For example `const { page = 1, limit = 10 } = req.query` takes page and limit, with default values. With arrays it goes by position: `const [users, count] = await Promise.all([...])`."

```js
const { name, address: { city }, role = "user" } = user;
const [first, , third] = arr;
```

### Optional chaining and nullish coalescing

> 🎤 **Interview answer:** "`?.` lets me read a deep property safely. If something in the middle is missing, I get `undefined` instead of a crash. `??` gives a default value only when the value is `null` or `undefined`. `||` gives the default also for `0`, `''` and `false`. So when 0 is a valid value, I use `??`."

```js
user?.address?.city                   // undefined instead of crash
const limit = req.query.limit ?? 10;  // only null/undefined use 10
const limit2 = req.query.limit || 10; // 0 and "" also use 10
```

### Higher-order functions

> 🎤 **Interview answer:** "A higher-order function is a function that takes another function as input, or returns a function. `map` and `filter` take a function. Debounce takes a function and returns a new one. In Express, `authorize('admin')` returns a middleware function."

### Pure functions

> 🎤 **Interview answer:** "A pure function always gives the same output for the same input, and does not change anything outside itself: no API call, no DB write. They are very easy to test. Redux reducers must be pure. In my backend I keep calculations like price or discount as pure functions."

### Currying

> 🎤 **Interview answer:** "Currying means turning `f(a, b, c)` into `f(a)(b)(c)`: a function that takes one argument at a time. It is useful to make pre-filled functions. For example, `discount(10)` gives back a function that takes 10% off any price."

```js
const discount = (pct) => (price) => price - price * pct / 100;
const tenOff = discount(10);
tenOff(500); // 450
```

### IIFE (Immediately Invoked Function Expression)

> 🎤 **Interview answer:** "An IIFE is a function that runs as soon as it is written. We wrap it in brackets and call it straight away. Earlier it was used to keep variables private. Today I mostly see it to run `await` code at the top of a script: `(async () => { await main(); })()`."

```js
(function () { /* runs once, right away */ })();
```

### Callback hell

> 🎤 **Interview answer:** "Callback hell is when we put callbacks inside callbacks inside callbacks. The code goes deeper and deeper, and is hard to read and to handle errors in. The fix is promises and async/await, which keep the code flat."

### Event bubbling, capturing and delegation (browser)

> 🎤 **Interview answer:** "When I click on a button, the click event also reaches its parent elements, going upward. That is **bubbling**. The opposite direction, from top down to the button, is **capturing**.
>
> **Event delegation** uses bubbling: instead of adding a click listener on 100 list items, I add one listener on the parent and check `e.target` to see which item was clicked.
>
> `stopPropagation()` stops the event from going up. `preventDefault()` stops the browser's default action, like submitting a form."

### Map vs Object, Set, WeakMap

> 🎤 **Interview answer:**
> - "`Map` – like an object, but keys can be any type, even objects. It keeps the order of insertion and has `.size`.
> - `Set` – stores only unique values. `[...new Set(arr)]` removes duplicates.
> - `WeakMap` – keys must be objects. If that object is not used anywhere else, it is removed from memory automatically, so it doesn't cause memory leaks."

### Object.freeze vs Object.seal

> 🎤 **Interview answer:** "`freeze` – you cannot add, delete or change anything. `seal` – you cannot add or delete properties, but you **can** change existing values. Both work only on the top level."

### CommonJS vs ES Modules

> 🎤 **Interview answer:** "CommonJS is Node's old way: `require` and `module.exports`. ES Modules is the new standard way: `import` and `export`. ES Modules can remove unused code at build time and allow `await` at the top level. In Node, we turn them on with `"type": "module"` in package.json."

| CommonJS | ES Modules |
|---|---|
| `require` / `module.exports` | `import` / `export` |
| Loaded while the code runs | Checked before the code runs |
| Default in older Node | Use `"type": "module"` or `.mjs` |

### Generators

> 🎤 **Interview answer:** "A generator is a function written with `function*`. It can pause at every `yield` and continue later when we call `next()`. It is useful when we want values one at a time instead of all at once, like reading a big list page by page."

```js
function* ids() { yield 1; yield 2; yield 3; }
const gen = ids();
gen.next().value; // 1
gen.next().value; // 2
```

### Garbage collection and memory leaks

> 🎤 **Interview answer:** "JavaScript frees memory by itself. If nothing in the code can reach an object anymore, it is removed. This is garbage collection.
>
> A memory leak happens when we keep holding things we don't need. Common reasons in Node:
> - an in-memory cache or array that keeps growing with no limit,
> - `setInterval` that is never cleared,
> - event listeners that are never removed.
>
> To find a leak, I watch memory with `process.memoryUsage()` or PM2, and compare heap snapshots in Chrome DevTools. The fix is usually adding a size limit or expiry to the cache, or moving the cache to Redis."

### Strict mode

> 🎤 **Interview answer:** "`'use strict'` makes JavaScript throw errors for silent mistakes, like using a variable that was never declared. Classes and ES modules are always in strict mode."

### Error handling in JS

> 🎤 **Interview answer:** "For normal code and `await`, I use `try/catch`. For `.then` chains, I use `.catch`. If an async function throws, it doesn't crash right away. It returns a failed promise, so someone must catch it.
>
> In my backend, I make my own error classes like `NotFoundError` with a status code. Services throw them, and one central Express error middleware sends the right response and logs the error."

- An async function that throws returns a **rejected promise** – see [11 Scenarios → Q11](11-scenario-questions.md#11-what-happens-when-an-async-function-throws-an-error).

### Memoization

> 🎤 **Interview answer:** "Memoization means saving the result of a function for an input. Next time the same input comes, we return the saved result instead of calculating again. It uses a closure to keep the saved results. React's `useMemo` uses the same idea. On the backend, Redis caching is the same idea on a bigger scale."

Example in [02 Coding](02-javascript-coding.md#q15-memoize-function).

---

[Home](README.md) | Next: [02 JavaScript Coding →](02-javascript-coding.md)
