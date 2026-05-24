# 04 — Data Layer

---

## 4.1 JPA, Hibernate, and Spring Data — How They Relate

```
Your Code
  └── Spring Data JPA (repositories — auto-generated CRUD implementations)
        └── JPA (specification — defines EntityManager, JPQL, etc.)
              └── Hibernate (implementation of JPA — the actual SQL generator)
                    └── JDBC Driver (talks to the actual database)
```

### How It Works

Think of this like a restaurant kitchen:

- **JPA** is the recipe book — a specification that says "here's how persistence should work, here are the rules." It defines interfaces like `EntityManager`, annotations like `@Entity`, and query language JPQL. But JPA alone is just rules on paper. It cannot persist anything.

- **Hibernate** is the actual chef — it reads the JPA recipe book and implements every rule. When you do `entityManager.persist(user)`, it's Hibernate that decides what SQL to generate, how to batch it, how to cache it, and when to flush it to the database.

- **Spring Data JPA** is the kitchen manager — it takes Hibernate's power and adds productivity. You write `interface UserRepository extends JpaRepository<User, Long>`, declare `Optional<User> findByEmail(String email)`, and Spring generates the full implementation (SQL and all) at startup. You never write boilerplate CRUD.

- **JDBC Driver** is the delivery guy — it physically sends the SQL string over a TCP connection to your database.

### Why This Layering Exists

You could use JDBC directly, but then you're writing raw SQL, managing `ResultSet`, manually mapping columns to fields, and handling connections by hand. JPA/Hibernate abstract that entire mechanical layer. Spring Data JPA abstracts it even further.

In an SDE 2 interview, the question "what is the difference between JPA and Hibernate" is very common. The answer is: JPA is the interface, Hibernate is the implementation. You write code against JPA, but Hibernate runs it.

> **Pitfall:** A lot of developers confuse "JPA annotations" with "Hibernate annotations". `@Entity`, `@Id`, `@OneToMany` are **JPA** — they live in `javax.persistence` (or `jakarta.persistence`). `@CreationTimestamp`, `@BatchSize`, `@NaturalId` are **Hibernate-specific** — they're in `org.hibernate.annotations`. If you ever switch from Hibernate to another JPA provider (rare, but possible), Hibernate-specific annotations won't work.

---

## 4.2 Entity Mapping

```java
@Entity                  // Marks this class as a JPA entity — Hibernate maps it to a DB table
@Table(name = "users", indexes = {  // Specifies the exact DB table name and declares any indexes to be created
    @Index(name = "idx_users_email", columnList = "email")  // Creates a DB index on the email column for fast lookup queries
})
// Represents a single user row in the "users" table; its lifecycle is fully managed by JPA/Hibernate
public class User {

    @Id                  // Marks this field as the table's primary key
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // Tells the DB to auto-increment this value on each INSERT
    private Long id;     // Unique identifier for this user — assigned by the DB, never set manually

    @Column(nullable = false, length = 100)  // Maps to a NOT NULL VARCHAR(100) column in the DB
    private String name;   // User's full name — required, max 100 characters

    @Column(nullable = false, unique = true, length = 255)  // NOT NULL + UNIQUE constraint enforced at the DB level
    private String email;  // User's email address — must be unique across all user rows

    @Enumerated(EnumType.STRING)  // Stores the enum as its name ("ACTIVE") not its ordinal (0) — safe against enum reordering
    @Column(nullable = false)
    private UserStatus status;  // Current account state (e.g. ACTIVE, INACTIVE) stored as a readable string

    @CreationTimestamp               // Hibernate-specific: auto-sets this to the current timestamp just before INSERT
    @Column(updatable = false)       // Prevents this column from being changed after the row is first created
    private LocalDateTime createdAt; // Timestamp of when this user was first persisted — never changes after creation

    @UpdateTimestamp                 // Hibernate-specific: auto-sets this to the current timestamp on every UPDATE
    private LocalDateTime updatedAt; // Timestamp of the most recent change to this user's data

    @Version                         // Enables optimistic locking — Hibernate adds "AND version = ?" to every UPDATE to detect concurrent modifications
    private Long version;            // Auto-incremented by Hibernate on each UPDATE; a stale read throws OptimisticLockException
}
```

### How It Works

When Spring Boot starts, Hibernate scans your classpath for classes annotated with `@Entity`. For each one, it reads all the annotations and builds an internal model called a **mapping metadata**. From this, it can:

1. Generate `CREATE TABLE` DDL (if `spring.jpa.hibernate.ddl-auto=create` or `update`)
2. Know how to translate between Java objects and DB rows
3. Know which column maps to which field, what constraints to apply, and how to generate IDs

`@Table(name = "users")` explicitly sets the table name. Without it, Hibernate uses the class name by default — `User` becomes `user` or `users` depending on the naming strategy. Always be explicit in production.

`@Column(nullable = false)` tells Hibernate to add `NOT NULL` to DDL. It also tells Hibernate's schema validation to check for this constraint. It does **not** validate at the Java level — you still need Bean Validation (`@NotNull`) if you want that.

### `GenerationType` Strategies — In Depth

| Strategy | Behavior | When to Use |
|---|---|---|
| `IDENTITY` | DB auto-increment (MySQL, PostgreSQL SERIAL) | Most common for single-node apps |
| `SEQUENCE` | DB sequence object | PostgreSQL sequences, batch inserts |
| `TABLE` | Extra DB table for ID generation | Avoid — has locking issues |
| `UUID` | Assigned UUID | Distributed systems, no ordering needed |

**`SEQUENCE` vs `IDENTITY` — a real gotcha:**

With `IDENTITY`, Hibernate must insert a row to get the ID. This **disables batch inserts** because Hibernate needs to flush after each insert to get the generated key. With `SEQUENCE`, Hibernate can pre-fetch a block of IDs from the sequence and assign them in memory, allowing batch operations.

```java
// With SEQUENCE — enables batching
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "user_id_seq", allocationSize = 50)
// @SequenceGenerator: defines a DB sequence object for ID generation; allocationSize pre-fetches a block of IDs so Hibernate doesn't hit the DB for every single insert
private Long id;
// allocationSize = 50 means Hibernate fetches 50 IDs at a time, reducing DB round-trips
```

