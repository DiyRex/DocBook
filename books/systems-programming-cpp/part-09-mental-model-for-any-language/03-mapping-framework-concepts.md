# Chapter 93 — Mapping Framework Concepts Across Languages

Every web framework reinvents the same six concepts under different names. Once you can map them, learning a new framework takes hours, not days.

You already know that controllers translate protocol, services orchestrate logic, and repositories hide storage. You know that dependency injection wires services together, that middleware runs before and after request handling, and that templates render views. These patterns are universal across Spring, Laravel, Rails, Django, ASP.NET Core, Express, NestJS, Phoenix, and a dozen others.

The names change. The concepts do not.

This chapter teaches you to recognize those concepts regardless of their names, so that when you pick up a new framework, you are not learning six new things—you are recognizing six old things with unfamiliar labels. The skill is translation, not invention.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Name the universal six concepts and define each one in framework-agnostic terms.
2. Map those concepts across at least six different frameworks and languages.
3. Recognize when two frameworks use different terminology for the same idea.
4. Understand why frameworks choose different philosophies (conventions vs. annotations, immutability vs. mutability) and what that means for how you write code.
5. Predict how a new framework will solve a routing or middleware problem based on its philosophy.
6. Avoid forcing one framework's idioms onto another and instead adapt to each framework's design intent.

---

## The Universal Six

Every web framework, regardless of language or philosophy, solves six problems:

1. **Routing**: matching an HTTP request path to a handler
2. **Controller**: translating protocol to a method call
3. **Middleware**: running logic before/after request handling
4. **Dependency Injection**: wiring services together
5. **Model/Repository**: hiding data storage
6. **View/Template**: rendering responses

These are not names invented by any one framework. They are the six unavoidable responsibilities of a web application. Different frameworks emphasize them differently and name them differently, but all six exist, explicitly or implicitly, in every framework.

A framework's philosophy—whether it enforces conventions or requires annotations, whether it favors functional or object-oriented style, whether it isolates layers or blurs them—shapes how these six are expressed. But the problems they solve are the same.

---

## The Cross-Language Mapping Table

Here is how nine major frameworks name the universal six:

| Concept | Spring Java | Laravel PHP | Rails Ruby | Django Python | ASP.NET Core C# | Express Node.js | NestJS TypeScript | Phoenix Elixir | Actix Rust |
|---------|-------------|-------------|-----------|---------------|-----------------|-----------------|------------------|---------------|-----------|
| **Routing** | `@RequestMapping`, `@GetMapping` | `Route::get()`, `Route::post()` | `get '/path'` in `routes.rb` | `path('path', views.handler)` in `urls.py` | `[HttpGet("path")]` attribute | `app.get('path', handler)` | `@Get('path')` decorator | `get("/path", handler)` in router | `#[get("/path")]` macro |
| **Controller** | `@RestController`, `@PostMapping` on class method | `class OrderController` method | `class OrdersController < ApplicationController` | `class OrderView(View)` | `[ApiController]` class, method | `app.get('path', (req, res) => {})` handler function | `@Controller` class, method | `def handle(conn, _params)` in controller module | Handler function in a route |
| **Middleware** | `HandlerInterceptor`, `@Component`, filter servlet | `Middleware` class, `register()` | Rack middleware, `use` | WSGI middleware, decorator, view | `IMiddleware`, added in `app.UseMiddleware<T>()` | `app.use(middleware)` | `@Injectable()`, `NestMiddleware` or guards | Plug protocol, `plug :name` | Middleware trait, `wrap()` |
| **Dependency Injection** | `@Autowired`, `@Inject`, container wires types | `$this->app->make()`, service providers | Rails `initialize` with params, implicit lookup | Django depends on explicit passing or settings | `IServiceCollection`, registered in Startup | Manual or Express middleware | `@Injectable()`, module declarations | Dependency passed as argument | Manual or compile-time macros |
| **Model/Repository** | `@Entity`, JPA `interface OrderRepository` | Eloquent `Model`, or separate repository | ActiveRecord `class Order < ApplicationRecord` | Django ORM `Model`, or custom manager | Entity Framework `DbContext`, `DbSet<T>` | Manual or external ORM (Sequelize, Mongoose) | TypeORM entity, custom repository | `schema` and context module | Manual or Diesel ORM |
| **View/Template** | JSP, Thymeleaf, Spring View Resolver | Blade template engine | ERB or HAML | Django template language | Razor `.cshtml` | EJS, Handlebars, Pug | Template engine (EJS, Handlebars) or JSON serialization | EEx template engine | Serialization (JSON/XML) |

### Immediate Observations

**1. Annotation-based routing vs. convention-based routing**

Spring, ASP.NET, NestJS, and Actix use annotations or decorators on methods to declare routes. Laravel, Rails, Django, and Phoenix use explicit route files. Express and basic Node.js use imperative code.

The consequence: annotation-based frameworks make the route visible next to the handler (tight coupling in structure, but easy to navigate). Convention-based frameworks centralize all routes (one place to understand the whole API). Imperative frameworks are most explicit but require the most boilerplate.

**2. Container-driven DI vs. explicit passing**

Spring, ASP.NET Core, and NestJS have mandatory DI containers that resolve the dependency graph automatically. Express, basic Node.js, and Phoenix typically expect you to wire dependencies manually or at the framework level.

