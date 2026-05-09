# Chapter 13 — Architectural Decision Records (and Part 4 Synthesis)

This chapter does two things. First, it introduces you to the lightweight habit that holds long-lived architectures together — the ADR, a simple document format that captures why you made a decision and what it costs. Then it walks through one architectural choice across every concept in Part 4, showing you how all these ideas are not independent tools but answers to the same question: how should this system be organized, and what are you trading off to organize it that way?

By the end of this chapter you will be able to:

1. Write a well-formed Architecture Decision Record and recognize one that is clear vs vague.
2. Judge when to write an ADR (expensive-to-change decisions) and when to skip it (implementation details).
3. Apply every tool from Part 4 — layers, services, repositories, models, dependency injection, web frameworks, file organization — to a real system, showing how they fit together.
4. Recognize where architecture cannot help you and where you must rely on discipline, domain understanding, and honest tradeoff analysis.

---

## 13.1 Why Decisions Get Lost

Pick a codebase that has been alive for three years. Ask the engineers why it is organized the way it is. You will get answers like:

- "That service used to be on the same machine, but someone split it out."
- "I think we use PostgreSQL because that's what the first architect wanted, but I'm not sure."
- "The dependency injection container seemed like a good idea at the time."
- "I don't know why we have both a repository and a service layer. The tests just pass through them."

This is not stupidity or carelessness. It is entropy. Without a record, the *why* of an architecture evaporates within 6 to 12 months. New engineers cannot tell which choices are load-bearing and which are vestigial. People make sense of the system by reading code, which tells them *what* is there but not *why* it is there. They guess. They refactor based on guesses. They undo decisions that were load-bearing, and the cost compounds.

The cheaper-than-you-think cure is a single habit: when you make a significant architectural decision, write it down. Not in Confluence or Notion (those rot behind paywalls and vanish when someone leaves). In a markdown file, in the repo, linked from PRs and pull requests, immutable and searchable.

---

## 13.2 The ADR Format

The Architecture Decision Record format comes from Michael Nygard's 2011 proposal. It is simple enough to create no overhead, expressive enough to capture what matters.

Each ADR has this shape:

```
# ADR-NNN: Title

## Status
Proposed / Accepted / Superseded by ADR-MMM / Deprecated

## Context
Why does this decision matter?
What is the problem or constraint we are facing?
What assumptions are we making?

## Decision
What choice are we making?
State it as a declaration: "We will use...", "We will not...", "We choose to..."

## Rationale
What are the alternatives?
Why is this better than the alternatives?
What tradeoffs are we making (quality attributes)?
What are we optimizing for, and what are we accepting as cost?

## Consequences
What becomes easier?
What becomes harder?
What will we need to revisit or monitor?
What future decisions does this constrain?
```

The length is one page, rarely two. You are not writing a design document. You are capturing the decision, the context that made it necessary, and the tradeoff. That is all.

Here is a concrete example:

```
# ADR-007: Synchronous REST API, no async, for the first year

## Status
Accepted

## Context
We are building a lending platform with 6 engineers.
We have 50,000 users, growing 5% per month.
Most API calls (load application, get loan status, pay bill) complete in under 200ms.
The team knows Django well; we have an operational Ruby on Rails codebase that uses synchronous handlers.
Our cloud provider charges by compute-hours, not request volume.

## Decision
We will implement the API as synchronous request/response handlers.
No async, no background workers, no message queues.
All business logic runs in the HTTP request thread.

## Rationale
Simplicity is our highest-priority quality attribute at this stage.
With 50,000 users and 5% monthly growth, we will not hit synchronous bottlenecks for at least 12 months.
(50K * 1.05^12 = ~90K users, assuming even distribution, that is still well under the 1000 req/sec that one machine handles.)

Asynchronous handlers (async/await, RQ, Celery) add:
- Infrastructure (message queue, monitoring)
- Conceptual overhead (distributed tracing, delayed failure modes, retries)
- Testing complexity (no longer straightforward)
- Team overhead (context switching, debugging is harder)

For a 6-person team, that overhead dominates the benefit.
We chose to optimize for time-to-market and team velocity.

We considered a hybrid (async for slow operations), but rejected it because:
- It adds just enough complexity to be dangerous, not enough to solve the problem.
- It commits us to debugging async failures without hitting the threshold where async matters.
- Our slow operations are rare (bulk report generation), not on the critical path.

## Consequences
We will complete feature work 20% faster in the short term (no infrastructure setup).
Code review will be simpler (no async context to understand).
Performance troubleshooting will start easy (flame graphs, not distributed tracing).

We are betting on growth staying predictable.
If we hit 500 req/sec to one endpoint, we will exceed capacity, and the remedy is vertical scaling (bigger machine) or caching, not async.

In ~12 months, we should revisit this ADR.
Indicators to watch: p99 latency trending up, single-endpoint request rates, queue times on any operation.
If we see those trends, we write ADR-NNN to supersede this one.
```

