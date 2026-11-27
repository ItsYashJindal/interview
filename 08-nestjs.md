[Home](README.md) | [← 07 APIs & Advanced Backend](07-apis-backend.md) | Next: [09 Kafka, Redis & Messaging →](09-kafka-redis-messaging.md)

# 08 – NestJS (added – it is in your resume title)

Your resume says "Node.js · TypeScript · NestJS" and you built 2 NestJS microservices at KindSwap. Expect at least 5–6 NestJS questions.

## Contents
1. [What is NestJS and why use it](#1-what-is-nestjs-and-why-use-it)
2. [Modules, controllers, providers](#2-modules-controllers-providers)
3. [Dependency injection](#3-dependency-injection)
4. [Request lifecycle order](#4-request-lifecycle-order)
5. [Middleware vs guards vs interceptors vs pipes vs filters](#5-middleware-vs-guards-vs-interceptors-vs-pipes-vs-filters)
6. [DTOs and validation](#6-dtos-and-validation)
7. [Guards and RBAC with Auth0](#7-guards-and-rbac-with-auth0)
8. [Exception filters](#8-exception-filters)
9. [Microservices in NestJS](#9-microservices-in-nestjs)
10. [Config, TypeORM, testing](#10-config-typeorm-testing)
- [More NestJS questions](#more-nestjs-questions)

---

### 1. What is NestJS and why use it

> 🎤 **Interview answer:** "NestJS is a Node.js framework written in TypeScript. It runs on top of Express, and gives a **fixed structure**: modules, controllers, services, dependency injection and decorators.
>
> Why use it instead of plain Express?
> - **Same structure everywhere** – every developer knows where each piece of code goes. This really helps big teams and new joiners.
> - **Easy testing** – because of dependency injection, I can swap a real database with a fake one in tests.
> - **Lots built in** – validation, guards, Swagger docs, and support for Redis, Kafka and other message systems.
>
> The downside: more code to write, and it takes time to learn. For a tiny service, plain Express is enough. I used NestJS to build the Referral and Staking microservices at KindSwap."

---

### 2. Modules, controllers, providers

> 🎤 **Interview answer:** "These are the three main building blocks:
> - **Module** – a box that groups everything for one feature: its controller, its services, and what it shares with other modules. `AppModule` is the main box.
> - **Controller** – handles the HTTP part. It defines routes with `@Get`, `@Post`, reads params and body, and calls a service. I keep controllers thin, with no business logic.
> - **Provider** – anything Nest can inject, mostly **services**, which hold the business logic.
>
> For example, in the Referral service I had a `ReferralModule` with a `ReferralController`, a `ReferralService`, and the database repository for referrals."

```ts
// referral.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([Referral]), UsersModule],
  controllers: [ReferralController],
  providers: [ReferralService],
  exports: [ReferralService],   // other modules can use it
})
export class ReferralModule {}

// referral.controller.ts – handles HTTP, no business logic
@Controller("referrals")
export class ReferralController {
  constructor(private readonly referralService: ReferralService) {}

  @Get(":id")
  findOne(@Param("id", ParseUUIDPipe) id: string) {
    return this.referralService.findOne(id);
  }

  @Post()
  create(@Body() dto: CreateReferralDto, @Req() req) {
    return this.referralService.create(dto, req.user.sub);
  }
}

// referral.service.ts – business logic
@Injectable()
export class ReferralService {
  constructor(@InjectRepository(Referral) private repo: Repository<Referral>) {}

  findOne(id: string) {
    return this.repo.findOneByOrFail({ id });
  }
}
```

- **Global module** – `@Global()`, so you don't need to import it everywhere (e.g. Config, Redis).
- **Dynamic module** – a module you configure when importing it, like `TypeOrmModule.forRoot({...})`.

---

### 3. Dependency injection

**Q. What is DI?**

> 🎤 **Interview answer:** "Dependency injection means a class **doesn't create the things it needs** itself with `new`. Instead, it just asks for them in its constructor, and NestJS creates them and passes them in.
>
> Example: `ReferralService` needs a repository. It doesn't do `new Repository()`. It just writes it in the constructor, and Nest gives it.
>
> Benefits:
> - Classes are loosely connected.
> - Easy to swap one implementation for another.
> - **Easy testing**: in a test, I give the service a fake (mock) repository instead of the real database."

```ts
// In a test – give a fake repository instead of the real one
const module = await Test.createTestingModule({
  providers: [
    ReferralService,
    { provide: getRepositoryToken(Referral), useValue: { findOneByOrFail: jest.fn() } },
  ],
}).compile();
```

**Provider types:**

> 🎤 **Interview answer:** "Besides normal services, I can tell Nest exactly what to inject:
> - `useClass` – use this class. For example, the token `PAYMENT_GATEWAY` → `RazorpayGateway`.
> - `useValue` – use this fixed value, like a config object, or a mock in tests.
> - `useFactory` – build it with a function, like creating a Redis client using the URL from config."

```ts
{ provide: "PAYMENT_GATEWAY", useClass: RazorpayGateway }   // class
{ provide: "CONFIG", useValue: { retries: 3 } }             // fixed value
{
  provide: "REDIS",                                         // built with a function
  useFactory: (cfg: ConfigService) => new Redis(cfg.get("REDIS_URL")),
  inject: [ConfigService],
}

// Inject it
constructor(@Inject("PAYMENT_GATEWAY") private gateway: PaymentGateway) {}
```

**Scopes:**

> 🎤 **Interview answer:** "By default, Nest creates **one instance** of each service for the whole app (singleton). This is the fastest and what I use almost always. `REQUEST` scope creates a new instance for every request, which is slower. `TRANSIENT` gives each user of the service its own copy. I avoid request scope unless I really need it."

**Circular dependency:**

> 🎤 **Interview answer:** "A circular dependency is when service A needs B, and B needs A. Nest doesn't know which to create first. The quick fix is `forwardRef(() => B)`. But it usually means the design is wrong. The better fix is to move the shared logic into a third service that both use."

---

### 4. Request lifecycle order

> 🎤 **Interview answer:** "When a request comes in, it goes through these steps in order:
> 1. **Middleware** – general things like logging and request id.
> 2. **Guards** – is this user allowed? (JWT check, role check)
> 3. **Interceptors (before)** – things like starting a timer.
> 4. **Pipes** – validate and convert the input.
> 5. **Controller → Service** – the real work.
> 6. **Interceptors (after)** – change the response, like wrapping it in a standard format.
> 7. **Exception filters** – only if an error was thrown anywhere; they build the error response."

Very commonly asked:

```
Incoming request
 → Middleware
 → Guards                 (is the user allowed? true / false)
 → Interceptors (before)  (logging, timing)
 → Pipes                  (validate / convert params and body)
 → Controller method → Service
 → Interceptors (after)   (change the response, e.g. wrap in { data })
 → Exception filters      (only if an error was thrown)
 → Response
```

---

### 5. Middleware vs guards vs interceptors vs pipes vs filters

> 🎤 **Interview answer:**
> - "**Middleware** – same as Express middleware. Runs first. Good for general things like logging.
> - **Guard** – answers one question: **is this request allowed?** Returns true or false. Used for auth and roles.
> - **Interceptor** – runs code **before and after** the controller. Used to change the response, measure time, or add caching.
> - **Pipe** – **checks and converts input**, like validating the body with a DTO, or turning `'5'` into the number 5.
> - **Exception filter** – **turns errors into a proper response**.
>
> Using the right one keeps each job in its own place."

| | Job | Example |
|---|---|---|
| Middleware | Same as Express middleware | Logging, request id |
| Guard | Allow or block the request | JWT auth, roles |
| Interceptor | Code before and after the handler | Response wrapper, timing, cache |
| Pipe | Validate and convert input | `ValidationPipe`, `ParseIntPipe` |
| Exception filter | Turn errors into responses | Custom error format |

**Interceptor example – response time and wrapper:**

> 🎤 **Interview answer:** "In an interceptor, code before `next.handle()` runs before the controller. `next.handle()` gives the controller's result, and I can change it. Here, I wrap every response in `{ success: true, data }` and log how long it took. I used a global interceptor like this so all our microservices returned the same response shape."

```ts
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler) {
    const start = Date.now();                               // before the controller
    return next.handle().pipe(
      map((data) => ({ success: true, data })),             // change the response
      tap(() => console.log(`took ${Date.now() - start}ms`)), // after the controller
    );
  }
}
```
(`next.handle()` returns an RxJS Observable – that's why we use `.pipe(map, tap)`.)

---

### 6. DTOs and validation

> 🎤 **Interview answer:** "A DTO (Data Transfer Object) is a class that says **exactly what data an API accepts**. I add validation decorators to its fields, like `@IsUUID()` or `@IsInt()`.
>
> Then I turn on a global `ValidationPipe` with:
> - `whitelist: true` – removes any extra fields that aren't in the DTO. This stops users from sending things like `role: 'admin'`.
> - `transform: true` – converts the input into the DTO class with the right types.
>
> A DTO must be a **class**, not an interface, because interfaces disappear when the code runs, and the validators need something that exists at runtime. For update DTOs I use `PartialType(CreateDto)`, which makes all fields optional."

```ts
export class CreateStakeDto {
  @IsUUID() poolId: string;
  @IsNumberString() amount: string;
  @IsOptional() @IsInt() @Min(1) lockDays?: number;
}

// Update DTO – same fields, all optional
export class UpdateStakeDto extends PartialType(CreateStakeDto) {}
```
```ts
// main.ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,             // remove fields not in the DTO
  forbidNonWhitelisted: true,  // or give an error for extra fields
  transform: true,             // convert to the DTO class / types
}));
```

---

### 7. Guards and RBAC with Auth0

> 🎤 **Interview answer:** "At KindSwap, we used two guards:
> 1. **JWT guard** – checks the token that Auth0 gave the user. It gets Auth0's public key, verifies the token is real and not expired, and puts the user (with their permissions) on the request.
> 2. **Roles guard** – I made a `@Roles()` decorator to mark which permission a route needs, like `@Roles('payout:approve')`. The guard reads this and checks if the user has that permission.
>
> If the token is invalid → 401. If the permission is missing → the guard returns false → 403."

```ts
// roles.decorator.ts – marks which permissions a route needs
export const Roles = (...roles: string[]) => SetMetadata("roles", roles);

// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(ctx: ExecutionContext): boolean {
    // 1. read what the route needs
    const required = this.reflector.getAllAndOverride<string[]>("roles", [
      ctx.getHandler(),
      ctx.getClass(),
    ]);
    if (!required) return true; // no rule → allow

    // 2. read what the user has (set by the JWT guard)
    const user = ctx.switchToHttp().getRequest().user;
    const userPerms: string[] = user?.permissions ?? [];

    // 3. allow if the user has at least one
    return required.some((r) => userPerms.includes(r));
  }
}

// controller
@UseGuards(AuthGuard("jwt"), RolesGuard)
@Roles("payout:approve")
@Post("payouts/batch")
approveBatch() { /* ... */ }
```

---

### 8. Exception filters

> 🎤 **Interview answer:** "NestJS has ready-made errors like `NotFoundException`, `BadRequestException` and `UnauthorizedException`. When I throw one, Nest sends the right HTTP status automatically.
>
> To make **all errors look the same**, I write one global exception filter that catches everything. If it's a known HTTP error, it uses its status and message. If it's an unknown error, it returns 500 with a general message, so internal details aren't shown. It also logs the full error."

```ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse();

    const isHttp = exception instanceof HttpException;
    const status = isHttp ? exception.getStatus() : 500;
    const message = isHttp ? exception.message : "Internal server error";

    // log the full error here
    res.status(status).json({ success: false, statusCode: status, message });
  }
}
app.useGlobalFilters(new AllExceptionsFilter());
```

---

### 9. Microservices in NestJS

> 🎤 **Interview answer:** "NestJS can connect services using Redis, Kafka, RabbitMQ, TCP and others, and the code looks the same for all of them. There are two styles:
> - **Request–response** – `client.send()` + `@MessagePattern`. The caller **waits** for a reply.
> - **Event** – `client.emit()` + `@EventPattern`. The caller **doesn't wait**. Fire and forget.
>
> At KindSwap, the Staking and Referral services talked through **Redis pub/sub**. For example, when a stake was completed, the Staking service sent an event, and the Referral service gave a reward to the person who referred that user.
>
> One honest limitation: Redis pub/sub **doesn't save messages**. If the Referral service is down at that moment, it misses the event. For events that must never be lost, I'd use Kafka, RabbitMQ or Redis Streams, or save the event in the DB and retry."

```ts
// Staking service sends the event
this.client.emit("stake.completed", { userId, amount });

// Referral service listens
@EventPattern("stake.completed")
handleStake(@Payload() data: StakeCompletedEvent) {
  return this.referralService.creditReferrer(data);
}
```
See [09 → Redis pub/sub](09-kafka-redis-messaging.md#redis-pubsub) for the follow-up question.

**Hybrid app:** one Nest app that serves HTTP **and** listens to messages (`app.connectMicroservice()`).

---

### 10. Config, TypeORM, testing

> 🎤 **Interview answer:**
> - "**Config** – I use `ConfigModule` with a validation schema, so the app stops at startup if a required setting like `DATABASE_URL` is missing.
> - **Database** – at KindSwap I used TypeORM. Tables are defined as classes with decorators, and I inject repositories into services. In production I keep `synchronize: false` and use migrations, because `synchronize` can delete columns. I store money and crypto amounts as `numeric`, never as JavaScript decimal numbers, because those have rounding errors.
> - **Testing** – `Test.createTestingModule` with fake providers for unit tests, and `supertest` for API tests.
> - **Cron jobs** – `@Cron` from `@nestjs/schedule`. But with many servers, **every** server runs the cron. So I need a lock or leader election, which is exactly the problem I solved with Redis."

**Config:**
```ts
ConfigModule.forRoot({
  isGlobal: true,
  validationSchema: Joi.object({ DATABASE_URL: Joi.string().required() }),
});
```

**TypeORM:**
```ts
@Entity()
export class Stake {
  @PrimaryGeneratedColumn("uuid") id: string;
  @Column("numeric", { precision: 36, scale: 18 }) amount: string; // crypto amount – never float
  @ManyToOne(() => User, (u) => u.stakes) user: User;
  @CreateDateColumn() createdAt: Date;
}
```
- Repository: `@InjectRepository(Stake) repo: Repository<Stake>`.
- Transactions: `this.dataSource.transaction(async (manager) => {...})`.
- Keep `synchronize: false` in production. Use migrations.

**Scheduling:** `@Cron("0 0 * * *")`. With many servers, it runs on every server → use a lock / leader election. See [09 → Leader election](09-kafka-redis-messaging.md#leader-election-with-redis).

---

## More NestJS questions

### Q. What does `@Injectable()` do?

> 🎤 **Interview answer:** "`@Injectable()` tells Nest: 'this class can be created and injected by you'. Nest then creates it (one shared copy by default), and passes it to any class that asks for it in its constructor."

### Q. How does Nest know what to inject?

> 🎤 **Interview answer:** "TypeScript can save the **types of constructor parameters** as hidden information when the code is compiled (the `emitDecoratorMetadata` setting). Nest reads this information and sees, for example, 'this constructor needs a `UserRepository`'. Then it finds and passes that provider. Interfaces don't exist at runtime, so for them we use a custom token with `@Inject('TOKEN')`."

### Q. Custom decorators

> 🎤 **Interview answer:** "Custom decorators remove repeated code. A common one is `@CurrentUser()`, which gets the logged-in user from the request, so the controller doesn't need `@Req()`. I also made `@Roles()` and `@Public()` decorators that guards read."

```ts
export const CurrentUser = createParamDecorator(
  (_, ctx: ExecutionContext) => ctx.switchToHttp().getRequest().user
);

// use
@Get("me")
findMine(@CurrentUser() user) { /* ... */ }
```

### Q. Lifecycle hooks

> 🎤 **Interview answer:** "Nest calls special methods at certain moments:
> - At startup: `OnModuleInit`, `OnApplicationBootstrap` – good for connecting to services.
> - At shutdown: `OnModuleDestroy`, `OnApplicationShutdown` – good for closing connections.
>
> The shutdown ones work only if I call `app.enableShutdownHooks()`. I used them to close Redis, and in the price service, to **release the leader lock** on shutdown, so another server could take over immediately instead of waiting for the lock to expire."

### Q. Global vs scoped guards/pipes

> 🎤 **Interview answer:** "Three ways to apply a guard:
> - `app.useGlobalGuards()` in `main.ts` – applies to all routes, but can't use dependency injection.
> - `APP_GUARD` provider in a module – applies to all routes **and** can inject services. I prefer this.
> - `@UseGuards()` on a controller or method – only for those routes.
>
> Same idea for pipes, interceptors and filters."

### Q. Fastify vs Express adapter?

> 🎤 **Interview answer:** "Nest uses Express by default, which has the most middleware and community support. Fastify is faster. Switching is mostly one line. But for most of our services Express was fine, because the slow part was the database, not the framework."

```ts
NestFactory.create(AppModule, new FastifyAdapter());
```

### Q. How do you add Swagger?

> 🎤 **Interview answer:** "I install `@nestjs/swagger`, set it up in `main.ts` with a title and JWT auth, and serve the docs at `/docs`. Then I add `@ApiProperty` to DTO fields and `@ApiTags` to controllers. Since the docs come from the same DTOs used for validation, they're always in sync with the code. The frontend team used them as the contract."

```ts
const config = new DocumentBuilder().setTitle("Referral API").addBearerAuth().build();
SwaggerModule.setup("docs", app, SwaggerModule.createDocument(app, config));
```

---

[Home](README.md) | [← 07 APIs & Advanced Backend](07-apis-backend.md) | Next: [09 Kafka, Redis & Messaging →](09-kafka-redis-messaging.md)
