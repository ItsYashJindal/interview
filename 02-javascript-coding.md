[Home](README.md) | [← 01 JavaScript](01-javascript.md) | Next: [03 Node.js & Express →](03-nodejs-express.md)

# 02 – JavaScript Coding & Output Questions (4 years level)

These are the practical questions most asked in 3–5 year interviews. Practise writing each one **without looking**, then explain it out loud.

**Quick meaning of Big-O (used below):**
- **O(1)** – same time no matter how big the input is.
- **O(n)** – one loop over the input.
- **O(n²)** – a loop inside a loop (slow for big input).
- **O(n log n)** – the time sorting takes.

## Contents
- [Part A – Output guessing questions](#part-a--output-guessing-questions)
- [Part B – Common coding questions](#part-b--common-coding-questions)
- [Polyfills](#polyfills)
- [Part C – Async coding questions](#part-c--async-coding-questions)
- [Part D – Small data-structure questions](#part-d--small-data-structure-questions)

---

## Part A – Output guessing questions

**Rule to remember for all event-loop questions:**
1. Normal code runs first.
2. Then promise callbacks (`.then`, code after `await`, `queueMicrotask`).
3. Then `setTimeout` / `setInterval` callbacks.

### O1. Event loop order

> 🎤 **Interview answer:** "The output is A, E, C, D, B. A and E are normal code, so they print first. C (promise) and D (queueMicrotask) are both high-priority tasks, and they run in the order they were added. B is a setTimeout, so it runs last, even with 0 ms."

```js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
queueMicrotask(() => console.log("D"));
console.log("E");
```
**Output:** `A E C D B`

### O2. async/await order

> 🎤 **Interview answer:** "It prints 3, 1, 4, 2. First 3 prints. Then `f()` is called. Code before the first `await` runs straight away, so 1 prints. At `await`, the function pauses and the rest of it is put in the queue. So control comes back and 4 prints. After that, the function continues and 2 prints. `await` pauses only that function, not the whole program."

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

### O3. Node.js – nextTick vs promise vs setImmediate vs setTimeout

> 🎤 **Interview answer:** "In Node, `process.nextTick` has the highest priority, even higher than promises. So 'nextTick' prints first, then 'promise'. After that, setTimeout 0 and setImmediate. Usually 'timeout' prints before 'immediate', but in the main file this order is not guaranteed. Inside a file-read callback, setImmediate always comes first."

```js
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
Promise.resolve().then(() => console.log("promise"));
process.nextTick(() => console.log("nextTick"));
```
**Output:** `nextTick promise timeout immediate` (timeout and immediate can swap in the main file).

**Priority order in Node:** normal code → `nextTick` → promises → `setTimeout` → `setImmediate`.

### O4. var in loop with setTimeout

> 🎤 **Interview answer:** "It prints 3, 3, 3. With `var`, there is only one `i` for the whole loop. The setTimeout callbacks run after the loop has finished, and by then `i` is 3. The fix is to use `let`, which gives each round its own `i`."

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i));
```
**Output:** `3 3 3`. Fix: use `let` → `0 1 2`.

### O5. Hoisting

> 🎤 **Interview answer:** "It prints `undefined`, `'function'` and `undefined`.
> - `var a` is known early but has no value yet, so `undefined`.
> - `foo` is a normal function, which is fully available early, so `'function'`.
> - `bar` is a `var` that holds an arrow function. Only the variable is known early, with value `undefined`, so `typeof bar` is `'undefined'`."

```js
console.log(a); var a = 1;
console.log(typeof foo); function foo() {}
console.log(typeof bar); var bar = () => {};
```
**Output:** `undefined`, `function`, `undefined`

### O6. Function inside function hoisting

> 🎤 **Interview answer:** "It prints `undefined`, not 1. Inside `test`, there is a local `var x`. Because of hoisting, this local `x` exists from the first line of the function, and it hides the outer `x`. At the `console.log` line it has no value yet, so we get `undefined`."

```js
var x = 1;
function test() {
  console.log(x);
  var x = 2;
}
test();
```
**Output:** `undefined`

### O7. this with arrow vs normal

> 🎤 **Interview answer:** "It prints `'A'`, `undefined` and `['A']`.
> - `obj.regular()` → normal method, `this` is `obj`, so `'A'`.
> - `obj.arrow()` → an arrow function has no own `this`. It takes `this` from outside the object, which is not `obj`, so `undefined`.
> - `obj.nested()` → the arrow inside `map` takes `this` from the `nested` method, which is `obj`, so `['A']`.
>
> Lesson: don't use arrow functions as object methods, but they are great as callbacks inside methods."

```js
const obj = {
  name: "A",
  regular() { return this.name; },
  arrow: () => this?.name,
  nested() { return [1].map(() => this.name); },
};
console.log(obj.regular(), obj.arrow(), obj.nested());
```
**Output (in Node):** `A undefined ["A"]`

### O8. Losing this

> 🎤 **Interview answer:** "It gives `undefined` (or an error in strict mode). When I copy `user.greet` into `g` and call `g()`, there is no `user.` in front, so `this` is not `user` anymore. Fix: `user.greet.bind(user)`."

```js
const user = { name: "Yash", greet() { return this.name; } };
const g = user.greet;
console.log(g());
```
**Output:** `undefined`. Fix: `const g = user.greet.bind(user)`.

### O9. Object reference

> 🎤 **Interview answer:** "First it prints 2. `b = a` does not copy the object. Both `a` and `b` point to the same object, so changing `b.x` also changes `a.x`. Then it prints false, because `===` checks if it is the **same object**, not if they look the same. These are two different objects."

```js
const a = { x: 1 };
const b = a;
b.x = 2;
console.log(a.x);
console.log({ x: 1 } === { x: 1 });
```
**Output:** `2`, `false`

### O10. Type coercion

> 🎤 **Interview answer:** "JavaScript changes types automatically in some cases:
> - `+` with a string joins text, so `1 + '2'` is `'12'`.
> - `-` only works with numbers, so `'5' - 2` is `3`.
> - `NaN` is never equal to anything, even itself. Use `Number.isNaN`.
> - `0.1 + 0.2` is not exactly `0.3` because of how computers store decimals. That's why I store money in paise as whole numbers."

```js
console.log(1 + "2");            // "12"
console.log("5" - 2);            // 3
console.log([] + []);            // ""
console.log([] == false);        // true
console.log(null == undefined);  // true
console.log(null === undefined); // false
console.log(NaN === NaN);        // false  (use Number.isNaN)
console.log(0.1 + 0.2 === 0.3);  // false
```

### O11. Closure counter

> 🎤 **Interview answer:** "It prints 1, 2, 1. Every time we call `outer()`, a new `count` is created. So `c1` and `c2` have their own separate counters. `c1` goes 1, 2. `c2` starts fresh at 1."

```js
function outer() {
  let count = 0;
  return () => ++count;
}
const c1 = outer();
const c2 = outer();
console.log(c1(), c1(), c2());
```
**Output:** `1 2 1`

### O12. Promise chain

> 🎤 **Interview answer:** "It prints 'fail' and then 5. The second `.then` throws an error, so the chain jumps straight to `.catch`. That's why 'not run' never prints. `.catch` prints 'fail' and returns 5. After a `.catch` returns a value, the chain continues normally, so the last `.then` gets 5."

```js
Promise.resolve(1)
  .then(x => x + 1)
  .then(x => { throw new Error("fail") })
  .then(x => console.log("not run"))
  .catch(e => { console.log(e.message); return 5; })
  .then(x => console.log(x));
```
**Output:** `fail`, `5`

### O13. Spread shallow copy

> 🎤 **Interview answer:** "It prints 1 and 2. Spread makes a shallow copy. `n` is a simple number, so it is really copied, and changing `b.n` doesn't affect `a`. But `inner` is an object, so `a.inner` and `b.inner` point to the same object. Changing `b.inner.m` also changes `a.inner.m`."

```js
const a = { n: 1, inner: { m: 1 } };
const b = { ...a };
b.n = 2; b.inner.m = 2;
console.log(a.n, a.inner.m);
```
**Output:** `1 2`

### O14. Array tricks

> 🎤 **Interview answer:** "`+` turns arrays into text, so `[1,2,3] + [4]` becomes `'1,2,34'`. The default `sort` compares items as text, so `10` comes before `2`. For numbers I must pass `(a, b) => a - b`. `typeof []` is `'object'`, so to check for an array I use `Array.isArray`."

```js
console.log([1, 2, 3] + [4]);              // "1,2,34"
console.log([10, 1, 2].sort());            // [1, 10, 2]  (sorted as text!)
console.log([10, 1, 2].sort((a,b)=>a-b));  // [1, 2, 10]
console.log(typeof []);                    // "object"
console.log(Array.isArray([]));            // true
```

### O15. setTimeout inside promise

> 🎤 **Interview answer:** "It prints 1, 3, 5, 4, 2. The important point: the code inside `new Promise(...)` runs **immediately**, so 3 prints right after 1. Then 5. Then the `.then` prints 4. setTimeout prints 2 last."

```js
console.log(1);
setTimeout(() => console.log(2), 0);
new Promise((res) => {
  console.log(3);   // runs immediately!
  res();
}).then(() => console.log(4));
console.log(5);
```
**Output:** `1 3 5 4 2`

---

## Part B – Common coding questions

### Q1. Reverse a string

> 🎤 **Interview answer:** "Easiest way: split into letters, reverse the array, join back. If they say no built-in methods, I loop from the last letter to the first and build a new string. Both take one pass, O(n)."

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

> 🎤 **Interview answer:** "A palindrome reads the same from both sides, like 'madam'. First I make the string lowercase and keep only letters and digits. Then I put one pointer at the start and one at the end. I compare them and move both towards the middle. If any pair is different, it's not a palindrome. One loop, O(n)."

```js
function isPalindrome(str) {
  const allowed = "abcdefghijklmnopqrstuvwxyz0123456789";

  // 1. keep only letters and digits
  let s = "";
  for (const ch of str.toLowerCase()) {
    if (allowed.includes(ch)) s += ch;
  }

  // 2. compare from both ends
  let left = 0;
  let right = s.length - 1;
  while (left < right) {
    if (s[left] !== s[right]) return false;
    left++;
    right--;
  }
  return true;
}

isPalindrome("A man, a plan, a canal: Panama"); // true
```

### Q3. Remove duplicates from array

> 🎤 **Interview answer:** "Simplest way: `[...new Set(arr)]`. A Set keeps only unique values. Without Set, I can use `filter` and keep an item only if this is the first time it appears (`indexOf(v) === i`), but that is slower. For an array of objects, I use a Map with the id as the key, so only one object per id is kept."

```js
const unique = [...new Set(arr)];

// without Set
const unique2 = arr.filter((v, i) => arr.indexOf(v) === i);

// array of objects – unique by id
const map = new Map();
for (const u of users) map.set(u.id, u);
const uniqueById = [...map.values()];
```

### Q4. Count frequency of characters / words

> 🎤 **Interview answer:** "I loop over the string once and keep a counter object. For each character, I add 1 to its count. This 'counter object' idea is used in many other questions, like first unique character, anagram and most repeated character."

```js
function frequency(str) {
  const count = {};
  for (const ch of str) {
    count[ch] = (count[ch] || 0) + 1;
  }
  return count;
}
frequency("hello"); // { h:1, e:1, l:2, o:1 }

// For words: split on spaces and skip empty parts
function wordFrequency(sentence) {
  const count = {};
  for (const word of sentence.toLowerCase().split(" ")) {
    if (word === "") continue;
    count[word] = (count[word] || 0) + 1;
  }
  return count;
}
```

### Q5. First non-repeating character

> 🎤 **Interview answer:** "Two loops. The first loop counts every character. The second loop goes through the string again from the start and returns the first character whose count is 1. If none, return null. O(n)."

```js
function firstUnique(str) {
  const count = {};
  for (const c of str) count[c] = (count[c] || 0) + 1;
  for (const c of str) if (count[c] === 1) return c;
  return null;
}
firstUnique("aabbcdd"); // "c"
```

### Q6. Check anagram

> 🎤 **Interview answer:** "Two words are anagrams if they have the same letters the same number of times, like 'listen' and 'silent'. Simple way: sort the letters of both and compare. Faster way: one counter object. Add 1 for each letter of the first word, subtract 1 for each letter of the second word. At the end, every count must be 0."

```js
// Simple – sort and compare
const isAnagram = (a, b) =>
  a.length === b.length && a.split("").sort().join("") === b.split("").sort().join("");

// Faster – counter
function isAnagram2(a, b) {
  if (a.length !== b.length) return false;
  const count = {};
  for (const c of a) count[c] = (count[c] || 0) + 1;
  for (const c of b) {
    if (!count[c]) return false; // letter missing or used too many times
    count[c]--;
  }
  return true;
}
```

### Q7. Flatten a nested array

> 🎤 **Interview answer:** "Built-in: `arr.flat(Infinity)`. To write it myself: I loop over the array. If an item is itself an array, I call the same function on it (recursion) and add its result. If not, I just add the item."

```js
arr.flat(Infinity); // built-in

function flatten(arr) {
  const result = [];
  for (const item of arr) {
    if (Array.isArray(item)) {
      result.push(...flatten(item)); // go inside the inner array
    } else {
      result.push(item);
    }
  }
  return result;
}
flatten([1, [2, [3, [4]]]]); // [1, 2, 3, 4]
```

### Q8. Flatten a nested object

> 🎤 **Interview answer:** "I loop over the keys and build a path like 'b.d.e'. If the value is another object, I call the same function again with the longer path. If it's a normal value, I save it in the result with that path. I check for `null` separately, because `typeof null` is `'object'`. This is useful for MongoDB `$set` updates with dot paths."

```js
function flattenObj(obj, prefix = "", result = {}) {
  for (const key in obj) {
    const path = prefix ? prefix + "." + key : key;
    const value = obj[key];
    const isPlainObject = typeof value === "object" && value !== null && !Array.isArray(value);

    if (isPlainObject) {
      flattenObj(value, path, result); // go deeper
    } else {
      result[path] = value;
    }
  }
  return result;
}
flattenObj({ a: 1, b: { c: 2, d: { e: 3 } } }); // { a:1, "b.c":2, "b.d.e":3 }
```

### Q9. Second largest number

> 🎤 **Interview answer:** "Instead of sorting, I loop once and keep two variables: `first` (biggest) and `second` (second biggest). If a number is bigger than `first`, the old `first` becomes `second`, and the new number becomes `first`. Else, if it is bigger than `second` and not equal to `first`, it becomes `second`. One loop, O(n)."

```js
function secondLargest(arr) {
  let first = -Infinity;
  let second = -Infinity;
  for (const n of arr) {
    if (n > first) {
      second = first;
      first = n;
    } else if (n > second && n !== first) {
      second = n;
    }
  }
  return second === -Infinity ? null : second;
}
secondLargest([5, 1, 9, 9, 7]); // 7
```

### Q10. Group by a key

> 🎤 **Interview answer:** "I loop over the items. For each item, I check its key value. If there is no group for it yet, I create an empty array. Then I push the item into its group. I used this to group orders by status for a dashboard."

```js
function groupBy(arr, key) {
  const groups = {};
  for (const item of arr) {
    const k = item[key];
    if (!groups[k]) groups[k] = [];
    groups[k].push(item);
  }
  return groups;
}
groupBy(orders, "status"); // { paid: [...], pending: [...] }
```

### Q11. Chunk an array

> 🎤 **Interview answer:** "I loop with a step equal to the chunk size and take `slice(i, i + size)` each time. The last chunk just has whatever is left. I've used this in real work: sending notifications to thousands of users in batches of 100, so we don't overload the provider."

```js
function chunk(arr, size) {
  const result = [];
  for (let i = 0; i < arr.length; i += size) {
    result.push(arr.slice(i, i + size));
  }
  return result;
}
chunk([1,2,3,4,5], 2); // [[1,2],[3,4],[5]]
```

### Q12. Deep clone an object

> 🎤 **Interview answer:** "If the value is not an object (number, string, null), I return it as it is. If it's a Date, I make a new Date. If it's an array or object, I create an empty copy and call the same function for every property. In real projects I simply use `structuredClone`, but this shows I understand how it works."

```js
function deepClone(obj) {
  if (obj === null || typeof obj !== "object") return obj; // simple value
  if (obj instanceof Date) return new Date(obj);

  const copy = Array.isArray(obj) ? [] : {};
  for (const key in obj) {
    copy[key] = deepClone(obj[key]); // copy each level
  }
  return copy;
}
```
*Extra point to mention:* if an object points to itself (circular reference), this function would loop forever. To handle that, keep a `Map` of objects already copied.

### Q13. Deep equal (compare two objects)

> 🎤 **Interview answer:** "If both are exactly equal with `===`, return true. If either one is not an object or is null, return false. Then check both have the same number of keys, and compare every key by calling the same function again."

```js
function deepEqual(a, b) {
  if (a === b) return true;
  if (typeof a !== "object" || typeof b !== "object" || a === null || b === null) return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  for (const key of keysA) {
    if (!deepEqual(a[key], b[key])) return false;
  }
  return true;
}
```

### Q14. Debounce and throttle

> 🎤 **Interview answer:** "Debounce: every call cancels the old timer and starts a new one. So the real function runs only when calls stop for the given time. Throttle: I keep a 'waiting' flag. The first call runs and sets the flag. All calls are ignored until the timer resets the flag. Debounce is for a search box, throttle is for scroll."

Written in [01 JavaScript → Debouncing](01-javascript.md#7-debouncing-and-throttling). Be ready to write both from memory – **asked in almost every interview**.

### Q15. Memoize function

> 🎤 **Interview answer:** "I return a new function that keeps a cache (a Map) in a closure. When it's called, I turn the arguments into a key with `JSON.stringify`. If the key is already in the cache, I return the saved result. If not, I call the real function, save the result, and return it. In production I'd add a size limit so the cache doesn't grow forever."

```js
function memoize(fn) {
  const cache = new Map();
  return function (...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key); // already calculated
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}
const slowSquare = (n) => n * n;
const fastSquare = memoize(slowSquare);
```

### Q16. Currying – sum(1)(2)(3)()

> 🎤 **Interview answer:** "Each call returns a new function that remembers the total so far. When it is called with no value, it returns the total instead of another function.
>
> For a general `curry` function: I keep collecting arguments. `fn.length` tells me how many arguments the original function needs. When I have enough, I call the original function. If not, I return a function that waits for more."

```js
function sum(a) {
  return function (b) {
    if (b === undefined) return a; // no more values → give the total
    return sum(a + b);
  };
}
sum(1)(2)(3)(); // 6

// General curry
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn(...args);       // enough args → run
    return (...more) => curried(...args, ...more);           // else wait for more
  };
}
const add3 = curry((a, b, c) => a + b + c);
add3(1)(2)(3);  // 6
add3(1, 2)(3);  // 6
```

### Q17. once() – function that runs only one time

> 🎤 **Interview answer:** "I keep a `done` flag and the `result` in a closure. On the first call, I run the function, save the result and set `done = true`. Every later call just returns the saved result without running again. Real use: connect to the database only once, even if many places call `init()`."

```js
function once(fn) {
  let done = false;
  let result;
  return function (...args) {
    if (!done) {
      done = true;
      result = fn(...args);
    }
    return result;
  };
}
const init = once(() => console.log("DB connected"));
init(); // DB connected
init(); // nothing
```

### Q18. compose and pipe

> 🎤 **Interview answer:** "Both join many functions into one. `pipe` runs them left to right: the output of the first becomes the input of the next. `compose` does the same, but right to left. So `pipe(add1, double)(5)` = double(add1(5)) = 12."

```js
const pipe = (...fns) => (x) => fns.reduce((value, fn) => fn(value), x);         // left to right
const compose = (...fns) => (x) => fns.reduceRight((value, fn) => fn(value), x); // right to left
pipe(x => x + 1, x => x * 2)(5); // 12
```

### Q19. Find missing number in 1..n

> 🎤 **Interview answer:** "The sum of numbers from 1 to n is `n × (n + 1) / 2`. I calculate what the sum should be, subtract the real sum of the array, and the difference is the missing number. One loop, O(n)."

```js
function missing(arr, n) {
  const expected = (n * (n + 1)) / 2;
  let actual = 0;
  for (const num of arr) actual += num;
  return expected - actual;
}
missing([1, 2, 4, 5], 5); // 3
```

### Q20. Move all zeros to the end

> 🎤 **Interview answer:** "I keep a position `pos` that starts at 0. I loop through the array, and every non-zero number is written at `pos`, then `pos` moves forward. After the loop, I fill the rest with zeros. The order of non-zero numbers stays the same."

```js
function moveZeros(arr) {
  let pos = 0;
  for (const n of arr) {
    if (n !== 0) {
      arr[pos] = n;
      pos++;
    }
  }
  while (pos < arr.length) {
    arr[pos] = 0;
    pos++;
  }
  return arr;
}
moveZeros([0, 1, 0, 3, 12]); // [1, 3, 12, 0, 0]
```

### Q21. Two sum (return indexes)

> 🎤 **Interview answer:** "Slow way: two loops, check every pair, O(n²). Better way: one loop with a Map. For each number, I calculate what I still need: `target - number`. If I have already seen that number, I return both indexes. If not, I save the current number and its index in the Map. One loop, O(n)."

```js
function twoSum(nums, target) {
  const seen = new Map(); // number -> index
  for (let i = 0; i < nums.length; i++) {
    const need = target - nums[i];
    if (seen.has(need)) return [seen.get(need), i];
    seen.set(nums[i], i);
  }
  return [];
}
twoSum([2, 7, 11, 15], 9); // [0, 1]
```

### Q22. Sort array of objects

> 🎤 **Interview answer:** "I pass a compare function to `sort`. For numbers, `a.age - b.age` sorts small to big. For text, I use `localeCompare`. Remember: `sort` changes the original array, so I copy it first if I need the original."

```js
users.sort((a, b) => a.age - b.age);                 // by number
users.sort((a, b) => a.name.localeCompare(b.name));  // by text
const sortedCopy = [...users].sort((a, b) => a.age - b.age); // original stays same
```

### Q23. Capitalize first letter of each word

> 🎤 **Interview answer:** "I split the sentence on spaces. For each word, I make the first letter uppercase and add the rest of the word. I skip empty parts that come from double spaces. Then I join the words back with a space."

```js
function titleCase(sentence) {
  const words = sentence.split(" ");
  const result = [];
  for (const word of words) {
    if (word === "") continue; // skip extra spaces
    result.push(word[0].toUpperCase() + word.slice(1));
  }
  return result.join(" ");
}
titleCase("hello  world"); // "Hello World"
```

### Q24. Max occurring character

> 🎤 **Interview answer:** "One loop with a counter object. Each time I increase a character's count, I also check if it is now the highest count. If yes, I remember that character. So I don't need a second loop."

```js
function maxChar(str) {
  const count = {};
  let maxCh = "";
  let maxCount = 0;
  for (const c of str) {
    count[c] = (count[c] || 0) + 1;
    if (count[c] > maxCount) {
      maxCount = count[c];
      maxCh = c;
    }
  }
  return maxCh;
}
maxChar("javascript"); // "a"
```

### Q25. Fibonacci (with memo)

> 🎤 **Interview answer:** "Plain recursion is very slow, because it calculates the same values again and again. With memo, I save each answer in an object, so every value is calculated only once. Even simpler: a loop with two variables, which is fast and uses almost no memory."

```js
// With memo
function fib(n, memo = {}) {
  if (n <= 1) return n;
  if (memo[n]) return memo[n];
  memo[n] = fib(n - 1, memo) + fib(n - 2, memo);
  return memo[n];
}

// With a loop (best)
function fibLoop(n) {
  let a = 0, b = 1;
  for (let i = 0; i < n; i++) {
    const next = a + b;
    a = b;
    b = next;
  }
  return a;
}
fibLoop(10); // 55
```

### Q26. Intersection and difference of arrays

> 🎤 **Interview answer:** "Intersection = items in both arrays. Difference = items in the first but not the second. I put the second array in a Set once, before the loop, because checking a Set is very fast. Using `includes` inside a loop would be a loop inside a loop, which is slow."

```js
const setB = new Set(b);                          // create once
const intersection = a.filter(x => setB.has(x));  // in both
const difference   = a.filter(x => !setB.has(x)); // only in a
```

### Q27. Get value from nested object by path – get(obj, "a.b.c")

> 🎤 **Interview answer:** "I split the path on dots to get the keys. Then I go one level deeper for each key. If at any point the value is missing, I stop and return the default value instead of crashing."

```js
function get(obj, path, defaultValue) {
  const keys = path.split(".");
  let current = obj;
  for (const key of keys) {
    if (current === null || current === undefined) return defaultValue;
    current = current[key];
  }
  return current === undefined ? defaultValue : current;
}
get({ a: { b: { c: 5 } } }, "a.b.c");      // 5
get({ a: {} }, "a.b.c", "not found");      // "not found"
```

---

## Polyfills

A polyfill = writing your own version of a built-in method. Interviewers love these at 4 years. Know `map`, `filter`, `reduce`, `bind`, `Promise.all`.

**Key idea:** when we add a method to `Array.prototype`, every array gets it, and inside the method `this` is the array.

### P1. Array.prototype.map

> 🎤 **Interview answer:** "I add `myMap` to `Array.prototype`, so `this` is the array. I create an empty result array, loop over each item, call the callback with the item, index and the array, and save what it returns at the same index. The original array is not changed."

```js
Array.prototype.myMap = function (callback) {
  const result = [];
  for (let i = 0; i < this.length; i++) {
    result.push(callback(this[i], i, this));
  }
  return result;
};
[1, 2, 3].myMap(n => n * 2); // [2, 4, 6]
```

### P2. Array.prototype.filter

> 🎤 **Interview answer:** "Same as map, but I push the item only if the callback returns true. So the result can be shorter than the original."

```js
Array.prototype.myFilter = function (callback) {
  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (callback(this[i], i, this)) result.push(this[i]);
  }
  return result;
};
[1, 2, 3, 4].myFilter(n => n % 2 === 0); // [2, 4]
```

### P3. Array.prototype.reduce

> 🎤 **Interview answer:** "The tricky part is the starting value. If it is given, I start with it from index 0. If not, I use the first item as the start and loop from index 1. If the array is empty and there is no starting value, I throw an error, just like the real reduce. In each loop, the result of the callback becomes the new total."

```js
Array.prototype.myReduce = function (callback, initial) {
  let total = initial;
  let start = 0;

  if (total === undefined) {
    if (this.length === 0) throw new TypeError("Reduce of empty array with no initial value");
    total = this[0];
    start = 1;
  }

  for (let i = start; i < this.length; i++) {
    total = callback(total, this[i], i, this);
  }
  return total;
};
[1, 2, 3].myReduce((sum, n) => sum + n, 0); // 6
```

### P4. Function.prototype.bind

> 🎤 **Interview answer:** "`bind` does not run the function. It returns a new function. Inside `myBind`, `this` is the original function, so I save it in `fn`. I return a new function that, when called later, runs `fn` with the given object as `this`, plus the arguments given now and the arguments given later."

```js
Function.prototype.myBind = function (context, ...firstArgs) {
  const fn = this; // the original function
  return function (...laterArgs) {
    return fn.apply(context, [...firstArgs, ...laterArgs]);
  };
};