The consequence: container-driven frameworks scale better (100+ services become manageable), but startup is slower and errors are runtime. Manual wiring is faster and more transparent but becomes tedious at scale.

**3. Model-centric vs. separation of concerns**

Rails and Laravel blur the line between model, repository, and validator. A Rails model is the data, the queries, and the validation all in one. Spring, ASP.NET, and NestJS separate entity, repository, and service explicitly.

The consequence: Rails and Laravel are fast to write for CRUD operations but can accumulate logic in the model layer. Spring et al. encourage more structure upfront but require more files.

**4. Functional vs. object-oriented**

Phoenix and Elixir favor immutable data structures and pure functions. Rails and Django favor object mutation. Express lies in between—you can write it either way.

The consequence: functional frameworks are easier to reason about and test (immutability eliminates side effects) but require learning functional thinking. OO frameworks feel familiar if you know object-oriented languages but can hide mutations.

---

## Idiomatic Differences: Philosophy Shapes Implementation

A framework's philosophy is the second-most-important thing to learn after the names. Using Spring idioms in Express will work but will feel wrong. Using Phoenix idioms in Rails will feel foreign.

### Spring (Annotations, IoC Container Central)

Spring is built around annotations and automatic dependency wiring. Everything you declare becomes a bean, and the container resolves the graph.

```java
// Spring idiom: annotate, let the container wire
@RestController
@RequestMapping("/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest req) {
        Order order = orderService.createOrder(req.getUserId(), req.getProductId(), req.getQuantity());
        return ResponseEntity.status(201).body(order);
    }
}

@Service
@Transactional
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    public Order createOrder(String userId, String productId, int quantity) {
        // Service logic orchestrates the workflow
        return orderRepository.save(new Order(userId, productId, quantity));
    }
}

@Repository
public interface OrderRepository extends JpaRepository<Order, String> {
    // Spring data JPA generates the SQL
}
```

**Spring's philosophy:** Annotations are metadata. The container scans them, wires automatically, and manages the full lifecycle. You declare *what*, not *how*.

**When to use:** You have 100+ services, complex dependency graphs, and you want the container to handle the wiring. You are willing to learn Spring's idioms and accept runtime DI overhead.

**When not to:** You are building a small microservice, a CLI tool, or a system where startup time is critical. Spring's container adds overhead.

### Laravel (Service Providers, Convention Over Configuration)

Laravel uses service providers to organize registration. You write less explicit wiring and more rely on conventions (controllers in `app/Http/Controllers`, models in `app/Models`).

```php
// Laravel idiom: register in service providers, then use contracts
namespace App\Http\Controllers;

class OrderController {
    
    public function store(Request $request, OrderService $service) {
        // Laravel auto-resolves OrderService from the container
        $validated = $request->validate([
            'user_id' => 'required|uuid',
            'product_id' => 'required|uuid',
            'quantity' => 'required|int|min:1'
        ]);
        
        $order = $service->createOrder(
            $validated['user_id'],
            $validated['product_id'],
            $validated['quantity']
        );
        
        return response()->json(['order_id' => $order->id], 201);
    }
}

namespace App\Services;

class OrderService {
    
    public function __construct(OrderRepository $repository) {
        $this->repository = $repository;
    }
    
    public function createOrder($userId, $productId, $quantity) {
        return $this->repository->create([
            'user_id' => $userId,
            'product_id' => $productId,
            'quantity' => $quantity,
        ]);
    }
}

namespace App\Models;

class Order extends Model {
    // Model is the data + queries (ActiveRecord pattern)
    public function user() {
        return $this->belongsTo(User::class);
    }
    
    public static function forUser($userId) {
        return static::where('user_id', $userId)->get();
    }
}
```

**Laravel's philosophy:** Register services once, then use type hints. The framework auto-resolves types. Models are self-sufficient (include both data access and relationships). Validation is declarative (rules in arrays, not code).

**When to use:** You are building a monolithic web app and want rapid development. Laravel's conventions and built-in validation, querying, and routing make CRUD fast.

**When not to:** You have a complex domain with many business rules that don't fit neatly into models. You need strict separation of concerns. You want immutability (Laravel models are mutable).

### Rails (Convention Over Configuration, Implicit Workflows)

Rails takes convention even further. Controllers, models, routes, and views all follow strict naming conventions. If a model is `Order`, the table is `orders`, the controller is `OrdersController`, etc.

```ruby
# Routes (config/routes.rb) — convention declares the path
Rails.application.routes.draw do
  resources :orders
end

# Controller (app/controllers/orders_controller.rb)
class OrdersController < ApplicationController
  def create
    @order = Order.create(order_params)
    render json: @order, status: 201
  end
  
  private
  
  def order_params
    params.require(:order).permit(:user_id, :product_id, :quantity)
  end
end

# Model (app/models/order.rb) — ActiveRecord, blurs data and logic
class Order < ApplicationRecord
  belongs_to :user
  has_many :items
  
  validates :quantity, presence: true, numericality: { greater_than: 0 }
  
  scope :recent, -> { order(created_at: :desc) }
  
  def total_price
    items.sum(&:price)
  end
end
```