### `@Enumerated` — Critical Pitfall

```java
// DANGEROUS — DO NOT DO THIS
@Enumerated(EnumType.ORDINAL)  // stores 0, 1, 2...
private UserStatus status;

// If you add a new enum value BETWEEN existing ones, your DB data becomes wrong:
// Before: ACTIVE=0, INACTIVE=1
// After:  ACTIVE=0, PENDING=1, INACTIVE=2  ← all INACTIVE rows now read as PENDING!

// ALWAYS do this instead:
@Enumerated(EnumType.STRING)  // stores "ACTIVE", "INACTIVE"
private UserStatus status;
// String is robust to reordering; only renaming breaks it
```

> **Tricky Question:** If `@Column(nullable = false)` is on a field, will inserting a `null` via Java throw an exception? **Not necessarily at the Java level.** Hibernate may let you set `null` in Java and only fail when the SQL hits the DB. For Java-level null checks, add `@NotNull` from `jakarta.validation.constraints` alongside `@Column(nullable = false)`.

> **Pitfall:** `@CreationTimestamp` and `@UpdateTimestamp` are Hibernate-specific and set the value just before Hibernate issues the SQL. Do **not** also set these fields manually in your code — Hibernate will overwrite them.

---

## 4.3 Entity Relationships

### `@ManyToOne` / `@OneToMany` — The Most Common Pattern

```java
@Entity  // Maps Order to its own DB table (e.g. "order")
// Represents a customer's purchase order; owns the relationship with OrderItem by holding the FK column on the item side
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)  // Auto-incremented primary key
    private Long id;  // Unique identifier for this order — set by the DB on INSERT

    @ManyToOne(fetch = FetchType.LAZY)  // Many orders belong to one customer; LAZY = don't load Customer data until a field on it is actually accessed
    @JoinColumn(name = "customer_id", nullable = false)  // Creates a "customer_id" FK column in the orders table; NOT NULL enforced at DB level
    private Customer customer;  // The customer who placed this order

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    // mappedBy="order": FK lives on OrderItem, not here; CascadeType.ALL: save/delete propagates to items; orphanRemoval: removing an item from this list also deletes its DB row
    private List<OrderItem> items = new ArrayList<>();  // All line items belonging to this order

    // Adds an item and sets the back-reference so both sides of the bidirectional relationship stay consistent
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);  // sets the FK column on the owning side — required for Hibernate to persist the relationship
    }

    // Removes an item and nulls its back-reference; with orphanRemoval=true Hibernate then deletes that row from the DB
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);  // clears FK; Hibernate treats this item as an orphan and issues a DELETE
    }
}

@Entity  // Maps OrderItem to its own DB table (e.g. "order_item")
// Represents a single line item within an order; this is the owning side — it holds the "order_id" FK column
public class OrderItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)  // Auto-incremented primary key
    private Long id;  // Unique identifier for this line item

    @ManyToOne(fetch = FetchType.LAZY)  // Many items belong to one order; LAZY avoids loading the entire Order when only the item is needed
    @JoinColumn(name = "order_id")  // "order_id" FK column lives in this table — this is the owning side Hibernate reads/writes for the relationship
    private Order order;  // The order this item belongs to; Hibernate syncs the FK from this field
}
```

### How It Works — Owning Side vs Inverse Side

This is one of the most misunderstood concepts in JPA. In a bidirectional relationship, Hibernate needs to know **which side owns the foreign key column**.

- The **owning side** is the one with `@JoinColumn`. It has the actual `order_id` column in its table. Hibernate syncs the FK from this side.
- The **inverse side** is the one with `mappedBy = "..."`. It tells Hibernate "the FK is managed by the `order` field on the other side — look there."

**The critical implication:** If you only update the inverse side, Hibernate ignores it.

```java
// BROKEN — only updating inverse side
order.getItems().add(item);  // ← Hibernate ignores this for FK persistence
orderRepository.save(order); // order_id column in order_items stays NULL

// CORRECT — must update the owning side
item.setOrder(order);        // ← sets the FK column
itemRepository.save(item);   // order_id is populated

// BEST — use the helper method which updates BOTH sides
order.addItem(item);         // sets item.order AND adds to order.items
orderRepository.save(order); // with CascadeType.ALL, saves item too
```

### `@ManyToMany` — And When to Avoid It

```java
@Entity  // Maps Student to the "student" table
// Represents a student; owns the many-to-many relationship with Course through the join table
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)  // Auto-incremented primary key
    private Long id;  // Unique identifier for this student

    @ManyToMany  // A student can enroll in many courses; a course can have many students
    @JoinTable(  // Defines the intermediate join table that stores the student↔course pairings
        name = "student_courses",                             // Name of the join table in the DB
        joinColumns = @JoinColumn(name = "student_id"),       // FK column in the join table pointing to Student
        inverseJoinColumns = @JoinColumn(name = "course_id")  // FK column in the join table pointing to Course
    )
    private Set<Course> courses = new HashSet<>();  // Courses this student is enrolled in; Set prevents duplicate entries
}

@Entity  // Maps Course to the "course" table
// Represents a course; the inverse (non-owning) side of the student↔course relationship — Student manages the join table
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)  // Auto-incremented primary key
    private Long id;  // Unique identifier for this course

    @ManyToMany(mappedBy = "courses")  // Inverse side — Student owns the join table; changes only to this collection are NOT persisted by Hibernate
    private Set<Student> students = new HashSet<>();  // Students enrolled in this course
}
```

**When to replace `@ManyToMany` with an explicit join entity:**

If you need to store data on the relationship itself (e.g., the date a student enrolled in a course, or a grade), `@ManyToMany` can't do that. Replace it with an explicit `@Entity` for the join table:

```java
@Entity  // Explicit join entity replacing @ManyToMany — allows extra attributes to be stored on the relationship itself
// Represents a student's enrollment in a specific course, capturing when they enrolled and what grade they received
public class Enrollment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)  // Auto-incremented primary key
    private Long id;  // Unique identifier for this enrollment record

    @ManyToOne(fetch = FetchType.LAZY)  // Many enrollments can belong to one student
    private Student student;  // The student who enrolled — FK column auto-created by Hibernate

    @ManyToOne(fetch = FetchType.LAZY)  // Many enrollments can belong to one course
    private Course course;  // The course the student enrolled in — FK column auto-created by Hibernate

    private LocalDate enrolledAt;  // Date the student registered for this course
    private String grade;          // Grade earned (nullable — stays null until the instructor submits grades)
}
```

This pattern (turning the join table into an entity) is more common in real production apps.

### FetchType Defaults — Critical to Know

| Relationship | Default FetchType | Should Change? |
|---|---|---|
| `@ManyToOne` | **EAGER** | **Yes — always change to LAZY** |
| `@OneToOne` | **EAGER** | **Yes — always change to LAZY** |
| `@OneToMany` | LAZY | Keep as is |
| `@ManyToMany` | LAZY | Keep as is |

**Rule:** Always declare `FetchType.LAZY` on all associations. Fetch eagerly only in specific queries that need it via `JOIN FETCH` or `@EntityGraph`. This keeps your default load path predictable.

> **Tricky Question:** `@OneToOne` has EAGER as default, just like `@ManyToOne`. This means loading a `User` will always also load its `Address` even if you never use it. This also can't be easily fixed with `JOIN FETCH` (because `@OneToOne` lazy loading requires bytecode enhancement or proxy tricks). This is called the **`@OneToOne` lazy loading problem** and is a common performance trap.

> **Pitfall:** Using `List` vs `Set` for collections matters. With `@OneToMany`, if you use `List` and have `CascadeType.MERGE`, Hibernate may delete and re-insert all items when you merge the parent — this is called the **bag semantics problem**. For bidirectional `@ManyToMany`, always use `Set` to avoid duplicates and Hibernate treating it as a "bag" (list without order, allows duplicates).

---

## 4.4 CascadeType — Propagating Operations

Cascade means: "when I do operation X to the parent, also do X to children automatically."

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items;
```

### How It Works

Without cascade, if you save an `Order`, Hibernate saves only the `Order` row. You'd have to manually call `itemRepository.save(item)` for every item. With `CascadeType.PERSIST`, saving the `Order` also saves all `items` automatically.

```java
// Without cascade:
Order order = new Order();
OrderItem item1 = new OrderItem(...);
order.addItem(item1);
orderRepository.save(order); // only saves Order row
itemRepository.save(item1);  // must manually save each item

// With CascadeType.PERSIST (or ALL):
Order order = new Order();
order.addItem(new OrderItem(...));
orderRepository.save(order); // saves Order AND all items in one go
```

| CascadeType | What it does |
|---|---|
| `PERSIST` | Saving parent also saves new children |
| `MERGE` | Merging (updating) parent also merges children |
| `REMOVE` | Deleting parent also deletes children |
| `REFRESH` | Refreshing parent also refreshes children from DB |
| `DETACH` | Detaching parent also detaches children from persistence context |
| `ALL` | All of the above |

### `orphanRemoval = true` — What it Actually Does

This is different from `CascadeType.REMOVE`. With `orphanRemoval = true`, if you **remove a child from the parent's collection** (without deleting the parent), that child is deleted from the DB.

```java
Order order = orderRepository.findById(orderId).get();
OrderItem item = order.getItems().get(0);
order.removeItem(item);       // removes from collection
orderRepository.save(order);  // with orphanRemoval=true, item is also DELETEd from DB
                               // without orphanRemoval, item row stays — dangling FK!
```

Use `orphanRemoval = true` only when children have **no independent lifecycle** — e.g., `OrderItem` only makes sense attached to an `Order`.

> **Pitfall:** `CascadeType.REMOVE` on `@ManyToMany` is almost always wrong. It doesn't just remove the join table rows — it deletes the related entities themselves. If you cascade REMOVE from `Student` to `Course`, deleting a student deletes the actual course (and all other students enrolled in it!). For `@ManyToMany`, typically use no cascade or only `PERSIST`/`MERGE`.

> **Tricky Question:** What is the difference between `CascadeType.REMOVE` and `orphanRemoval = true`? 
> - `REMOVE`: if you delete the parent entity, children are also deleted.
> - `orphanRemoval`: if you remove a child from the parent's collection (parent survives), the child is deleted.
> - Setting `orphanRemoval = true` implies `CascadeType.REMOVE`, but not vice versa.

---

## 4.5 Repository Pattern

### Repository Hierarchy

```
CrudRepository<T, ID>              — save, findById, findAll, delete, count
  └── PagingAndSortingRepository   — findAll(Pageable), findAll(Sort)
        └── JpaRepository<T, ID>   — flush, saveAll, deleteAllInBatch, getReferenceById
```

```java
// Spring Data repository for the User entity — Spring generates the full CRUD + query implementations at startup; no boilerplate SQL or DAO classes needed
public interface UserRepository extends JpaRepository<User, Long> {
    // Spring Data auto-implements these from the method name:

    // Finds a single user by their unique email; Optional prevents NullPointerException when no match exists
    Optional<User> findByEmail(String email);

    // Returns all users with a given status whose age is strictly greater than the provided threshold
    List<User> findByStatusAndAgeGreaterThan(UserStatus status, int age);

    // Returns true if any user row has this email — runs a COUNT query, cheaper than loading the full entity
    boolean existsByEmail(String email);

    // Returns the count of users in the given status — issues a COUNT(*) query, not a full SELECT
    long countByStatus(UserStatus status);

    // Deletes the user row matching this email — issues a DELETE query directly
    void deleteByEmail(String email);
}
```

### How It Works — The Magic Behind Method Names

At startup, Spring Data scans all interfaces extending repository types. For each method, it parses the name using a set of keyword conventions and builds an AST (abstract syntax tree) that gets compiled into a JPQL query. You never write the implementation — Spring generates it.

```java
findByStatusAndAgeGreaterThan(UserStatus status, int age)
// Parsed as: SELECT u FROM User u WHERE u.status = ?1 AND u.age > ?2
```

This is called **derived query methods**. Under the hood, Spring Data uses a `SimpleJpaRepository` class which does the EntityManager calls. If you ever need to debug, enable SQL logging — you'll see the generated SQL.

### Derived Query Keywords

```java
// Comparison
findByAgeGreaterThan(int age)
findByAgeLessThanEqual(int age)
findByAgeBetween(int min, int max)
findByNameContaining(String part)       // LIKE %part%
findByNameStartingWith(String prefix)   // LIKE prefix%
findByNameEndingWith(String suffix)     // LIKE %suffix