function hello(greeting) { return greeting + " " + this.name; }
const sayHi = hello.myBind({ name: "Yash" }, "Hi");
sayHi(); // "Hi Yash"
```

### P5. call and apply

> 🎤 **Interview answer:** "The trick: if I call a function as `obj.fn()`, then `this` is `obj`. So I put the function on the object for a moment, call it, then remove it. I use a `Symbol` as the key so I don't overwrite any real property of the object. `apply` is the same, but the arguments come as an array."

```js
Function.prototype.myCall = function (context, ...args) {
  const key = Symbol();       // unique key, won't clash with real properties
  context[key] = this;        // put the function on the object
  const result = context[key](...args); // call it → 'this' is context
  delete context[key];        // clean up
  return result;
};

Function.prototype.myApply = function (context, args = []) {
  return this.myCall(context, ...args);
};
```

### P6. Promise.all

> 🎤 **Interview answer:** "I return a new Promise. I keep a results array and a counter of how many are done. For each promise, when it succeeds, I save the value at the **same index** (so the order is correct) and increase the counter. When the counter equals the total, I resolve with all results. If any one fails, I reject straight away. If the input is empty, I resolve with an empty array."

```js
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let doneCount = 0;

    if (promises.length === 0) return resolve([]);

    promises.forEach((p, i) => {
      Promise.resolve(p)
        .then((value) => {
          results[i] = value; // keep the same order
          doneCount++;
          if (doneCount === promises.length) resolve(results);
        })
        .catch(reject);       // first error → reject everything
    });
  });
}
```

### P7. Promise.allSettled

> 🎤 **Interview answer:** "I change every promise into one that never fails. If it passes, it gives `{ status: 'fulfilled', value }`. If it fails, I catch it and give `{ status: 'rejected', reason }`. Then `Promise.all` on these always succeeds and gives me the status of every promise."

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

> 🎤 **Interview answer:** "I return a new Promise and connect its resolve and reject to every input promise. Whichever finishes first decides the result. A promise can finish only once, so the later ones are ignored."

```js
function race(promises) {
  return new Promise((resolve, reject) => {
    for (const p of promises) {
      Promise.resolve(p).then(resolve, reject);
    }
  });
}
```

---

## Part C – Async coding questions

### A1. sleep / delay

> 🎤 **Interview answer:** "I return a promise that resolves after `setTimeout`. Then `await sleep(1000)` waits 1 second, but only that function waits. The rest of the app keeps running. I use it for waiting between retries."

```js
const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));
await sleep(1000);
```

### A2. Retry a failing async function

> 🎤 **Interview answer:** "I try the function in a loop, up to a max number of times. If it works, I return the result. If it fails on the last try, I throw the error. Otherwise I wait and try again, and each time I wait longer: 500 ms, then 1000, then 2000. This is called exponential backoff. I only retry errors that can fix themselves, like timeouts or 5xx errors, not 400 errors. For payments, I retry only with an idempotency key so the customer is never charged twice."

```js
async function retry(fn, retries = 3, delay = 500) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === retries) throw err;    // last try → give up
      await sleep(delay * 2 ** (attempt - 1)); // wait 500, 1000, 2000...
    }
  }
}
```

### A3. Timeout for a promise

> 🎤 **Interview answer:** "I create a second promise that fails after N ms. Then I use `Promise.race` between the real promise and the timeout promise. Whichever finishes first wins. If the real call is too slow, the timeout wins and we get an error."

```js
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), ms)
  );
  return Promise.race([promise, timeout]);
}
await withTimeout(fetch("/slow-api"), 3000);
```

### A4. Run promises one after another (sequential)

> 🎤 **Interview answer:** "I loop with `for...of` and `await` each task before starting the next. The tasks must be **functions** that return promises, not promises, because a promise starts running as soon as it is created. I use this when order matters."

```js
async function runSequential(tasks) {
  const results = [];
  for (const task of tasks) {
    results.push(await task());
  }
  return results;
}
```

### A5. Run with a concurrency limit (asked a lot at 4+ years)

> 🎤 **Interview answer:** "Example: upload 1000 files, but only 5 at the same time. I start 5 'workers'. Each worker runs a loop: pick the next task from a shared counter, run it, save the result, and pick the next one, until no tasks are left. So at most 5 tasks run at any moment. As soon as one finishes, that worker picks the next one. The library `p-limit` does the same thing."

```js
async function runWithLimit(tasks, limit) {
  const results = [];
  let next = 0; // index of the next task to pick

  async function worker() {
    while (next < tasks.length) {
      const current = next;
      next++;
      results[current] = await tasks[current]();
    }
  }

  const workers = [];
  for (let i = 0; i < limit; i++) workers.push(worker());
  await Promise.all(workers);
  return results;
}
```

### A6. Convert callback function to promise (promisify)

> 🎤 **Interview answer:** "I return a new function. When called, it returns a Promise. Inside, I call the original function and add my own callback at the end. Node callbacks get `(err, data)`: if there is an error I reject, otherwise I resolve with the data. Node already has `util.promisify` for this."

```js
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, data) => {
        if (err) reject(err);
        else resolve(data);
      });
    });
  };
}
const readFile = promisify(require("fs").readFile);
```

### A7. Print 1 to 5 with 1 second gap

> 🎤 **Interview answer:** "Simple way: a `let` loop where each setTimeout waits `i * 1000` ms, so they print at 1, 2, 3, 4 and 5 seconds. Cleaner way: an async loop with `await sleep(1000)` before each print."

```js
for (let i = 1; i <= 5; i++) setTimeout(() => console.log(i), i * 1000);

