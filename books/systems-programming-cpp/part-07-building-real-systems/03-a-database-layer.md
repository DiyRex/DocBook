# Chapter 70 — A Database Layer

A database layer is the boundary between two worlds: your domain model and the row/column world of SQL. Done well, neither side knows about the other. The service has no idea whether data lives in PostgreSQL, SQLite, or a file on disk. The database has no idea that a row represents a `Book` object with behavior and rules. Done poorly, both leak everywhere: services write SQL, repositories apply business logic, and a simple schema change cascades through fifteen files.

This chapter teaches you to build that boundary correctly.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Understand connection pools, their lifecycle, and the deadlock patterns they create.
2. Explain why repositories return domain objects, not rows.
3. Map rows to domain objects without falling into the N+1 trap or hiding the cost of loading.
4. Write transactions that preserve invariants across multiple writes.
5. Design and version database schemas as code, not as manual operations.
6. Build parameterized queries that are safe from injection.
7. Cache data in front of the database without creating stale-data bugs.
8. Implement a concrete `BookRepository` using a real database library.

---

## 70.1 Connection Pools

A database is a shared resource. Creating a new TCP connection and running the handshake for every query is prohibitively expensive. Most applications maintain a pool of open connections, reusing them across requests.

### How a Pool Works

A connection pool is simple in theory: maintain a set of open connections; when a request comes in, grab an unused one; when the request finishes, return it to the pool.

```cpp
class ConnectionPool {
private:
    std::queue<std::shared_ptr<DatabaseConnection>> available_;
    std::vector<std::shared_ptr<DatabaseConnection>> all_;
    std::mutex mu_;
    int pool_size_;
    std::string connection_string_;
    
public:
    ConnectionPool(int size, const std::string& connstr) 
        : pool_size_(size), connection_string_(connstr) {
        for (int i = 0; i < size; ++i) {
            auto conn = std::make_shared<DatabaseConnection>(connstr);
            all_.push_back(conn);
            available_.push(conn);
        }
    }
    
    std::shared_ptr<DatabaseConnection> acquire() {
        std::unique_lock<std::mutex> lock(mu_);
        while (available_.empty()) {
            // PROBLEM: if all connections are in use and we need another,
            // we block. If the thread holding the connection is also waiting
            // for something else, we deadlock.
        }
        auto conn = available_.front();
        available_.pop();
        return conn;
    }
    
    void release(std::shared_ptr<DatabaseConnection> conn) {
        std::unique_lock<std::mutex> lock(mu_);
        available_.push(conn);
    }
};
```

### Pool Size and Deadlock

The single most common bug: **pool size is smaller than the thread pool size**.

Imagine you have:
- 10 worker threads handling HTTP requests.
- Database connection pool of size 5.

Scenario:
1. Threads A, B, C, D, E each grab a connection. All 5 are in use.
2. Threads F, G, H, I, J arrive and need a connection.
3. They block in `acquire()`, waiting for a connection to be released.
4. But none of the threads holding connections are waiting; they are running queries or waiting for other I/O.
5. Suddenly, thread A's query takes an unexpectedly long time. All 10 worker threads are now blocked.

This is a deadlock: threads are waiting for a resource, but the resource holder is not making progress because it is waiting for something else.

**Rule: Pool size should be at least as large as your thread pool size.**

In practice, a good heuristic is to make it slightly larger (e.g., thread pool size + 2 or 3). This allows for brief contention when a request is finishing but has not yet released its connection.

Why does this matter? Because:

1. **You cannot predict query times.** An author query that normally takes 10ms might take 500ms if the database disk is busy or the network is congested.

2. **External services block threads.** If a request calls a payment API and waits for a response, it holds the connection the whole time. If the payment API is slow, many threads pile up.

3. **Lock contention.** If two transactions try to update the same row, one will block while waiting for the other. Its connection is held but idle.