// Boolean / Null
findByActiveTrue()
findByDeletedFalse()
findByDeletedAtIsNull()
findByDeletedAtIsNotNull()

// Ordering and limiting
findTop5ByOrderByCreatedAtDesc()
findFirstByEmailOrderByIdAsc(String email)

// Multiple conditions — AND has higher precedence than OR
findByStatusOrActiveAndAge(UserStatus s, boolean a, int age)
// Parsed as: status = s OR (active = a AND age = age)
// Careful with AND/OR precedence — use @Query for complex logic
```

> **Pitfall:** Method names can get very long and unreadable: `findByStatusAndCreatedAtAfterAndAgeGreaterThanOrderByNameAsc`. This compiles and works but is hard to maintain. As soon as a query has 3+ conditions, switch to `@Query` with JPQL or native SQL. The derived method name approach is great for simple lookups, terrible for complex filters.

> **Tricky Question:** Does `JpaRepository.save()` always execute an `INSERT`? **No.** `save()` calls either `persist` (for new entities) or `merge` (for existing ones). An entity is considered "new" if its `@Id` field is `null` (for objects) or `0` (for primitives). If you set the ID manually before calling `save()`, Hibernate will try to `merge` — which means a `SELECT` first, then an `UPDATE` or `INSERT`. This is the `save()` SELECT+INSERT pattern: a performance trap if you're inserting new entities with pre-assigned IDs.

---

## 4.6 Custom Queries

### JPQL (Object-Level SQL)

JPQL operates on **entities and their fields**, not table names and column names. This makes it database-agnostic.

```java
// Spring Data repository for Order — adds custom JPQL queries on top of the standard JpaRepository CRUD operations
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Fetches orders for a specific customer filtered by status using an explicit JPQL query (operates on entity fields, not raw table columns)
    @Query("SELECT o FROM Order o WHERE o.customer.id = :customerId AND o.status = :status")
    // @Query: lets you write explicit JPQL or SQL instead of relying on method-name parsing — use this for any query with 3+ conditions or joins
    List<Order> findByCustomerAndStatus(
            @Param("customerId") Long customerId,  // @Param: binds this method argument to the :customerId named parameter in the JPQL string
            @Param("status") OrderStatus status);  // Binds this argument to the :status named parameter

    // Loads orders AND their items together in one SQL JOIN — eliminates the N+1 problem for this query
    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.customer.id = :customerId")
    List<Order> findByCustomerWithItems(@Param("customerId") Long customerId);

    // Returns lightweight DTOs via a JPQL constructor expression — avoids loading full entities into the persistence context
    @Query("SELECT new com.example.dto.OrderSummary(o.id, o.status, o.totalAmount) " +
           "FROM Order o WHERE o.customer.id = :customerId")
    List<OrderSummary> findSummariesByCustomer(@Param("customerId") Long customerId);
}
```

### How JPQL Projections Work

When you fetch a full `@Entity`, Hibernate loads it into the **persistence context** (first-level cache), tracks all changes to it, and manages its lifecycle. This has overhead. If you only need a few fields for a read-only API response, projections are far more efficient:

```java
// Option 1: DTO constructor expression in JPQL (shown above)
// Hibernate creates a plain Java object — no persistence tracking at all

// Option 2: Interface projection — Spring generates a dynamic proxy that selects only these columns from the DB (no SELECT *)
public interface OrderSummary {
    Long getId();                  // Returns the order's primary key
    OrderStatus getStatus();       // Returns the order's current status
    BigDecimal getTotalAmount();   // Returns the total monetary value of the order
}

// Repository
List<OrderSummary> findByCustomerId(Long customerId);
// Spring Data infers: SELECT o.id, o.status, o.totalAmount — not SELECT *
```

Interface projections are convenient but slightly slower than DTO projections because of the proxy overhead.

### Native SQL Query

Use native SQL when you need DB-specific features (window functions, CTEs, JSON operators) that JPQL doesn't support:

```java
@Query(value = "SELECT * FROM orders WHERE YEAR(created_at) = :year",
       nativeQuery = true)
List<Order> findByYear(@Param("year") int year);
```

With `nativeQuery = true`, Hibernate passes the SQL directly to the JDBC driver without translation. Column names must match your actual table, not entity fields.

### Modifying Queries

```java
@Modifying          // Signals this is a write query (UPDATE/DELETE) — Spring calls executeUpdate() instead of executeQuery() on the PreparedStatement
@Transactional      // @Transactional: wraps this method in a DB transaction; commits on success, rolls back on RuntimeException by default
@Query("UPDATE User u SET u.status = :status WHERE u.lastLoginAt < :cutoff")
int deactivateInactiveUsers(
        @Param("status") UserStatus status,       // New status to apply to all matched users
        @Param("cutoff") LocalDateTime cutoff);   // Users who haven't logged in since before this date are deactivated; returns the count of updated rows
