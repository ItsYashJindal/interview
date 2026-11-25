[Home](README.md) | [← 01 JavaScript](01-javascript.md) | Next: [03 Node.js & Express →](03-nodejs-express.md)

# 02 – JavaScript Coding & Output Questions (4 years level)

These are the practical questions most asked in 3–5 year interviews. Practise writing each one **without looking**, then explain it out loud.

## Contents
- [Part A – Output guessing questions](#part-a--output-guessing-questions)
- [Part B – Common coding questions](#part-b--common-coding-questions)
- [Polyfills](#polyfills)
- [Part C – Async coding questions](#part-c--async-coding-questions)
- [Part D – Small data-structure questions](#part-d--small-data-structure-questions)

---

## Part A – Output guessing questions

### O1. Event loop order

> 🎤 **Interview answer:** "The output is A, E, C, D, B. A and E are synchronous, so they print first. The promise `.then` and `queueMicrotask` both go into the microtask queue, and they run in the order they were added, so C then D. The setTimeout is a macrotask, so even with 0 milliseconds it runs only after the microtask queue is empty. That gives B last."

```js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
queueMicrotask(() => console.log("D"));
console.log("E");
```
**Output:** `A E C D B`
Sync first (A, E), then all microtasks in order (C, D), then macrotask (B).

### O2. async/await order

> 🎤 **Interview answer:** "It prints 3, 1, 4, 2. 3 is logged first. When `f()` is called, everything before the first `await` runs synchronously, so 1 prints. At `await null`, the rest of the function is scheduled as a microtask and control returns to the caller, so 4 prints. Once the stack is empty, the microtask resumes `f` and prints 2. The key point is that `await` pauses only that function, not the whole program."

```js
async function f() {
  console.log(1);
  await null;
  console.log(2);
}
console.log(3);
f();
console.log(4);
```
**Output:** `3 1 4 2`
Code before the first `await` runs synchronously. The rest goes to the microtask queue.

### O3. Node.js – nextTick vs promise vs setImmediate vs setTimeout

> 🎤 **Interview answer:** "In Node, `process.nextTick` has the highest priority. Its queue is drained even before promise microtasks, so 'nextTick' comes first, then 'promise'. After that, between setTimeout 0 and setImmediate, the order isn't guaranteed in the main module, because it depends on whether the 1 ms timer is ready when the event loop enters the timers phase. Usually you see 'timeout' and then 'immediate'. Inside an I/O callback, though, setImmediate always runs first, because the check phase comes right after the poll phase."

```js
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
Promise.resolve().then(() => console.log("promise"));
process.nextTick(() => console.log("nextTick"));
```
**Output:** `nextTick promise timeout immediate` (timeout vs immediate order can change in the main module; inside an I/O callback, `immediate` always comes first).

### O4. var in loop with setTimeout

> 🎤 **Interview answer:** "It prints 3, 3, 3, because `var` creates one function-scoped `i` that all three callbacks share, and by the time they run the loop has already made `i` equal to 3. The cleanest fix is `let`, which creates a fresh binding per iteration. Before ES6, people used an IIFE to capture the current value."

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i));
```
**Output:** `3 3 3`. Fix: use `let`, or an IIFE `((j) => setTimeout(() => console.log(j)))(i)`.

### O5. Hoisting

> 🎤 **Interview answer:** "It prints `undefined`, `'function'` and `undefined`. `var a` is hoisted and initialised to `undefined`, so logging before the assignment gives undefined. Function declarations are hoisted fully, so `typeof foo` is already 'function'. `bar` is a `var` holding an arrow function, so only the variable is hoisted, as undefined, and `typeof bar` is 'undefined'. If it were `let` or `const`, it would throw a ReferenceError because of the TDZ."

```js
console.log(a); var a = 1;
console.log(typeof foo); function foo() {}
console.log(typeof bar); var bar = () => {};
```
**Output:** `undefined`, `function`, `undefined`

### O6. Function inside function hoisting

> 🎤 **Interview answer:** "It prints `undefined`, not 1. Inside `test`, the local `var x` is hoisted to the top of the function, so the local `x` shadows the global one from the very first line, and at the time of `console.log` it's still undefined. This is a classic example of why `var` hoisting is confusing."

```js
var x = 1;
function test() {
  console.log(x);
  var x = 2;
}
test();
```
**Output:** `undefined` – the local `var x` is hoisted inside `test` and hides the outer `x`.

### O7. this with arrow vs normal

> 🎤 **Interview answer:** "In Node it prints 'A', undefined and ['A']. `regular()` is called as `obj.regular()`, so `this` is `obj`. `arrow` is an arrow function defined in the object literal, and object literals don't create a scope, so it takes `this` from the module level, which is an empty `module.exports` object, so the name is undefined. In `nested`, the arrow inside `map` takes `this` from the `nested` method, which is `obj`, so it returns ['A']. So arrow functions are good as callbacks inside methods, but not as methods themselves."

```js
const obj = {
  name: "A",
  regular() { return this.name; },
  arrow: () => this?.name,
  nested() { return [1].map(() => this.name); },
};
console.log(obj.regular(), obj.arrow(), obj.nested());
```
**Output (in Node):** `A undefined ["A"]` – the arrow property takes `this` from the file scope, not from `obj`. The arrow inside a method takes the method's `this`. (In a browser, `this` at top level is `window`, and `window.name` is usually `""`.)

### O8. Losing this

> 🎤 **Interview answer:** "It returns undefined, or throws a TypeError in strict mode. When we assign `user.greet` to a variable and call `g()` on its own, there's no object before the dot, so `this` isn't `user` any more. The fix is `user.greet.bind(user)`, or calling it as `user.greet()`. The same bug shows up when you pass a method as a callback to setTimeout or an event listener."

```js
const user = { name: "Yash", greet() { return this.name; } };
const g = user.greet;
console.log(g());
```
**Output:** `undefined` in Node (in strict mode it throws a TypeError because `this` is `undefined`). Fix: `user.greet.bind(user)`.

### O9. Object reference

> 🎤 **Interview answer:** "First it prints 2, because `b = a` copies the reference, not the object, so both point to the same object in memory. Then it prints false, because `===` on objects compares references, and two separate object literals are different objects even with the same content. To compare content you need a deep-equal function."

```js
const a = { x: 1 };
const b = a;
b.x = 2;
console.log(a.x);
console.log({ x: 1 } === { x: 1 });
```
**Output:** `2`, `false` (different objects in memory).

### O10. Type coercion

> 🎤 **Interview answer:** "These all come from JavaScript's coercion rules. `+` with a string concatenates, so `1 + '2'` is '12'. `-` only works on numbers, so `'5' - 2` is 3. Arrays are converted to strings with `+`, so `[] + []` is an empty string. `[] == false` is true because both become 0. `null == undefined` is true by spec, but not with `===`. `NaN` is never equal to itself, so we use `Number.isNaN`. `0.1 + 0.2` isn't exactly 0.3 because of binary floating point. That's why I store money in paise as integers."

```js
console.log(1 + "2");   // "12"
console.log("5" - 2);   // 3
console.log([] + []);   // ""
console.log([] == false); // true
console.log(null == undefined);  // true
console.log(null === undefined); // false
console.log(NaN === NaN); // false  (use Number.isNaN)
console.log(0.1 + 0.2 === 0.3); // false
```

### O11. Closure counter

> 🎤 **Interview answer:** "It prints 1, 2, 1. Every call to `outer()` creates a new execution context with its own `count`, so `c1` and `c2` have independent closures. `c1` increments its own count to 1 and then 2, while `c2` starts fresh at 1."

```js
function outer() {
  let count = 0;
  return () => ++count;
}
const c1 = outer();
const c2 = outer();
console.log(c1(), c1(), c2());
```
**Output:** `1 2 1` – each call to `outer` makes a new closure.

### O12. Promise chain

> 🎤 **Interview answer:** "It prints 'fail' and then 5. The first `.then` returns 2, and the second throws, so the chain becomes rejected and skips the next `.then`, which is why 'not run' never prints. `.catch` handles the error, logs 'fail' and returns 5. Returning a value from catch turns the chain back into a fulfilled one, so the final `.then` receives 5."

```js
Promise.resolve(1)
  .then(x => x + 1)
  .then(x => { throw new Error("fail") })
  .then(x => console.log("not run"))
  .catch(e => { console.log(e.message); return 5; })
  .then(x => console.log(x));
```
**Output:** `fail`, `5` – after `.catch` returns a value, the chain continues.

### O13. Spread shallow copy

> 🎤 **Interview answer:** "It prints 1 and 2. Spread makes a shallow copy, so `n`, a primitive, is copied by value, and changing `b.n` doesn't affect `a`. But `inner` is an object, so both `a.inner` and `b.inner` point to the same object, and changing `b.inner.m` changes `a.inner.m` too. For an independent copy I'd use `structuredClone`."

```js
const a = { n: 1, inner: { m: 1 } };
const b = { ...a };
b.n = 2; b.inner.m = 2;
console.log(a.n, a.inner.m);
```
**Output:** `1 2`

### O14. Array tricks

> 🎤 **Interview answer:** "The `+` operator converts arrays to strings, so `[1,2,3] + [4]` becomes '1,2,3' plus '4', which is '1,2,34'. The default `sort` compares items as strings, so 10 comes before 2. For numbers you must pass a comparator like `(a, b) => a - b`. `typeof []` is 'object', so to check for an array I use `Array.isArray`. Also remember that `sort` changes the original array, so I use `toSorted` or copy first if I need the original."

```js
console.log([1, 2, 3] + [4]);        // "1,2,34"
console.log([10, 1, 2].sort());      // [1, 10, 2]  (sorts as strings!)
console.log([10, 1, 2].sort((a,b)=>a-b)); // [1, 2, 10]
console.log(typeof []);              // "object"
console.log(Array.isArray([]));      // true
```

### O15. setTimeout inside promise

> 🎤 **Interview answer:** "It prints 1, 3, 5, 4, 2. The important point is that the Promise executor function runs synchronously, so 3 prints right after 1. Then 5 is synchronous. After the stack is empty, the `.then` microtask prints 4, and finally the setTimeout macrotask prints 2."

```js
console.log(1);
setTimeout(() => console.log(2), 0);
new Promise((res) => {
  console.log(3);   // executor runs synchronously!
  res();
}).then(() => console.log(4));
console.log(5);
```
**Output:** `1 3 5 4 2`

---

## Part B – Common coding questions

### Q1. Reverse a string

> 🎤 **Interview answer:** "The quickest way is split into characters, reverse the array, and join. If they say no built-ins, I loop from the last index to the first and build a new string. Both are O(n). One edge case worth mentioning: `split('')` breaks emojis and other multi-byte characters, so `[...str]` or `Array.from` is safer for Unicode."

```js
const reverse = (s) => s.split("").reverse().join("");
// without built-ins
function reverse2(s) {
  let out = "";
  for (let i = s.length - 1; i >= 0; i--) out += s[i];
  return out;
}
```

### Q2. Check palindrome

> 🎤 **Interview answer:** "First I normalise the string: lowercase it and remove non-alphanumeric characters, so 'A man, a plan…' works. Then I use two pointers, one at the start and one at the end, moving inward and comparing. If any pair doesn't match, I return false. That's O(n) time and O(1) extra space, which is better than reversing the whole string. Empty strings and single characters are palindromes."

```js
function isPalindrome(str) {
  const s = str.toLowerCase().replace(/[^a-z0-9]/g, "");
  let l = 0, r = s.length - 1;
  while (l < r) if (s[l++] !== s[r--]) return false;
  return true;
}
```

### Q3. Remove duplicates from array

> 🎤 **Interview answer:** "For primitives, the simplest is `[...new Set(arr)]`. It's O(n) and keeps the first-occurrence order. Without Set, I can filter where `indexOf(v) === i`, but that's O(n²). For an array of objects, Set doesn't work because objects compare by reference, so I use a Map keyed by id, which keeps one object per id. In real code I'd also ask whether to keep the first or the last duplicate."

```js
const unique = [...new Set(arr)];
// without Set
const unique2 = arr.filter((v, i) => arr.indexOf(v) === i);
// array of objects by id
const uniqueById = [...new Map(users.map(u => [u.id, u])).values()];
```

### Q4. Count frequency of characters / words

> 🎤 **Interview answer:** "I loop over the string once and keep a counter object, or a Map, where each character's count is incremented. It's O(n) time. For words, I'd first split on whitespace and lowercase them. This frequency-map pattern is the base for many other questions, like first unique character, anagram and max occurring character."

```js
function frequency(str) {
  const map = {};
  for (const ch of str) map[ch] = (map[ch] || 0) + 1;
  return map;
}
frequency("hello"); // { h:1, e:1, l:2, o:1 }
```

### Q5. First non-repeating character

> 🎤 **Interview answer:** "I do two passes. The first pass builds a frequency map of every character. The second pass goes through the string again in order and returns the first character whose count is 1. That's O(n) time and gives the correct order, because the second pass follows the original string. If nothing is unique, I return null."

```js
function firstUnique(str) {
  const freq = {};
  for (const c of str) freq[c] = (freq[c] || 0) + 1;
  for (const c of str) if (freq[c] === 1) return c;
  return null;
}
```

### Q6. Check anagram

> 🎤 **Interview answer:** "Two strings are anagrams if they have the same characters with the same counts. The simple way is to check the lengths, sort both strings and compare, which is O(n log n). The better way is one frequency map: increment for the first string, decrement for the second, and check everything is zero. That's O(n). I'd also clarify whether to ignore case and spaces."

```js
const isAnagram = (a, b) =>
  a.length === b.length && [...a].sort().join("") === [...b].sort().join("");
```

### Q7. Flatten a nested array

> 🎤 **Interview answer:** "Built-in, it's `arr.flat(Infinity)`. To write it myself I use recursion with reduce: for each item, if it's an array I recursively flatten it and concat, otherwise I just add it. Time is O(total elements). For extremely deep nesting, recursion could hit the stack limit, so an iterative version with a stack is the safer alternative."

```js
arr.flat(Infinity); // built-in

function flatten(arr) {
  return arr.reduce(
    (acc, item) => acc.concat(Array.isArray(item) ? flatten(item) : item),
    []
  );
}
flatten([1, [2, [3, [4]]]]); // [1, 2, 3, 4]
```

### Q8. Flatten a nested object

> 🎤 **Interview answer:** "I write a recursive function that carries a prefix path. For each key, I build the path like 'b.d.e'. If the value is a plain object, I recurse with that path. Otherwise, for primitives, arrays and null, I set `result[path] = value`. I have to check for `null` separately because `typeof null` is 'object'. This is useful in real work for building MongoDB `$set` updates with dot notation, or for flattening data for CSV exports."

```js
function flattenObj(obj, prefix = "", res = {}) {
  for (const key in obj) {
    const path = prefix ? `${prefix}.${key}` : key;
    if (typeof obj[key] === "object" && obj[key] !== null && !Array.isArray(obj[key])) {
      flattenObj(obj[key], path, res);
    } else {
      res[path] = obj[key];
    }
  }
  return res;
}
flattenObj({ a: 1, b: { c: 2, d: { e: 3 } } }); // { a:1, "b.c":2, "b.d.e":3 }
```

### Q9. Second largest number

> 🎤 **Interview answer:** "Instead of sorting, which is O(n log n), I do one pass with two variables, `first` and `second`, both starting at minus Infinity. If a number is bigger than first, the old first moves to second. Otherwise, if it's bigger than second and not equal to first, it becomes second. That's O(n) time and O(1) space. The edge cases are arrays with fewer than two distinct values, where I return null, and duplicates of the max."

```js
function secondLargest(arr) {
  let first = -Infinity, second = -Infinity;
  for (const n of arr) {
    if (n > first) { second = first; first = n; }
    else if (n > second && n !== first) second = n;
  }
  return second === -Infinity ? null : second;
}
```

### Q10. Group by a key

> 🎤 **Interview answer:** "I use reduce with an object as the accumulator. For each item, I take its key value, create an empty array for it if needed, and push the item. That's one pass, O(n). In real code I've used this to group orders by status for a dashboard. Modern JavaScript also has `Object.groupBy` now."

```js
function groupBy(arr, key) {
  return arr.reduce((acc, item) => {
    (acc[item[key]] ||= []).push(item);
    return acc;
  }, {});
}
groupBy(orders, "status");
```

### Q11. Chunk an array

> 🎤 **Interview answer:** "I loop with a step of `size` and push `arr.slice(i, i + size)` each time. The last chunk just has whatever is left. It's O(n). I've used this in production: when sending notifications to thousands of users, I chunk them into batches of 100 and process each batch with `Promise.allSettled`, so we don't overload the provider."

```js
function chunk(arr, size) {
  const res = [];
  for (let i = 0; i < arr.length; i += size) res.push(arr.slice(i, i + size));
  return res;
}
chunk([1,2,3,4,5], 2); // [[1,2],[3,4],[5]]
```
(Used in real work: sending notifications in batches of 100.)

### Q12. Deep clone an object

> 🎤 **Interview answer:** "Recursion: if the value is a primitive or null, return it as is. Dates need special handling, so I create a new Date. For arrays and objects, I create an empty copy and recursively clone each property. To handle circular references, I keep a WeakMap of objects already cloned, and if I see one again I return the existing copy instead of recursing forever. In real projects I'd simply use `structuredClone`, but this shows I understand the edge cases."

```js
function deepClone(obj, seen = new WeakMap()) {
  if (obj === null || typeof obj !== "object") return obj;
  if (obj instanceof Date) return new Date(obj);
  if (seen.has(obj)) return seen.get(obj);          // handle circular reference
  const copy = Array.isArray(obj) ? [] : {};
  seen.set(obj, copy);
  for (const key of Object.keys(obj)) copy[key] = deepClone(obj[key], seen);
  return copy;
}
```

### Q13. Deep equal (compare two objects)

> 🎤 **Interview answer:** "First, if they're strictly equal, return true, which also covers the same reference and equal primitives. If either one isn't an object, or is null, return false. Then compare the number of keys, and recursively check every key with deepEqual. Extra edge cases I'd mention: arrays versus objects, Dates, NaN, and circular references, which need a 'seen' map."

```js
function deepEqual(a, b) {
  if (a === b) return true;
  if (typeof a !== "object" || typeof b !== "object" || !a || !b) return false;
  const ka = Object.keys(a), kb = Object.keys(b);
  if (ka.length !== kb.length) return false;
  return ka.every(k => deepEqual(a[k], b[k]));
}
```

### Q14. Debounce and throttle

> 🎤 **Interview answer:** "For debounce: I return a function that clears the existing timer and starts a new setTimeout on every call, so the real function runs only after calls stop for the delay. For throttle: I keep a 'waiting' flag. The first call runs immediately and sets the flag, and further calls are ignored until a timeout resets it. Both use closures and `fn.apply(this, args)` to keep `this` and the arguments. Debounce is for search input, throttle is for scroll."

Written in [01 JavaScript → Debouncing](01-javascript.md#7-debouncing-and-throttling). Be ready to write both from memory – **asked in almost every interview**.

### Q15. Memoize function

> 🎤 **Interview answer:** "I return a wrapper function that keeps a Map cache in a closure. On each call, I build a key from the arguments, usually with `JSON.stringify`. If the key exists I return the cached result, otherwise I call the original function, store the result and return it. It only makes sense for pure functions. In production I'd add a size limit or TTL so the cache doesn't grow forever, which is basically an LRU cache."

```js
function memoize(fn) {
  const cache = new Map();
  return function (...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
const slowSquare = (n) => n * n;
const fastSquare = memoize(slowSquare);
```

### Q16. Currying – sum(1)(2)(3)()

> 🎤 **Interview answer:** "For the infinite `sum`, each call returns a new function that remembers the running total in a closure. When it's called with no argument, it returns the total instead of another function. For a generic `curry`, I compare the arguments collected so far with `fn.length`, the number of declared parameters. If we have enough, I call the original function. Otherwise I return a function that collects more arguments and tries again. That supports both `add(1)(2)(3)` and `add(1, 2)(3)`."

```js
function sum(a) {
  return function (b) {
    if (b === undefined) return a;
    return sum(a + b);
  };
}
sum(1)(2)(3)(); // 6

// Generic curry
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn.apply(this, args);
    return (...next) => curried.apply(this, [...args, ...next]);
  };
}
const add3 = curry((a, b, c) => a + b + c);
add3(1)(2)(3); add3(1, 2)(3); // 6
```

### Q17. once() – function that runs only one time

> 🎤 **Interview answer:** "I keep two variables in a closure: a `done` flag and the `result`. On the first call I set done to true, run the function and store the result. Every later call just returns the stored result without running it again. A real use is initialising something once, like a DB connection or an SDK client, even if many places call init."

```js
function once(fn) {
  let done = false, result;
  return function (...args) {
    if (!done) { done = true; result = fn.apply(this, args); }
    return result;
  };
}
const init = once(() => console.log("DB connected"));
```

### Q18. compose and pipe

> 🎤 **Interview answer:** "Both combine several functions into one. `pipe` runs them left to right: I reduce over the functions, passing each function's output as the next one's input. `compose` is the same but right to left, using `reduceRight`, which is the mathematical order. So `pipe(add1, double)(5)` gives 12. It's useful for building data transformation steps or a chain of validations."

```js
const pipe = (...fns) => (x) => fns.reduce((v, f) => f(v), x);       // left to right
const compose = (...fns) => (x) => fns.reduceRight((v, f) => f(v), x); // right to left
pipe(x => x + 1, x => x * 2)(5); // 12
```

### Q19. Find missing number in 1..n

> 🎤 **Interview answer:** "The sum of 1 to n is n times n plus 1, divided by 2. I calculate that expected sum, subtract the actual sum of the array, and the difference is the missing number. O(n) time, O(1) space. For very large n, the sum could overflow safe integers, so XOR of all numbers is an alternative that avoids big sums."

```js
function missing(arr, n) {
  const expected = (n * (n + 1)) / 2;
  return expected - arr.reduce((a, b) => a + b, 0);
}
```

### Q20. Move all zeros to the end

> 🎤 **Interview answer:** "I use a write pointer. I loop through the array, and every non-zero number is written at the pointer position and the pointer moves forward. After the loop, I fill the remaining positions with zeros. This keeps the order of the non-zero elements, works in place, and is O(n) time with O(1) extra space."

```js
function moveZeros(arr) {
  let pos = 0;
  for (const n of arr) if (n !== 0) arr[pos++] = n;
  while (pos < arr.length) arr[pos++] = 0;
  return arr;
}
```

### Q21. Two sum (return indexes)

> 🎤 **Interview answer:** "The brute-force approach is two nested loops, which is O(n²). The better approach uses a Map from value to index. For each number, I calculate what I need, target minus the current number. If that's already in the map, I return both indexes. Otherwise I store the current number and its index. That's one pass, O(n) time and O(n) space. I check before inserting so the same element isn't used twice."

```js
function twoSum(nums, target) {
  const seen = new Map();
  for (let i = 0; i < nums.length; i++) {
    const need = target - nums[i];
    if (seen.has(need)) return [seen.get(need), i];
    seen.set(nums[i], i);
  }
  return [];
}
```
O(n) time using a Map instead of two loops (O(n²)).

### Q22. Sort array of objects

> 🎤 **Interview answer:** "I pass a comparator to sort. For numbers, `a.age - b.age` gives ascending order, and `b - a` gives descending. For strings I use `localeCompare`, which handles case and accents properly. For multiple keys, I compare the first key and, if equal, fall back to the second. And I remember that `sort` mutates the array, so I copy it first if needed."

```js
users.sort((a, b) => a.age - b.age);                 // number
users.sort((a, b) => a.name.localeCompare(b.name));  // string
```

### Q23. Capitalize first letter of each word

> 🎤 **Interview answer:** "I split the sentence on spaces, map each word to its first character uppercased plus the rest of the word, and join back with spaces. I use optional chaining on the first character so empty strings from double spaces don't crash. For messy input I'd split on a whitespace regex instead."

```js
const titleCase = (s) => s.split(" ").map(w => w[0]?.toUpperCase() + w.slice(1)).join(" ");
```

### Q24. Max occurring character

> 🎤 **Interview answer:** "One pass with a frequency object. As I increment each character's count, I also check if it's now greater than the current max count, and if so I update the max character. That way I don't need a second loop. It's O(n) time. If there's a tie, this returns the character that reached the max count first."

```js
function maxChar(str) {
  const f = {};
  let max = "", count = 0;
  for (const c of str) {
    f[c] = (f[c] || 0) + 1;
    if (f[c] > count) { count = f[c]; max = c; }
  }
  return max;
}
```

### Q25. Fibonacci (with memo)

> 🎤 **Interview answer:** "Plain recursive Fibonacci is O(2^n), because it recalculates the same values again and again. With memoization I store each computed fib(n) in an object, so every value is calculated once and it becomes O(n). An iterative version with two variables is even better, O(n) time and O(1) space, with no recursion depth problem."

```js
function fib(n, memo = {}) {
  if (n <= 1) return n;
  if (memo[n]) return memo[n];
  return (memo[n] = fib(n - 1, memo) + fib(n - 2, memo));
}
```

### Q26. Intersection and difference of arrays

> 🎤 **Interview answer:** "For intersection, I filter the first array to keep items that exist in the second. For difference, I keep items that don't exist in the second. The important optimisation is to create the Set from the second array once, outside the filter, so the lookups are O(1) and the whole thing is O(n + m) instead of O(n × m) with `includes`."

```js
const intersection = a.filter(x => new Set(b).has(x));
const difference   = a.filter(x => !b.includes(x));
```

### Q27. Get value from nested object by path – get(obj, "a.b.c")

> 🎤 **Interview answer:** "I split the path on dots and reduce over the keys, going one level deeper each time. If at any point the current value is null or undefined, I return undefined instead of crashing. At the end, `??` returns the default value if nothing was found. It's like lodash's `get`, or what optional chaining does, but with a dynamic path string."

```js
const get = (obj, path, def) =>
  path.split(".").reduce((o, k) => (o == null ? undefined : o[k]), obj) ?? def;
```

---

## Polyfills

Interviewers love these at 4 years. Know `map`, `filter`, `reduce`, `bind`, `Promise.all`.

### P1. Array.prototype.map

> 🎤 **Interview answer:** "I add a method to `Array.prototype`, so inside it `this` is the array. I create a new result array, loop over the indexes, call the callback with the element, the index and the array itself, using `call` so the optional `thisArg` works, and store the return value at the same index. The `i in this` check skips holes in sparse arrays, like the real map. It never changes the original array."

```js
Array.prototype.myMap = function (cb, thisArg) {
  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this) result[i] = cb.call(thisArg, this[i], i, this);
  }
  return result;
};
```

### P2. Array.prototype.filter

> 🎤 **Interview answer:** "Same structure as map, except I only push the element into the result when the callback returns a truthy value. The callback gets the element, the index and the array, with `thisArg` support. The result can be shorter than the original, and the original stays unchanged."

```js
Array.prototype.myFilter = function (cb, thisArg) {
  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this && cb.call(thisArg, this[i], i, this)) result.push(this[i]);
  }
  return result;
};
```

### P3. Array.prototype.reduce

> 🎤 **Interview answer:** "The tricky part is the initial value. If one is given, the accumulator starts with it and I loop from index 0. If not, I use the first element as the accumulator and start from index 1, and if the array is empty with no initial value, I throw a TypeError just like the native method. Then for each element, the accumulator becomes the callback's return value, and I return it at the end."

```js
Array.prototype.myReduce = function (cb, initial) {
  let acc = initial;
  let start = 0;
  if (acc === undefined) {
    if (this.length === 0) throw new TypeError("Reduce of empty array with no initial value");
    acc = this[0];
    start = 1;
  }
  for (let i = start; i < this.length; i++) acc = cb(acc, this[i], i, this);
  return acc;
};
```

### P4. Function.prototype.bind

> 🎤 **Interview answer:** "`bind` returns a new function and doesn't call it immediately. Inside `myBind`, `this` is the original function, so I save it. I return a new function that, when called, uses `apply` to call the original with the given context and the preset arguments followed by any later ones. That supports partial application like `bind(ctx, a)(b)`. A fully spec-compliant version also handles being called with `new`, which I'd mention."

```js
Function.prototype.myBind = function (context, ...preset) {
  const fn = this;
  return function (...later) {
    return fn.apply(context, [...preset, ...later]);
  };
};
```

### P5. call and apply

> 🎤 **Interview answer:** "The trick is to make the function temporarily a method of the context object, because calling `context.fn()` makes `this` equal to the context. I use a Symbol as the property name so I don't overwrite any existing property, call it with the arguments, delete the property, and return the result. `apply` is the same but takes the arguments as an array, so I just spread them into `myCall`."

```js
Function.prototype.myCall = function (context = globalThis, ...args) {
  const key = Symbol();
  context[key] = this;       // make the function a method of context
  const res = context[key](...args);
  delete context[key];
  return res;
};
Function.prototype.myApply = function (context, args = []) {
  return this.myCall(context, ...args);
};
```

### P6. Promise.all

> 🎤 **Interview answer:** "I return a new Promise. I keep a results array and a counter. For each input I wrap it with `Promise.resolve`, so plain values also work. On success I store the value at its original index, which keeps the order even if they finish in a different order, and increment the counter. When the counter equals the length, I resolve with the results. If any one rejects, I reject immediately, which is the fail-fast behaviour. The edge case is an empty array, which should resolve immediately with an empty array."

```js
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let done = 0;
    if (promises.length === 0) return resolve([]);
    promises.forEach((p, i) => {
      Promise.resolve(p)
        .then((val) => {
          results[i] = val;          // keep order
          if (++done === promises.length) resolve(results);
        })
        .catch(reject);              // first error rejects all
    });
  });
}
```

### P7. Promise.allSettled

> 🎤 **Interview answer:** "I map each promise to one that never rejects: on success it resolves to `{ status: 'fulfilled', value }`, and on failure it catches and resolves to `{ status: 'rejected', reason }`. Then `Promise.all` over those always succeeds and gives the status of every promise. I use allSettled for bulk operations like sending notifications, where one failure shouldn't stop the report for the others."

```js
function allSettled(promises) {
  return Promise.all(
    promises.map(p =>
      Promise.resolve(p)
        .then(value => ({ status: "fulfilled", value }))
        .catch(reason => ({ status: "rejected", reason }))
    )
  );
}
```

### P8. Promise.race

> 🎤 **Interview answer:** "I return a new Promise and attach its resolve and reject to every input promise. Whichever settles first wins, and since a promise can only settle once, the later ones are ignored. The common real use is a timeout, racing an API call against a promise that rejects after N milliseconds."

```js
const race = (promises) =>
  new Promise((resolve, reject) => promises.forEach(p => Promise.resolve(p).then(resolve, reject)));