**Rails' philosophy:** Conventions eliminate boilerplate. If you follow them, the framework "just works." Models are objects that know how to persist themselves (ActiveRecord). Routes are declarative resource descriptions. Validation is metadata.

**When to use:** You are building a monolithic web app fast and your team is familiar with Rails. You want convention to drive architecture.

**When not to:** Your domain is complex and doesn't fit neatly into models. You want explicit over implicit. You need the code to run fast (Rails startup and runtime are slow compared to compiled languages). You want to test without a database.

### Django (Explicit, MVT, ORM-First)

Django uses explicit URL routing and form-centric validation. Models define the database schema. Views are request handlers. Templates render the response.

```python
# URLs (urls.py) — explicit mapping
from django.urls import path
from . import views

urlpatterns = [
    path('orders/', views.OrderListCreateView.as_view(), name='order_list_create'),
]

# Views (views.py) — class-based views handle requests
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class OrderListCreateView(APIView):
    def post(self, request):
        serializer = OrderSerializer(data=request.data)
        if serializer.is_valid():
            order = serializer.save()
            return Response({'order_id': order.id}, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

# Models (models.py) — define schema and queries
from django.db import models

class Order(models.Model):
    user_id = models.CharField(max_length=36)
    product_id = models.CharField(max_length=36)
    quantity = models.IntegerField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        db_table = 'orders'
    
    def __str__(self):
        return f"Order {self.id}"

# Serializers (serializers.py) — validation + schema
from rest_framework import serializers

class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'user_id', 'product_id', 'quantity']
        
        def validate_quantity(self, value):
            if value <= 0:
                raise serializers.ValidationError("Quantity must be positive")
            return value
```

**Django's philosophy:** Explicit over implicit. Routes are explicit mappings. Models define the database. Serializers handle validation. Views are request handlers. There is a right place for everything.

**When to use:** You are building an API or web app and want structure. Django's organization is clear even to newcomers. You want an ORM that works well (Django ORM is mature and powerful).

**When not to:** You need extreme performance (Django startup is slow). You want functional programming (Django is imperative and OO). Your use case doesn't fit the MVT pattern.

### ASP.NET Core (Attributes, Functional Composition)

ASP.NET Core uses attributes to decorate methods with metadata (route, HTTP method, authorization). Dependency injection is mandatory and configured fluently.

```csharp
// Startup (Program.cs or Startup.cs) — fluent DI registration
var services = new ServiceCollection();

services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<OrderService>();
services.AddControllers();

var app = services.BuildServiceProvider().CreateApplicationBuilder();
app.UseRouting();
app.UseEndpoints(endpoints => {
    endpoints.MapControllers();
});

// Controller (OrdersController.cs) — attributes declare route and method
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase {
    
    private readonly OrderService _service;
    
    public OrdersController(OrderService service) {
        _service = service;
    }
    
    [HttpPost]
    public async Task<IActionResult> CreateOrder(CreateOrderRequest req) {
        var order = await _service.CreateOrderAsync(req.UserId, req.ProductId, req.Quantity);
        return CreatedAtAction(nameof(CreateOrder), new { id = order.Id }, order);
    }
}

// Service (OrderService.cs)
public class OrderService {
    
    private readonly IOrderRepository _repository;
    
    public OrderService(IOrderRepository repository) {
        _repository = repository;
    }
    
    public async Task<Order> CreateOrderAsync(string userId, string productId, int quantity) {
        var order = new Order { UserId = userId, ProductId = productId, Quantity = quantity };
        await _repository.SaveAsync(order);
        return order;
    }
}

// Repository (IOrderRepository.cs)
public interface IOrderRepository {
    Task<Order> FindByIdAsync(string id);
    Task SaveAsync(Order order);
}
```

**ASP.NET Core's philosophy:** Fluent configuration API. Attributes provide metadata. Async/await is the default. Strong typing and inheritance (ControllerBase). Dependency injection is wired via interfaces.

**When to use:** You are building in C# and want a modern, performant web framework. ASP.NET Core is fast, has good tooling, and scales well.

**When not to:** You are not using C# (the framework is C#-only). You prefer lightweight frameworks (ASP.NET has more ceremony than Express).

### Express (Minimal, Middleware-First, Functional)

Express is minimal by design. You define routes imperatively and attach middleware as functions. There is no mandatory DI container, no models, no convention. You wire everything manually.

```javascript
// Basic setup
const express = require('express');
const app = express();

app.use(express.json());

// Middleware (runs before handlers)
const authorize = (req, res, next) => {
    if (!req.headers.authorization) {
        return res.status(401).json({ error: 'Unauthorized' });
    }
    next();
};

app.use(authorize);

// Route with handler (controller function)
app.post('/orders', async (req, res) => {
    const { user_id, product_id, quantity } = req.body;
    
    if (!user_id || !product_id || quantity <= 0) {
        return res.status(400).json({ error: 'Invalid input' });
    }
    
    try {
        const order = await orderService.createOrder(user_id, product_id, quantity);
        res.status(201).json({ order_id: order.id });
    } catch (err) {
        res.status(400).json({ error: err.message });
    }
});

// Service (plain JavaScript class)
class OrderService {
    constructor(repository) {
        this.repository = repository;
    }
    
    async createOrder(userId, productId, quantity) {
        return this.repository.save({
            user_id: userId,
            product_id: productId,
            quantity,
        });
    }
}

// Repository (plain class)
class OrderRepository {
    constructor(db) {
        this.db = db;
    }
    
    async save(order) {
        return this.db.insert('orders', order);
    }
}

// Manual wiring in main
const db = new Database('postgresql://localhost');
const orderRepo = new OrderRepository(db);
const orderService = new OrderService(orderRepo);

app.listen(3000, () => console.log('Server running on port 3000'));
```

