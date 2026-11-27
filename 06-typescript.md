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

> 🎤 **Interview answer:** "TypeScript is JavaScript **plus types**. We write the type of every variable and function, like `name: string`. Before running, TypeScript converts the code to normal JavaScript, and the types are removed.
>
> The main benefit: it **catches mistakes while we write code**, before it runs. For example a typo, passing a number where a string is expected, or using something that might be `undefined`. It also gives better auto-complete, and makes big changes safer. This really helps in big teams and microservices.
>
> One important point: TypeScript checks only **while writing / building**. Data from outside, like the request body or a third-party API, can still be wrong at runtime. So I still validate it with zod, Joi or class-validator. I've used TypeScript in all my recent work: NestJS at KindSwap and the Node services at Air India."

---

### 1. Type vs interface

> 🎤 **Interview answer:** "Both can describe the shape of an object, and for normal objects either one works.
>
> Differences:
> - `type` can also describe things that aren't objects, like `type Status = 'paid' | 'failed'`. An interface can't.
> - If I declare the same `interface` twice, TypeScript **merges** them. This is how we add `user` to Express's `Request`. `type` can't do this.
> - Interface extends with `extends`. Type combines with `&`.
>
> My rule: `interface` for object shapes like DTOs, `type` for unions and everything else."

```ts
interface User { id: number; name: string; }
type User2 = { id: number; name: string; };
```

| | interface | type |
|---|---|---|
| Object shapes | Yes | Yes |
| Union like `"a" \| "b"` | No | Yes |
| Extend | `interface B extends A` | `type B = A & {...}` |
| Same name declared twice gets merged | Yes | No (error) |

---

### 2. Union and intersection types

> 🎤 **Interview answer:**
> - "**Union `|`** means 'one of these'. `string | number` means it can be a string **or** a number. I use it a lot for status values: `'pending' | 'paid' | 'failed'`. Before using a union, I check which one it is with `typeof`.
> - **Intersection `&`** means 'all of these together'. The value must have all fields of both types.
>
> A very useful pattern is the **discriminated union**: every option has a common field like `type`. When I `switch` on that field, TypeScript knows exactly which other fields exist. I've used this for payment webhook events and Kafka messages."

**Union (`|`)** – one of these types.
```ts
type Status = "pending" | "paid" | "failed";

function format(id: string | number) {
  if (typeof id === "string") return id.toUpperCase(); // here TS knows it's a string
  return id.toFixed(0);                                // here TS knows it's a number
}
```

**Intersection (`&`)** – has all fields of both.
```ts
type Timestamps = { createdAt: Date; updatedAt: Date };
type Order = { id: string; amount: number } & Timestamps;
```

**Discriminated union** (great for events and API responses):
```ts
type PaymentEvent =
  | { type: "success"; txnId: string }
  | { type: "failed"; reason: string };

function handle(e: PaymentEvent) {
  switch (e.type) {
    case "success": return e.txnId;   // TS knows txnId exists here
    case "failed":  return e.reason;  // TS knows reason exists here
  }
}
```

---

### 3. Generics

> 🎤 **Interview answer:** "Generics let me write a function or type **once** and use it with **many types**, while still keeping type safety. Think of `<T>` as a 'type variable', a placeholder that gets filled in when the function is used.
>
> Simple example: `first<T>(arr: T[]): T`. If I pass numbers, it returns a number. If I pass strings, it returns a string.
>
> In backend code I use it for a common API response shape, `ApiResponse<T>`, so every endpoint returns the same structure with the right data type inside. I also use it for a `BaseRepository<T>` with common CRUD methods for every model."

```ts
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
first([1, 2]);      // returns number
first(["a", "b"]);  // returns string
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

**Limiting what T can be (`extends`):**
```ts
// T must have an id
function getId<T extends { id: string }>(obj: T) {
  return obj.id;
}