```

`@Modifying` is required for `UPDATE`/`DELETE` queries — it tells Spring Data this is a write query, not a read. `@Transactional` is required because modifying queries need an active transaction. The return type `int` gives you the number of affected rows.

> **Pitfall:** `@Modifying` queries bypass the persistence context — they go straight to the DB as SQL. This means if you load a `User` entity in the same transaction and then bulk-update it via `@Modifying`, the in-memory entity won't reflect the change. You'd need to call `entityManager.refresh(user)` or `entityManager.clear()` to re-sync. Add `clearAutomatically = true` to `@Modifying` to auto-clear the context: `@Modifying(clearAutomatically = true)`.

> **Tricky Question:** Can you use `@Modifying` with a `SELECT` query? No — it's for `UPDATE`, `DELETE`, and `INSERT ... SELECT`. Using it on a SELECT causes an exception. `@Modifying` tells Spring Data to execute the query with `executeUpdate()` instead of `executeQuery()` on the underlying JDBC `PreparedStatement`.

---

## 4.7 Pagination and Sorting

```java
// Controller
@GetMapping("/users")  // @GetMapping: maps HTTP GET /users requests to this method
public Page<UserResponse> listUsers(
    @RequestParam(defaultValue = "0") int page,           // @RequestParam: reads ?page= from the URL; defaults to 0 (first page) if absent
    @RequestParam(defaultValue = "20") int size,          // Number of items per page; defaults to 20 if the client doesn't specify
    @RequestParam(defaultValue = "createdAt") String sortBy,   // Field name to sort results by; defaults to creation date
    @RequestParam(defaultValue = "desc") String sortDir) {     // Sort direction "asc" or "desc"; defaults to newest-first

    Sort sort = sortDir.equalsIgnoreCase("asc")
        ? Sort.by(sortBy).ascending()
        : Sort.by(sortBy).descending();   // Builds an immutable Sort object from the client-supplied direction

    Pageable pageable = PageRequest.of(page, size, sort);  // Bundles page index, page size, and sort order into one Pageable passed to the repository
    return userService.list(pageable);
}

// Repository
// Spring Data repository for User — supports paginated queries on top of standard CRUD
public interface UserRepository extends JpaRepository<User, Long> {
    // Returns a page of users filtered by status; Spring automatically runs both the data query and the count query
    Page<User> findByStatus(UserStatus status, Pageable pageable);
}

// Service
// Retrieves all users as a paginated page, mapping each User entity to its API response DTO
public Page<UserResponse> list(Pageable pageable) {
    return userRepository.findAll(pageable)
        .map(userMapper::toResponse);  // Converts each User entity to a UserResponse DTO using the mapper
}
```

### How `Page<T>` Works Under the Hood

When you call a method returning `Page<T>`, Spring Data executes **two queries**:

1. The data query: `SELECT ... FROM users ORDER BY created_at DESC LIMIT 20 OFFSET 0`
2. The count query: `SELECT COUNT(*) FROM users` — to compute `totalElements` and `totalPages`

The count query can be slow on large tables. If you don't need `totalPages` (e.g., infinite scroll), use `Slice<T>` instead — it only fires the data query (it checks for one extra row to determine `hasNext()`).

```java
// Use Slice when you only need hasNext(), not total count
Slice<User> findByStatus(UserStatus status, Pageable pageable);
```

`Page<T>` response fields: `content`, `totalElements`, `totalPages`, `number` (current page, 0-indexed), `size`, `last`, `first`, `numberOfElements`.

### Sorting by User Input — Security Pitfall

```java
// DANGEROUS — never sort directly by user input
Sort sort = Sort.by(request.getParam("sortBy"));

// Safe — whitelist allowed sort fields
private static final Set<String> ALLOWED_SORT_FIELDS = Set.of("createdAt", "name", "email");

public Page<UserResponse> list(String sortBy, Pageable pageable) {
    if (!ALLOWED_SORT_FIELDS.contains(sortBy)) {
        throw new IllegalArgumentException("Invalid sort field");
    }
    // now safe to use
}
```

If you allow arbitrary field names in `Sort.by(...)`, an attacker can inject field names that cause Hibernate to generate SQL with unexpected column names (Hibernate will throw a mapping exception, but it's still a vulnerability surface).

> **Pitfall:** Spring Data's default `Pageable` resolver in Spring MVC uses 0-based page index. Your frontend may expect 1-based. Standardize this in your API contract. Also, never let users request `size=10000` — cap it server-side to avoid memory spikes.

> **Tricky Question:** You have a `@OneToMany` relationship and use a `JOIN FETCH` with `Pageable` — what happens? Hibernate warns: **"HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!"** This means Hibernate loads ALL rows into memory and paginates there — not in the DB. Fix: use two queries, or use `@EntityGraph` with `countQuery` specification, or use a subquery.

---

## 4.8 The N+1 Problem — Critical Interview Topic

### What It Is

```java
// You load 100 orders:
List<Order> orders = orderRepository.findAll();
// Hibernate fires: SELECT * FROM orders  — 1 query

// Then you access each order's customer (lazy):
orders.forEach(o -> System.out.println(o.getCustomer().getName()));
// Hibernate fires: SELECT * FROM customers WHERE id = ?  — 100 times!
// Total: 101 queries instead of 1 or 2
```

The name "N+1" refers to: 1 query to fetch the list, then N queries (one per item) to fetch each association. With 1000 orders this becomes 1001 queries.

### Why It Happens

When you access `o.getCustomer()` on a lazily-loaded entity, Hibernate sees a proxy object that hasn't been loaded yet. It fires a SELECT to populate it. Because this happens in a loop, you get N individual SELECTs.

### Fix 1: `JOIN FETCH` in JPQL

```java
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();
// Fires: SELECT o.*, c.* FROM orders o JOIN customers c ON o.customer_id = c.id
// 1 query, full data loaded
```

`DISTINCT` is needed because a JOIN can produce duplicate `Order` rows if one order has multiple items (Hibernate returns the same order object once per join row). `DISTINCT` in JPQL deduplicates in memory, not necessarily in SQL.

### Fix 2: `@EntityGraph`

```java
@EntityGraph(attributePaths = {"customer", "items", "items.product"})
// @EntityGraph: overrides the LAZY fetch strategy for the listed association paths — fetches them eagerly in a single JOIN for this query only, without changing the entity mapping
@Query("SELECT o FROM Order o")
// Loads all orders together with their customer, items, and each item's product in one SQL statement — eliminates N+1 for this call site
List<Order> findAllWithDetails();
```

Or as a named graph on the entity:

```java
@Entity
@NamedEntityGraph(name = "Order.withCustomer",  // @NamedEntityGraph: defines a reusable named fetch plan on the entity class — referenced by name in repository methods instead of repeating attributePaths
    attributeNodes = @NamedAttributeNode("customer"))  // @NamedAttributeNode: names the association field to eagerly fetch when this graph is applied
// Order entity annotated with a named fetch plan; repositories can reference "Order.withCustomer" by name
public class Order { ... }