```

---

## Part C – Async coding questions

### A1. sleep / delay

> 🎤 **Interview answer:** "I return a promise that resolves after `setTimeout` with the given milliseconds. Then `await sleep(1000)` pauses only the current async function, not the event loop. I use this for retry backoff and for slowing down batch jobs so we don't hit third-party rate limits."

```js
const sleep = (ms) => new Promise(res => setTimeout(res, ms));
await sleep(1000);
```

### A2. Retry a failing async function

> 🎤 **Interview answer:** "I loop up to the maximum number of retries. In each attempt I try to `await fn()`, and if it succeeds I return the result. If it fails and it's the last attempt, I throw the error. Otherwise I wait with exponential backoff, like 500, then 1000, then 2000 milliseconds, and try again. In production I'd add jitter so many clients don't retry at the same moment, and I'd only retry errors that can recover, like timeouts or 5xx, not 400s. For payment calls, I only retry together with an idempotency key so we never double-charge."

```js
async function retry(fn, retries = 3, delay = 500) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === retries) throw err;
      await sleep(delay * 2 ** (attempt - 1)); // exponential backoff: 500, 1000, 2000
    }
  }
}
```
Real use: calling a payment gateway or third-party API.

### A3. Timeout for a promise

> 🎤 **Interview answer:** "I create a timeout promise that rejects after N milliseconds and use `Promise.race` between it and the real promise, so whichever finishes first decides the result. Two improvements I'd mention: clear the timer when the real promise wins, so we don't leave timers running, and for HTTP calls use an AbortController so the underlying request is actually cancelled, not just ignored."

```js
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), ms)
  );
  return Promise.race([promise, timeout]);
}
```

### A4. Run promises one after another (sequential)

> 🎤 **Interview answer:** "I loop through the task functions with `for...of` and await each one before starting the next, collecting the results. The tasks have to be functions that return promises, not promises that already exist, because a promise starts running as soon as it's created. I use this when order matters or when each step depends on the previous one, like database migrations."

```js
async function runSequential(tasks) {
  const results = [];
  for (const task of tasks) results.push(await task());
  return results;
}
```

### A5. Run with a concurrency limit (asked a lot at 4+ years)

> 🎤 **Interview answer:** "The idea is a worker pool. I start N workers with `Promise.all`. Each worker runs a loop: take the next index from a shared counter, run that task, store the result at that index, and continue until no tasks are left. Since JavaScript is single-threaded, `index++` is safe without locks. So at most N tasks run at any moment, and as soon as one finishes, the worker picks the next one. In real work I've used this for bulk uploads and third-party API calls, where unlimited parallelism would hit rate limits or exhaust connections. Libraries like `p-limit` do the same thing."

Run many async tasks but only N at the same time (e.g. upload 1000 files, 5 at a time).
```js
async function runWithLimit(tasks, limit) {
  const results = [];
  let index = 0;

  async function worker() {
    while (index < tasks.length) {
      const current = index++;
      results[current] = await tasks[current]();
    }
  }

  await Promise.all(Array.from({ length: limit }, worker));
  return results;
}
```

### A6. Convert callback function to promise (promisify)

> 🎤 **Interview answer:** "I return a wrapper function that returns a new Promise. Inside it, I call the original function with the given arguments plus one extra callback in the Node error-first style: if `err` is set I reject, otherwise I resolve with the data. In real code Node gives us `util.promisify`, and most core modules have promise versions now, like `fs/promises`."

```js
function promisify(fn) {
  return (...args) =>
    new Promise((resolve, reject) => {
      fn(...args, (err, data) => (err ? reject(err) : resolve(data)));
    });
}
const readFile = promisify(require("fs").readFile);
```
(Node already has `util.promisify`.)

### A7. Print 1 to 5 with 1 second gap

> 🎤 **Interview answer:** "The simplest is a `let` loop where each setTimeout delay is `i * 1000`, so they fire at 1, 2, 3, 4 and 5 seconds, and `let` makes sure each callback has its own `i`. The cleaner async way is a for loop with `await sleep(1000)` before each log, which really waits between steps. If they ask for it with `var`, I'd use an IIFE or pass `i` as the third argument to setTimeout."

```js
for (let i = 1; i <= 5; i++) setTimeout(() => console.log(i), i * 1000);
// or
(async () => { for (let i = 1; i <= 5; i++) { await sleep(1000); console.log(i); } })();
```

---

## Part D – Small data-structure questions

### D1. Simple EventEmitter

> 🎤 **Interview answer:** "I keep an object that maps event names to arrays of listener functions. `on` pushes a listener, `off` filters it out, and `emit` calls every listener for that event with the arguments. `once` wraps the listener in a function that calls it and then removes itself. Node's built-in EventEmitter works like this. One thing I'd mention is memory leaks: listeners that are never removed keep their closures alive, which is why Node warns when there are more than 10 listeners."

```js
class EventEmitter {
  constructor() { this.events = {}; }
  on(name, fn) { (this.events[name] ||= []).push(fn); return this; }
  off(name, fn) { this.events[name] = (this.events[name] || []).filter(f => f !== fn); }
  emit(name, ...args) { (this.events[name] || []).forEach(fn => fn(...args)); }
  once(name, fn) {
    const wrapper = (...args) => { fn(...args); this.off(name, wrapper); };
    this.on(name, wrapper);
  }
}
```

### D2. LRU Cache (Least Recently Used)

> 🎤 **Interview answer:** "I use a JavaScript Map, because it keeps insertion order, so the first key is always the least recently used. On `get`, if the key exists, I delete it and set it again so it moves to the end as the most recent. On `put`, if the key exists I delete it first, and if the cache is full I remove the first key from `map.keys()`. Both operations are O(1). The classic version is a hashmap plus a doubly linked list, and the Map gives us that behaviour for free. Redis has the same idea with its allkeys-lru eviction policy."

`Map` keeps insertion order, so the first key is the oldest.
```js
class LRUCache {
  constructor(limit) { this.limit = limit; this.map = new Map(); }
  get(key) {
    if (!this.map.has(key)) return -1;
    const val = this.map.get(key);
    this.map.delete(key);
    this.map.set(key, val);         // move to newest
    return val;
  }
  put(key, val) {
    if (this.map.has(key)) this.map.delete(key);
    else if (this.map.size === this.limit) this.map.delete(this.map.keys().next().value); // remove oldest
    this.map.set(key, val);
  }
}
```

### D3. Simple in-memory rate limiter (fixed window)

> 🎤 **Interview answer:** "For each IP I store a count and the window start time. On every request, if the current window has expired I reset the count, then I increment it and allow the request only if the count is within the limit. It's simple, but it has two problems: bursts at window edges, and it only works on a single server. In production I implement it in Redis with `INCR` plus `EXPIRE`, or a sliding window with sorted sets, so all instances share the same counters. I'd also clean up old entries, otherwise this Map grows forever."

```js
const hits = new Map();
function isAllowed(ip, limit = 100, windowMs = 60_000) {
  const now = Date.now();
  const entry = hits.get(ip) || { count: 0, start: now };
  if (now - entry.start > windowMs) { entry.count = 0; entry.start = now; }
  entry.count++;
  hits.set(ip, entry);
  return entry.count <= limit;
}
```
For multiple servers use Redis – see [03 → Rate limiting](03-nodejs-express.md#10-api-rate-limiting).

### D4. Valid parentheses

> 🎤 **Interview answer:** "I use a stack. When I see an opening bracket, I push it. When I see a closing bracket, I pop and check it matches the right opening bracket using a pairs map, and if not, it's invalid. At the end the stack must be empty, otherwise there are unclosed brackets. O(n) time and O(n) space."

```js
function isValid(s) {
  const stack = [], pairs = { ")": "(", "]": "[", "}": "{" };
  for (const c of s) {
    if ("([{".includes(c)) stack.push(c);
    else if (stack.pop() !== pairs[c]) return false;
  }
  return stack.length === 0;
}
```

### D5. Longest substring without repeating characters

> 🎤 **Interview answer:** "Sliding window. I keep a `start` pointer and a map of each character's last seen index. As I move the end pointer, if the current character was seen inside the current window, I move `start` just past its last position. Then I update the last index and the best length. Each character is processed once, so it's O(n) instead of checking every substring."

```js
function longestUnique(s) {
  const last = new Map();
  let start = 0, best = 0;
  for (let i = 0; i < s.length; i++) {
    if (last.has(s[i]) && last.get(s[i]) >= start) start = last.get(s[i]) + 1;
    last.set(s[i], i);
    best = Math.max(best, i - start + 1);
  }
  return best;
}
```

---

**Practice tip:** For each question say: (1) approach in one line, (2) write code, (3) time complexity, (4) one edge case (empty input, duplicates, negative numbers).

[Home](README.md) | [← 01 JavaScript](01-javascript.md) | Next: [03 Node.js & Express →](03-nodejs-express.md)