// or
async function print() {
  for (let i = 1; i <= 5; i++) {
    await sleep(1000);
    console.log(i);
  }
}
```

---

## Part D – Small data-structure questions

### D1. Simple EventEmitter

> 🎤 **Interview answer:** "I keep an object where each event name has a list of listener functions. `on` adds a listener to the list. `off` removes it. `emit` calls every listener for that event. `once` wraps the listener so that it removes itself after running once. Node's own EventEmitter works the same way."

```js
class EventEmitter {
  constructor() {
    this.events = {}; // { eventName: [fn1, fn2] }
  }
  on(name, fn) {
    if (!this.events[name]) this.events[name] = [];
    this.events[name].push(fn);
  }
  off(name, fn) {
    if (!this.events[name]) return;
    this.events[name] = this.events[name].filter(f => f !== fn);
  }
  emit(name, ...args) {
    if (!this.events[name]) return;
    for (const fn of this.events[name]) fn(...args);
  }
  once(name, fn) {
    const wrapper = (...args) => {
      fn(...args);
      this.off(name, wrapper); // remove after first run
    };
    this.on(name, wrapper);
  }
}
```

### D2. LRU Cache (Least Recently Used)

> 🎤 **Interview answer:** "An LRU cache has a size limit. When it is full, it removes the item that was **used longest ago**. I use a JavaScript Map, because a Map remembers the order in which keys were added. So the first key is always the oldest.
> - On `get`: if the key exists, I delete it and add it again, so it moves to the end as 'recently used'.
> - On `put`: if the cache is full, I delete the first key (the oldest), then add the new one.
>
> Redis uses the same idea when its memory is full."

```js
class LRUCache {
  constructor(limit) {
    this.limit = limit;
    this.map = new Map();
  }