// In repository:
@EntityGraph("Order.withCustomer")  // References the named graph defined on the entity — fetches customer eagerly for this query only
List<Order> findByStatus(OrderStatus status);
```

`@EntityGraph` is more readable than `JOIN FETCH` for commonly needed combinations and can be defined once on the entity.

### Fix 3: `@BatchSize`

Instead of loading one association per query, load them in batches:

```java
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
@BatchSize(size = 25)  // Hibernate-specific: when lazily loading these items, fetch up to 25 uninitialized proxies at once via WHERE id IN (...) — 100 items = ~4 queries instead of 100
private List<OrderItem> items;  // Lazily loaded line items; fetched in batches of 25 when accessed to reduce query count
```

Hibernate collects the IDs of all unloaded entities and fetches them in a single `WHERE id IN (1, 2, 3, ..., 25)` query per batch. This doesn't fully solve N+1 but massively reduces query count.

### Fix 4: Second-level `@Query` — Load IDs First

For complex cases, load IDs with the paginated query, then batch-fetch the entities:

```java
// Step 1: paginated ID query (fast)
@Query("SELECT o.id FROM Order o WHERE o.status = :status")
List<Long> findIdsByStatus(OrderStatus status, Pageable pageable);

// Step 2: load with JOIN FETCH for the smaller set
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.customer WHERE o.id IN :ids")
List<Order> findByIdsWithCustomer(List<Long> ids);
```

### Detecting N+1

Enable SQL logging:
```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        generate_statistics: true   # enables Hibernate stats
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE   # logs parameter values
    org.hibernate.stat: DEBUG            # logs query statistics
```

Or use a tool like **p6spy** or **datasource-proxy** in tests to assert query counts:

```java
// With datasource-proxy in tests:
assertSelectCount(1);  // fails the test if more than 1 SELECT was fired
```

> **Pitfall:** N+1 doesn't always show up in unit tests — it only manifests with real data. Always test with realistic data volumes. Even a simple `findAll()` on an entity with 3 EAGER associations fires `3N + 1` queries.

> **Tricky Question:** Can `JOIN FETCH` cause a different kind of problem? Yes — a **Cartesian product problem**. If you `JOIN FETCH o.customer JOIN FETCH o.items`, and an order has 10 items, Hibernate returns 10 rows for that order (one per item), each with the full customer data duplicated. For 100 orders with 10 items each, you get 1000 rows. `DISTINCT` deduplicates in Java, but 1000 rows still travel over the network. At some point, multiple targeted queries beat one massive join.

---

## 4.9 Transactions Deep Dive

### `@Transactional` Defaults

```java
@Transactional(
    propagation = Propagation.REQUIRED,      // join existing or create new
    isolation = Isolation.DEFAULT,           // use DB default (usually READ_COMMITTED)
    readOnly = false,                        // set true for read-only service methods
    rollbackFor = RuntimeException.class,    // default: rolls back on unchecked exceptions only
    noRollbackFor = {},
    timeout = -1                             // no timeout
)
```

### How `@Transactional` Actually Works

Spring uses **AOP (Aspect-Oriented Programming)** to wrap your bean in a **proxy** at startup. When you call a method on that bean from outside the class, the call goes through the proxy, which:
1. Checks if a transaction exists (based on `propagation`)
2. Opens one if needed (calls `DataSourceTransactionManager.begin()`)
3. Calls your actual method
4. If method throws, rolls back
5. If method returns, commits

This means: **`@Transactional` only works when called from outside the class.** Calls within the same class bypass the proxy entirely.

### Propagation Types — With Real Examples

```java
@Service  // @Service: marks this class as a Spring-managed service bean — Spring creates one instance and injects it wherever needed
// Orchestrates the full order placement: saves the order, charges payment, and sends a notification
public class OrderService {

    // Wraps the entire workflow in transaction T1; payment joins T1, notification runs in its own T2 so a failed email doesn't roll back the order
    @Transactional  // REQUIRED — creates transaction T1
    public void placeOrder(Order order) {
        orderRepository.save(order);
        paymentService.charge(order);       // joins T1 (REQUIRED) — a payment failure rolls back the order too
        notificationService.notify(order);  // opens T2, suspends T1 (REQUIRES_NEW) — notification failure leaves the order intact
    }
}

// Service responsible for charging the customer's payment method
@Service
public class PaymentService {
    // Runs inside the caller's existing transaction (T1) — a failure here rolls back everything the caller did
    @Transactional(propagation = Propagation.REQUIRED)
    public void charge(Order order) {
        // runs inside T1 — if this throws, T1 rolls back entirely
    }
}

// Service responsible for sending order confirmation notifications
@Service
public class NotificationService {
    // Runs in its own independent transaction (T2) — a failed notification does not roll back the order saved in T1
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void notify(Order order) {
        // T1 is suspended, T2 is opened
        // if notify() throws and T2 rolls back, T1 is NOT affected
        // useful: even if email notification fails, order should be saved
    }
}
```

| Propagation | Existing TX | No Existing TX |
|---|---|---|
| `REQUIRED` | Join it | Create new |
| `REQUIRES_NEW` | Suspend it, create new | Create new |
| `MANDATORY` | Join it | Throw `IllegalTransactionStateException` |
| `SUPPORTS` | Join it | Run without TX |
| `NOT_SUPPORTED` | Suspend it, run without TX | Run without TX |
| `NEVER` | Throw exception | Run without TX |
| `NESTED` | Create savepoint | Create new |

`NESTED` uses a savepoint — if the nested transaction fails, it rolls back to the savepoint, but the outer transaction can still commit. This is JDBC savepoint behavior and isn't supported by all databases/providers.

### Isolation Levels — What Problem They Solve

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible |
| `READ_COMMITTED` (PG default) | Prevented | Possible | Possible |
| `REPEATABLE_READ` (MySQL default) | Prevented | Prevented | Possible |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

- **Dirty read:** TX1 reads data written by TX2 before TX2 commits. If TX2 rolls back, TX1 has phantom data.
- **Non-repeatable read:** TX1 reads a row, TX2 updates it and commits, TX1 reads the same row again — gets different values.
- **Phantom read:** TX1 runs a range query (e.g., `WHERE age > 30`), TX2 inserts a new row matching that range, TX1 runs the same query — gets an extra row.

### The 4 Transaction Pitfalls

#### 1. Self-Invocation (Most Common Interview Question)

```java
@Service
// BROKEN: both methods live in the same class — the REQUIRES_NEW on sendWelcomeEmail is silently ignored due to self-invocation
public class UserService {

