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

> 🎤 **Interview answer:** "NestJS is a progressive Node.js framework written in TypeScript. It runs on top of Express by default, or Fastify, and gives an opinionated architecture inspired by Angular: modules, controllers, providers, dependency injection and decorators. Compared to plain Express, the big advantage is consistency: every developer knows where code goes, which helps a lot in larger teams and when mentoring juniors. Built-in DI makes unit testing and mocking easy, and it comes with validation pipes, guards, interceptors, Swagger integration and microservice transports like Redis, Kafka and gRPC out of the box. The trade-off is more boilerplate and a learning curve, so for a tiny service Express can be enough. I used NestJS to build the Referral and Staking microservices at KindSwap."

A Node.js framework written in TypeScript. It runs on top of Express (default) or Fastify and gives a fixed structure inspired by Angular: **modules, controllers, services, dependency injection, decorators**.

**Why (vs plain Express):**
- Clear structure – every developer puts code in the same place (helps big teams and mentoring juniors).
- Built-in DI → easy testing and mocking.
- Built-in validation, guards, interceptors, Swagger, microservice transports (Redis, Kafka, RabbitMQ, gRPC).
- TypeScript first.

**Downside:** more boilerplate and learning curve; for a very small service Express may be enough.

---

### 2. Modules, controllers, providers

> 🎤 **Interview answer:** "These are the three building blocks. A module, decorated with `@Module`, groups everything for one feature: its controllers, its providers, what it imports from other modules and what it exports for others to use. AppModule is the root. A controller handles the HTTP layer: it maps routes with decorators like `@Get` and `@Post`, extracts params and body, and delegates to a service. I keep controllers thin, with no business logic. A provider is anything Nest can inject: services with the business logic, repositories, helpers, factories. There are also global modules with `@Global`, for things like config and Redis, and dynamic modules configured at import time, like `TypeOrmModule.forRootAsync`. In the Referral service, for example, I had a ReferralModule with its controller, its service and the TypeORM repository for the Referral entity."

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
  findOne(id: string) { return this.repo.findOneByOrFail({ id }); }
}
```

- **Module** – groups related code (a feature). `AppModule` is the root.
- **Controller** – receives requests, calls service, returns response.
- **Provider** – anything injectable: services, repositories, helpers, factories.
- **Global module** – `@Global()` so you don't import it everywhere (e.g. ConfigModule, Redis module).
- **Dynamic module** – a module configured at import time (`TypeOrmModule.forRoot({...})`, `forRootAsync`).

---

### 3. Dependency injection

**Q. What is DI?**

> 🎤 **Interview answer:** "Dependency injection means a class doesn't create its own dependencies with `new`. It declares them, usually in the constructor, and the framework's IoC container creates and supplies them. In NestJS, I mark a class with `@Injectable`, register it as a provider in a module, and Nest resolves the whole dependency graph, creating singletons by default. The benefits are loose coupling, the ability to swap implementations, like a different payment gateway behind the same token, and very easy testing, because in `Test.createTestingModule` I can replace a real repository with a mock using `useValue`."

Instead of a class creating its own dependencies (`new UserRepository()`), they are **given to it** (usually through the constructor). Nest's IoC container creates and passes them.

**Benefits:** loose coupling, easy to swap implementations, easy to mock in tests.

```ts
// test
const module = await Test.createTestingModule({
  providers: [
    ReferralService,
    { provide: getRepositoryToken(Referral), useValue: { findOneByOrFail: jest.fn() } },
  ],
}).compile();
```

**Provider types:**

> 🎤 **Interview answer:** "Besides normal class providers, there are custom providers. `useClass` maps a token to a class, which is useful for choosing an implementation, like a PaymentGateway token mapped to RazorpayGateway. `useValue` provides a fixed value, a config object or a mock in tests. `useFactory` builds the provider with a function, which can be async and can inject other providers, like creating a Redis client from ConfigService. `useExisting` creates an alias. For string or symbol tokens, I inject with `@Inject('TOKEN')`."

```ts
{ provide: "PAYMENT_GATEWAY", useClass: RazorpayGateway }   // class
{ provide: "CONFIG", useValue: { retries: 3 } }             // value
{ provide: "REDIS", useFactory: (cfg: ConfigService) => new Redis(cfg.get("REDIS_URL")), inject: [ConfigService] }
```
Inject with `@Inject("PAYMENT_GATEWAY") private gateway: PaymentGateway`.

**Scopes:** `DEFAULT` (singleton – one instance for the app, best performance), `REQUEST` (new per request), `TRANSIENT` (new for each consumer).

> 🎤 **Interview answer:** "The default scope is singleton: one instance shared by the whole app, which is the most efficient and what I use almost always. REQUEST scope creates a new instance per incoming request, useful for per-request context like a tenant id, but it costs performance and the scope bubbles up to everything that injects it. TRANSIENT gives each consumer its own instance. So I avoid request scope unless it's really needed, and pass context explicitly or use AsyncLocalStorage instead."

**Circular dependency:** A needs B and B needs A → use `forwardRef(() => B)`, but better to refactor (move shared logic to a third service).

> 🎤 **Interview answer:** "A circular dependency is when service A injects B and B injects A, so Nest can't decide which to create first. The quick fix is `forwardRef(() => B)` on both sides. But I treat it as a design smell: the better fix is to extract the shared logic into a third service that both depend on, or use events to decouple them."

---

### 4. Request lifecycle order

> 🎤 **Interview answer:** "The order is: incoming middleware first, then guards, which decide whether the request is allowed at all, like JWT and role checks. Then interceptors run their 'before' logic, like logging or timing. Then pipes validate and transform the params and body. Then the controller method runs and calls the service. On the way back, interceptors run their 'after' logic, like wrapping the response in a standard format. If an exception is thrown anywhere along the way, exception filters catch it and build the error response. Within each type, global ones run first, then controller-level, then route-level."

Very commonly asked:

```
Incoming request
 → Middleware
 → Guards                 (can this user access? – return true/false)
 → Interceptors (before)  (logging, timing, caching)
 → Pipes                  (validate / transform params and body)
 → Controller method → Service
 → Interceptors (after)   (change response, e.g. wrap in { data })
 → Exception filters      (only if an error was thrown anywhere)
 → Response