**Express's philosophy:** Minimal framework, maximum flexibility. No conventions. You choose your own structure. Middleware is a function. Routes are imperative. DI is manual or delegated to user choice.

**When to use:** You are building a small API or microservice. You want full control over structure. You don't want ceremony or a DI container. You are comfortable wiring dependencies manually.

**When not to:** You have a large team that needs convention to stay aligned. You are building a monolithic web app with many features (Express's minimalism becomes a burden).

### NestJS (Framework Within Node.js, Angular-Like)

NestJS brings Spring-like structure to Node.js. It uses TypeScript decorators, a DI container, and a module system. It is opinionated like Spring but for the JavaScript ecosystem.

```typescript
// Decorator-based routing and DI
import { Controller, Post, Body, Injectable, Module } from '@nestjs/common';

@Injectable()
export class OrderService {
    constructor(private repository: OrderRepository) {}
    
    async createOrder(userId: string, productId: string, quantity: number) {
        return this.repository.save({
            userId,
            productId,
            quantity,
        });
    }
}

@Controller('orders')
export class OrderController {
    constructor(private service: OrderService) {}
    
    @Post()
    async create(@Body() req: CreateOrderRequest) {
        const order = await this.service.createOrder(
            req.userId,
            req.productId,
            req.quantity
        );
        return { orderId: order.id };
    }
}

@Module({
    providers: [OrderService, OrderRepository],
    controllers: [OrderController],
    exports: [OrderService],
})
export class OrderModule {}

// Main app bootstraps modules
@Module({
    imports: [OrderModule],
})
export class AppModule {}
```

**NestJS's philosophy:** Structure and DI like Spring, but on Node.js. Modules organize features. Decorators declare intent. The framework handles the boilerplate.

**When to use:** You are writing Node.js in TypeScript and want the structure of Spring or ASP.NET Core. You have a large team and want enforced patterns.

**When not to:** You prefer Express's minimalism. You want the simplest possible code (NestJS adds ceremony). You are prototyping and want to move fast.

### Phoenix (Functional, Immutable Data, Pattern Matching)

Phoenix uses Elixir's functional paradigm. There are no mutable models. Data is passed through pipelines (functions). Controllers are modules with functions. The pattern is explicit and functional.

```elixir
# Router (router.ex) — explicit path-to-handler mapping
defmodule MyApp.Router do
  use MyApp, :router

  pipeline :api do
    plug :accepts, ["json"]
  end

  scope "/api", MyApp.API do
    pipe_through :api
    post "/orders", OrderController, :create
  end
end

# Controller (controllers/order_controller.ex) — function receives conn + params
defmodule MyApp.API.OrderController do
  use MyApp, :controller
  
  def create(conn, %{"user_id" => user_id, "product_id" => product_id, "quantity" => quantity}) do
    # Service is called with data, returns new data
    case OrderService.create_order(user_id, product_id, quantity) do
      {:ok, order} ->
        conn
        |> put_status(:created)
        |> json(%{order_id: order.id})
      
      {:error, reason} ->
        conn
        |> put_status(:bad_request)
        |> json(%{error: reason})
    end
  end
end

# Service (services/order_service.ex) — pure functions, no mutation
defmodule MyApp.OrderService do
  
  def create_order(user_id, product_id, quantity) do
    order = %Order{
      user_id: user_id,
      product_id: product_id,
      quantity: quantity
    }
    
    case OrderRepository.save(order) do
      {:ok, saved_order} -> {:ok, saved_order}
      {:error, reason} -> {:error, reason}
    end
  end
end

# Repository (repos/order_repository.ex) — queries are atoms, not SQL strings
defmodule MyApp.OrderRepository do
  
  def save(order) do
    case Repo.insert(order) do
      {:ok, saved_order} -> {:ok, saved_order}
      {:error, changeset} -> {:error, changeset}
    end
  end
end
```

**Phoenix's philosophy:** Immutability and pure functions. Data flows through pipelines. Errors are explicit (`:ok` / `:error` tuples, not exceptions). The request is a connection struct that is transformed by middleware and handlers.

**When to use:** You are building a real-time system (Elixir shines for concurrency). You want immutability and functional purity. Your team knows or is willing to learn Elixir.

**When not to:** You need the JavaScript or Python ecosystems. You want mutable state. Your team is inexperienced with functional programming.

### Actix (Rust, Compile-Time, Type-Safe)

Actix is Rust's web framework. It uses macros to declare routes and relies on Rust's type system for safety. Middleware is middleware traits. There is no runtime DI container; dependencies are explicit.

```rust
// Handler (defined with a macro)
use actix_web::{web, HttpResponse, post};

struct OrderService {
    repository: OrderRepository,
}

#[post("/orders")]
async fn create_order(
    service: web::Data<OrderService>,
    req: web::Json<CreateOrderRequest>,
) -> HttpResponse {
    match service.create_order(&req.user_id, &req.product_id, req.quantity).await {
        Ok(order) => HttpResponse::Created().json(order),
        Err(e) => HttpResponse::BadRequest().json(json!({ "error": e })),
    }
}

// Service (plain Rust struct)
impl OrderService {
    async fn create_order(&self, user_id: &str, product_id: &str, quantity: i32) -> Result<Order, String> {
        self.repository.save(Order {
            user_id: user_id.to_string(),
            product_id: product_id.to_string(),
            quantity,
        }).await
    }
}

// Main app setup
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let order_service = web::Data::new(OrderService {
        repository: OrderRepository::new(),
    });
    
    HttpServer::new(move || {
        App::new()
            .app_data(order_service.clone())
            .service(create_order)
    })
    .bind("127.0.0.1:3000")?
    .run()
    .await
}
```

**Actix's philosophy:** Compile-time safety. Type system prevents errors. Macros are compile-time code generation. Manual DI (explicit wiring via `web::Data`). Performance is the default (Actix is very fast).

**When to use:** You are building in Rust and want the fastest possible web framework. You want compile-time guarantees. You are comfortable with Rust's learning curve.

**When not to:** You are not using Rust. You want to move fast in a dynamic language. Rust's borrow checker and type system have a steep learning curve.

---

## Routing Variations: How You Declare Which Path Goes Where

Routing is the entry point. Every request arrives with a path, and the router's job is to match it to a handler. Frameworks approach this three ways.

### Annotation-Based (Spring, ASP.NET, NestJS, Actix)

Routes are declared as attributes or macros on the handler function:

```java
// Spring
@PostMapping("/orders/{id}")
public ResponseEntity<Order> getOrder(@PathVariable String id) { }

// ASP.NET Core
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(string id) { }

// NestJS
@Get(':id')
getOrder(@Param('id') id: string) { }

// Actix
#[get("/orders/{id}")]
async fn get_order(id: web::Path<String>) -> HttpResponse { }
```

**Consequence:** The route is visible next to the handler. Navigating from route to code is easy (click the decorator, jump to the method). Adding a new route requires adding a decorator. Refactoring a route means finding all decorated methods.

**Discovery:** Tools and IDEs can easily find all routes (search for the annotation), but the routes are scattered across files.

### Explicit Route Files (Laravel, Rails, Django, Phoenix)

Routes are declared in a central file, mapping path to handler:

```ruby
# Rails
Rails.application.routes.draw do
  resources :orders
  get '/orders/:id', to: 'orders#show'
  post '/orders', to: 'orders#create'
end

# Django
urlpatterns = [
    path('orders/<int:id>/', views.order_detail),
    path('orders/', views.order_list_create),
]

# Laravel
Route::get('/orders/{id}', [OrderController::class, 'show']);
Route::post('/orders', [OrderController::class, 'store']);

# Phoenix
scope "/api" do
  get "/orders/:id", OrderController, :show
  post "/orders", OrderController, :create
end
```

**Consequence:** All routes are in one place (easy to understand the API shape). Adding or changing a route is easy (edit one file). Finding which code handles a route requires looking up the controller method.

**Discovery:** Tools can parse the route file and show you the entire API. But navigating from route to code requires manual lookup.

### Imperative Registration (Express, Basic Node.js)

Routes are registered in code, usually in the order they are declared:

```javascript
// Express
app.get('/orders/:id', getOrder);
app.post('/orders', createOrder);
app.put('/orders/:id', updateOrder);

const getOrder = (req, res) => { };
const createOrder = (req, res) => { };
const updateOrder = (req, res) => { };
```

**Consequence:** Routes and handlers are defined together (explicit and co-located). Middleware can wrap routes locally. There is no central registry (routes are scattered). Adding a route requires adding code in the handler section.

**Discovery:** No central file. You must grep for route registrations. IDEs have harder time finding all routes.

### Choosing Based on Philosophy

- **Annotation-based** works best when you have many routes and want the route visible next to the logic.
- **Explicit route files** work best when you want to understand the whole API in one place and when routes are stable.
- **Imperative** works best when you are building a small API or want fine-grained control over middleware per route.

---

## Middleware Variations: Intercepting Request/Response

Middleware runs before and after request handling. Frameworks conceptualize it differently.

### Onion (Express, NestJS)

Middleware wraps the request. It runs in order on the way in, then in reverse order on the way out.

```javascript
// Express: middleware wraps the handler
app.use(logRequest);  // Runs first on request in
app.use(authenticate);  // Runs second on request in
app.post('/orders', createOrder);  // Handler
// Then middleware runs in reverse on response out

const logRequest = (req, res, next) => {
    console.log('Request:', req.method, req.path);
    next();  // Pass to next middleware
};

const authenticate = (req, res, next) => {
    if (!req.headers.authorization) {
        return res.status(401).json({ error: 'Unauthorized' });
    }
    next();
};
```

**Consequence:** Middleware forms layers. Outer layers run first on the way in, last on the way out. You can intercept both the request and the response.

### Filter Chain (Spring)

Middleware is a chain of filters. Each filter can transform the request before delegating to the next.

```java
// Spring: filter intercepts request and response
@Component
public class AuthenticationFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        
        // Before: check authorization
        if (!req.getHeader("Authorization").startsWith("Bearer ")) {
            res.setStatus(401);
            res.getWriter().write("Unauthorized");
            return;
        }
        
        // Delegate to the next filter
        chain.doFilter(req, res);
        
        // After: log response
        System.out.println("Response status: " + res.getStatus());
    }
}
```

**Consequence:** Filters form a chain. Each filter can decide whether to pass to the next or short-circuit. Before and after hooks are explicit in a single method.

### Plug (Phoenix)

Phoenix's "plugs" are pipelines of transformations. Each plug transforms the connection.

```elixir
# Phoenix: plugs transform the connection
defmodule MyApp.Router do
  use MyApp, :router
  
  pipeline :api do
    plug :accepts, ["json"]
    plug :authenticate
  end
  
  defp authenticate(conn, _opts) do
    case get_authorization(conn) do
      {:ok, user} -> assign(conn, :user, user)
      :error -> send_resp(conn, 401, "Unauthorized")
    end
  end
end
```

**Consequence:** Plugs are pure functions that transform a connection struct. The connection flows through the pipeline. No explicit `next()` calls; the pipeline is composed declaratively.

### Middleware in Action

Despite the differences, they all do the same thing: run code before and after the handler. Choose the framework's idiom:

- **Express onion**: intuitive for small pipelines, explicit about before/after.
- **Spring filters**: good for global concerns (security, logging), explicit chain.
- **Phoenix plugs**: functional and composable, part of the larger functional paradigm.

---

## Dependency Injection Variations: Wiring Services

Every framework needs to wire services together. Frameworks differ in how much is automatic and how much is manual.

### Annotation-Driven Auto-Wiring (Spring, NestJS)

The container scans for types and wires them based on type hints.

```java
// Spring: @Autowired and types are enough
@Service
public class OrderService {
    @Autowired
    private OrderRepository repository;  // Scanned and wired
}

// Or in constructor (preferred)
@Service
public class OrderService {
    private final OrderRepository repository;
    
    @Autowired
    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

**Cost:** Requires reflection and a DI container. Startup scans the classpath. Errors are runtime (the container might fail to resolve the graph).

**Benefit:** Minimal boilerplate. Adding a new service just requires writing the class and using the annotation.

### Explicit Registration (ASP.NET Core, Laravel)

You tell the container what to wire via a fluent API or configuration method.

```csharp
// ASP.NET Core: explicit registration
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<OrderService>();
```

**Cost:** You must explicitly register everything. More boilerplate.

**Benefit:** Clear what is wired. Errors are caught early. No scanning required; startup is faster.

### Manual Wiring (Express, Phoenix, Actix)

You wire services explicitly in the main setup.

```javascript
// Express: manual wiring
const db = new Database('postgresql://localhost');
const orderRepo = new OrderRepository(db);
const orderService = new OrderService(orderRepo);

app.use((req, res, next) => {
    req.services = { orderService };
    next();
});
```

**Cost:** Very explicit and verbose. Every service must be instantiated in main.

**Benefit:** Zero magic. No reflection, no container, no scanning. Startup is fast. Errors are clear.

### Hybrid: Convention + Explicit (Laravel, Rails)

The framework auto-resolves types by convention, but you can override with explicit configuration.

```php
// Laravel: auto-resolve by type hint, plus manual bindings
$container->bind('stripe', function () {
    return new StripeClient(config('stripe.key'));
});

public function __construct(StripeClient $client) {
    // Laravel resolves this from the binding
    $this->client = $client;
}
```

**Cost:** Sometimes implicit (auto-resolve), sometimes explicit (manual bindings). Can be confusing when the boundary is unclear.

**Benefit:** Best of both worlds: auto-resolve simple cases, explicit control when needed.

---

## ORM Variations: Hiding Data Storage

Every framework needs to hide the database. Two main patterns exist.

### ActiveRecord (Rails, Laravel, Django)

The model is the data, the queries, and the validation. It knows how to save itself.

```ruby
# Rails: model does it all
class Order < ApplicationRecord
  validates :quantity, presence: true, numericality: { greater_than: 0 }
  
  def self.for_user(user_id)
    where(user_id: user_id)
  end
  
  def total_price
    items.sum(&:price)
  end
end

# Usage
order = Order.create(user_id: 'u123', product_id: 'p456', quantity: 2)
order.save!
orders = Order.for_user('u123')
```

**Consequence:** Models are fat. They contain data, queries, and logic. This is fast for CRUD but can accumulate too much responsibility.

**Benefit:** Intuitive for CRUD. Minimal boilerplate. Relations are built-in (` order.items` loads related items).

### Data Mapper (Spring JPA, .NET Entity Framework, Elixir Ecto)

Entities are data. Repositories handle queries. They are separate.

```java
// Spring: Entity is data, Repository is queries
@Entity
public class Order {
    @Id
    private String id;
    
    private String userId;
    private String productId;
    private int quantity;
    
    // Getters and setters only
}

@Repository
public interface OrderRepository extends JpaRepository<Order, String> {
    List<Order> findByUserId(String userId);
}

// Usage
Order order = new Order();
order.setUserId('u123');
repository.save(order);

List<Order> orders = repository.findByUserId('u123');
```

**Consequence:** Separation of concerns. Entities are POJOs (Plain Old Java Objects). Repositories are query objects. This is more structured.

**Benefit:** Testable (mock repositories easily). Clear boundaries. Entities remain simple.

### Choosing Between Patterns

- **ActiveRecord** is faster for CRUD and small apps. Models are intuitive.
- **Data Mapper** scales better for complex domains. Entities remain simple.

Most teams start with ActiveRecord and graduate to Data Mapper as the domain grows.

---

## Worked Example: Listing Orders in Four Frameworks

Let's trace the same feature—list all orders for a user—through four different frameworks. This shows how the universal six appear in different contexts.

### Laravel (ActiveRecord)

```php
// 1. Route (routes/api.php)
Route::get('/users/{id}/orders', [OrderController::class, 'index']);

// 2. Controller (app/Http/Controllers/OrderController.php)
class OrderController {
    public function index($userId) {
        // Fetch via model's static method
        $orders = Order::where('user_id', $userId)->get();
        
        return response()->json([
            'orders' => $orders,
        ]);
    }
}

// 3. Model (app/Models/Order.php) — includes query methods
class Order extends Model {
    protected $table = 'orders';
    
    public static function forUser($userId) {
        return static::where('user_id', $userId)->get();
    }
}
```

**Flow:** Route → Controller (request protocol) → Model (query via relation) → Database.

**Responsibility:** Controller is thin (translates HTTP). Model has data + queries.

### Spring (Data Mapper + Service)

```java
// 1. Route and Controller (annotation-based)
@RestController
@RequestMapping("/users/{userId}/orders")
public class OrderController {
    @Autowired
    private OrderService service;
    
    @GetMapping
    public ResponseEntity<List<OrderDTO>> listOrders(@PathVariable String userId) {
        // Delegate to service
        List<Order> orders = service.listOrdersForUser(userId);
        
        // Convert to DTO
        List<OrderDTO> dtos = orders.stream()
            .map(OrderDTO::from)
            .collect(Collectors.toList());
        
        return ResponseEntity.ok(dtos);
    }
}

// 2. Service (orchestrates use case)
@Service
public class OrderService {
    @Autowired
    private OrderRepository repository;
    
    public List<Order> listOrdersForUser(String userId) {
        return repository.findByUserId(userId);
    }
}

// 3. Repository (query interface)
@Repository
public interface OrderRepository extends JpaRepository<Order, String> {
    List<Order> findByUserId(String userId);
}

// 4. Entity (data only)
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private String id;
    
    private String userId;
    private String productId;
    private int quantity;
    
    // Getters only
}
```

**Flow:** Route (annotation) → Controller (translate HTTP) → Service (orchestrate) → Repository (query) → Entity (data) → Database.

**Responsibility:** Each layer has one job. Service orchestrates the logic. Repository hides SQL. Entity is a data holder.

### Express (Minimal, Manual)

```javascript
// 1. Route and handler (one step)
app.get('/users/:userId/orders', async (req, res) => {
    const { userId } = req.params;
    
    try {
        // Fetch directly
        const orders = await orderService.listOrdersForUser(userId);
        res.json({ orders });
    } catch (err) {
        res.status(400).json({ error: err.message });
    }
});