// key must be a real key of obj
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
getProp({ name: "Yash", age: 27 }, "name"); // OK, returns string
getProp({ name: "Yash" }, "email");          // Error – no such key
```

**Generic repository (common in NestJS/TypeORM projects):**
```ts
class BaseRepository<T> {
  constructor(private model: Model<T>) {}
  findById(id: string): Promise<T | null> {
    return this.model.findById(id);
  }
}
```

---

### 4. Enums

> 🎤 **Interview answer:** "An enum is a fixed list of named values, like `OrderStatus.Pending` or `OrderStatus.Paid`. I prefer **string enums**, like `Paid = 'PAID'`, because the value is readable in logs and in the database. Number enums (0, 1, 2) are hard to understand when you see them in the DB.
>
> Many teams, including me, often use a simple union instead: `type Status = 'PENDING' | 'PAID'`. It adds no extra JavaScript code. I use real enums when a framework needs them, like NestJS Swagger or TypeORM enum columns."

```ts
enum OrderStatus {
  Pending = "PENDING",
  Paid = "PAID",
  Shipped = "SHIPPED",
}
order.status = OrderStatus.Paid;

// Simple alternative
type Status = "PENDING" | "PAID" | "SHIPPED";
```

---

### 5. Type assertions

> 🎤 **Interview answer:** "A type assertion with `as` tells TypeScript: 'trust me, I know this type'. But it does **no real check** when the code runs. If I'm wrong, the bug just shows up later at runtime.
>
> So I avoid it for outside data. Instead, I validate the data with zod, or write a **type guard**: a small function that really checks the data and tells TypeScript the type."

```ts
const input = document.getElementById("email") as HTMLInputElement;
const data = JSON.parse(body) as User; // risky – no real check
```
- `user!.name` – the `!` tells TS "this is not null". Use rarely.
- `as unknown as X` – a sign something is wrong.

**Type guard (better than `as`):**
```ts
function isUser(x: unknown): x is User {
  return typeof x === "object" && x !== null && "id" in x;
}

if (isUser(data)) {
  data.id; // TS now knows it's a User
}
```

**`satisfies`** – checks that a value matches a type, but keeps its exact type:
```ts
const config = { port: 3000, env: "prod" } satisfies Record<string, string | number>;
```

---

### 6. Optional properties

> 🎤 **Interview answer:** "`phone?: string` means the field **may be missing**. So its type is `string | undefined`, and TypeScript makes me handle the missing case, usually with `?.` or `??`. Functions can have optional parameters the same way, or default values. `readonly` means the field can't be changed after it is created, like `createdAt`."

```ts
interface User {
  id: string;
  email: string;
  phone?: string;            // may be missing
  readonly createdAt: Date;  // can't be changed after creation
}

function greet(name: string, title?: string) {}   // optional parameter
function greet2(name: string, title = "Mr") {}    // default value

user.phone?.trim();               // safe if phone is missing
const p = user.phone ?? "N/A";    // default if missing
```

---

### 7. Utility types

> 🎤 **Interview answer:** "Utility types are ready-made helpers that create a new type from an existing one, so I don't have to write the same fields again. The ones I use most:
> - `Partial<User>` – all fields optional. Perfect for an 'update user' DTO.
> - `Omit<User, 'password'>` – same type without the password. I use it for API responses.
> - `Pick<User, 'id' | 'name'>` – only the chosen fields.
> - `Record<Status, number>` – an object with these keys and these value types.
>
> The benefit: if the main `User` type changes, all these types update automatically."

| Utility | What it does | Example use |
|---|---|---|
| `Partial<T>` | All fields optional | Update DTO: `Partial<User>` |
| `Required<T>` | All fields required | |
| `Readonly<T>` | No field can be changed | Config object |
| `Pick<T, K>` | Keep only chosen fields | `Pick<User, "id" \| "name">` |
| `Omit<T, K>` | Remove fields | `Omit<User, "password">` |
| `Record<K, V>` | Object with keys K and values V | `Record<OrderStatus, number>` |
| `Exclude<U, X>` | Remove options from a union | `Exclude<Status, "failed">` |
| `NonNullable<T>` | Remove null and undefined | |
| `ReturnType<F>` | The return type of a function | `ReturnType<typeof createUser>` |
| `Awaited<T>` | The type inside a Promise | |

```ts
interface User { id: string; name: string; email: string; password: string; }