```

---

### 5. Middleware vs guards vs interceptors vs pipes vs filters

> 🎤 **Interview answer:** "Middleware is the same as Express middleware. It runs first and doesn't know which handler will execute, so it's for generic things like request ids and logging. Guards answer one question: can this request proceed? They return true or false, and they have access to the ExecutionContext, so they can read metadata like required roles. That's where auth and RBAC go. Interceptors wrap the handler before and after using RxJS, so they can transform responses, measure time, add caching or timeouts. Pipes validate and transform input, like ValidationPipe with DTOs or ParseUUIDPipe. Exception filters convert thrown errors into consistent HTTP responses. Picking the right one keeps responsibilities clean."

| | Purpose | Knows which handler will run? | Example |
|---|---|---|---|
| Middleware | Same as Express middleware | No | Logging, request id, cors |
| Guard | Allow or deny the request | Yes (`ExecutionContext`) | JWT auth, roles |
| Interceptor | Run code before and after the handler, change result | Yes | Response wrapper, timing, cache, timeout |
| Pipe | Validate and transform input | Yes | `ValidationPipe`, `ParseIntPipe` |
| Exception filter | Turn errors into responses | Yes | Custom error format |

**Interceptor example – response time and wrapper:**

> 🎤 **Interview answer:** "An interceptor implements `intercept(context, next)`. Code before `next.handle()` runs before the controller, and `next.handle()` returns an RxJS Observable of the response. So with operators like `map` I can wrap every response in a standard `{ success: true, data }` format, and with `tap` I can log how long it took. Other uses are `timeout` to fail slow handlers, `catchError` to map errors, or caching. I applied a global transform interceptor so all our microservices returned the same response shape."

```ts
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler) {
    const start = Date.now();
    return next.handle().pipe(
      map((data) => ({ success: true, data })),
      tap(() => console.log(`took ${Date.now() - start}ms`)),
    );
  }
}
```
Interceptors use **RxJS Observables** (`next.handle()` returns an Observable).

---

### 6. DTOs and validation

> 🎤 **Interview answer:** "A DTO, data transfer object, is a class that defines exactly what an endpoint accepts. I decorate its properties with class-validator decorators like `@IsUUID`, `@IsInt` and `@Min`, and enable a global ValidationPipe with `whitelist: true`, which strips any properties not in the DTO and protects against mass assignment, `forbidNonWhitelisted` to reject them instead, and `transform: true` to convert the payload into the DTO class with proper types. DTOs have to be classes, not interfaces, because interfaces disappear at runtime and the validators need the class metadata. For update DTOs I use `PartialType(CreateDto)`, and the same DTOs drive the Swagger documentation."

**DTO (Data Transfer Object)** – a class that defines what data the API accepts.
```ts
export class CreateStakeDto {
  @IsUUID() poolId: string;
  @IsNumberString() amount: string;
  @IsOptional() @IsInt() @Min(1) lockDays?: number;
}
```
```ts
// main.ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,             // remove fields not in DTO (stops mass assignment)
  forbidNonWhitelisted: true,  // or throw error for extra fields
  transform: true,             // convert to DTO class / types
}));
```
Why a class and not an interface? Interfaces disappear at runtime; classes stay, so `class-validator` decorators can run.

`PartialType(CreateDto)`, `PickType`, `OmitType` from `@nestjs/mapped-types` / `@nestjs/swagger` – create update DTOs quickly.

---

### 7. Guards and RBAC with Auth0

> 🎤 **Interview answer:** "At KindSwap, authentication and RBAC worked in two guards. First, a JWT guard using passport-jwt with jwks-rsa: it fetches Auth0's public key from the tenant's JWKS endpoint, verifies the RS256 signature, expiry, audience and issuer, and attaches the decoded user, including the permissions Auth0 puts in the token, to the request. Second, a custom RolesGuard: I created a `@Roles()` decorator that stores the required permissions as metadata with SetMetadata, and the guard uses the Reflector to read that metadata from the handler or class and checks that the user has one of them. Routes like approving a payout batch were protected with `@Roles('payout:approve')`. If the token is invalid we return 401, and if the permission is missing, the guard returns false and Nest returns 403."

```ts
// roles.decorator.ts
export const Roles = (...roles: string[]) => SetMetadata("roles", roles);

// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  canActivate(ctx: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<string[]>("roles", [ctx.getHandler(), ctx.getClass()]);
    if (!required) return true;
    const user = ctx.switchToHttp().getRequest().user;    // set by JWT guard
    const userPerms: string[] = user?.permissions ?? [];  // Auth0 puts permissions in the token
    return required.some((r) => userPerms.includes(r));
  }
}

// controller
@UseGuards(AuthGuard("jwt"), RolesGuard)
@Roles("payout:approve")
@Post("payouts/batch")
approveBatch() { ... }
```
**JWT strategy for Auth0:** `passport-jwt` with `jwks-rsa` – fetches Auth0's public key from `https://<tenant>/.well-known/jwks.json`, verifies RS256 signature, `audience` and `issuer`.

Guard returns `false` → Nest sends **403**. Throw `UnauthorizedException` → **401**.

---

### 8. Exception filters

> 🎤 **Interview answer:** "NestJS has built-in HTTP exceptions like NotFoundException, BadRequestException, UnauthorizedException and ConflictException, which automatically become proper HTTP responses. For a consistent error format across services, I write a global exception filter with `@Catch()` that catches everything. If it's an HttpException, it uses its status and message. Otherwise it returns a 500 with a generic message, so internals aren't leaked. It also logs the full error with context, and every error response has the same shape. I can also create domain-specific exceptions and filters, for example mapping a database unique-constraint error to 409 Conflict."

Nest has built-in HTTP exceptions: `BadRequestException`, `NotFoundException`, `UnauthorizedException`, `ForbiddenException`, `ConflictException`...

```ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse();
    const status = exception instanceof HttpException ? exception.getStatus() : 500;
    const message = exception instanceof HttpException ? exception.message : "Internal server error";
    // log full exception here
    res.status(status).json({ success: false, statusCode: status, message });
  }
}
app.useGlobalFilters(new AllExceptionsFilter());
```

---

### 9. Microservices in NestJS