  get(key) {
    if (!this.map.has(key)) return -1;
    const value = this.map.get(key);
    this.map.delete(key);      // remove...
    this.map.set(key, value);  // ...and add again → now it's the newest
    return value;
  }

  put(key, value) {
    if (this.map.has(key)) {
      this.map.delete(key);
    } else if (this.map.size === this.limit) {
      const oldestKey = this.map.keys().next().value; // first key = oldest
      this.map.delete(oldestKey);
    }
    this.map.set(key, value);
  }
}
```

### D3. Simple in-memory rate limiter (fixed window)

> 🎤 **Interview answer:** "For each IP, I store how many requests it made and when its time window started. On every request: if the window is over, I reset the count to 0. Then I add 1. If the count is more than the limit, I block the request. This works on one server only. With many servers I use Redis, so all servers share the same counts."

```js
const hits = new Map(); // ip -> { count, start }

function isAllowed(ip, limit = 100, windowMs = 60000) {
  const now = Date.now();
  let entry = hits.get(ip);

  if (!entry || now - entry.start > windowMs) {
    entry = { count: 0, start: now }; // new window
  }

  entry.count++;
  hits.set(ip, entry);
  return entry.count <= limit;
}
```
For multiple servers use Redis – see [03 → Rate limiting](03-nodejs-express.md#10-api-rate-limiting).

### D4. Valid parentheses

> 🎤 **Interview answer:** "I use a stack (an array where I add and remove from the end). When I see an opening bracket, I push it. When I see a closing bracket, I pop the last opening bracket and check that it matches. If it doesn't match, the string is invalid. At the end, the stack must be empty."

```js
function isValid(s) {
  const stack = [];
  const pairs = { ")": "(", "]": "[", "}": "{" };

  for (const c of s) {
    if (c === "(" || c === "[" || c === "{") {
      stack.push(c);
    } else {
      const last = stack.pop();
      if (last !== pairs[c]) return false;
    }
  }
  return stack.length === 0;
}
isValid("({[]})"); // true
isValid("(]");     // false
```

### D5. Longest substring without repeating characters

> 🎤 **Interview answer:** "I use a 'sliding window': a part of the string from `start` to the current position that has no repeated letters. I remember the last position of each letter in a Map. When I see a letter that is already inside my window, I move `start` to just after its last position. At every step, I update the best length. One loop, O(n)."

```js
function longestUnique(s) {
  const lastSeen = new Map(); // letter -> last index
  let start = 0;
  let best = 0;

  for (let i = 0; i < s.length; i++) {
    const ch = s[i];
    if (lastSeen.has(ch) && lastSeen.get(ch) >= start) {
      start = lastSeen.get(ch) + 1; // move window past the repeat
    }
    lastSeen.set(ch, i);
    best = Math.max(best, i - start + 1);
  }
  return best;
}
longestUnique("abcabcbb"); // 3 ("abc")
```

---

**Practice tip:** For each question say: (1) the idea in one line, (2) write the code, (3) how fast it is (Big-O), (4) one edge case (empty input, duplicates, negative numbers).

[Home](README.md) | [← 01 JavaScript](01-javascript.md) | Next: [03 Node.js & Express →](03-nodejs-express.md)