// 2. Service (plain class)
class OrderService {
    constructor(repository) {
        this.repository = repository;
    }
    
    async listOrdersForUser(userId) {
        return this.repository.findByUserId(userId);
    }
}

// 3. Repository (plain class)
class OrderRepository {
    constructor(db) {
        this.db = db;
    }
    
    async findByUserId(userId) {
        return this.db.query('SELECT * FROM orders WHERE user_id = ?', [userId]);
    }
}

// 4. Main (manual wiring)
const db = new Database('postgresql://localhost');
const orderRepo = new OrderRepository(db);
const orderService = new OrderService(orderRepo);
```

**Flow:** Route (in-line) → Handler (request protocol) → Service (orchestration) → Repository (query) → Database.

**Responsibility:** Minimal. Service and Repository are plain classes. Handler is a function.

### Phoenix (Functional)

```elixir
# 1. Router (explicit path-to-handler mapping)
defmodule MyApp.Router do
  scope "/users" do
    get "/:user_id/orders", OrderController, :index
  end
end

# 2. Controller (function receives conn + params)
defmodule MyApp.OrderController do
  def index(conn, %{"user_id" => user_id}) do
    orders = OrderService.list_for_user(user_id)
    
    conn
    |> put_status(:ok)
    |> json(%{orders: orders})
  end