> 🎤 **Interview answer:** "NestJS has a microservices package with pluggable transports: TCP, Redis, Kafka, RabbitMQ, NATS and gRPC, so the handler code looks the same regardless of the broker. There are two communication styles. Request-response uses `@MessagePattern` with `client.send()`, which waits for a reply. Event-based uses `@EventPattern` with `client.emit()`, which is fire and forget. At KindSwap, the Staking and Referral services communicated through Redis pub/sub with events, for example when a stake completed, the Referral service received the event and credited the referrer.
>
> I'm also honest about the limitation: Redis pub/sub doesn't persist messages, so if a subscriber is down it misses events. For flows that must not lose events, I'd use Kafka, RabbitMQ or Redis Streams, or an outbox table with a reconciliation job. Nest also supports hybrid apps that serve HTTP and listen to a transport at the same time."

Nest supports transports: TCP, Redis, Kafka, RabbitMQ, NATS, gRPC.

**Two styles:**
- **Request–response** – `@MessagePattern("get_user")` + `client.send()` → waits for reply.
- **Event-based** – `@EventPattern("referral_rewarded")` + `client.emit()` → fire and forget.

```ts
// Staking service publishes
this.client.emit("stake.completed", { userId, amount });

// Referral service listens
@EventPattern("stake.completed")
handleStake(@Payload() data: StakeCompletedEvent) {
  return this.referralService.creditReferrer(data);
}
```
At KindSwap, Referral and Staking services talked through **Redis pub/sub**. Note: Redis pub/sub does **not store messages** – if a subscriber is down it misses them. For events that must not be lost, use Kafka/RabbitMQ/Redis Streams or add a DB-backed retry/outbox. (Be ready for this follow-up – see [09 → Redis pub/sub](09-kafka-redis-messaging.md#redis-pubsub).)

**Hybrid app:** one Nest app that serves HTTP and also listens to a microservice transport (`app.connectMicroservice()`).

---

### 10. Config, TypeORM, testing

> 🎤 **Interview answer:** "For configuration, I use `ConfigModule.forRoot` with `isGlobal` and a Joi validation schema, so the app fails at startup if a required env variable is missing. For the database at KindSwap, I used TypeORM: entities defined with decorators, repositories injected with `@InjectRepository`, relations like ManyToOne, and transactions through `dataSource.transaction`. Two important practices: `synchronize: false` in production with proper migrations, because synchronize can drop columns, and storing crypto or money amounts as `numeric` with high precision, never as JavaScript floats.
>
> For testing, I use `Test.createTestingModule` with mocked providers for unit tests, and supertest against `app.getHttpServer()` for e2e tests. For scheduled jobs, `@nestjs/schedule` with `@Cron`, keeping in mind that with multiple instances every instance runs the cron, so I need a lock or leader election, which is exactly the problem I solved with Redis."

**Config:**
```ts
ConfigModule.forRoot({ isGlobal: true, validationSchema: Joi.object({ DATABASE_URL: Joi.string().required() }) })
```

**TypeORM:**
```ts
@Entity()
export class Stake {
  @PrimaryGeneratedColumn("uuid") id: string;
  @Column("numeric", { precision: 36, scale: 18 }) amount: string;   // crypto amounts – don't use float
  @ManyToOne(() => User, (u) => u.stakes) user: User;
  @CreateDateColumn() createdAt: Date;
}
```
- Repository pattern: `@InjectRepository(Stake) repo: Repository<Stake>`.
- Transactions: `this.dataSource.transaction(async (manager) => {...})`.
- Use **migrations**, keep `synchronize: false` in production (it can drop columns).
- Money/crypto amounts: store as `numeric`/`decimal` or integer smallest unit, never JS float (`0.1 + 0.2` problem).

**Testing:** `Test.createTestingModule` for unit tests with mocked providers; `supertest` + `app.getHttpServer()` for e2e.

**Scheduling:** `@nestjs/schedule` – `@Cron("0 0 * * *")`. With multiple instances, cron runs on every instance → use a lock / leader election (exactly the problem I solved – see [09 → Leader election](09-kafka-redis-messaging.md#leader-election-with-redis)).

---

## More NestJS questions

### Q. What does `@Injectable()` do?

> 🎤 **Interview answer:** "`@Injectable()` marks a class as a provider that the Nest IoC container can manage. Nest can create it, cache it as a singleton by default, and inject it wherever it's needed. It also lets the class itself receive injected dependencies through its constructor, because it triggers the emission of the constructor's type metadata."

Marks a class as a provider that Nest's DI container can manage and inject.

### Q. How does Nest know what to inject?

> 🎤 **Interview answer:** "With `emitDecoratorMetadata` enabled, the TypeScript compiler emits the constructor parameter types as metadata on decorated classes. Nest reads that metadata using `reflect-metadata`, sees, for example, that the constructor needs a UserRepository, and looks up the provider registered under that class token in the module's scope. For interfaces or primitives, there's no runtime type, which is why we use custom tokens with `@Inject('TOKEN')`."

TypeScript emits constructor parameter types as metadata (`emitDecoratorMetadata`). Nest reads it with `reflect-metadata` and finds the matching provider.

### Q. Custom decorators

> 🎤 **Interview answer:** "Custom decorators remove repetition. A common one is a param decorator, `@CurrentUser()`, created with `createParamDecorator`, which pulls `request.user` out of the ExecutionContext, so controllers get the user directly instead of using `@Req()`. I also create metadata decorators like `@Roles()` or `@Public()` with SetMetadata, read by guards. And `applyDecorators` lets me combine several decorators, like guards plus Swagger auth, into one."

```ts
export const CurrentUser = createParamDecorator((_, ctx: ExecutionContext) => ctx.switchToHttp().getRequest().user);
// use: findMine(@CurrentUser() user)
```

### Q. Lifecycle hooks

> 🎤 **Interview answer:** "Nest calls lifecycle hooks on providers and modules at specific moments: `OnModuleInit` and `OnApplicationBootstrap` during startup, which are good for connecting to services or starting consumers, and `OnModuleDestroy`, `BeforeApplicationShutdown` and `OnApplicationShutdown` during shutdown. The shutdown hooks only run if `app.enableShutdownHooks()` is called. I used them for graceful shutdown: closing Redis connections and, in the price service, releasing the leader lock so another instance could take over immediately instead of waiting for the TTL."

`OnModuleInit`, `OnApplicationBootstrap`, `OnModuleDestroy`, `BeforeApplicationShutdown`, `OnApplicationShutdown`. Enable `app.enableShutdownHooks()` for graceful shutdown (close Redis, release leader lock).

### Q. Global vs scoped guards/pipes

> 🎤 **Interview answer:** "There are three ways. `app.useGlobalGuards()` in main.ts applies it globally, but it's created outside the DI container, so it can't inject dependencies. Registering it as the `APP_GUARD` provider in a module also applies it globally, but with full DI, so it can inject services like the Reflector or ConfigService. That's what I prefer. Or `@UseGuards()` on a controller or method, to scope it locally. The same applies to pipes, interceptors and filters, with APP_PIPE, APP_INTERCEPTOR and APP_FILTER."

`app.useGlobalGuards()` (no DI) vs providing `APP_GUARD` in a module (with DI) vs `@UseGuards()` on controller/method.

### Q. Fastify vs Express adapter?

> 🎤 **Interview answer:** "Nest is platform-agnostic. Express is the default and has the biggest middleware ecosystem and community. Fastify has lower overhead and better raw throughput, plus schema-based serialisation. Switching is mostly one line with FastifyAdapter, but Express-specific middleware or `req` and `res` usage would need changes. For most of our services Express was fine, because the bottleneck was the database, not the framework."

Fastify is faster with lower overhead; Express has a bigger middleware ecosystem. Switch with `NestFactory.create(AppModule, new FastifyAdapter())`.

### Q. How do you add Swagger?

> 🎤 **Interview answer:** "I install `@nestjs/swagger`, build a document config in main.ts with DocumentBuilder, setting the title, version and `addBearerAuth` for JWT, and serve it with `SwaggerModule.setup('docs', app, document)`. Then I annotate DTOs with `@ApiProperty` and controllers with `@ApiTags`, `@ApiBearerAuth` and response decorators. The CLI plugin can even generate many annotations automatically. Since the docs come from the same DTOs used for validation, they stay in sync with the code, and the frontend team used them as the contract."

```ts
const config = new DocumentBuilder().setTitle("Referral API").addBearerAuth().build();
SwaggerModule.setup("docs", app, SwaggerModule.createDocument(app, config));
```

---

[Home](README.md) | [← 07 APIs & Advanced Backend](07-apis-backend.md) | Next: [09 Kafka, Redis & Messaging →](09-kafka-redis-messaging.md)
