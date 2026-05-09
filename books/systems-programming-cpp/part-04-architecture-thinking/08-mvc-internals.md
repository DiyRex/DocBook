# Chapter 40 — MVC Internals

MVC is one of the oldest architectural patterns and one of the most misused. The original idea, born in Smalltalk-80 in 1979, is not what most web frameworks call MVC today. The difference matters. If you don't know which one is in front of you, you will make design decisions that don't fit the actual architecture, and your code will feel wrong. This chapter shows both, so you can recognize them and know what each one demands.

## Learning Objectives

By the end of this chapter you will be able to:

1. Describe the original Smalltalk MVC pattern — what the Model, View, and Controller did in 1979.
2. Explain how web MVC differs: the roles shift, the data flow changes, and the patterns that follow.
3. Identify which variant you are working with in a real codebase, based on how data flows and who observes whom.
4. Recognize how MVC relates to the Controllers-Services-Repositories pattern from Chapter 38 and where they overlap.
5. Understand why MVC without Dependency Injection becomes untestable.
6. Decide when MVC is the right pattern and when it is not (and when you should admit you don't have a View at all).

---

## 40.1 The Original Smalltalk MVC (1979)

In Xerox Smalltalk-80, MVC solved a specific problem: how to allow multiple Views of the same underlying Model, and how to keep all of them in sync.

### The Architecture

**Model**: The Model is the state and the rules. It knows how to load data from storage, apply business rules, validate invariants, and notify observers when state changes. The Model never knows about any View.

**View**: The View is a visual representation. It observes the Model and updates itself whenever the Model changes. If the Model's temperature changes, every Celsius display and Fahrenheit display update automatically. The View never modifies the Model directly.

**Controller**: The Controller receives input from the user (mouse clicks, key presses) and translates them into method calls on the Model. It decides what the user's gesture means, then tells the Model to change. If the user clicks "increase temperature," the Controller calls `model.increaseTemperature()`. The Controller never updates the View directly; the View observes the Model and updates itself.

### The Data Flow

```
User Input
  ↓
[Controller] →→→ model.setTemperature(newValue)
                  ↓
              [Model]
                  ↓ (notifies observers)
                  ↓
[View listens] ←←← (setState based on new Model)
  ↓
Screen Update
```

The triangle is not synchronous handoff. It is:

1. **Controller** sees input, calls Model.
2. **Model** changes internally, broadcasts change.
3. **View** reacts to the broadcast and redraws.

### The Key Insight

The decoupling is radical: **the View never talks to the Controller, and the Controller never talks to the View.** Communication happens through the Model as the hub. This means you can have many Views (a chart, a table, a gauge, all showing the same data) and they all stay in sync because they all observe the same Model.

### Cost and Benefit

**Benefits:**
- Multiple independent Views can coexist without knowing about each other.
- The Model is fully testable in isolation (it has no UI dependencies).
- Adding a new View does not require changing the Model or the existing Views.

**Costs:**
- You need an observer pattern implementation (callbacks, events, or some equivalent).
- The flow of control becomes less obvious; multiple Views responding to the same Model change can be hard to trace.
- If the Model broadcasts too frequently (e.g., on every keystroke), the Views can thrash.

---

## 40.2 Web MVC (Rails, Laravel, Spring MVC, ASP.NET)

Web MVC flattened the triangle. It solved a different problem: how to structure a request-response cycle when the response involves data, logic, and template rendering.

### The Architecture

**Model**: The Model is usually domain state — User, Order, Product, etc. It knows how to load and persist itself. In frameworks like Rails, the Model is an ORM (ActiveRecord, Eloquent); in Spring, it is a POJO with annotations. The Model does NOT broadcast changes; it is just data + some behavior.

**View**: The View is a template — HTML, JSON, XML — that takes data and renders it. Crucially, the View does NOT observe the Model. It is rendered once per request and discarded. The template is filled with data from the Controller.

**Controller**: The Controller orchestrates the request. It parses the HTTP request, loads Models from the database (or has the Service load them), applies business rules, and passes data to the View. The Controller decides what data the View gets to see.

### The Data Flow

```
HTTP Request
  ↓
[Router matches route]
  ↓
[Controller action]
  ├─→ parse request
  ├─→ load Model (or call Service, which loads Model)
  ├─→ apply business logic / call service
  ├─→ pass data to View
  ↓
[View] renders template with data
  ↓
HTTP Response (HTML/JSON)
```

### Key Differences from Smalltalk MVC

**1. Request-based, not event-based.** In Smalltalk, the UI event loop ran continuously. In web MVC, each HTTP request is independent; the Controller runs once, passes data to the View once, and the response goes back. There is no persistent Model running in the browser.

**2. The View is not an observer.** In Smalltalk, the View listened to the Model continuously. In web MVC, the View is given data and renders it. There is no subscription mechanism; the data is passed as arguments.

**3. The Controller is the orchestrator, not the dispatcher.** In Smalltalk, the Controller merely translated input to a Model method call. In web MVC, the Controller does heavy lifting: it loads data, applies logic (or delegates to a Service), shapes data for the View, and handles errors.

**4. Template language, not programmatic View.** In Smalltalk, a View was a class (possibly with its own state). In web MVC, the View is often a string template (ERB, Jinja, Blade, Razor) that the Controller fills in. The template is stateless.

### Example: Rails-like Web MVC

```ruby
# config/routes.rb
get "/books/:id", to: "books#show"

# app/controllers/books_controller.rb
class BooksController < ApplicationController
  def show
    # 1. Parse the request (routing gave us :id)
    book_id = params[:id]
    
    # 2. Load the Model (or call a Service)
    @book = Book.find(book_id)
    
    # 3. If business logic is involved, call a Service (from Ch 38)
    @recommendation_service = RecommendationService.new
    @similar_books = @recommendation_service.findSimilar(@book)
    
    # 4. Pass data to the View
    # (Rails automatically renders app/views/books/show.html.erb)
    # with @book and @similar_books available
  end
end

# app/views/books/show.html.erb
<div class="book">
  <h1><%= @book.title %></h1>
  <p>Author: <%= @book.author %></p>
  <p>Price: $<%= @book.price %></p>
  
  <h2>Similar Books</h2>
  <ul>
    <% @similar_books.each do |book| %>
      <li><a href="/books/<%= book.id %>"><%= book.title %></a></li>
    <% end %>
  </ul>
</div>

# app/models/book.rb
class Book < ApplicationRecord
  # ORM mapping, associations, validations
  has_many :authors
  validates :title, presence: true
  
  def discounted_price
    price * 0.9
  end
end
```

Notice:
- The Controller loads data and passes it to the View as instance variables.
- The View (template) renders static HTML mixed with data.
- The Model (Book) is ORM-mapped, with associations and validations.
- There is no observer pattern; the View does not listen to anything.

---

## 40.3 MVC + Services + Repositories = Common Modern Stack

In practice, modern web frameworks often layer three patterns together:

1. **MVC** for the request-response cycle.
2. **Services** (from Chapter 38) for business logic and workflows.
3. **Repositories** (from Chapter 38) for data access.

The flow becomes:

```
HTTP Request
  ↓
[Controller]
  ├─→ parse request
  ├─→ call Service
  │    ├─→ call Repository (load data)
  │    ├─→ apply business rules
  │    ├─→ call another Repository (save data)
  │    └─→ return Result
  ├─→ pass Result to View
  ↓
[View] renders Result
  ↓
HTTP Response
```

The Controller is thin (parse → delegate → render). The Service handles orchestration. The Repository handles persistence. The Model is just the domain data shape.

This is not pure MVC. It is MVC + a separation of concerns from the Service/Repository pattern. But it is what most web applications do today, and it is a sensible structure.

---

## 40.4 What Each Component Owns

When MVC is working well, the ownership is clear:

### Controller

**Owns:**
- HTTP request parsing (query params, body, headers, cookies).
- Parameter validation (is this field present? is it the right type?).
- Routing dispatch (which action should handle this?).
- Response formatting (what status code? what headers? what content type?).

**Does not own:**
- Business rules (those live in the Service or Model).
- Database queries (those live in the Repository).
- External API calls without orchestration (those belong in a Service).

### Model

**Owns:**
- Data shape and relationships (what fields does a Book have? how does a Book relate to an Author?).
- Validation rules that are intrinsic to the domain (a Book's title cannot be empty; a User's email must be unique).
- Calculations that make sense only in the domain (a Subscription's isActive status based on its expiration date).

**Does not own:**
- HTTP status codes.
- Serialization details (JSON, XML, Protocol Buffers — those are View concerns).
- Orchestration of multiple Models (that is the Service's job).

### View

**Owns:**
- How to display data (HTML structure, CSS classes, JavaScript interactivity).
- Presentation logic (should this be bold? should that be hidden?).
- Format translation (how does a datetime print? how are numbers formatted?).

**Does not own:**
- Business rules (a View should never ask "is this user allowed to see this?"; the Controller or Service should filter before passing data).
- Data fetching (the View should never query the database; data comes from the Controller).
- State mutation (the View should never call model.save() or model.delete()).

---

## 40.5 Anti-Patterns and Their Failures

### Anti-Pattern 1: SQL in the View Template

```erb
<!-- WRONG -->
<table>
  <% @users = User.where("age > 18 AND status = ?", "active") %>
  <% @users.each do |user| %>
    <tr><td><%= user.name %></td><td><%= user.email %></td></tr>
  <% end %>
</table>
```

**What breaks:**
- If the filtering rule changes, you must hunt through all templates that use it.
- The rule is implemented in one template, ignored in another. Inconsistency.
- You cannot reuse the same filtered list in an API response or a report.
- Testing: you cannot test the filtering logic without rendering HTML.

**Correct approach:**
```ruby
# Controller
@users = userService.findActiveAdults()

# View
<table>
  <% @users.each do |user| %>
    <tr><td><%= user.name %></td><td><%= user.email %></td></tr>
  <% end %>
</table>

# Service
def findActiveAdults
  userRepository.findByAgeGreaterThan(18).select { |u| u.status == "active" }
end
```

### Anti-Pattern 2: Business Rules in the Controller

```cpp
// WRONG
class OrderController {
  HttpResponse createOrder(const HttpRequest& req) {
    auto userId = req.parameters()["user_id"];
    auto user = database->findUser(userId);
    
    // Business logic: check subscription
    if (!user.hasActiveSubscription()) {
      return HttpResponse(403, "Subscription required");
    }
    
    // More business logic: check quota
    if (user.monthlyOrderCount() >= user.monthlyLimit()) {
      return HttpResponse(403, "Monthly limit exceeded");
    }
    
    // ... create the order
  }
};
```

**What breaks:**
- The same order-creation logic might be called from a batch job, an API, or an event handler. If those paths skip the Controller, they skip the business rules.
- Changes to the rules require updating the Controller; the Service layer becomes useless.
- Testing: you must HTTP-mock to test business logic.

**Correct approach:**
```cpp
// Controller: thin, mechanical
class OrderController {
  HttpResponse createOrder(const HttpRequest& req) {
    auto userId = req.parameters()["user_id"];
    auto result = orderService->createOrder(userId, /* ... */);
    
    if (!result.isSuccess()) {
      return HttpResponse(400, result.error());
    }
    
    return HttpResponse(201, json(result.order()));
  }
};

// Service: owns business logic
class OrderService {
  Result<Order> createOrder(string userId, /* ... */) {
    auto user = userRepo->findById(userId);
    
    if (!user.hasActiveSubscription()) {
      return Error("Subscription required");
    }
    if (user.monthlyOrderCount() >= user.monthlyLimit()) {
      return Error("Monthly limit exceeded");
    }
    
    // ... persist and return
  }
};
```

### Anti-Pattern 3: Templates Calling Model Methods with Side Effects

```erb
<!-- WRONG -->
<div>
  <h1><%= @order.title %></h1>
  
  <!-- This method increments a counter; calling it while rendering breaks idempotence -->
  <p>Views: <%= @order.incrementViewCount() %></p>
  
  <!-- This sends an email; rendering the same template twice sends it twice -->
  <%= @order.sendNotification() %>
</div>
```

**What breaks:**
- The template is no longer idempotent. Rendering the same page twice has side effects.
- The View is responsible for business outcomes (sending emails), not just presentation.
- Debugging: you see the side effect happen, but the stack trace points to the template renderer, not the Controller or Service.

**Correct approach:**
```ruby
# Controller
@order = orderService.fetchOrder(order_id)
orderService.incrementViewCount(@order)

# View
<div>
  <h1><%= @order.title %></h1>
  <p>Views: <%= @order.view_count %></p>
</div>
```

The side effect happens in the Service, and the View just displays the data.

---

## 40.6 MVVM, MVP, and When They Matter

The web MVC pattern has cousins that solve different problems.

### MVVM (Model-View-ViewModel)

**Where it appears:** WPF (Windows), SwiftUI, Angular, Vue.js, React (loosely).

**The idea:** The ViewModel is an intermediate object that the View *binds* to. When you change a text field in the View, it automatically updates the ViewModel. When the ViewModel changes, the View automatically re-renders. The relationship is *two-way*.

```
[View] ←→ (two-way binding) ←→ [ViewModel]
                                   ↓ (loads from / saves to)
                                [Model]
```

**Why it exists:** Web UI frameworks offer *reactive* or *data-binding* primitives (Vue's `v-model`, SwiftUI's `@State`, React's `useState`). These let you write the View as a function of the ViewModel, and the View re-runs whenever the ViewModel changes. MVVM formalizes that relationship.

**Tradeoff:** MVVM is powerful when the UI framework's binding model works smoothly, but the two-way binding can hide implicit dependencies. You must trace not just what calls what, but also what binds to what.

### MVP (Model-View-Presenter)

**Where it appears:** Older Win32 / Java Swing applications, some iOS patterns.

**The idea:** The Presenter is the orchestrator (like a Service in Chapter 38). The View is passive: it has no logic, just methods like `displayUser(name, email)` that the Presenter calls. The Presenter loads Models, applies logic, and pushes data to the View.

```
[View] (passive, calls back on user input)
  ↑
  ↓ (Presenter calls these methods)
[Presenter] ←→ (loads / saves)
                [Model]
```

**Why it exists:** Some UI frameworks (notably older ones) offer no binding support. MVP lets you write a thin View that just receives commands from the Presenter, and a Presenter that orchestrates everything. The result is testable: you mock the View and test the Presenter in isolation.

**Tradeoff:** MVP requires explicit method calls from Presenter to View (`view->displayUser()`, `view->showError()`). As the application grows, the Presenter can become huge and tightly coupled to the View's methods.

### When to Use Which

- **MVC**: Server-rendered HTML (Rails, Laravel), or stateless HTTP APIs.
- **MVVM**: Reactive UI frameworks (Vue, React, SwiftUI) where binding is available and makes sense.
- **MVP**: Older desktop apps, or situations where the View must be testable in isolation and binding is not available.

In practice, most modern web applications are MVC servers with JavaScript MVVM frontends, each handling their own layer.

---

## 40.7 Why MVC Without Dependency Injection Becomes Untestable

Here is a tightly coupled Controller:

```cpp
class OrderController {
  Result createOrder(const Request& req) {
    // Problem 1: Controller creates the Service
    OrderService service;
    
    // Problem 2: Service creates the Repository
    // (happens inside OrderService's constructor)
    
    auto result = service.createOrder(userId, productId);
    return result;
  }
};

class OrderService {
  OrderService() {
    // Problem 3: Service creates the real Database
    orderRepo = new PostgreSQLOrderRepository(
      new DatabaseConnection("prod_host", "prod_db")
    );
  }
};
```

**To test the Controller:**
1. You must instantiate a real OrderService.
2. Which instantiates a real PostgreSQLOrderRepository.
3. Which connects to a real database.
4. You cannot test error handling; you cannot mock the database; you cannot test in isolation.

**With Dependency Injection:**

```cpp
class OrderController {
private:
  OrderService& orderService;  // Injected, not created
  
public:
  OrderController(OrderService& service) : orderService(service) {}
  
  Result createOrder(const Request& req) {
    auto result = orderService.createOrder(userId, productId);
    return result;
  }
};

// In tests:
class MockOrderService : public OrderService { /* ... */ };

OrderController controller(mockService);  // Inject the mock
auto response = controller.createOrder(request);
// Now you can test without a database
```

**The principle:** if a class creates its own dependencies, you cannot test it in isolation. Dependency Injection means the Controller receives its dependencies (the Service) from the outside, so tests can inject mocks.

See Chapter 32 for a full treatment of DI and its role in testability.

---

## 40.8 Worked Example: List and Show

Let's trace a "list books" and "show book details" pair, showing the request-response flow and where anti-patterns can hide.

### Route Definitions

```ruby
# Rails routes
get "/books", to: "books#list"
get "/books/:id", to: "books#show"
```

### Controller

```ruby
class BooksController < ApplicationController
  def initialize(book_service)
    @book_service = book_service
  end
  
  # GET /books
  def list
    # 1. Parse request (query parameters for filtering, pagination)
    filter = params[:filter]
    page = params[:page].to_i || 1
    per_page = params[:per_page].to_i || 20
    
    # 2. Validate shape (all params should be present or have defaults)
    if per_page > 100
      per_page = 100  # Prevent abuse
    end
    
    # 3. Delegate to service
    @books = @book_service.listBooks(filter, page, per_page)
    
    # 4. View renders (Rails automatically renders app/views/books/list.html.erb)
  end
  
  # GET /books/:id
  def show
    # 1. Parse request
    book_id = params[:id]
    
    # 2. Validate shape (id must be present)
    if book_id.blank?
      render json: { error: "Book ID required" }, status: 400
      return
    end
    
    # 3. Delegate to service
    @book = @book_service.getBook(book_id)
    
    # 4. Handle error from service
    if @book.nil?
      render json: { error: "Book not found" }, status: 404
      return
    end
    
    # 5. Load related data (through service, not directly)
    @similar_books = @book_service.findSimilar(@book)
    @reviews = @book_service.getReviews(book_id, limit: 5)
    
    # 6. View renders (Rails automatically renders app/views/books/show.html.erb)
  end
end
```

### Service

```ruby
class BookService
  def initialize(book_repo, review_repo, recommendation_engine)
    @book_repo = book_repo
    @review_repo = review_repo
    @recommendation_engine = recommendation_engine
  end
  
  def listBooks(filter, page, per_page)
    # Business logic: how to filter
    if filter == "bestsellers"
      @book_repo.findBestsellersPaginated(page, per_page)
    elsif filter == "new"
      @book_repo.findNewestPaginated(page, per_page)
    else
      @book_repo.findAllPaginated(page, per_page)
    end
  end
  
  def getBook(book_id)
    @book_repo.findById(book_id)
  end
  
  def findSimilar(book)
    # Orchestrate: load book metadata, run algorithm, filter results
    metadata = @book_repo.getMetadata(book.id)
    candidates = @book_repo.findByGenre(book.genre, limit: 100)
    
    # Apply business logic: sort by recommendation score
    similar = @recommendation_engine.rank(book, candidates)
    
    # Filter to top 5
    similar.take(5)
  end
  
  def getReviews(book_id, limit: 10)
    @review_repo.findByBookId(book_id, limit: limit)
  end
end
```

### Repository

```ruby
class BookRepository
  def initialize(database)
    @db = database
  end
  
  def findById(book_id)
    row = @db.query("SELECT * FROM books WHERE id = ?", [book_id])
    row.empty? ? nil : Book.new(row[0])
  end
  
  def findAllPaginated(page, per_page)
    offset = (page - 1) * per_page
    rows = @db.query(
      "SELECT * FROM books ORDER BY created_at DESC LIMIT ? OFFSET ?",
      [per_page, offset]
    )
    rows.map { |row| Book.new(row) }
  end
  
  def findBestsellersPaginated(page, per_page)
    offset = (page - 1) * per_page
    rows = @db.query(
      "SELECT b.* FROM books b JOIN sales s ON b.id = s.book_id " +
      "GROUP BY b.id ORDER BY SUM(s.quantity) DESC LIMIT ? OFFSET ?",
      [per_page, offset]
    )
    rows.map { |row| Book.new(row) }
  end
  
  def findByGenre(genre, limit: 10)
    rows = @db.query(
      "SELECT * FROM books WHERE genre = ? LIMIT ?",
      [genre, limit]
    )
    rows.map { |row| Book.new(row) }
  end
  
  def getMetadata(book_id)
    @db.query("SELECT * FROM book_metadata WHERE book_id = ?", [book_id])
  end
end

class ReviewRepository
  def initialize(database)
    @db = database
  end
  
  def findByBookId(book_id, limit: 10)
    rows = @db.query(
      "SELECT * FROM reviews WHERE book_id = ? ORDER BY created_at DESC LIMIT ?",
      [book_id, limit]
    )
    rows.map { |row| Review.new(row) }
  end
end
```

### Views

```erb
<!-- app/views/books/list.html.erb -->
<div class="books-list">
  <h1>Books</h1>
  
  <div class="filters">
    <a href="/books?filter=all">All Books</a>
    <a href="/books?filter=bestsellers">Bestsellers</a>
    <a href="/books?filter=new">New Releases</a>
  </div>
  
  <div class="books">
    <% @books.each do |book| %>
      <div class="book-card">
        <h3><a href="/books/<%= book.id %>"><%= book.title %></a></h3>
        <p>Author: <%= book.author %></p>
        <p>Price: $<%= book.price %></p>
      </div>
    <% end %>
  </div>
  
  <div class="pagination">
    <a href="/books?page=<%= current_page - 1 %>">← Previous</a>
    <span><%= current_page %></span>
    <a href="/books?page=<%= current_page + 1 %>">Next →</a>
  </div>
</div>

<!-- app/views/books/show.html.erb -->
<div class="book-detail">
  <h1><%= @book.title %></h1>
  <p class="author">by <%= @book.author %></p>
  <p class="price">$<%= @book.price %></p>
  
  <div class="description">
    <%= @book.description %>
  </div>
  
  <h2>Similar Books</h2>
  <ul class="similar">
    <% @similar_books.each do |book| %>
      <li><a href="/books/<%= book.id %>"><%= book.title %></a></li>
    <% end %>
  </ul>
  
  <h2>Reviews</h2>
  <div class="reviews">
    <% @reviews.each do |review| %>
      <div class="review">
        <p class="rating">★ <%= review.rating %>/5</p>
        <p class="text"><%= review.text %></p>
        <p class="author">— <%= review.author %></p>
      </div>
    <% end %>
  </div>
</div>
```

### What Each Layer Owns

| Layer | Responsibility |
|-------|---|
| **Controller** | Parse HTTP params (`filter`, `page`, `id`). Validate shape. Delegate to Service. Format response status. |
| **Service** | Implement filtering logic (what does "bestsellers" mean?). Load data from repositories. Orchestrate related data (get book + reviews + similar books). |
| **Repository** | Translate domain queries to SQL. Map rows to domain objects. Hide database details. |
| **View** | Render data as HTML. Apply presentation logic (page numbers, links). |

### The Anti-Pattern Version

```ruby
# WRONG: Business logic in controller
def show
  @book = Book.find(params[:id])  # Controller touches database
  
  # Business logic: calculate similar books
  similar = Book.where(genre: @book.genre).limit(5).sort_by do |b|
    (b.num_reviews - @book.num_reviews).abs
  end
  
  @similar_books = similar
end

# WRONG: Queries in view
<div>
  <h1><%= @book.title %></h1>
  
  <!-- View queries the database -->
  <% reviews = Review.where(book_id: @book.id).limit(5) %>
  <h2>Reviews (<%= reviews.count %>)</h2>
  <% reviews.each do |review| %>
    <p><%= review.text %></p>
  <% end %>
</div>
```

This version breaks because:
1. The filtering logic is in the Controller; if you build a book API, you must duplicate it.
2. The reviews query is in the View; if you add a new View (XML, API), you must duplicate it.
3. Changes to the reviews logic require hunting through templates.
4. Testing: you cannot test the filtering logic without HTTP-mocking and database-mocking.

---

## 40.9 When MVC Is the Wrong Pattern

MVC assumes the system has three distinct concerns: data, presentation, and protocol translation. But not all systems have all three.

### Single-Page Applications (SPAs): MVC Moves to the Browser

```
Traditional Web MVC:
  Browser ← [HTTP] ← Server [Controller] [Service] [Repository] [View renders HTML]

SPA (React, Vue, Angular):
  Browser [React components] [Redux store / Vuex] ← [HTTP API] ← Server [API controller, no View]
```

In an SPA, the server no longer renders Views. It is just an API that returns JSON. The frontend is an MVVM or Redux-like system running in JavaScript.

The server MVC pattern becomes just "Model + Repository," and the Controller becomes thin (just API endpoints). The interesting complexity moves to the browser.

### APIs Without UI

```
Traditional MVC assumes you have a View.
REST API with no UI:
  Client → [HTTP] → Server [Controller] → [Service] → [Repository] → database
  
  Where is the View? There is no View.
  The Controller returns JSON, but that is not a "View" in the MVC sense.
```

An API-only service does not need the View layer. The Controller takes a request and returns JSON. You still have Service and Repository (orchestration and persistence), but the MVC triangle is broken. This is just "API + Service + Repository."

### CLI Tools

```
CLI app:
  User types a command → [argument parser / command dispatcher] → [Service] → [Repository] → database
  
  Where is the View? The "output" is just text to stdout.
  Where is the Model? There is data, but often no persistent storage.
```

A CLI tool is typically just a dispatcher and some business logic. It may not have a Service layer at all; the dispatcher directly calls functions. Forcing MVC here adds needless complexity.

---

## 40.10 Tradeoffs

| Variant | Pros | Cons | When to Use |
|---------|------|------|-------------|
| **Smalltalk MVC (observer-based)** | Multiple Views stay in sync automatically. Model is fully independent. Easy to add new Views. | Requires observer infrastructure. Control flow is implicit. Broadcast storms if Model changes too often. | Desktop apps with persistent Models and multiple Views (rarely in modern web). |
| **Web MVC (request-response)** | Clean separation of concerns. Each layer is testable in isolation. Stateless; scales horizontally. Standard in most frameworks. | Requires HTTP infrastructure. View is re-rendered per request (performance cost). Harder to do real-time updates. | Server-rendered web apps. Traditional web services. |
| **MVVM (data binding)** | Reactive. Changes propagate automatically. Clean declarative View logic. | Implicit dependencies (binding can hide what updates what). Two-way binding can be surprising. | SPAs with reactive frameworks (Vue, React, SwiftUI). |
| **MVP (presenter orchestration)** | Testable in isolation. Clear control flow. Works with non-reactive UI frameworks. | Presenter can grow large. Explicit View methods (`displayUser`, `showError`) are verbose. | Older desktop apps. Testing-first development with mock Views. |
| **No MV* (small scripts, CLIs)** | No abstraction overhead. Simple and direct. | Poor separation of concerns. Hard to change. Not scalable. | Throwaway scripts, simple utilities, one-off tools. |

---

## 40.11 Common Misconceptions

| Misconception | Reality |
|---|---|
| "MVC means three folders." | MVC is about responsibility separation, not file organization. You might have one folder per layer, or they might be interleaved. What matters is the dependencies. |
| "The Model is just the database table." | The Model is the domain entity and its rules. The database is the Repository's concern. |
| "Views are optional; an API is just MVC without the V." | An API is a different pattern. You have Controller (HTTP → JSON) and Repository (JSON ← database), but no View in the MVC sense. The M (Model/domain object) may not exist explicitly. |
| "Smalltalk MVC is the same as web MVC." | They are opposite. Smalltalk's View observes the Model; web's View does not. Smalltalk's Controller dispatches input; web's Controller orchestrates. Know which one you have. |
| "DI is optional; you can test without it." | You can test the system as a whole (integration tests) without DI. You cannot test the Controller in isolation without DI. |
| "Business logic should be in the Model, not the Service." | Depends on the system. In a rich domain model (DDD), the Model owns rules. In an anemic model (CRUD-heavy), the Service owns orchestration. Neither is wrong; know which you have. |

---

## 40.12 Exercises

1. **Identify the pattern.** Take a web framework you know (Rails, Laravel, Spring, ASP.NET, Django). Write a simple CRUD operation (create a Post, show it, edit it, delete it). Now trace the data flow: where does the HTTP request go? Which layer transforms it? Where does the View template get called? Which pattern (MVC, MVVM, MVP) is it?

2. **Refactor an anti-pattern.** Find code in a real codebase where SQL is in a template, or business logic is in a Controller, or the Model does orchestration. Write the corrected version with business logic in the Service, data access in the Repository, and the Controller thin.

3. **Test a tightly coupled Controller.** Take a Controller that creates its own dependencies (new Service(), new Repository()). Write unit tests for it; note what mocking is required. Now inject the dependencies and rewrite the tests. Compare: how much simpler is the second version?

4. **Compare to Observer.** Write a small GUI application (in Python with Tkinter, or C++ with Qt, or JavaScript) using Smalltalk-style MVC: the Model broadcasts, the View observes. Compare the code to a request-response web example. What changed? What got simpler?

5. **Design an API.** You have a REST API returning JSON. There is no server-rendered View. Is this still MVC? What layers do you need? Sketch the architecture: what does the Controller do? What does the Service do? Where is the Model?

6. **Multi-View Consistency.** Imagine an application where multiple parts of the UI (a sidebar, a main panel, a status bar) all show different aspects of the same data. How would you implement this in (a) Smalltalk MVC, (b) a web MVC with AJAX, (c) a Vue.js component system? What are the tradeoffs?

---

## 40.13 Summary

MVC is not one pattern; it is two, with the same name. **Smalltalk MVC** (1979) solved the problem of keeping multiple Views in sync with a Model by making the View an observer. **Web MVC** (Rails, Laravel, etc.) solved a different problem: translating HTTP requests to responses by separating protocol (Controller), orchestration (Service), persistence (Repository), and rendering (View).

Know which one you have. In modern web development, you almost always have the second. The key insight is that each layer has a clear boundary and responsibility: the Controller translates protocol, the Service orchestrates, the Repository hides storage, and the View renders. Violating those boundaries — SQL in templates, business logic in Controllers, orchestration in Models — creates code that is hard to test, hard to reuse, and hard to change.

Combine MVC with Dependency Injection (Chapter 32), and you get testability. Combine it with Services and Repositories (Chapter 38), and you get scalability. Recognize when you don't have all three layers (SPAs have MVC in the browser, APIs have no View, CLIs have no Model), and don't force the pattern when it doesn't fit.

---

> **[← Previous: Chapter 39 — Why Models Exist](07-why-models-exist.md)** · **[↑ Part 4](README.md)** · **[Next: Chapter 41 — Domain-Driven Thinking →](09-domain-driven-thinking.md)**