    // Creates the user and tries to send a welcome email in a separate transaction — the @REQUIRES_NEW won't take effect
    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
        sendWelcomeEmail(user);  // ← this.sendWelcomeEmail() — goes directly to the real object, bypasses the Spring AOP proxy!
    }

    // Intended to run in a brand-new transaction, but since it's called via "this" the proxy is never involved
    @Transactional(propagation = Propagation.REQUIRES_NEW)  // USELESS — never invoked via proxy
    public void sendWelcomeEmail(User user) {
        // This runs in the SAME transaction as createUser, not a new one
        // If this throws, createUser's transaction also rolls back
    }
}
```

**Why:** The Spring AOP proxy wraps `UserService`. External callers call `proxy.createUser()`. But `createUser` calling `sendWelcomeEmail` is literally `this.sendWelcomeEmail()` — it goes to the real object, not the proxy. The proxy never intercepts it.

**Fix:** Extract `sendWelcomeEmail` to a separate bean (`EmailService`). Then Spring proxies `emailService.sendWelcomeEmail()`.

```java
@Service
// Corrected: sendWelcomeEmail is extracted to a separate bean so Spring can proxy it and honour REQUIRES_NEW
public class UserService {
    private final EmailService emailService;  // Separate Spring-managed bean — calls to its methods go through the AOP proxy

    // Creates the user and sends a welcome email; the email runs in its own independent transaction via EmailService
    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
        emailService.sendWelcomeEmail(user);  // goes through EmailService's Spring proxy — REQUIRES_NEW is honoured here
    }
}
```

#### 2. Checked Exceptions Don't Rollback by Default

```java
// BROKEN: InsufficientFundsException is a checked exception — Spring won't roll back on it by default, so the debit commits even if the credit fails
@Transactional
public void transferMoney(Long fromId, Long toId, BigDecimal amount) throws InsufficientFundsException {
    accountService.debit(fromId, amount);    // deducts money from the source account
    accountService.credit(toId, amount);     // if this throws InsufficientFundsException (checked),
                                             // DEFAULT behavior = COMMIT (!) — money is deducted but never credited!
}

// Fix: explicitly tell Spring that this checked exception should also trigger a rollback
@Transactional(rollbackFor = InsufficientFundsException.class)
public void transferMoney(...) throws InsufficientFundsException { ... }
// Now a checked exception also causes rollback
```

Spring's default is: rollback on `RuntimeException` and `Error`, commit on checked exceptions. This is a JEE convention that causes real production bugs. Whenever you use checked exceptions in transactional code, always specify `rollbackFor`.

#### 3. Transaction Too Wide — Holding DB Connections During Slow Operations

```java
// BAD: @Transactional spans the entire method — a DB connection is held open during the slow HTTP call
@Transactional  // BAD — holds DB connection for the entire method duration
public void processOrder(Long orderId) {
    Order order = orderRepository.findById(orderId);  // checks out a connection from the pool here
    String externalData = externalApi.call();  // HTTP call — could take 3-10 seconds; DB connection is idle but blocked
    order.enrich(externalData);
    orderRepository.save(order);
}
// Connection pool has 10 connections. Under load, 10 concurrent calls here = 0 available connections = 503 errors
```

```java
// Fix: make the external call before opening any transaction so no DB connection is held during the wait
public void processOrder(Long orderId) {
    String externalData = externalApi.call();         // no transaction open — no DB connection consumed
    updateOrderWithData(orderId, externalData);       // transaction is scoped only to the actual DB work
}

// Applies the enriched data to the order inside a short-lived transaction
@Transactional  // only holds connection during the 2 DB ops
private void updateOrderWithData(Long orderId, String data) {
    // ...but wait, self-invocation! This @Transactional won't work.
    // Move to a separate service or inject self via ApplicationContext
}
```

This runs into the self-invocation problem again — combine both lessons and use a dedicated `OrderUpdateService`.

#### 4. Swallowing Exceptions Prevents Rollback

```java
// BAD: catching and swallowing the exception hides it from Spring — the transaction commits even though the charge failed
@Transactional
public void processPayment(Payment payment) {
    try {
        paymentRepository.save(payment);            // persists the payment intent to the DB
        externalGateway.charge(payment);            // calls the payment gateway; may throw RuntimeException
    } catch (Exception e) {
        log.error("Payment failed", e);
        // Exception swallowed — Spring thinks no exception occurred → COMMITS
        // You now have a saved payment row with no actual charge against the card
    }
}

// Fix option 1: rethrow the exception so Spring's transaction interceptor can see it and roll back
} catch (Exception e) {
    log.error("Payment failed", e);
    throw e;  // re-propagates to Spring, which then rolls back the transaction
}

// Fix option 2: explicitly mark the transaction for rollback without letting the exception propagate
} catch (Exception e) {
    log.error("Payment failed", e);
    TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();  // tells Spring to roll back even though no exception escapes this method
}
```

> **Tricky Question:** Is `@Transactional` on a `private` method effective? **No.** Spring AOP proxies can only intercept `public` method calls from outside the class. `@Transactional` on private or package-private methods is silently ignored. Always put `@Transactional` on `public` methods.

> **Pitfall:** Putting `@Transactional` on the repository layer AND the service layer means service calls nest inside the repository's transaction (since `REQUIRED` joins an existing TX). This is usually fine — but if you call multiple repository methods in a service method without `@Transactional` on the service, each repository call gets its own transaction. If the second one fails, the first one already committed — partial update. Always put `@Transactional` at the service layer for multi-step operations.

---

## 4.10 HikariCP Connection Pool

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: user
    password: pass
    hikari:
      maximum-pool-size: 10           # max connections to DB
      minimum-idle: 5                 # min idle connections kept alive
      connection-timeout: 30000       # max wait for a connection (ms) before throwing
      idle-timeout: 600000            # how long idle connection can stay in pool (10 min)
      max-lifetime: 1800000           # max connection lifetime — recycle before DB kills it (30 min)
      pool-name: MyHikariPool
```

