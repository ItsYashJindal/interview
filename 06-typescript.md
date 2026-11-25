[Home](README.md) | [← 05 React](05-react.md) | Next: [07 APIs & Advanced Backend →](07-apis-backend.md)

# 06 – TypeScript

## Contents
1. [Type vs interface](#1-type-vs-interface)
2. [Union and intersection types](#2-union-and-intersection-types)
3. [Generics](#3-generics)
4. [Enums](#4-enums)
5. [Type assertions](#5-type-assertions)
6. [Optional properties](#6-optional-properties)
7. [Utility types](#7-utility-types)
8. [TypeScript with Express](#8-typescript-with-express)
- [Extra TypeScript questions (added)](#extra-typescript-questions)

---

**Q. What is TypeScript and why use it?**

> 🎤 **Interview answer:** "TypeScript is a superset of JavaScript that adds static types. It's compiled with `tsc` into plain JavaScript, and the types are erased at runtime. The main benefit is catching bugs at compile time, like typos, wrong argument types, or accessing something that might be undefined, before the code ever runs. It also gives much better autocomplete and safer refactoring, and the types act as documentation, which really helps in large teams and microservices where many services share contracts.
>
> One thing I always mention: TypeScript only checks at compile time. Data from outside, like request bodies, third-party APIs or the DB, still needs runtime validation with zod, Joi or class-validator. I've used TypeScript in all my recent work: the NestJS microservices at KindSwap and the Node services at Air India."

TypeScript is JavaScript + static types. It is compiled (`tsc`) to plain JavaScript. Types are removed at runtime.
Benefits: catches bugs before running (typos, wrong arguments, `undefined` access), better autocomplete and refactoring, code is self-documenting. Very useful in large teams and microservices.

**Remember:** TypeScript checks only at **compile time**. Data coming from outside (API body, DB, third-party API) must still be validated at **runtime** (class-validator, zod, joi).

---

### 1. Type vs interface

> 🎤 **Interview answer:** "Both can describe the shape of an object, and for most object types they're interchangeable. The differences are: `type` can also describe unions, primitives, tuples and function types, like `type Status = 'paid' | 'failed'`, which an interface can't. Interfaces support declaration merging, so declaring the same interface twice merges them. That's exactly how we add `user` to Express's Request. And interfaces extend with `extends`, while types compose with `&`.
>
> My rule: interfaces for object shapes and public contracts like DTOs and class shapes, and types for unions, tuples, function types and utility or mapped types. Consistency within the codebase matters more than which one you pick."

Both describe the shape of an object.
```ts
interface User { id: number; name: string; }
type User2 = { id: number; name: string; };
```

| | interface | type |
|---|---|---|
| Object shapes | Yes | Yes |
| Union / primitive / tuple | No | Yes (`type Status = "a" \| "b"`) |
| Extend | `interface B extends A` | `type B = A & {...}` |
| Declaration merging (same name declared twice gets merged) | Yes | No (error) |
| Class `implements` | Yes | Yes (if object type) |

**When to use:** `interface` for object shapes and public contracts (DTOs, class shapes – also needed for merging, e.g. extending Express `Request`). `type` for unions, tuples, function types, and mapped/utility types. Consistency in the codebase matters more than the choice.

```ts
// declaration merging
interface Window { myValue: string; } // adds to existing Window type
```

---

### 2. Union and intersection types

> 🎤 **Interview answer:** "A union, with the pipe symbol, means the value can be one of several types, like `string | number`, or a literal union like `'pending' | 'paid' | 'failed'`, which I use a lot for statuses. Before using a union value, we narrow it with checks like `typeof`. An intersection, with `&`, combines types, so the value must have all properties of both, like an Order type combined with a Timestamps type.
>
> The most useful pattern is the discriminated union: each variant has a common literal field like `type`, and in a switch on that field TypeScript knows exactly which properties exist. I've used this for payment webhook events and Kafka message types."

**Union (`|`)** – value can be **one of** these types.
```ts
type Status = "pending" | "paid" | "failed";   // literal union
function format(id: string | number) {
  if (typeof id === "string") return id.toUpperCase(); // narrowing
  return id.toFixed(0);
}
```

**Intersection (`&`)** – value has **all** properties of both types.
```ts
type Timestamps = { createdAt: Date; updatedAt: Date };
type Order = { id: string; amount: number } & Timestamps;
```

**Discriminated union** (very useful for API responses / events):
```ts
type PaymentEvent =
  | { type: "success"; txnId: string }
  | { type: "failed"; reason: string };

function handle(e: PaymentEvent) {
  switch (e.type) {
    case "success": return e.txnId;   // TS knows txnId exists here
    case "failed":  return e.reason;
  }
}
```

---

### 3. Generics

> 🎤 **Interview answer:** "Generics are type parameters. They let me write a function, class or interface once and use it with many types without losing type safety, unlike `any`. A simple example is `first<T>(arr: T[]): T`, where the return type follows the input. In backend code I use them for a standard `ApiResponse<T>` or `Paginated<T>` wrapper, so every endpoint returns a consistent, typed shape, and for a generic `BaseRepository<T>` with common CRUD methods. With constraints like `T extends { id: string }` or `K extends keyof T`, I can restrict what types are allowed. For example, `getProp(obj, key)` only accepts keys that really exist on the object, and the return type is exactly that property's type."

Generics let you write reusable code that works with **many types** while keeping type safety. Think of them as "type parameters".

```ts
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
first<number>([1, 2]);  // number
first(["a", "b"]);      // string (inferred)
```

**Real backend example – standard API response:**
```ts
interface ApiResponse<T> {
  success: boolean;
  data: T;
  message?: string;
}
const res: ApiResponse<User[]> = { success: true, data: users };

interface Paginated<T> {
  items: T[];
  nextCursor: string | null;
}
```

**Constraints (`extends`):**
```ts
function getId<T extends { id: string }>(obj: T) { return obj.id; }

function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
getProp({ name: "Yash", age: 27 }, "name"); // string
getProp({ name: "Yash" }, "email");          // error
```

**Generic repository (common in NestJS/TypeORM projects):**
```ts
class BaseRepository<T> {
  constructor(private model: Model<T>) {}
  findById(id: string): Promise<T | null> { return this.model.findById(id); }
}
```

---

### 4. Enums

> 🎤 **Interview answer:** "An enum is a named set of constants, like `OrderStatus.Pending` or `OrderStatus.Paid`. Numeric enums auto-increment from 0 and have reverse mapping, but they're less safe and unclear in logs and the database. So I prefer string enums, like `Paid = 'PAID'`. `const enum` gets inlined at compile time with no runtime object.
>
> Honestly, many teams, including me, often prefer a union of string literals instead, because it adds no extra JavaScript and works naturally with JSON. I use real enums when I need a runtime object to iterate over, or when the framework expects them, like NestJS Swagger docs or TypeORM enum columns."

A set of named constants.
```ts
enum OrderStatus {
  Pending = "PENDING",
  Paid = "PAID",
  Shipped = "SHIPPED",
}
order.status = OrderStatus.Paid;
```
- **Numeric enum** (`enum A { X, Y }` → 0, 1) – has reverse mapping, but any number can be assigned in older TS → less safe.
- **String enum** – clearer in logs and DB. Preferred.
- **const enum** – removed at compile time (inlined), no runtime object.

**Enum vs union of strings?**
Many teams prefer `type Status = "PENDING" | "PAID"` (no extra JS code, works well with JSON). Use enums when you want a runtime object to loop over or when the framework expects them (NestJS Swagger, TypeORM enum columns).

---

### 5. Type assertions

> 🎤 **Interview answer:** "A type assertion, with `as`, tells the compiler 'trust me, I know this type'. It does no runtime check at all, so if I'm wrong, the bug just moves to runtime. The non-null assertion `!` and especially `as unknown as X` are warning signs in code review. So instead of asserting API or JSON data, I prefer validating it with zod, which also gives me the type, or writing a type guard function with an `x is User` return type that actually checks at runtime. The newer `satisfies` operator is also useful: it checks that a value matches a type but keeps the precise inferred type."

Tell the compiler "trust me, I know the type better".
```ts
const input = document.getElementById("email") as HTMLInputElement;
const data = JSON.parse(body) as User;
```
- Assertions do **no runtime check**. If you are wrong, bugs happen at runtime. Prefer validation or type guards.
- **Non-null assertion** `user!.name` – tells TS it's not null. Use rarely.
- `as unknown as X` – double assertion; a sign something is wrong.

**Type guard (better than assertion):**
```ts
function isUser(x: unknown): x is User {
  return typeof x === "object" && x !== null && "id" in x;
}
```

**`satisfies` operator (TS 4.9+)** – checks a value matches a type but keeps the exact inferred type:
```ts
const config = { port: 3000, env: "prod" } satisfies Record<string, string | number>;
```

---

### 6. Optional properties

> 🎤 **Interview answer:** "An optional property, like `phone?: string`, means the key may be missing, so its type is effectively `string | undefined`, and TypeScript forces me to handle the undefined case, usually with optional chaining or `??`. Functions can have optional parameters or default values the same way. `readonly` marks properties that can't be reassigned after creation, like `createdAt`. There's a subtle difference: `phone?: string` means the key can be absent, while `phone: string | undefined` means the key must be present but can hold undefined."

```ts
interface User {
  id: string;
  email: string;
  phone?: string;            // may be missing (type is string | undefined)
  readonly createdAt: Date;  // can't be changed after creation
}

function greet(name: string, title?: string) {}   // optional parameter
function greet2(name: string, title = "Mr") {}    // default value

user.phone?.trim();          // optional chaining
const p = user.phone ?? "N/A";
```
`phone?: string` – the key may not exist. `phone: string | undefined` – key must exist but value can be undefined. (Different with `exactOptionalPropertyTypes`.)

---

### 7. Utility types

> 🎤 **Interview answer:** "Utility types are built-in generic types that transform other types, so I don't have to duplicate definitions. The ones I use most: `Partial<T>` makes all fields optional, which is perfect for update DTOs. `Pick` and `Omit` select or remove fields, so `Omit<User, 'password'>` is my public user response type. `Record<K, V>` is for maps, like counts per status. `Readonly`, `Required` and `NonNullable` adjust modifiers. And `ReturnType`, `Parameters` and `Awaited` extract types from functions and promises. That way, if the base User type changes, all derived DTOs update automatically. Internally they're just mapped types, for example Partial is `{ [K in keyof T]?: T[K] }`."

| Utility | What it does | Example use |
|---|---|---|
| `Partial<T>` | All fields optional | Update DTO: `Partial<User>` |
| `Required<T>` | All fields required | Config after defaults applied |
| `Readonly<T>` | All fields readonly | Immutable config |
| `Pick<T, K>` | Only chosen fields | `Pick<User, "id" \| "name">` for public profile |
| `Omit<T, K>` | Remove fields | `Omit<User, "password">` for API response |
| `Record<K, V>` | Object with keys K and values V | `Record<OrderStatus, number>` counts |
| `Exclude<U, X>` | Remove types from a union | `Exclude<Status, "failed">` |
| `Extract<U, X>` | Keep only matching union members | |
| `NonNullable<T>` | Remove null and undefined | |
| `ReturnType<F>` | Return type of a function | `ReturnType<typeof createUser>` |
| `Parameters<F>` | Parameter types as tuple | |
| `Awaited<T>` | Type inside a Promise | `Awaited<ReturnType<typeof fetchUser>>` |

```ts
interface User { id: string; name: string; email: string; password: string; }

type CreateUserDto = Omit<User, "id">;
type UpdateUserDto = Partial<Omit<User, "id">>;
type PublicUser = Omit<User, "password">;
```

**How Partial works inside (mapped type):**
```ts
type MyPartial<T> = { [K in keyof T]?: T[K] };
```

---

### 8. TypeScript with Express

> 🎤 **Interview answer:** "I install `@types/express` and `@types/node`, enable `strict` mode in tsconfig, and type the handlers with Request, Response and NextFunction. Request is generic over params, response body, request body and query, so `Request<{id: string}, {}, CreateOrderBody>` gives typed `req.params` and `req.body`. To add `req.user` after JWT verification, I use declaration merging in a `.d.ts` file on the global Express namespace's Request interface.
>
> Since types disappear at runtime, I validate bodies with zod in a `validate(schema)` middleware, and infer the TypeScript type from the same schema with `z.infer`, so validation and types never get out of sync. In production I compile with `tsc` to `dist` and run the JavaScript with PM2 or Docker."

**Setup:**
```bash
npm i express
npm i -D typescript @types/express @types/node ts-node-dev
npx tsc --init
```

**Typed route handler:**
```ts
import express, { Request, Response, NextFunction } from "express";

interface CreateOrderBody { productId: string; qty: number; }
interface OrderParams { id: string; }

app.post("/orders", async (req: Request<{}, {}, CreateOrderBody>, res: Response) => {
  const { productId, qty } = req.body;   // typed
  res.status(201).json({ productId, qty });
});

app.get("/orders/:id", (req: Request<OrderParams>, res: Response) => {
  res.json({ id: req.params.id });
});
```
`Request<Params, ResBody, ReqBody, Query>` – the generic order.

**Adding `user` to Request (asked often):**
```ts
// src/types/express.d.ts
import { JwtPayload } from "./auth";
declare global {
  namespace Express {
    interface Request {
      user?: JwtPayload;
    }
  }
}
export {};
```
This uses **declaration merging** – that's why `Request` is an interface.

**Typed middleware and error handler:**
```ts
export const auth = (req: Request, res: Response, next: NextFunction) => { ... };

export const errorHandler = (err: AppError, req: Request, res: Response, _next: NextFunction) => {
  res.status(err.statusCode ?? 500).json({ message: err.message });
};
```

**Runtime validation with zod (types + validation from one place):**
```ts
import { z } from "zod";
const CreateOrder = z.object({ productId: z.string(), qty: z.number().int().positive() });
type CreateOrder = z.infer<typeof CreateOrder>;

const validate = (schema: z.ZodSchema) => (req: Request, res: Response, next: NextFunction) => {
  const result = schema.safeParse(req.body);
  if (!result.success) return res.status(400).json(result.error.flatten());
  req.body = result.data;
  next();
};
```

**Important tsconfig options:** `"strict": true` (turn on all strict checks), `"target"`, `"module"`, `"outDir": "dist"`, `"rootDir": "src"`, `"esModuleInterop": true`, `"sourceMap": true`.

---

## Extra TypeScript questions

### Q. any vs unknown vs never

> 🎤 **Interview answer:** "`any` switches off type checking completely, so it spreads unsafety, and I avoid it. `unknown` also means the value could be anything, but TypeScript won't let me use it until I narrow it with a check. It's the safe choice for JSON input and caught errors. `never` represents something that can never happen, like a function that always throws, or the impossible branch of a switch. I use it in an `assertNever` helper in the default case of switches on discriminated unions, so if someone adds a new event type and forgets to handle it, the build fails."

- `any` – turns off type checking. Avoid.
- `unknown` – "could be anything", but you must check the type before using it. Safe version of `any` (use for `catch (e)` and JSON input).
- `never` – value that never happens: function that always throws, or the impossible case in a switch.
```ts
function assertNever(x: never): never { throw new Error("Unexpected: " + x); }
```
Use `assertNever` in the default of a switch on a discriminated union – if someone adds a new type and forgets to handle it, TS gives an error.

### Q. void vs undefined

> 🎤 **Interview answer:** "`void` as a return type means the function doesn't return anything meaningful, and callers shouldn't use its result. `undefined` is an actual value and type. A function typed to return `undefined` must explicitly return it. In practice, `void` is used for function return types and `undefined` for values that may be absent."

`void` – function returns nothing useful. `undefined` – an actual value.

### Q. keyof and typeof

> 🎤 **Interview answer:** "`typeof`, in a type position, gets the type of an existing value, and `keyof` gets a union of the keys of a type. Together, `keyof typeof roles` turns an object's keys into a literal union like `'admin' | 'user'`. It's a nice way to keep a runtime object and its type in sync from a single source, and it's also useful for typing config lookups."

```ts
const roles = { admin: 1, user: 2 };
type Role = keyof typeof roles; // "admin" | "user"
```

### Q. Type narrowing

> 🎤 **Interview answer:** "Narrowing is how TypeScript refines a broad type, like a union or unknown, to a more specific one inside a code block, based on runtime checks. It understands `typeof`, `instanceof`, the `in` operator, equality and truthiness checks, discriminant fields in discriminated unions, and custom type guards that return `x is Type`. After the check, I can safely use the properties of the narrowed type without assertions."

TS figures out the exact type inside a block using `typeof`, `instanceof`, `in`, equality checks, discriminant fields, custom type guards.

### Q. Access modifiers

> 🎤 **Interview answer:** "`public` is the default and accessible anywhere. `private` is only inside the class. `protected` is the class and its subclasses. `readonly` can't be reassigned after construction. TypeScript's `private` is compile-time only, while JavaScript's `#field` is truly private at runtime. In NestJS I use parameter properties all the time, like `constructor(private readonly userRepo: UserRepository)`, which declares and assigns the injected dependency in one line."

`public` (default), `private` (only inside class), `protected` (class + subclasses), `readonly`. TS `private` is compile-time only; `#field` is real JS private.
```ts
class UserService {
  constructor(private readonly repo: UserRepository) {} // parameter property – used everywhere in NestJS
}
```

### Q. Abstract class vs interface

> 🎤 **Interview answer:** "An interface is just a type contract. It has no implementation and disappears completely after compilation. An abstract class can contain real implemented methods along with abstract ones that subclasses must implement. It can't be instantiated with `new`, and it exists at runtime. That runtime presence matters in NestJS: since interfaces vanish, you can't use them as injection tokens directly, but an abstract class can be used as a token. I've used that pattern for a `PaymentGateway` abstraction with an implementation per gateway."

- Interface – only a shape, no code, removed at runtime.
- Abstract class – can have code + abstract methods; cannot be created with `new`; exists at runtime (NestJS can use it as an injection token).

### Q. Function overloads

> 🎤 **Interview answer:** "Function overloads let me declare multiple call signatures for one function, so the return type depends on the argument types. For example, `parse(string)` returns a number and `parse(number)` returns a string. I write the overload signatures first, then one implementation signature that handles all cases. Callers only see the overloads. In many cases generics or union types are simpler, so I use overloads only when the input-output relationship really differs."

```ts
function parse(x: string): number;
function parse(x: number): string;
function parse(x: any) { return typeof x === "string" ? Number(x) : String(x); }
```

### Q. Tuples

> 🎤 **Interview answer:** "A tuple is a fixed-length array where each position has a known type, like `[number, number]` for coordinates, or `[Error | null, Data]`. I've used them for GeoJSON coordinates, which are in `[longitude, latitude]` order, and React's `useState` also returns a tuple. They can be labelled and readonly for clarity."

Fixed-length arrays with known types: `const point: [number, number] = [77.1, 28.4];`

### Q. Decorators

> 🎤 **Interview answer:** "Decorators are functions, prefixed with `@`, that attach metadata or behaviour to classes, methods, properties or parameters. NestJS and TypeORM are built on them: `@Controller`, `@Get`, `@Injectable`, `@UseGuards`, `@Entity` and `@Column`. Nest reads that metadata to do routing and dependency injection. It needs `experimentalDecorators` and `emitDecoratorMetadata` enabled in tsconfig. I've also written custom decorators, like a `@Roles()` decorator used by an RBAC guard."

Functions that add metadata or behaviour to classes, methods, properties – `@Controller()`, `@Get()`, `@Injectable()`, `@Column()`. Heavily used in NestJS and TypeORM. Need `"experimentalDecorators": true` and `"emitDecoratorMetadata": true` for Nest.

### Q. Declaration files (.d.ts)

> 🎤 **Interview answer:** "Declaration files contain only type information, with no implementation, so TypeScript knows the types of JavaScript code. Most libraries ship their own types, or we install them from DefinitelyTyped, like `@types/express`. For an untyped internal or third-party library, I write a small `.d.ts` with `declare module`. I also use them for global augmentations, like adding `user` to Express's Request."

Contain only types for JS libraries. `@types/express` gives types for Express. You can write your own `.d.ts` for untyped libraries.

### Q. How is TS code run in production?

> 🎤 **Interview answer:** "In production I don't run TypeScript directly. The build step compiles with `tsc`, or the Nest CLI, into JavaScript in a `dist` folder, usually in CI or a multi-stage Docker build, and then we run `node dist/main.js` under PM2 or in a container. ts-node or tsx is only for local development, because compiling at runtime adds startup time and memory. I enable source maps so error stack traces point to the TypeScript lines."

Compile with `tsc` to `dist/`, run `node dist/main.js` (with PM2/Docker). Don't run `ts-node` in production.

### Q. Mapped and conditional types (basic)

> 🎤 **Interview answer:** "Mapped types build a new type by looping over the keys of an existing one, like `{ [K in keyof T]: T[K] | null }` to make every field nullable. That's how Partial and Readonly are built. Conditional types choose a type based on a condition, `T extends X ? A : B`, like a type-level ternary. With `infer` they can extract types, which is how ReturnType works. In day-to-day work I mostly use the built-in utilities, but knowing these explains how they work."

```ts
type Nullable<T> = { [K in keyof T]: T[K] | null };       // mapped
type IsString<T> = T extends string ? "yes" : "no";        // conditional
```

### Q. Index signature

> 🎤 **Interview answer:** "An index signature, like `{ [key: string]: string }`, describes an object whose keys aren't known in advance but whose values all share a type, such as HTTP headers or a dictionary loaded from config. For a known set of keys, I prefer `Record<'a' | 'b', T>`. With `noUncheckedIndexedAccess`, TypeScript adds `undefined` to lookups, which is safer because the key might not exist."

```ts
interface Headers { [key: string]: string; }
```

---

[Home](README.md) | [← 05 React](05-react.md) | Next: [07 APIs & Advanced Backend →](07-apis-backend.md)