Set your pool size conservatively. Monitor thread pool saturation in production. If threads are frequently waiting for a connection, increase the pool size or reduce thread count.

### Connection Lifecycle

Connections are not infinite. A few events that matter:

1. **Idle timeout**: If a connection has not been used for N seconds, close it. Databases often close their end after even longer periods, so the connection becomes half-dead (writes fail, reads might hang).

2. **Max age**: Connections can accumulate state. Some database drivers or proxies (like PgBouncer) reset the connection after N hours. You should refresh old connections.

3. **Validation on reuse**: Before handing out a connection from the pool, test it with a simple query like `SELECT 1;` to ensure it is still alive.

```cpp
class ConnectionPool {
private:
    struct PooledConnection {
        std::shared_ptr<DatabaseConnection> conn;
        std::chrono::system_clock::time_point acquired_at;
        std::chrono::system_clock::time_point last_used;
    };
    
    std::queue<PooledConnection> available_;
    std::mutex mu_;
    std::chrono::seconds idle_timeout_{300};
    std::chrono::hours max_age_{24};
    
public:
    std::shared_ptr<DatabaseConnection> acquire() {
        std::unique_lock<std::mutex> lock(mu_);
        
        auto now = std::chrono::system_clock::now();
        while (!available_.empty()) {
            auto pooled = available_.front();
            available_.pop();
            
            // Check: is this connection too old?
            if (now - pooled.acquired_at > max_age_) {
                // Discard and create a new one
                pooled.conn->close();
                return createNewConnection();
            }
            
            // Check: is this connection idle too long?
            if (now - pooled.last_used > idle_timeout_) {
                // Discard and create a new one
                pooled.conn->close();
                return createNewConnection();
            }
            
            // Validate: can we still use it?
            if (!pooled.conn->isHealthy()) {
                pooled.conn->close();
                return createNewConnection();
            }
            
            // Reuse
            pooled.last_used = now;
            return pooled.conn;
        }
        
        // Pool is empty; create a new connection
        return createNewConnection();
    }
};
```

The lesson: **A connection pool is not a set-and-forget data structure. It requires monitoring, validation, and lifecycle management.**

---

## 70.2 Repositories — Reprise

We covered this in Chapter 38, but it bears repeating here in the context of the database boundary.

A repository is a **domain-shaped interface to storage**. It gives the service a collection-like interface (findById, save, findByUserId) while hiding whether data comes from a database, a cache, an API, or a file.

The key idea: **a repository returns domain objects, not rows**.

```cpp
// Good
class BookRepository {
public:
    std::optional<Book> findById(BookId id) {
        // Implementation queries the database
        // But returns a Book domain object
    }
};

// Bad
class BookRepository {
public:
    std::optional<Row> findById(BookId id) {
        // Returns a raw row; caller must know how to turn it into a Book
    }
};
```

Why? Because the service should not know the shape of a row. If the database schema changes (normalized into two tables instead of one), the repository changes, but the service does not.

---

## 70.3 Mapping Rows to Domain Objects

This is where the rubber meets the road. A row from the database is a collection of columns and types. A domain object is an entity with identity, state, behavior, and invariants. The mapping between them is not trivial.

The core tension: **a row is just data, but a domain object enforces rules**. When you load a `Book` from the database, you must rebuild not just its fields, but its invariants. If the database stores `price_cents` as an integer, you must turn it into a `Price` value object that knows prices are never negative. If the database stores `status` as a string, you must turn it into an enum with no invalid states.

This is not optional. Leaving a row in its raw form (as a map of fields) until late in your service is a common way to introduce bugs. A typo in a string key (`book['price']` vs `book['prices']`) is silent; a type mismatch in a domain object is a compiler error.

### Hand-Rolled Mappers

The simplest approach: write the mapping by hand.