### How It Works

Creating a DB connection involves a TCP handshake, authentication, and session setup — it can take 20-100ms. A connection pool solves this by keeping a set of open connections ready to use. When your code calls `dataSource.getConnection()`, HikariCP checks out an idle connection from the pool. When your transaction ends, the connection is returned (not closed) to the pool.

Without a pool, every request would spend ~50ms just opening a connection. Under 100 req/s, that's 5000ms wasted just on setup.

### Sizing the Pool

The common formula from the HikariCP documentation:

```
max_pool_size = (number_of_cores * 2) + effective_spindle_count
```

For a 4-core server with SSDs (spindle count = 0): `max = 4*2 + 0 = 8 → round up to 10`.

Counterintuitively, **more connections is not always better**. Each DB connection uses memory on the DB server. PostgreSQL, for example, forks a process per connection. 1000 connections = 1000 DB processes. Too many can degrade DB performance and cause context-switching overhead.

Start with 10, watch your `HikariCP` metrics (available via Spring Actuator), and tune based on:
- Pool utilization (if always near max, increase)
- Connection wait time (if high, either increase pool or find slow queries)

### Pool Exhaustion — What It Looks Like

```
HikariPool-1 - Connection is not available, request timed out after 30000ms.
```

This means all 10 connections are in use and 30 seconds passed before one was returned. Root causes:
1. A transaction is holding a connection too long (long transaction, external HTTP call inside transaction)
2. A deadlock between transactions
3. Pool size genuinely too small for the load

Check with actuator endpoint: `GET /actuator/metrics/hikaricp.connections.active`

> **Pitfall:** If `max-lifetime` is longer than the DB server's `wait_timeout` (MySQL) or `tcp_keepalives_idle` (PostgreSQL), the DB may silently close a connection while it's idle in the pool. The next time it's checked out, it appears valid but throws an exception on first use. Set `max-lifetime` shorter than the DB's idle timeout. HikariCP's default `max-lifetime` of 30 minutes is safe for most configs, but always check your DB timeout settings.

> **Tricky Question:** What happens if `minimum-idle` equals `maximum-pool-size`? HikariCP will maintain exactly that many connections at all times — effectively a fixed-size pool. There's no scaling up or down. This is actually recommended by HikariCP's author for most production setups because it gives predictable behavior.

---

## Quick-Fire Tricky Questions

**Q: What is the difference between `save()` and `saveAndFlush()` in `JpaRepository`?**

`save()` adds the entity to the persistence context. The SQL may be batched and sent to the DB later — at transaction commit or when Hibernate decides to flush. `saveAndFlush()` immediately sends the SQL to the database. Use it when you need the auto-generated ID back immediately, or when a subsequent operation in the same transaction needs to see that row.

---

**Q: You call `userRepository.findById(1L)` twice in the same transaction. How many SQL queries are fired?**

**One.** The first call hits the DB and stores the entity in the first-level cache (persistence context). The second call finds it there and returns it without querying the DB. The first-level cache is per-transaction, per-EntityManager.

---

**Q: What is the difference between `getReferenceById()` and `findById()`?**

- `findById()` — immediately executes a SELECT. Returns `Optional<T>`. If not found, `Optional.empty()`.
- `getReferenceById()` — returns a lazy proxy without hitting the DB. A SELECT fires only when you access a field. If the entity doesn't exist, you get `EntityNotFoundException` when you touch the proxy.

```java
// Efficient: no SELECT for Order — you only need the reference for a FK
Order order = orderRepository.getReferenceById(orderId);
item.setOrder(order);      // sets FK column without loading Order entity
itemRepository.save(item);
```

---

**Q: What is optimistic locking and how does `@Version` work?**

Optimistic locking assumes conflicts are rare. `@Version` adds a version column to the table. Before any UPDATE, Hibernate checks that the version hasn't changed since you read it.

```java
@Entity  // Maps Product to the "product" table
// Represents a product with inventory; uses @Version for optimistic locking so concurrent stock updates don't silently overwrite each other
public class Product {
    @Id private Long id;             // Primary key — uniquely identifies this product
    private int stock;               // Current available inventory count; decremented on each purchase
    @Version private Long version;   // Optimistic lock counter — Hibernate auto-increments this on each UPDATE; never set manually
}

// Thread 1: reads product with version=5, decrements stock to 9
// Thread 2: reads same product with version=5, decrements stock to 9

// Thread 1 commits: UPDATE products SET stock=9, version=6 WHERE id=1 AND version=5 → 1 row updated ✓
// Thread 2 commits: UPDATE products SET stock=9, version=6 WHERE id=1 AND version=5 → 0 rows updated!
// → Hibernate throws OptimisticLockException → catch it and retry
```

**Pessimistic locking** (`@Lock(LockModeType.PESSIMISTIC_WRITE)`) issues `SELECT ... FOR UPDATE`, blocking other transactions from reading/writing that row. Use when conflicts are frequent or when you absolutely cannot afford a retry.

---

**Q: Why should you never use `@ManyToOne(fetch = FetchType.EAGER)` in production?**

1. Every load of the entity loads the association too — even when unused
2. A `findAll()` on 1000 rows with 3 EAGER associations fires 3000 additional queries (or 1 massive join)
3. The behavior is encoded in the mapping, not the query — you can't easily opt out of it per call site
4. If the associated entity also has EAGER associations, you get a cascading load of your entire object graph

Always use LAZY and fetch eagerly only in specific queries via `JOIN FETCH` or `@EntityGraph`.

---

**Q: What happens if you annotate a `private` method with `@Transactional`?**

Nothing. Spring AOP uses either JDK dynamic proxies or CGLIB proxies. Both can only intercept `public` method calls made through the proxy reference. Private methods are called directly — the proxy is bypassed. `@Transactional` is silently ignored. Always place `@Transactional` on `public` methods.