type CreateUserDto = Omit<User, "id">;            // everything except id
type UpdateUserDto = Partial<Omit<User, "id">>;   // same, but all optional
type PublicUser    = Omit<User, "password">;      // safe to send in API
```

---

### 8. TypeScript with Express

> 🎤 **Interview answer:** "I install `@types/express` and `@types/node`, and turn on `strict` mode in tsconfig. Then I type the handlers with `Request` and `Response`. `Request` can take types for params, body and query, so `req.params` and `req.body` are typed.
>
> To add `req.user` after checking the JWT, I write a small `.d.ts` file that adds `user` to Express's `Request` interface. This works because interfaces can be merged.
>
> Since types are removed at runtime, I validate the body with **zod**. Nice trick: I create the TypeScript type from the same zod schema with `z.infer`, so the validation and the type always match. In production, I compile to JavaScript with `tsc` and run it with PM2 or Docker."

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

// Request<Params, ResponseBody, RequestBody, Query>
app.post("/orders", async (req: Request<{}, {}, CreateOrderBody>, res: Response) => {
  const { productId, qty } = req.body;   // typed
  res.status(201).json({ productId, qty });
});

app.get("/orders/:id", (req: Request<OrderParams>, res: Response) => {
  res.json({ id: req.params.id });
});
```

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

**Typed middleware and error handler:**
```ts
export const auth = (req: Request, res: Response, next: NextFunction) => { /* ... */ };

export const errorHandler = (err: AppError, req: Request, res: Response, _next: NextFunction) => {
  res.status(err.statusCode ?? 500).json({ message: err.message });
};
```

**Validation with zod (type + check from one place):**
```ts
import { z } from "zod";

const CreateOrder = z.object({
  productId: z.string(),
  qty: z.number().int().positive(),
});
type CreateOrder = z.infer<typeof CreateOrder>; // TS type made from the schema

const validate = (schema: z.ZodSchema) => (req: Request, res: Response, next: NextFunction) => {
  const result = schema.safeParse(req.body);
  if (!result.success) return res.status(400).json(result.error.flatten());
  req.body = result.data;
  next();
};

app.post("/orders", validate(CreateOrder), createOrder);
```

**Important tsconfig options:** `"strict": true` (turn on all checks), `"outDir": "dist"`, `"rootDir": "src"`, `"esModuleInterop": true`, `"sourceMap": true`.

---

## Extra TypeScript questions

### Q. any vs unknown vs never

> 🎤 **Interview answer:**
> - "`any` – turns off type checking completely. I avoid it.
> - `unknown` – 'could be anything', but TypeScript **won't let me use it** until I check its type. The safe version of `any`. I use it for JSON input and caught errors.
> - `never` – something that can never happen, like a function that always throws.
>
> A nice trick with `never`: in the `default` case of a switch, I call `assertNever(x)`. If someone adds a new event type and forgets to handle it, the build fails."

```ts
function assertNever(x: never): never {
  throw new Error("Unexpected: " + x);
}
```

### Q. void vs undefined

> 🎤 **Interview answer:** "`void` means a function returns nothing useful. `undefined` is an actual value. I use `void` for function return types."

### Q. keyof and typeof

> 🎤 **Interview answer:** "`typeof` gets the type of an existing variable. `keyof` gets the keys of a type as a union. Together, `keyof typeof roles` turns the keys of an object into `'admin' | 'user'`. So the object and the type always stay in sync."

```ts
const roles = { admin: 1, user: 2 };
type Role = keyof typeof roles; // "admin" | "user"
```

### Q. Type narrowing