This is an ADR. Notice what it does and what it does not:

- **It does**: captures the problem, the choice, alternatives, tradeoff, and consequences.
- **It does not**: describe the implementation in detail. "How do we build the API?" is not here. "What endpoints exist?" is not here. This is decision, not design.
- **It does not**: nitpick. The status is either Accepted, Proposed (under discussion), Superseded (replaced by a later ADR), or Deprecated (we tried it, we don't recommend it). You don't track "blocked" or "pending"; either decide it or don't.

---

## 13.3 When to Write One

Not every decision deserves an ADR. You don't write ADRs for:

- **Implementation details**: "Should this class use `std::vector` or `std::deque`?" Answer: measure it, refactor if it matters. No ADR.
- **Everyday design choices**: "Should this method be public or private?" No ADR.
- **Bug fixes**: "We discovered X was broken. We fixed it." No ADR.
- **Refactoring**: "We reorganized the handlers into smaller files." No ADR.

You **do** write ADRs for decisions that are expensive to change (Chapter 33 again):

- **Database choice**: PostgreSQL vs MongoDB vs DynamoDB. Expensive to reverse.
- **Service boundaries**: "Should the notification system be in-process or a separate service?" Expensive to reverse.
- **Synchronous vs asynchronous**: As above. Affects every handler in the system.
- **Shared data model**: "Will services share a database or have their own?" Expensive to reverse.
- **Framework choice**: Django vs FastAPI, Rails vs Sinatra. Some cost to change.
- **Authentication and authorization model**: Token-based vs session-based, centralized vs distributed. Expensive to reverse.
- **Caching strategy**: Local, Redis, CDN, or none. Affects concurrency, consistency, testing.
- **Deployment topology**: Monolith vs microservices. Very expensive to reverse.

The test: **if reversing the decision after deployment would cost more than a few engineering-weeks, write an ADR.**

---

## 13.4 Where ADRs Live

In the repo, under `/docs/adr/`, or `/architecture/decisions/`, or similar. Numbered sequentially: `ADR-001.md`, `ADR-002.md`, etc. Never update an existing ADR. If the decision changes, write a new ADR and reference the old one.

```
# ADR-018: Add async handlers for batch operations (supersedes ADR-007)

## Status
Accepted

## Context
(Revisit triggers from ADR-007 have materialized: p99 latency trending up, batch operation is now on critical path.)

...
```

Link ADRs from pull requests that implement them:

```
Implements ADR-007: Synchronous REST API.
This PR adds the initial handler structure, no async framework,
all business logic in request threads.
```

Link ADRs from README and architecture overviews. New engineers should be able to find them easily.

If you have 15+ ADRs, consider a summary table in an `ADR-index.md`:

| ADR | Title | Status | Date | Impact |
|---|---|---|---|---|
| 001 | PostgreSQL for primary store | Accepted | 2023-01 | High |
| 007 | Sync REST, no async | Superseded by 018 | 2023-02 | High |
| 018 | Async for batch ops | Accepted | 2025-01 | Medium |

---

## 13.5 A Real ADR: Bookshelf Service (Full Example)

Let me write out one complete ADR to show the format in practice. This is a lending library service:

```
# ADR-009: Three-layer architecture with repositories and dependency injection

## Status
Accepted

## Context
The Bookshelf service manages a library's catalog and checkout history.
Core entities: Book, Member, Loan, Hold, Return.
We anticipate 100,000 books, 50,000 active members.
The service must be testable (unit tests, no real database).
It will integrate with a billing system (notify when a book is overdue).
The team has two senior engineers and two junior engineers.
We are building this from scratch with a clean slate.

## Decision
We will organize the code in three layers:
- **HTTP/Web layer** (controllers): accept requests, parse, call services, return JSON.
- **Domain/Business logic layer** (services): orchestrate business rules, call repositories.
- **Persistence layer** (repositories): abstract database access.

We will use a dependency injection container to wire components.
We will inject repositories into services and services into controllers.
No singletons. No global state.

## Rationale
Three layers are the minimum stable configuration for a service like this.
They are not arbitrary; they fall out of the problem:

1. **HTTP is noise**: the transport mechanism should not leak into business logic.
   By separating controllers from services, we can test "loan a book" without spinning up a web server.

2. **Persistence is a detail**: we might start with PostgreSQL, later move to a different database.
   Repositories let us swap the database without touching business logic.

3. **Business logic is fragile**: loans, holds, returns have rules (can you return a book you don't have? can you hold a book that's checked out to you?).
   These rules live in one place (services and models), not scattered across HTTP handlers.

4. **Testing at layers**: we can test business logic without touching the database (mock repositories).
   We can test the database with integration tests (real database, real repositories).
   We can test HTTP handlers with handlers tests (mock services).

We considered two alternatives and rejected them:

**Alternative 1: Layerless, handler-per-endpoint**
Each HTTP handler calls the database directly, implements business logic inline.
This works for tiny services (< 5 endpoints) and fails spectacularly at 20 endpoints.
Business logic is tangled with HTTP parsing. Testing requires a real database. Changes ripple everywhere.
Rejected: we are not tiny, and we anticipate growth.

**Alternative 2: Single service layer, no repositories**
Services call the database directly via an ORM.
This saves the repositories-as-abstraction layer.
But it makes testing harder (every service test needs a database) and couples business logic to the database schema.
Rejected: we want to test without a database.

We are trading some boilerplate (repository interfaces, DI configuration) for testability and decoupling.

## Consequences
**Easier**:
- Testing business logic in isolation (mock repositories).
- Changing database technology (new repository implementation, no service changes).
- Onboarding junior engineers (clear separation of concerns).
- Debugging (a bug in business logic is in one place).

**Harder**:
- Small services feel over-engineered (repository for 5 lines of SQL feels redundant).
- Performance: an extra layer of indirection (service calls repository calls ORM calls database).
- Code navigation: to understand a request, you jump between three files.

**To monitor**:
- If 80% of repositories are trivial (one-line wrapper around ORM), we may have over-abstracted. At that point, consider dropping repositories for a simpler model.
- If we add a second language or a microservice, the boundaries may need rethinking.
- If we hit performance problems, profile before optimizing; the layer may not be the bottleneck.

## See also
- ADR-010: Dependency Injection with Autofac
- ADR-011: PostgreSQL as primary store
- ADR-012: Model-rich domain objects, not anemic data transfer objects
```

This is a real ADR. It is explicit about the tradeoff (boilerplate for testability), honest about what gets harder, and clear about when to revisit.

---

## 13.6 Part 4 Synthesis: The Bookshelf Service Walk-Through

Now let's see every concept from Part 4 in one architecture. I'll trace the request "Loan a book to a member" through every layer and every concept.

### The Request and the HTTP Layer

A member hits `POST /loans` with:

```json
{
  "member_id": 42,
  "book_id": 1001,
  "duration_days": 14
}
```

This arrives at a **controller** (Chapter 40, Web MVC):

```cpp
class LoanController {
  LoanService* loan_service_;  // injected
  MemberService* members_;     // injected

public:
  LoanController(LoanService* svc, MemberService* m) 
    : loan_service_(svc), members_(m) {}

  Response CreateLoan(const Request& req) {
    int member_id = req.GetInt("member_id");
    int book_id = req.GetInt("book_id");
    int days = req.GetInt("duration_days", 14);  // default

    // Validation is light here: just parsing. Domain validation happens below.
    if (member_id <= 0 || book_id <= 0 || days <= 0) {
      return Response::BadRequest("Invalid IDs or duration");
    }

    // Delegate to business logic.
    try {
      Loan loan = loan_service_->CreateLoan(member_id, book_id, days);
      return Response::Created(JsonEncode(loan));
    } catch (const DomainException& e) {
      return Response::BadRequest(e.what());
    }
  }
};
```

The controller is thin. It parses the request, calls a service, catches errors, returns JSON. This is the **web layer** (Chapter 40) — HTTP is handled here and nowhere else.

### The Service Layer and Domain Language

The request goes to `LoanService::CreateLoan`:

```cpp
class LoanService {
  LoanRepository* loans_;        // injected
  BookRepository* books_;        // injected
  MemberRepository* members_;    // injected
  HoldService* holds_;           // injected
  NotificationService* notify_;  // injected

public:
  Loan CreateLoan(int member_id, int book_id, int duration_days) {
    // Load the book and member.
    Book book = books_->FindByIdOrThrow(book_id);
    Member member = members_->FindByIdOrThrow(member_id);

    // Check business rules.
    if (!member.can_borrow()) {
      throw DomainException("Member has outstanding fines; cannot borrow");
    }
    if (book.IsBorrowed()) {
      throw DomainException("Book is already borrowed");
    }

    // Create the loan using a domain model (Chapter 39).
    Loan loan = Loan::Create(member, book, duration_days);

    // Persist it (Chapter 38).
    loans_->Save(loan);

    // Notify the billing system (Chapter 43 — boundary crossing).
    notify_->SendLoanStarted(loan);

    return loan;
  }
};
```

Notice the **domain language** (Chapter 41): `member.can_borrow()`, `book.IsBorrowed()`, `Loan::Create()`. The code reads like the business domain, not like a database query. An accountant could read this and verify it is correct.

This is the **service layer** (Chapter 38) — orchestration of domain logic, repositories, and external services.

### The Domain Model

The domain rules live in the `Loan` class (Chapter 39 — models with rules in them):

```cpp
class Loan {
  int id_;
  Member member_;
  Book book_;
  DateTime due_date_;
  DateTime returned_date_;  // null until returned

public:
  // Factory method: the only way to create a valid loan.
  static Loan Create(const Member& m, const Book& b, int duration_days) {
    // Rule: check limits before creating.
    if (m.active_loan_count() >= m.max_loans()) {
      throw DomainException("Member has reached maximum loans");
    }
    if (b.hold_queue().empty() == false && b.hold_queue().front() != m.id()) {
      throw DomainException("Book has holds; member is not first in queue");
    }

    // Rule: return date is today + duration.
    DateTime due = DateTime::Now().AddDays(duration_days);

    Loan loan{/*...*/};
    loan.id_ = 0;  // not yet persisted
    loan.member_ = m;
    loan.book_ = b;
    loan.due_date_ = due;
    loan.returned_date_ = DateTime::Null();

    return loan;
  }

  bool IsOverdue() const {
    if (returned_date_.IsNull()) {
      return DateTime::Now() > due_date_;
    }
    return false;
  }

  void Return() {
    if (!returned_date_.IsNull()) {
      throw DomainException("Loan already returned");
    }
    returned_date_ = DateTime::Now();
  }
};
```

The model is **rich**: it has rules, not just data. Rules for creating a loan, checking overdue status, returning. A junior engineer reading this class understands the domain. That is the goal.

### The Repository Layer

The repository abstracts persistence (Chapter 38 — the repository pattern):

```cpp
class LoanRepository {
public:
  virtual ~LoanRepository() = default;

  // Pure interface, no database details here.
  virtual void Save(const Loan& loan) = 0;
  virtual Loan FindById(int id) = 0;
  virtual std::vector<Loan> FindByMemberId(int member_id) = 0;
  virtual std::vector<Loan> FindOverdueLoans() = 0;
};

// Concrete implementation: PostgreSQL.
class PostgresLoanRepository : public LoanRepository {
  Database* db_;

public:
  PostgresLoanRepository(Database* db) : db_(db) {}

  void Save(const Loan& loan) override {
    // SQL here, no business logic.
    db_->Execute(
      "INSERT INTO loans (member_id, book_id, due_date) VALUES (?, ?, ?)",
      loan.member_id(), loan.book_id(), loan.due_date()
    );
  }

  Loan FindById(int id) override {
    // Query, reconstruct domain object.
    auto row = db_->QueryOne("SELECT * FROM loans WHERE id = ?", id);
    return Loan{row["id"], row["member_id"], row["book_id"], row["due_date"]};
  }
};
```

The repository is a **boundary** (Chapter 35 — interface abstraction): it hides SQL from the service layer. If we later switch to MongoDB, we write a new repository. Services don't change.

This is why repositories matter: they are the **coupling point** between the domain (timeless, important) and the database (an implementation detail, easy to change).

### Separation of Concerns and Layering

Notice how each layer has one job:

- **Controller**: parse HTTP, call service, format response. No business logic, no database.
- **Service**: orchestrate domain logic and repositories. No HTTP, no SQL.
- **Model**: encode domain rules. No HTTP, no SQL, no data transfer.
- **Repository**: query database, reconstruct models. No business logic, no HTTP.

This is **separation of concerns** (Chapter 36) in action. A bug in validation lives in one place. A change to the database schema affects only repositories. A business rule change affects only models and services.

### Dependency Injection

How do these layers find each other? Not by instantiating. By injection (Chapter 42):

```cpp
int main() {
  Database db("postgres://...");
  
  // Create repositories.
  auto loans = std::make_unique<PostgresLoanRepository>(&db);
  auto books = std::make_unique<PostgresBookRepository>(&db);
  auto members = std::make_unique<PostgresMemberRepository>(&db);

  // Create services.
  auto hold_svc = std::make_unique<HoldService>(loans.get(), books.get());
  auto notify_svc = std::make_unique<NotificationService>(/* ... */);
  auto loan_svc = std::make_unique<LoanService>(
    loans.get(), books.get(), members.get(), hold_svc.get(), notify_svc.get()
  );

  // Create controller.
  auto controller = std::make_unique<LoanController>(loan_svc.get(), members.get());

  // Wire to HTTP framework.
  app.Post("/loans", [&](const Request& req) {
    return controller->CreateLoan(req);
  });

  app.Run();
}
```

No global state, no service locator, no magic. Each component lists what it needs (its constructor parameters). The `main` function wires them together.

### Complexity Bounded by Feature Flags

If the business later says "we want a 30-day loan promotion in March," we add a **feature flag** (Chapter 34):

```cpp
Loan LoanService::CreateLoan(int member_id, int book_id, int duration_days) {
  // ...
  
  // Rule: apply promotion if active.
  if (features_.IsEnabled("extended_march_loans") && DateTime::Now().month() == 3) {
    duration_days = std::max(duration_days, 30);  // minimum 30 days
  }

  Loan loan = Loan::Create(member, book, duration_days);
  // ...
}
```

The feature flag is toggled at runtime without recompiling. Complexity is **bounded**: the flag is localized, tested separately, can be disabled if it causes problems.

### Organizing the Files

Where do these files live? By feature (Chapter 44 — feature-first folders):

```
src/
  loans/
    loan.hpp              # Domain model
    loan.cpp
    loan_repository.hpp   # Repository interface
    postgres_loan_repo.hpp
    postgres_loan_repo.cpp
    loan_service.hpp      # Service
    loan_service.cpp
    loan_controller.hpp   # Controller
    loan_controller.cpp
  books/
    book.hpp
    # ... same pattern
  members/
    member.hpp
    # ... same pattern
  shared/
    domain_exception.hpp   # Shared across features
```

Each feature (loans, books, members) has all its layers stacked together. A junior engineer opening `src/loans/` understands everything they need to understand about loaning books.

### Handling Concerns that Cross Features

The notification system is a **boundary crossing** (Chapter 43). When a loan is created, billing must be notified. The service calls a notification service:

```cpp
notify_->SendLoanStarted(loan);
```

This could be:

1. **In-process** (Chapter 43 — internal calls): the notification service runs in the same process, same transaction. If it fails, the loan creation fails.

2. **Asynchronous queue** (Chapter 43 — event publishing): the loan is saved, then an event "LoanStarted" is published to a message queue. A separate worker processes it. If the worker fails, the loan still exists; billing catches up later.

The choice depends on the tradeoff (ADR territory):

- In-process is simple, synchronous, guaranteed.
- Queue-based is more robust (notification failure doesn't fail the loan), but more complex (you need a queue, a worker, monitoring).

For now, we chose in-process. As we grow, we might write ADR-015 to supersede this choice.

### Testing at Each Layer

This architecture enables testing at multiple scales:

```cpp
// Test 1: Domain model (no dependencies).
TEST(LoanTest, CannotCreateLoanIfMemberAtLimit) {
  Member m = Member::Create("Alice");
  m.set_max_loans(2);
  m.add_loan(Loan::Create(m, book1, 14));
  m.add_loan(Loan::Create(m, book2, 14));

  EXPECT_THROW(
    Loan::Create(m, book3, 14),
    DomainException
  );
}

// Test 2: Service logic (mock repositories).
TEST(LoanServiceTest, CreatesLoanIfMemberCanBorrow) {
  MockLoanRepository loans;
  MockBookRepository books;
  MockMemberRepository members;
  MockNotificationService notify;

  books.SetFindByIdResult(book1);
  members.SetFindByIdResult(alice);

  LoanService svc(&loans, &books, &members, nullptr, &notify);
  Loan result = svc.CreateLoan(alice.id(), book1.id(), 14);

  EXPECT_EQ(result.due_date(), DateTime::Now().AddDays(14));
  EXPECT_TRUE(loans.SaveWasCalled());
}

// Test 3: Controller (mock service).
TEST(LoanControllerTest, ParsesRequestAndCallsService) {
  MockLoanService svc;
  svc.SetCreateLoanResult(expected_loan);

  LoanController controller(&svc, nullptr);
  Response resp = controller.CreateLoan(request);

  EXPECT_EQ(resp.status(), 201);
  EXPECT_TRUE(JsonDecode(resp.body()) == expected_loan);
}
```

Each test is fast (no database), focused (tests one layer), and independent of others.

---

## 13.7 What Architecture Cannot Do

The Bookshelf walk-through shows what good architecture *can* do: clarity, decoupling, testability, and the ability to evolve without breaking the world.

But architecture has limits. Understand them:

1. **Architecture cannot fix bad domain understanding.** If you don't know what a "hold" means in a library system, no amount of layering will help. You will encode the wrong rules, cleanly, but wrong.

2. **Architecture cannot replace honest tradeoff analysis.** A three-layer architecture assumes the cost of extra indirection is worth the benefit of testability. If your service is truly trivial (a thin ORM wrapper), you are over-engineered. Layers must fit the problem.

3. **Architecture cannot substitute for communication.** If your team doesn't talk about why you made these choices, they will undo them. "Why do we have repositories?" "I don't know." Six months later, someone refactors them away. ADRs help, but they are not magic.

4. **Architecture cannot prevent sloppy work.** If engineers treat repositories as optional (sometimes call them, sometimes call ORM directly), the abstraction is broken. Discipline and code review are not negotiable.

5. **Architecture cannot scale infinitely.** A three-layer monolith works up to a point. Beyond that, you need to think about services, caches, databases, and scaling. That is a later decision (Part 5 and beyond).

---

## 13.8 Part 4 in Retrospect

Part 4 began with a simple question: **what is software architecture?** The answer was: decisions that are expensive to change.

From there, we built up:

- **Chapter 33**: Architecture as the set of high-cost decisions. The Cost-of-Change Test as your guide.
- **Chapter 34**: Why large programs become complex, and what structure helps — the observation that every architecture has a "shape" that emerges from the problem.
- **Chapter 35**: How abstraction creates boundaries. Interfaces are the places where you can swap implementations.
- **Chapter 36**: Separation of concerns — each module has one reason to change.
- **Chapter 37**: Layered architectures — the most common shape, the most stable.
- **Chapter 38**: Repositories and services — the standard separation within layers.
- **Chapter 39**: Models that encode rules, not just data. The domain model as the heart of the system.
- **Chapter 40**: Web frameworks and MVC. How HTTP is handled at the boundary.
- **Chapter 41**: Domain language — code that reads like the business domain.
- **Chapter 42**: Dependency injection — how to wire components without coupling them.
- **Chapter 43**: Boundary crossings — how systems communicate with the outside world.
- **Chapter 44**: Organizing code by feature, not by layer.
- **Chapter 13** (this one): ADRs as the record of architectural decisions, and a synthesis showing all of Part 4 applied to one system.

What connects all these is the core insight: **architecture is decisions, and decisions have tradeoffs.** Every choice you make optimizes for some quality attribute at the cost of another. The skill is recognizing the tradeoff, making it consciously, and writing it down.

---

## 13.9 What Comes Next

Part 4 is about how code is organized *in space* — modules, layers, services, boundaries.

Part 5, Runtime & Concurrency, is about how code runs *in time* — threads, async, schedulers, locks, atomics, data races.

The architecture you choose has consequences for the runtime. A three-layer monolith is simpler to reason about synchronously. A service-oriented system forces you to think about distribution, timeouts, and eventual consistency. A single-threaded event loop service is easier to reason about than a multi-threaded one, but single-threaded has its own constraints.

By the end of Part 5, you will understand how architecture and runtime are two views of the same system: architecture asks "what is separated?", runtime asks "how does it run?". They are inseparable.

---

## 13.10 Exercises

1. **Write an ADR for your codebase.** Pick a significant architectural decision in a codebase you maintain or know well. Write a real ADR for it: context, decision, rationale (including alternatives rejected), and consequences. If the decision is no longer current, mark it as "Superseded" and write a follow-up ADR explaining the change.

2. **ADR audit.** Read your codebase's architecture. If it has ADRs, read them. If not, identify the top three architectural decisions (database choice, service boundaries, sync vs async). For each, write an ADR even if retroactively. This clarifies what was decided and why.

3. **Trace a request.** Pick a real request in your system (a form submission, an API call, a job). Trace it through every layer — HTTP handler, service, repository, database, and back. Identify where separation of concerns is working and where it is leaking. Are there layers that are doing more than one job?

4. **Identify the tradeoff.** Look at the architecture in your codebase. For each major decision (layering, services, dependency injection), list the quality attribute it optimizes for and the ones it sacrifices. Is the tradeoff still justified? Would you make the same choice today?

5. **Design a different service.** Pick a service you know (a booking system, a chat app, an e-commerce checkout). Design its architecture from scratch:
   - What are the layers?
   - What are the repositories?
   - What is the domain language?
   - How are components wired?
   - Where are the boundaries?
   
   Write the decision as an ADR.

6. **Supersede an ADR.** If your codebase has an ADR that is no longer current, write a new ADR that supersedes it. Explain what changed (scaling, team size, business requirements, technology) and why the old decision no longer fits.

---

## 13.11 Summary

Architectural Decision Records are a lightweight habit that preserves the reasoning behind high-cost decisions. Write them for decisions that are expensive to reverse. Keep them in the repo, numbered, immutable, and linked from PRs. They cost almost nothing and save enormous time when new engineers ask "why is it like this?"

The Bookshelf service walk-through showed how every concept from Part 4 works in concert: layers separate HTTP from logic. Repositories separate logic from persistence. Models encode rules. Dependency injection wires everything without coupling. Feature flags bound complexity. Feature-first folders organize the code. And ADRs record why these choices were made.

Architecture is not a diagram or a pattern. It is thinking about how the system is organized and why. This thinking scales from individual engineers making local decisions to teams designing service boundaries. The discipline is the same: understand the tradeoff, make it consciously, write it down.

---

## 13.12 What's Next

**[← Previous: Organizing Large Codebases](12-organizing-large-codebases.md)** · **[↑ Part 4](README.md)** · **[Next: Part 5 — Runtime & Concurrency](../part-05-runtime-concurrency/README.md)** *(coming soon)*