end

# 3. Service (pure functions)
defmodule MyApp.OrderService do
  def list_for_user(user_id) do
    OrderRepository.find_by_user(user_id)
  end
end

# 4. Repository (Ecto queries)
defmodule MyApp.OrderRepository do
  def find_by_user(user_id) do
    from(o in Order, where: o.user_id == ^user_id)
    |> Repo.all()
  end
end
```

**Flow:** Route (explicit mapping) → Controller (transform conn) → Service (pure logic) → Repository (query) → Database.

**Responsibility:** Clear and functional. No mutation. Each function takes input, returns output.

---

## Observations Across the Four

| Aspect | Laravel | Spring | Express | Phoenix |
|--------|---------|--------|---------|---------|
| **Route declaration** | Explicit file | Annotation | In-line | Explicit file |
| **Controller** | Method on class | Class with annotations | Function | Function |
| **Service** | Implicit (often in model) | Explicit class | Explicit class | Explicit function |
| **Data layer** | Model with queries | Separate entity + repository | Separate repository | Separate repository |
| **DI** | Auto-resolve by type | Container wires | Manual | Manual |
| **Paradigm** | OO + ActiveRecord | OO + Data Mapper | Functional + flexible | Pure functional |

Each framework's choices lead to different code shapes. Laravel is fast to write for CRUD. Spring enforces structure. Express is minimal and explicit. Phoenix is pure and testable.

---

## Common Misconceptions

**Misconception 1: "Frameworks with the same name for a layer do the same thing."**

False. A "service" in Rails (implicit, often in the model) is different from a "service" in Spring (explicit orchestrator). A "repository" in Express (plain class) is different from a "repository" in Spring (JPA interface). The name is a convention, not a law.

**Misconception 2: "You should force one framework's structure onto another."**

False. Rails idioms feel wrong in Spring. Spring idioms feel heavy in Express. Each framework is optimized for a philosophy. Use Express idiomatically: minimal, functional, explicit. Use Rails idiomatically: convention-driven, ActiveRecord, fast CRUD.

**Misconception 3: "More layers = better design."**

False. Express with three layers (controller, service, repository) is better than Express with one layer (route handler that queries the database). But four layers are overkill. Match the number of layers to the complexity of the domain.

**Misconception 4: "The framework determines the quality of the code."**

False. Bad code can be written in Spring, Express, or Rails. Good code can be written in any of them. The framework shapes the structure, but discipline shapes the quality.

**Misconception 5: "You must understand all six concepts to write code in a new framework."**

False. But you must understand where each concept lives in the framework you are using. If you don't understand where the routing happens, you will struggle to add a new endpoint.

---

## When To Translate vs. When To Adapt

Here is the crucial skill: knowing when to translate (recognize the pattern) and when to adapt (follow the framework's idiom).

**Translate when:**
- You are learning a new framework and you want to map it back to concepts you know.
- You are porting code from one framework to another. Translate the structure, then adapt the syntax.
- You are comparing two frameworks to decide which to use.

**Adapt when:**
- You are writing production code in the framework. Use its idioms, not your old framework's.
- You are reading someone else's code. Understand it in the framework's terms, not in translation.
- You are optimizing for the framework's strengths. Express is fast because it is minimal; adding Spring structure to Express defeats the purpose.

**Example:** You are an expert in Spring and are joining a Rails project. Do not try to write Spring code in Rails. Instead:

1. **Translate first:** Recognize that Rails models are like Spring services + repositories combined.
2. **Adapt second:** Write Rails code using ActiveRecord and models, not by separating repositories.
3. **Improve:** If the model becomes too complex, introduce a service layer—but do it the Rails way, not the Spring way.

---

## Exercises

**Exercise 1: Map a New Framework**

Pick a web framework you've never used (Fiber in Go, Rocket in Rust, Sails in Node.js). Read its documentation and identify where each of the universal six concepts lives. Create a mapping table like the one in this chapter. How does its philosophy differ from the frameworks here?

**Exercise 2: Translate Code**

Take the Spring example (ordering endpoint) and translate it to Laravel, then to Express, then to Phoenix. Focus on mapping responsibilities, not syntax. Which translation felt natural? Which felt forced? Why?

**Exercise 3: Identify Misconceptions**

Read the source code of a web application written in a framework you know. Find a place where the developer seems to have tried to force another framework's idioms. Explain why it doesn't fit. How would you restructure it to be idiomatic?

**Exercise 4: Compare Middleware**

Write the same middleware (logging, authentication) in Express (onion), Spring (filter), and Phoenix (plug). How do the implementations differ? Which feels most natural to you? Why?

**Exercise 5: Justify a Framework Choice**

You are choosing a framework for a new project. Your team has equal expertise in Spring, Rails, and Express. The project is a monolithic web app with complex business logic. Justify which framework you would choose based on the philosophies explained in this chapter.

---

## Summary

Every web framework solves the same six problems: routing, controllers, middleware, dependency injection, models/repositories, and views. The names change, the philosophy changes, but the problems are constant.

A framework's philosophy shapes how you solve these problems. Spring enforces structure through annotations and a DI container. Rails embraces convention and ActiveRecord. Express is minimal and explicit. Phoenix is functional and pure. Understanding these differences is more important than memorizing syntax. Once you recognize the pattern, a new framework is not a mystery—it is a familiar concept with an unfamiliar face.

Learn to translate (map concepts across frameworks) and to adapt (write idiomatic code in the framework you are using). The skill is not learning a framework; it is learning to learn frameworks.

---

> **[← Previous: Recognizing Common Runtime Patterns](02-recognizing-runtime-patterns.md)**  ·  **[↑ Part 9](README.md)**  ·  **[Next: Recognizing Architectural Patterns →](04-recognizing-architectural-patterns.md)**
