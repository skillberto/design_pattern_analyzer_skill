# Language Hints

Idiomatic syntax for before/after sketches. Read only the section(s) for the language(s) detected in Scan Workflow step 1. Every sketch must name its file(s) (Core Rule 7) and must never embed large templated blobs as strings (Core Rule 8).

---

## Cross-language rules

- **DI default**: constructor injection of an interface/abstraction. Avoid service locators unless the framework already uses one.
- **Registry default**: a map from key → handler/factory, populated at composition root, replacing `if/switch` dispatch.
- **Composition root**: object graph wiring lives in exactly one place (`main`, `bootstrap`, `Program.cs`, DI container config). Name that file in sketches.
- **Template externalization**: template lives in `templates/` next to the module that renders it; placeholders use the language's standard templating tool, not hand-rolled `replace()` chains when a real engine is available.
- **Singleton**: prefer "one instance registered in the DI container" over a self-managing static singleton. Flag hand-rolled singletons that hold mutable state as a likely anti-pattern.

---

## TypeScript

- Interfaces for contracts: `interface PaymentGateway { charge(amount: number): Promise<void>; }`
- DI via constructor parameter properties:
  ```ts
  // src/orders/OrderService.ts
  export class OrderService {
    constructor(private readonly gateway: PaymentGateway) {}
  }
  ```
- Registry: `const handlers: Record<EventType, Handler> = { ... }` or `Map<string, Handler>`; type the key as a union for exhaustiveness.
- Strategy: interface + one class per strategy file (`src/pricing/strategies/DiscountStrategy.ts`), or a typed function `type PricingFn = (o: Order) => number` for stateless cases.
- Builder: fluent class returning `this`; consider an options object `{ ... }: Partial<Options>` first — often enough in TS.
- Discriminated unions + `switch (x.kind)` with a `never` check is idiomatic, not a Visitor smell. Only flag `instanceof` chains.
- Frameworks: NestJS (`@Injectable()`, providers), Angular DI, InversifyJS. Use the project's container if present.
- Templates: `fs.readFile('templates/x.html')` + Handlebars/EJS/Nunjucks/Mustache. Tagged template literals are fine for short snippets only.

## JavaScript

- Same as TypeScript minus types. Express contracts as JSDoc `@typedef` / `@interface` when useful.
- ES modules (`export class` / `export default`) unless the project is CommonJS (`module.exports`) — match the project.
- DI via constructor args or factory function `createOrderService({ gateway, logger })`.
- Registry: plain object or `Map`. Strategy: object of functions is idiomatic.
- Module-level state is a de facto singleton (module cache) — treat as such when evaluating shared mutable state.

## PHP

- **Classes over loose functions** (Core Rule 6): wrap related functions in a class with constructor-injected dependencies, one class per file, PSR-4 (`src/Orders/OrderService.php` ↔ `namespace App\Orders;`).
- PHP 8+ constructor property promotion and `readonly`:
  ```php
  // src/Orders/OrderService.php
  final class OrderService
  {
      public function __construct(
          private readonly PaymentGatewayInterface $gateway,
      ) {}
  }
  ```
- Interfaces named `*Interface` (PSR convention); enums (8.1+) for strategy keys.
- Registry: `array<string, HandlerInterface>` injected via constructor, or framework tagged services (Symfony `!tagged_iterator`, Laravel `$this->app->tag()`).
- DI containers: Symfony DI (autowiring), Laravel service container, PHP-DI. Static facades (`Cache::get`) are service locators — note this, don't automatically flag as anti-pattern in Laravel.
- Repository: Doctrine repositories / Eloquent; flag Eloquent calls scattered in controllers.
- Templates: Twig (`templates/x.html.twig`), Blade (`resources/views/x.blade.php`), or plain `.tpl`/`.php` template rendered via `ob_start(); include $path; return ob_get_clean();` with extracted variables. **Never heredoc/nowdoc for large config, HTML or generated code.**
- Generated PHP code: `templates/user.php.tpl` with placeholders, filled via `strtr($tpl, ['{{class}}' => $name])` or Twig.

## Python