```cpp
struct BookRow {
    std::string id;
    std::string title;
    std::string author_id;
    float price;
    std::string isbn;
    int year_published;
};

class BookRepository {
private:
    DatabaseConnection& db_;
    
    Book mapRowToBook(const BookRow& row) {
        // Restore the domain object from the row
        Book b(BookId(row.id), row.title, AuthorId(row.author_id));
        b.setPrice(Price::inUSD(row.price));
        b.setISBN(ISBN(row.isbn));
        b.setYearPublished(row.year_published);
        return b;
    }
    
public:
    std::optional<Book> findById(BookId id) {
        auto result = db_.query(
            "SELECT id, title, author_id, price, isbn, year_published "
            "FROM books WHERE id = ?",
            id.value()
        );
        
        if (result.empty()) {
            return std::nullopt;
        }
        
        return mapRowToBook(result[0]);
    }
    
    void save(const Book& book) {
        auto existing = findById(book.id());
        
        if (existing) {
            db_.execute(
                "UPDATE books SET title = ?, author_id = ?, price = ?, "
                "isbn = ?, year_published = ? WHERE id = ?",
                book.title(), book.authorId().value(), book.price().cents(),
                book.isbn().value(), book.yearPublished(), book.id().value()
            );
        } else {
            db_.execute(
                "INSERT INTO books (id, title, author_id, price, isbn, year_published) "
                "VALUES (?, ?, ?, ?, ?, ?)",
                book.id().value(), book.title(), book.authorId().value(),
                book.price().cents(), book.isbn().value(), book.yearPublished()
            );
        }
    }
};
```

**Pros:** Total control. No magic. The mapping is explicit.

**Cons:** Boilerplate. If you add a field to Book, you must update `mapRowToBook` and both queries. Easy to forget.

### ORMs (Object-Relational Mappers)

ORMs like Hibernate or SQLAlchemy promise to automate this mapping.

```cpp
// Pseudocode: ORM-like interface in C++ (library-dependent)
class Book {
public:
    PERSIST_FIELD(std::string, id);
    PERSIST_FIELD(std::string, title);
    PERSIST_FIELD(std::string, author_id);
    PERSIST_FIELD(float, price);
};

// ORM handles querying and mapping automatically
auto book = orm.books().findById(42);  // Returns a Book
book.setTitle("New Title");
orm.books().save(book);  // ORM detects changes and issues an UPDATE
```

**Pros:** Less boilerplate. Changes to the schema can sometimes be auto-detected.

**Cons:** Magic. The ORM decides when to load related objects (lazy vs eager loading), what happens on delete, how transactions work. These decisions leak into your domain model design. You end up with an anemic model, not a rich one. See Chapter 39 for why this matters.

The biggest pitfall: ORMs hide the cost of operations. A single line of code (`book.author()`) might trigger a query if the author was lazy-loaded. Worse, if you load 1000 books and then access their authors, the ORM transparently runs 1000 queries. The code looks simple; the performance is silently terrible.

For systems where performance and control matter, hand-rolled mappers with explicit queries are often better than ORMs. You see the cost of every operation. You know exactly when a query runs.

### The N+1 Problem

Whether you hand-roll or use an ORM, you can easily trigger the N+1 problem.

```cpp
// Load all books
auto books = repo.findAll();  // 1 query: SELECT * FROM books

// For each book, load its author
for (auto& book : books) {
    auto author = authorRepo.findById(book.authorId());  // N queries
    book.setAuthor(author);
}

// Total: 1 + N queries, where N = number of books
```

If you have 1000 books, you just ran 1001 queries. The first query was cheap; the next 1000 were individually cheap but collectively expensive.

Solutions:

1. **Eager loading (join):** Load the author in the same query.
   ```cpp
   auto books = repo.findAllWithAuthors();
   // SELECT b.*, a.* FROM books b LEFT JOIN authors a ON b.author_id = a.id
   ```

2. **Batch loading:** Load all author IDs, then query for all authors once.
   ```cpp
   auto books = repo.findAll();
   std::set<AuthorId> authorIds;
   for (const auto& book : books) {
       authorIds.insert(book.authorId());
   }
   auto authors = authorRepo.findByIds(authorIds);  // 1 query
   for (auto& book : books) {
       book.setAuthor(authors[book.authorId()]);
   }
   ```