> 🎤 **Interview answer:** "Narrowing means TypeScript figures out a more exact type inside an `if` block, based on my check. For example, after `if (typeof id === 'string')`, TypeScript knows `id` is a string inside that block. It understands `typeof`, `instanceof`, `in`, `===` checks, and type guard functions."

### Q. Access modifiers

> 🎤 **Interview answer:**
> - "`public` – can be used anywhere (default).
> - `private` – only inside the class.
> - `protected` – inside the class and its child classes.
> - `readonly` – can't be changed after it is set.
>
> In NestJS I use `constructor(private readonly userRepo: UserRepository)` everywhere. This one line creates the property and saves the injected dependency."

```ts
class UserService {
  constructor(private readonly repo: UserRepository) {}
}
```

### Q. Abstract class vs interface

> 🎤 **Interview answer:**
> - "**Interface** – only describes a shape. No real code. It disappears after compiling.
> - **Abstract class** – can have real code plus 'abstract' methods that child classes must write. We can't create it with `new`. It still exists at runtime.
>
> This matters in NestJS: because interfaces disappear, Nest can't use them for dependency injection directly, but it can use an abstract class. I used an abstract `PaymentGateway` class, with one implementation per payment gateway."

### Q. Function overloads

> 🎤 **Interview answer:** "Overloads let one function have different input/output types. For example, `parse(string)` returns a number, and `parse(number)` returns a string. I write the signatures first, then one real function that handles all cases. Often, generics or unions are simpler, so I use overloads rarely."

```ts
function parse(x: string): number;
function parse(x: number): string;
function parse(x: any) {
  return typeof x === "string" ? Number(x) : String(x);
}
```

### Q. Tuples

> 🎤 **Interview answer:** "A tuple is an array with a fixed length, where each position has a known type. For example `[number, number]` for map coordinates `[longitude, latitude]`. React's `useState` also returns a tuple: `[value, setValue]`."

```ts
const point: [number, number] = [77.1, 28.4];
```

### Q. Decorators

> 🎤 **Interview answer:** "Decorators are functions with `@` in front that add extra info or behaviour to a class, method or property. NestJS and TypeORM are built on them: `@Controller`, `@Get`, `@Injectable`, `@Entity`, `@Column`. Nest reads this info to set up routes and dependency injection. I've also written my own, like a `@Roles('admin')` decorator used by a guard."

### Q. Declaration files (.d.ts)

> 🎤 **Interview answer:** "A `.d.ts` file has only types, no real code. It tells TypeScript the types of a JavaScript library. Most libraries include them, or we install them like `@types/express`. I also use them to add things to existing types, like `user` on Express's `Request`."

### Q. How is TS code run in production?

> 🎤 **Interview answer:** "In production I don't run TypeScript directly. I compile it to JavaScript with `tsc` into a `dist` folder during the build, and then run `node dist/main.js` with PM2 or Docker. `ts-node` is only for local development, because it's slower. I turn on source maps, so error messages point to the right TypeScript line."

### Q. Mapped and conditional types (basic)

> 🎤 **Interview answer:**
> - "A **mapped type** creates a new type by going over every key of another type. For example, make every field nullable. This is how `Partial` is built.
> - A **conditional type** is like an `if` for types: `T extends string ? 'yes' : 'no'`.
>
> Day to day I mostly use the built-in utility types, but this explains how they work."

```ts
type Nullable<T> = { [K in keyof T]: T[K] | null };  // mapped
type IsString<T> = T extends string ? "yes" : "no";  // conditional
```

### Q. Index signature

> 🎤 **Interview answer:** "An index signature describes an object where we **don't know the key names** in advance, but all values have the same type. For example HTTP headers: `{ [key: string]: string }`. If I know the keys, I use `Record<'a' | 'b', T>` instead."

```ts
interface Headers { [key: string]: string; }
```

---

[Home](README.md) | [← 05 React](05-react.md) | Next: [07 APIs & Advanced Backend →](07-apis-backend.md)