- **Classes over loose functions** (Core Rule 6): group functions sharing data into a class with `__init__` storing dependencies. Use `@dataclass` for value objects.
- Contracts: `typing.Protocol` (structural) or `abc.ABC` + `@abstractmethod` (nominal). Prefer `Protocol` for DI seams.
  ```python
  # app/orders/order_service.py
  class OrderService:
      def __init__(self, gateway: PaymentGateway) -> None:
          self._gateway = gateway
  ```
- Registry: `dict[str, type[Handler]]` or a decorator-based registry (`@register("csv")`) in `app/<domain>/registry.py`.
- Strategy: class implementing a Protocol; a plain callable is acceptable only when stateless — but still house it in a class-based module per Rule 6 when there are several.
- Builder: often replaced by `@dataclass` with defaults or keyword args; only suggest Builder when construction has steps/validation.
- Singleton: module-level instance is idiomatic; avoid metaclass singletons. Flag module-level mutable globals used across modules.
- DI: manual constructor injection; FastAPI `Depends`, `dependency-injector`, `punq` if present.
- Templates: Jinja2 (`templates/nginx.conf.j2` + `Environment(loader=FileSystemLoader("templates"))`) or `string.Template` for simple substitution. **Never large f-strings / triple-quoted strings for config, HTML, SQL or code.**
- SQL: separate `.sql` files or a query builder/ORM (SQLAlchemy), not f-strings (also an injection risk).

## C

- No classes: encapsulation = opaque struct + functions with a module prefix in a header/source pair (`order_service.h` / `order_service.c`).
- Interfaces/Strategy: struct of function pointers (vtable):
  ```c
  /* payment_gateway.h */
  typedef struct {
      int (*charge)(void *ctx, long amount);
      void *ctx;
  } payment_gateway;
  ```
- DI: pass the vtable struct into `order_service_create(const payment_gateway *gw)`.
- Registry: static array of `{ const char *key; handler_fn fn; }` with a lookup function.
- Builder: config struct with designated initializers (`(struct opts){ .timeout = 5 }`).
- Singleton: `static` file-scope variable + accessor; flag non-static globals declared `extern` across files.
- Templates: load from file (`fopen` + read) and substitute; for build-time generation, keep templates in `templates/` and generate via a script, not giant `sprintf` format strings.

## C#

- Interfaces prefixed `I` (`IPaymentGateway`). Constructor injection with `Microsoft.Extensions.DependencyInjection`; register in `Program.cs` (composition root).
  ```csharp
  // Orders/OrderService.cs
  public sealed class OrderService(IPaymentGateway gateway)   // C# 12 primary constructor
  {
  }
  // Program.cs
  builder.Services.AddScoped<IPaymentGateway, StripeGateway>();
  ```
- Registry/Strategy: inject `IEnumerable<IHandler>` and select by a `Key` property, or keyed services (.NET 8 `AddKeyedScoped`).
- Mediator: MediatR is common — detect `IRequest<T>` / `IRequestHandler<,>`.
- Repository/Unit of Work: EF Core `DbContext` already is UoW + repository; flag redundant generic repositories wrapping it as a possible anti-pattern `[weak match]`.
- Builder: fluent builder or `record` with `with` expressions / `init` properties.
- Singleton: `AddSingleton` in DI over `static Instance`.
- Templates: Razor, Scriban, or embedded resource / file under `Templates/`; not giant interpolated `$"""..."""` raw strings.

## Java

- Interfaces without `I` prefix (`PaymentGateway`), implementations named by detail (`StripePaymentGateway`). One public class per file, package path = directory.
- DI: constructor injection, `final` fields; Spring (`@Service`, `@Component`, no field `@Autowired`), Jakarta CDI, Guice, Dagger.
  ```java
  // src/main/java/com/acme/orders/OrderService.java
  @Service
  public class OrderService {
      private final PaymentGateway gateway;
      public OrderService(PaymentGateway gateway) { this.gateway = gateway; }
  }
  ```
- Registry: `Map<String, Handler>` injected by Spring (bean name → bean) or an `EnumMap`.
- Strategy: interface + implementations, or enum with abstract method for small closed sets.
- Builder: static nested `Builder` class or Lombok `@Builder`; records for immutable data.
- Singleton: enum singleton or container-managed bean; flag double-checked locking hand-rolls.
- Sealed interfaces + pattern-matching `switch` (Java 17/21) are idiomatic, not Visitor smells.
- Templates: `src/main/resources/templates/` with Thymeleaf/FreeMarker/Mustache; text blocks (`"""`) only for short snippets.