3. **Lazy loading with cache:** Don't load the author until asked, but cache it.
   ```cpp
   auto book = repo.findById(id);
   auto author = book.author();  // Loads on first access, cached after
   ```

The point: **Know the cost of loading.** If a query touches 1000 rows and each row loads 5 authors, you just paid for 5000 queries. Make this explicit in your code, not hidden in the ORM.

---

## 70.4 Transactions

A transaction is a sequence of database operations that either all succeed or all fail together. Transactions enforce the ACID properties (Atomicity, Consistency, Isolation, Durability), though not all databases support all levels.

A transaction is also your service's tool for preventing inconsistency. Without them, multi-step operations can leave the database in a partial state. With them, the database either sees the full operation or none of it.

This is not theoretical. Consider a real scenario:

1. A user initiates a refund.
2. Your service marks the order as refunded.
3. It sends a message to the payment system to reverse the charge.
4. Before the message is sent, the database connection fails.

Without a transaction: the database now has a refunded order, but the payment system has no idea. The user's refund never processes. With a transaction: the refund is either fully in the database (and the payment system will be notified) or fully rolled back (and the order stays charged).

### ACID, Briefly

- **Atomicity**: All-or-nothing. Either all writes in the transaction succeed, or none do.
- **Consistency**: The database's invariants hold before and after the transaction.
- **Isolation**: Concurrent transactions do not interfere with each other.
- **Durability**: Once committed, the write survives a crash.

### When You Need Transactions

A service that changes multiple aggregates must use a transaction to keep them consistent.

```cpp
class RefundService {
public:
    Result<void> refund(OrderId orderId) {
        auto order = orderRepo_.findById(orderId);
        if (!order) return Error("Order not found");
        
        auto payment = paymentRepo_.findByOrderId(orderId);
        if (!payment) return Error("Payment not found");
        
        // Both must succeed or both must fail
        try {
            transaction_.begin();
            
            order.markRefunded();
            orderRepo_.save(*order);
            
            payment.markRefunded();
            paymentRepo_.save(*payment);
            
            transaction_.commit();
        } catch (const std::exception& e) {
            transaction_.rollback();
            return Error("Refund failed: " + std::string(e.what()));
        }
        
        return Ok();
    }
};
```

Without the transaction:
- If the order save succeeds but the payment save fails, the database is inconsistent (order marked refunded, payment not).
- A crash between the two saves leaves the same inconsistency.

With the transaction:
- Both writes succeed atomically.
- Or neither does, and the database is left unchanged.

### Isolation Levels

By default, databases use **Read Committed** isolation: you can read only data that has been committed. This prevents dirty reads but allows phantom reads (a row you didn't see before appears in a range query if another transaction inserts it).

For more strictness, use **Serializable** isolation: transactions behave as if they ran one after another, with no concurrency. This prevents all anomalies but is slower and more prone to deadlock.

```cpp
// Pseudocode
transaction_.begin(IsolationLevel::ReadCommitted);  // Default
// or
transaction_.begin(IsolationLevel::Serializable);  // Stronger, slower
```

Choose based on what invariants you need to protect.

The distinction matters in practice:

- **Read Committed**: Good enough for most operations (create, update, delete). Prevents a transaction from seeing partial writes. But two concurrent reads of the same row might see different committed versions.

- **Repeatable Read**: A transaction sees a consistent snapshot of the database as of the moment it started. No phantom reads from concurrent inserts.

- **Serializable**: The strongest level. Transactions are completely isolated. Use when you have invariants like "only one active subscription per customer" that a concurrent transaction might violate.

Higher isolation costs more (lock contention, longer transactions, more deadlock). Profile your workload. Most systems run fine on Read Committed with careful design.

---

## 70.5 Migrations

A database schema is not hand-edited on the server. It is versioned, like code. This is perhaps the most important discipline for database hygiene.

When you change the schema by hand on production (adding a column, renaming a table), you create invisible state. The code expects one schema; the database has another. Someone deploys a new version of the code that expects a column that doesn't exist yet. The application crashes. Or worse: a migration on the staging server succeeds, then fails on production because the data distribution is different.

Migrations solve this by making schema changes explicit and version-controlled.

### Schema as Code

Migrations are SQL files, numbered, that run in order.

```
migrations/
  001_create_books_table.sql
  002_create_authors_table.sql
  003_add_isbn_to_books.sql
```

```sql
-- 001_create_books_table.sql
CREATE TABLE books (
    id UUID PRIMARY KEY,
    title TEXT NOT NULL,
    author_id UUID NOT NULL REFERENCES authors(id),
    price_cents INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 002_create_authors_table.sql
CREATE TABLE authors (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 003_add_isbn_to_books.sql
ALTER TABLE books ADD COLUMN isbn VARCHAR(13);
```

A migration tool (Flyway, Liquibase, Alembic, sqlx-migrate) tracks which migrations have run and applies the ones you have not seen.

### Forward-Only vs. Reversible

A **forward-only** migration cannot be undone:

```sql
ALTER TABLE books DROP COLUMN isbn;
```

This is fast and simple, but if you realize the migration is wrong, you have to write a new migration to add the column back.

A **reversible** migration has an up and a down:

```sql
-- Up
ALTER TABLE books DROP COLUMN isbn;

-- Down
ALTER TABLE books ADD COLUMN isbn VARCHAR(13);
```

This is safer for development, but irreversible migrations (like a data type change) still require custom down logic.

### Dangerous Migrations

Some migrations are slow or risky on large tables:

1. **NOT NULL on a big table**: Adds a column and sets a default on millions of rows.
   ```sql
   -- Slow on a 1M-row table
   ALTER TABLE books ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'active';
   ```
   Better: add nullable, back-fill in a separate job, then add NOT NULL.

2. **Dropping columns**: Irreversible without a backup. If you made a mistake, the data is gone.

3. **Renaming columns**: Some databases lock the table while renaming.

4. **Changing types**: Requires casting; can fail if existing values are incompatible.

For large tables, use online schema change tools (pt-online-schema-change, GitHub's gh-ost) that do not lock the table.

---

## 70.6 Query Building Safely

The enemy: SQL injection.

```cpp
// WRONG
std::string title = getUserInput("book title");
auto result = db_.query(
    "SELECT * FROM books WHERE title = '" + title + "'"
);
// If title = "'; DROP TABLE books; --", you are in trouble
```

The defense: **parameterized queries** (also called prepared statements).

```cpp
// Right
std::string title = getUserInput("book title");
auto result = db_.query(
    "SELECT * FROM books WHERE title = ?",
    title
);
// The database driver handles escaping and treats title as data, not code
```

Every database library has this. Use it always.

```cpp
// C++ with libpq (PostgreSQL)
db_.query("SELECT * FROM books WHERE id = $1 AND status = $2", id, status);

// C++ with SQLite
db_.query("SELECT * FROM books WHERE id = ? AND status = ?", id, status);
```

Stored procedures are a borderline case. If you write the procedure definition, you control the SQL. If you are calling a stored procedure and passing user input, use parameterized calls.

---

## 70.7 Caching In Front of The DB

Reading from the database is expensive. A typical query might take 1–10ms. If you serve thousands of requests per second and each does multiple queries, you are now spending hundreds of milliseconds on I/O that could be milliseconds if cached in memory.

Caching reads in memory (or in Redis, Memcached) can speed up the critical path dramatically. The tradeoff: **stale data**. A cached value might be out of sync with the database.

This tradeoff is real and must be made explicit in your design. "Let us cache this" is not a decision; "let us cache this for 5 seconds, accepting that users might see data up to 5 seconds old" is a decision.

### Read-Through Cache

When you query, check the cache first. If there is a miss, load from the database and populate the cache.

```cpp
class CachedBookRepository {
private:
    BookRepository& underlying_;
    std::unordered_map<std::string, Book> cache_;
    std::mutex mu_;
    
public:
    std::optional<Book> findById(BookId id) {
        {
            std::unique_lock<std::mutex> lock(mu_);
            auto it = cache_.find(id.value());
            if (it != cache_.end()) {
                return it->second;  // Cache hit
            }
        }
        
        // Cache miss: load from database
        auto book = underlying_.findById(id);
        if (book) {
            std::unique_lock<std::mutex> lock(mu_);
            cache_[id.value()] = *book;
        }
        return book;
    }
};
```

### Cache Invalidation

The hard part: **when does the cache become stale?**

Option 1: **TTL (Time-To-Live)**. Expire cache entries after N seconds.
```cpp
// Cache entry is valid for 300 seconds
if (now - entry.cached_at > 300s) {
    // Evict from cache
}
```
Simple, but data can be stale for up to 300 seconds.

Option 2: **Explicit invalidation**. When you save a Book, remove it from the cache.
```cpp
void save(const Book& book) {
    underlying_.save(book);
    {
        std::unique_lock<std::mutex> lock(mu_);
        cache_.erase(book.id().value());
    }
}
```
Accurate, but requires discipline. If you forget to invalidate one path, the cache is stale.

Option 3: **Write-through cache**. When you save a Book, update the cache and the database together.
```cpp
void save(const Book& book) {
    underlying_.save(book);
    {
        std::unique_lock<std::mutex> lock(mu_);
        cache_[book.id().value()] = book;
    }
}
```
Keeps cache fresh, but if the database write fails, the cache is inconsistent.

### Write-Back Cache

A write-back (or write-behind) cache buffers writes and flushes them to the database later. Fast writes, but data loss risk if the cache crashes before flushing.

Generally: use only for non-critical data (analytics, logs) or with persistent write buffers.

---

## 70.8 Worked Example: A BookRepository in C++

Let us build a concrete BookRepository using SQLite and a thin wrapper.

### The Domain Model

```cpp
// domain/book.h
class Book {
private:
    const BookId id_;
    std::string title_;
    AuthorId author_id_;
    Price price_;
    ISBN isbn_;
    int year_published_;
    
public:
    Book(BookId id, const std::string& title, AuthorId author_id)
        : id_(id), title_(title), author_id_(author_id) {}
    
    BookId id() const { return id_; }
    const std::string& title() const { return title_; }
    AuthorId authorId() const { return author_id_; }
    Price price() const { return price_; }
    void setPrice(Price p) { price_ = p; }
    
    ISBN isbn() const { return isbn_; }
    void setISBN(const ISBN& i) { isbn_ = i; }
    
    int yearPublished() const { return year_published_; }
    void setYearPublished(int y) { year_published_ = y; }
};
```

### The Repository

```cpp
// persistence/book_repository.h
class BookRepository {
private:
    sqlite3* db_;
    
    Book mapRowToBook(sqlite3_stmt* stmt) const {
        const char* id_str = reinterpret_cast<const char*>(
            sqlite3_column_text(stmt, 0));
        const char* title = reinterpret_cast<const char*>(
            sqlite3_column_text(stmt, 1));
        const char* author_id_str = reinterpret_cast<const char*>(
            sqlite3_column_text(stmt, 2));
        int price_cents = sqlite3_column_int(stmt, 3);
        const char* isbn_str = reinterpret_cast<const char*>(
            sqlite3_column_text(stmt, 4));
        int year = sqlite3_column_int(stmt, 5);
        
        Book book(BookId(id_str), title, AuthorId(author_id_str));
        book.setPrice(Price::fromCents(price_cents));
        book.setISBN(ISBN(isbn_str ? isbn_str : ""));
        book.setYearPublished(year);
        return book;
    }
    
public:
    BookRepository(sqlite3* db) : db_(db) {}
    
    std::optional<Book> findById(const BookId& id) {
        const char* sql = R"(
            SELECT id, title, author_id, price_cents, isbn, year_published
            FROM books
            WHERE id = ?
        )";
        
        sqlite3_stmt* stmt = nullptr;
        int rc = sqlite3_prepare_v2(db_, sql, -1, &stmt, nullptr);
        if (rc != SQLITE_OK) {
            throw std::runtime_error(sqlite3_errmsg(db_));
        }
        
        sqlite3_bind_text(stmt, 1, id.value().c_str(), -1, SQLITE_STATIC);
        
        std::optional<Book> result;
        if (sqlite3_step(stmt) == SQLITE_ROW) {
            result = mapRowToBook(stmt);
        }
        
        sqlite3_finalize(stmt);
        return result;
    }
    
    void save(const Book& book) {
        auto existing = findById(book.id());
        
        const char* sql = existing
            ? R"(
                UPDATE books
                SET title = ?, author_id = ?, price_cents = ?, isbn = ?, year_published = ?
                WHERE id = ?
              )"
            : R"(
                INSERT INTO books (id, title, author_id, price_cents, isbn, year_published)
                VALUES (?, ?, ?, ?, ?, ?)
              )";
        
        sqlite3_stmt* stmt = nullptr;
        int rc = sqlite3_prepare_v2(db_, sql, -1, &stmt, nullptr);
        if (rc != SQLITE_OK) {
            throw std::runtime_error(sqlite3_errmsg(db_));
        }
        
        if (existing) {
            sqlite3_bind_text(stmt, 1, book.title().c_str(), -1, SQLITE_TRANSIENT);
            sqlite3_bind_text(stmt, 2, book.authorId().value().c_str(), -1, SQLITE_TRANSIENT);
            sqlite3_bind_int(stmt, 3, book.price().cents());
            sqlite3_bind_text(stmt, 4, book.isbn().value().c_str(), -1, SQLITE_TRANSIENT);
            sqlite3_bind_int(stmt, 5, book.yearPublished());
            sqlite3_bind_text(stmt, 6, book.id().value().c_str(), -1, SQLITE_TRANSIENT);
        } else {
            sqlite3_bind_text(stmt, 1, book.id().value().c_str(), -1, SQLITE_TRANSIENT);
            sqlite3_bind_text(stmt, 2, book.title().c_str(), -1, SQLITE_TRANSIENT);
            sqlite3_bind_text(stmt, 3, book.authorId().value().c_str(), -1, SQLITE_TRANSIENT);
            sqlite3_bind_int(stmt, 4, book.price().cents());
            sqlite3_bind_text(stmt, 5, book.isbn().value().c_str(), -1, SQLITE_TRANSIENT);
            sqlite3_bind_int(stmt, 6, book.yearPublished());
        }
        
        rc = sqlite3_step(stmt);
        if (rc != SQLITE_DONE) {
            throw std::runtime_error(sqlite3_errmsg(db_));
        }
        
        sqlite3_finalize(stmt);
    }
};
```

### Testing with a Fake

```cpp
// Test fake: in-memory storage
class FakeBookRepository {
private:
    std::unordered_map<std::string, Book> storage_;
    
public:
    std::optional<Book> findById(const BookId& id) {
        auto it = storage_.find(id.value());
        if (it != storage_.end()) {
            return it->second;
        }
        return std::nullopt;
    }
    
    void save(const Book& book) {
        storage_[book.id().value()] = book;
    }
};

// Test
void test_book_service_with_fake() {
    FakeBookRepository fakeRepo;
    BookService service(fakeRepo);
    
    // Load a non-existent book
    auto book = fakeRepo.findById(BookId("nonexistent"));
    assert(!book);
    
    // Create and save a book
    Book newBook(BookId("uuid-1"), "The Pragmatic Programmer", AuthorId("author-1"));
    newBook.setPrice(Price::fromCents(4599));
    fakeRepo.save(newBook);
    
    // Load it back
    auto loaded = fakeRepo.findById(BookId("uuid-1"));
    assert(loaded);
    assert(loaded->title() == "The Pragmatic Programmer");
    assert(loaded->price().cents() == 4599);
}
```

The service that uses the repository does not care whether it is the real SQLite implementation or the fake. It sees only the `BookRepository` interface.

---

## 70.9 Tradeoffs

| Approach | Pros | Cons | When to Use |
|----------|------|------|------------|
| Hand-rolled SQL + mappers | Full control; explicit cost | Boilerplate; easy to forget fields | Small schemas; full-featured repositories |
| ORM | Less boilerplate; auto-migrations | Magic; anemic models; N+1 hidden | CRUD-heavy apps without domain logic |
| Query builder | SQL is safer; readable | Still need to know the schema | Medium complexity; avoiding manual SQL |
| NoSQL document store | Flexible schema; fast writes | No joins; eventual consistency risk | Analytics; write-heavy logs; cache layers |
| Event sourcing | Audit trail; temporal queries | Complex; eventual consistency | Systems with strict audit requirements |

---

## 70.10 Common Misconceptions

**Misconception 1: "An ORM handles the database for me."**

No. An ORM handles SQL generation. You still need to understand transactions, indexes, locking, and isolation levels. A slow query is still slow, whether written by hand or generated by an ORM. You still need to avoid N+1.

**Misconception 2: "Connection pools are tuned once."**

No. Pool size, idle timeout, and max age must be tuned based on your workload. Too small, and you deadlock. Too large, and you waste memory. Monitor and adjust.

**Misconception 3: "Schema migrations are a deployment step."**

Schema migrations are a **code change**. They should be versioned, reviewed, tested, and rolled forward only. Do not hand-edit the schema on the server. Do not have multiple developers running migrations in different orders.

**Misconception 4: "Caching is a free speed boost."**

Caching introduces staleness. Stale data can be worse than no cache (if a user sees an outdated balance and makes a bad decision). Choose between TTL and explicit invalidation; neither is perfect.

**Misconception 5: "Transactions fix concurrency problems."**

Transactions fix consistency within a single request. Concurrency problems arise from transactions interfering with each other. Higher isolation levels prevent more problems but are slower. Choose based on what invariants you must protect, not as a blanket solution.

---

## 70.11 Exercises

1. Write a `ConnectionPool` class in C++ with size, idle timeout, and max age. Test that it rejects more concurrent acquires than pool size.

2. Implement a `Repository<T>` template in C++ that uses a `FakeDatabase` (in-memory map) for testing. Show that a `Service<T>` can work with either the fake or a real database.

3. Write a migration from a single `users` table to a normalized schema with `users` and `user_profiles`. Show the forward and reverse migrations.

4. Build a parameterized SQL query generator: given a table name, column names, and values, generate a safe INSERT or UPDATE statement.

5. Implement a read-through cache wrapper around a `BookRepository`. Measure the time for a cache miss vs. a hit. Then add explicit invalidation on save.

---

## 70.12 Summary

A database layer is a boundary. Build it carefully so neither the domain nor the storage leaks across. Use connection pools to avoid exhausting resources; monitor their size and lifecycle. Return domain objects from repositories, not rows. Understand the cost of loading (N+1 is real). Write transactions for invariants that span multiple writes. Version your schema in code, not by hand. Use parameterized queries to prevent injection. Cache wisely, knowing the cost of staleness. Test with fakes so services do not depend on a running database.

---

> **[← Previous: Designing a Web Server](02-designing-a-web-server.md)**  ·  **[↑ Part 7](README.md)**  ·  **[Next: A Mini Framework →](04-a-mini-framework.md)**