## Go

- Small interfaces defined by the **consumer**, not the implementer ("accept interfaces, return structs").
  ```go
  // orders/service.go
  type PaymentGateway interface { Charge(ctx context.Context, amount int64) error }

  type Service struct{ gateway PaymentGateway }

  func NewService(g PaymentGateway) *Service { return &Service{gateway: g} }
  ```
- DI: constructor functions `NewX(deps...)`, wired in `cmd/<app>/main.go`; Wire or Fx only if already used.
- Builder: functional options (`func WithTimeout(d time.Duration) Option`) is idiomatic over Builder structs.
- Registry: `map[string]Handler` with a `Register` func; `init()`-based registration is a [weak match] smell (hidden global state).
- Singleton: `sync.Once` + package var; flag mutable package-level vars used across packages.
- No inheritance: Template Method → interface + composed struct; Decorator → wrap the interface (e.g. `http.Handler` middleware).
- Type switches on interfaces are fine for small closed sets; flag large ones as Strategy/Visitor candidates.
- Templates: `text/template` / `html/template` with files under `templates/`, embedded via `//go:embed templates/*`. Not `fmt.Sprintf` over big format strings.

## Kotlin

- Interfaces without `I` prefix (`PaymentGateway`); file name matches the main class (`orders/OrderService.kt`). Several small related declarations (a sealed hierarchy, a value class and its helpers) may share one file — still one responsibility per file (Core Rule 7).
- DI via primary-constructor `private val` parameters:
  ```kotlin
  // src/main/kotlin/com/acme/orders/OrderService.kt
  class OrderService(
      private val gateway: PaymentGateway,
  ) {
      suspend fun checkout(order: Order) = gateway.charge(order.total)
  }
  ```
- Containers: Spring (`@Service`, constructor injection — no `lateinit var` + `@Autowired` fields), Koin (`module { single<PaymentGateway> { StripeGateway() } }`), Hilt/Dagger on Android (`@Inject constructor`). Wire in one composition root (Koin `appModule.kt`, `Application` class, Spring config). Flag `KoinComponent` + `by inject()` inside business classes as a service-locator `[weak match]`.
- **Top-level functions**: stateless top-level and extension functions are idiomatic — Core Rule 6 does not apply. Flag them only when several share mutable top-level state or take the same dependencies as parameters; then suggest a class with constructor-injected deps.
- Singleton: `object` declaration is the idiomatic singleton. Fine for stateless utilities; flag an `object` holding mutable state or hardwired dependencies (untestable) and suggest a DI-managed instance instead.
- Factory: `companion object { fun create(...) }` or a top-level factory function; `operator fun invoke` on a companion is acceptable.
- Strategy: interface + implementations, a `fun interface` (SAM) for single-method strategies, or a function type `(Order) -> Money` when stateless.
- Registry: `Map<Key, Handler>` built in the composition root; prefer an `enum class` or sealed type as key over raw strings. Spring can inject `Map<String, Handler>`; Koin via `getAll<Handler>()`.
- `when` over a `sealed interface`/`sealed class` is exhaustive and idiomatic — not a Visitor/Strategy smell. Only flag `is` chains over open types, or `when` blocks that grow every time a new variant is added across many files.
- Builder: usually unnecessary — named and default arguments plus `data class` `copy()` replace it. Suggest a type-safe builder DSL (`buildX { ... }` with `@DslMarker`) only for genuinely nested construction.
- Decorator/Proxy: class delegation `class LoggingGateway(private val inner: PaymentGateway) : PaymentGateway by inner` — override only the decorated methods.
- Observer: `Flow`/`StateFlow`/`SharedFlow` over hand-rolled listener lists in coroutine code.
- Repository: interface in the domain package, implementation in data/infrastructure (Room DAO, Exposed, Spring Data `CrudRepository`). Flag DB calls inside ViewModels/controllers.
- Templates: resource files under `src/main/resources/templates/` loaded via `javaClass.getResource(...)!!.readText()` and rendered with Mustache, FreeMarker, Pebble or Thymeleaf; kotlinx.html DSL is acceptable for HTML built in code. **Never large raw strings (`"""..."""` with `${}` interpolation) or `buildString` for config, HTML, SQL or generated code.**
