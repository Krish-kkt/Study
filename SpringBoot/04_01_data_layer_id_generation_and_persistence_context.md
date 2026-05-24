# 04_01 — Data Layer: ID Generation and the Persistence Context

---

## 04_01.1 SEQUENCE vs IDENTITY — The Batch Insert Gotcha

### The Core Problem with IDENTITY

`GenerationType.IDENTITY` means the database assigns the ID after each INSERT. Hibernate needs the ID before it can track the entity in its Persistence Context (covered in 04_01.2). This creates a hard constraint: **Hibernate must flush one INSERT at a time** — it cannot batch.

`GenerationType.SEQUENCE` lets Hibernate call `NEXTVAL` upfront, receive the ID before any INSERT, register the entity in the Persistence Context immediately, and batch all the INSERTs at flush time.

### What Actually Happens at the DB Level

**With IDENTITY (100 users):**

```
INSERT INTO users (name) VALUES ('Alice');  ◄── DB returns id=1
INSERT INTO users (name) VALUES ('Bob');    ◄── DB returns id=2
... 98 more individual round-trips
Total: 100 round-trips
```

**With SEQUENCE (allocationSize=50, 100 users):**

```
SELECT NEXTVAL('user_id_seq');  ◄── returns 1, Hibernate caches IDs 1..50
SELECT NEXTVAL('user_id_seq');  ◄── returns 51, Hibernate caches IDs 51..100

INSERT INTO users (id, name) VALUES (1,'Alice'), (2,'Bob'), ..., (50,...);
INSERT INTO users (id, name) VALUES (51,...), ..., (100,...);
Total: 4 DB calls instead of 100
```

### Entity Setup

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
    @SequenceGenerator(
        name = "user_seq",
        sequenceName = "user_id_seq",
        allocationSize = 50   // pre-fetches 50 IDs per NEXTVAL call; DB sequence INCREMENT BY must match
    )
    private Long id;
}
```

### Required YAML — Without This, Batching Is Silently Disabled

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true   # groups inserts by entity type before batching
        order_updates: true
```

> **Pitfall:** Setting `batch_size` while using `IDENTITY` strategy has no effect. Hibernate silently falls back to one-by-one inserts. No error is thrown — you only notice in performance metrics. Always verify with `generate_statistics: true`.

### What allocationSize Actually Does

```
allocationSize = 50 means:
  - DB sequence has INCREMENT BY 50
  - First NEXTVAL returns 1 → Hibernate owns IDs 1..50 in memory
  - Next 49 inserts use IDs from memory — zero DB calls
  - 51st insert → NEXTVAL returns 51 → Hibernate owns IDs 51..100
```

> **Pitfall:** If you manage migrations manually (Flyway/Liquibase), your DB sequence `INCREMENT BY` must match `allocationSize`. A mismatch causes ID collisions across multiple app instances — silent data corruption.

### DDL Validate — You Must Create the Sequence Manually

With `spring.jpa.hibernate.ddl-auto=validate`, Hibernate validates tables and columns at startup but **does not create sequences**. Create it in a migration:

```sql
-- V2__add_user_id_sequence.sql
CREATE SEQUENCE user_id_seq
    START WITH 1
    INCREMENT BY 50;   -- must match allocationSize
```

Hibernate does not validate that the sequence exists or its increment value — startup succeeds even if the sequence is missing. The bug only surfaces at runtime on first insert.

---

## 04_01.2 The Persistence Context — Why Hibernate Buffers Instead of Firing Immediately

### What It Is

The Persistence Context is **Hibernate's in-memory map for the current transaction**:

```
Map<(EntityType + ID), Entity>

┌────────────────────────────────────────────────┐
│  (User, id=1)  →  User{name="Alice", ...}      │
│  (User, id=2)  →  User{name="Bob", ...}        │
│  (Order, id=5) →  Order{total=200}             │
└────────────────────────────────────────────────┘
```

The ID is the key. **Hibernate cannot register an entity without an ID** — this is why IDENTITY blocks batching.

### Why Not Fire SQL Immediately?

If Hibernate sent SQL on every operation, the same code would produce:

```java
user.setName("Alice");    // ──► UPDATE users SET name='Alice' WHERE id=1
user.setEmail("a@x.com"); // ──► UPDATE users SET email='a@x.com' WHERE id=1
user.setStatus(ACTIVE);   // ──► UPDATE users SET status='ACTIVE' WHERE id=1
// 3 round-trips for one logical change
```

The Persistence Context buffers everything and sends the minimum, correctly ordered SQL at flush time. Below are the five things this enables.

---

### Benefit 1: Write Coalescing — Multiple Changes Become One SQL

```java
user.setName("Alice");
user.setEmail("alice@x.com");
user.setStatus(ACTIVE);

// Hibernate diffs current state vs the snapshot taken when entity was loaded
// Sends exactly ONE UPDATE:
// ──► UPDATE users SET name='Alice', email='alice@x.com', status='ACTIVE' WHERE id=1
```

If you change a field 10 times inside a loop, only the final state is sent.

---

### Benefit 2: Correct SQL Ordering — Prevents FK Constraint Violations

```java
// You write these in this order:
orderItemRepository.delete(item);   // delete child
orderRepository.save(newOrder);     // save parent referencing item's table
```

Firing DELETE before INSERT of a referencing row would violate FK constraints. Hibernate reorders at flush:

```
Flush order Hibernate guarantees:
  1. INSERT new entities      (parents before children)
  2. UPDATE existing entities
  3. DELETE removed entities  (children before parents)
```

---

### Benefit 3: First-Level Cache — No Duplicate Queries Within a Transaction

```java
User u1 = userRepository.findById(1L);  // ──► SELECT fires, entity cached
User u2 = userRepository.findById(1L);  // ──► returns from cache, NO SELECT

u1 == u2  // TRUE — same Java object, not two copies
```

No risk of working with two stale copies of the same row within one transaction.

---

### Benefit 4: Dirty Tracking — You Never Manually Call UPDATE

```java
@Transactional
public void updateName(Long id, String newName) {
    User user = userRepository.findById(id).get();
    // Hibernate snapshots state: {name="Alice", email="a@x.com"}

    user.setName(newName);   // no save() needed

    // At commit: Hibernate diffs snapshot vs current state
    // ──► UPDATE users SET name='Bob' WHERE id=1
}
```

---

### Benefit 5: Transaction Atomicity — Nothing Reaches DB Until Commit

```java
// Without buffering — partial data on exception:
orderRepository.save(order);        // INSERT fires immediately ✓
paymentRepository.save(payment);    // INSERT fires immediately ✓
inventoryService.deduct(items);     // throws RuntimeException
// order and payment rows are already committed — data inconsistent

// With Persistence Context buffering:
// All three are queued. Exception → rollback → buffer discarded → DB untouched
```

---

## 04_01.3 Custom ID Generation (Timestamp + Random)

### Why This Approach

If you assign the ID in Java before calling `save()`, Hibernate knows the ID immediately — no sequence call needed, no IDENTITY flush needed. **Batch inserts work exactly like SEQUENCE.**

A ULID (Universally Unique Lexicographically Sortable Identifier) is the standardized form of timestamp + random:

```xml
<dependency>
    <groupId>com.github.f4b6a3</groupId>
    <artifactId>ulid-creator</artifactId>
    <version>5.2.0</version>
</dependency>
```

```java
user.setId(UlidCreator.getUlid().toString());
// "01ARZ3NDEKTSV4RRFFQ69G5FAV"
// First 10 chars = timestamp (sortable), last 16 = random (collision-safe)
```

| | DB SEQUENCE | Custom (ULID / Timestamp+Random) |
|---|---|---|
| DB setup needed | Yes — `CREATE SEQUENCE` | No |
| Batch inserts | Yes | Yes |
| Sortable by insert time | No | Yes |
| Reveals insert time in ID | No | Yes — consider for public APIs |
| Collision risk | Zero | Near-zero (depends on random bits) |

---

## 04_01.4 The `save()` persist vs merge Problem

### How Spring Data Decides: `persist` or `merge`

```java
// SimpleJpaRepository (Spring Data source)
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        em.persist(entity);   // just INSERT
        return entity;
    } else {
        return em.merge(entity);  // SELECT first, then INSERT or UPDATE
    }
}
```

Default `isNew()` check:

```java
public boolean isNew(T entity) {
    return getId(entity) == null;  // null ID → new; non-null → might already exist
}
```

### The Custom ID Trap

```java
User user = new User();
user.setId("01HX3K9_a7f3");   // ID is not null
userRepository.save(user);

// isNew() → false (ID is set)
// → em.merge(user)
//     → check Persistence Context → not there
//     ──► SELECT * FROM users WHERE id = '01HX3K9_a7f3'   ← extra DB call
//     ◄── 0 rows
//     ──► INSERT INTO users ...
// Total: SELECT + INSERT instead of just INSERT
```

For 1000 bulk imports this doubles DB calls: 2000 instead of 1000.

### Full Decision Flow

```
save(entity)
│
├─ isNew?  (id == null  OR  Persistable.isNew() == true)
│   YES → em.persist() → just INSERT
│
└─ NOT new?
    → em.merge()
        ├─ Check Persistence Context
        ├─ Not there → SELECT from DB
        │   ├─ Found → UPDATE
        │   └─ Not found → INSERT
        └─ Returns a NEW managed copy — the original object is now detached!
```

> **Pitfall:** `merge()` returns a new managed object. If you ignore the return value and modify the original, those changes are NOT tracked. Always use the returned object: `User saved = userRepository.save(user);`

---

## 04_01.5 Fixing Custom IDs with `Persistable<T>`

```java
@Entity
@Table(name = "users")
public class User implements Persistable<String> {

    @Id
    private String id;

    private String name;

    @Transient               // not a DB column — lives only in memory
    private boolean isNew = true;

    @Override
    public boolean isNew() {
        return isNew;        // Spring Data calls this instead of checking id == null
    }

    @PostPersist             // fires after INSERT — flip flag so subsequent save() calls UPDATE
    @PostLoad                // fires after loading from DB — flip flag so loaded entities UPDATE correctly
    void markNotNew() {
        this.isNew = false;
    }

    @Override
    public String getId() {
        return id;
    }
}
```

### Why Both `@PostPersist` and `@PostLoad` Are Needed

| Scenario | `isNew` at creation | What fires | `isNew` after |
|---|---|---|---|
| `new User()` in Java | `true` (field initializer) | — | `true` |
| `save()` on new user | `true` | `persist()` → INSERT → `@PostPersist` | `false` |
| `findById()` from DB | `true` (reflection + field initializer) | `@PostLoad` | `false` |
| `save()` on loaded user | `false` | `merge()` → UPDATE | `false` |

When Hibernate loads from DB it uses reflection — **field initializers still run** (`isNew = true`). Without `@PostLoad`, a loaded entity would have `isNew = true`, and calling `save()` on it would try to INSERT a duplicate row — **primary key violation**.

> **Pitfall:** Removing `@PostLoad` breaks updates on loaded entities. `save(loadedUser)` would call `persist()` and throw a duplicate key exception because Hibernate thinks it's a new entity.

---

## Quick-Fire Tricky Questions

**Q: I set `batch_size: 50` in YAML but bulk inserts are still slow. Why?**

You're probably using `GenerationType.IDENTITY`. `batch_size` has no effect with IDENTITY — Hibernate cannot batch because it needs each INSERT to return the generated key before it can proceed. Switch to `SEQUENCE` or custom IDs.

**Q: With `ddl-auto=validate`, will Hibernate catch a wrong `INCREMENT BY` on my sequence?**

No. Hibernate only validates tables and columns — not sequences or their increment values. A mismatched `INCREMENT BY` passes startup validation and only causes ID collisions at runtime.

**Q: Why does `save()` return a value? Can I ignore it?**

For new entities (`persist()`), the return value is the same object you passed in — safe to ignore. For existing entities (`merge()`), the return value is a **new managed object** — the one you passed in becomes detached. Modifying the original after `merge()` has no effect. Always capture the return: `User saved = userRepository.save(user)`.

**Q: Can I use ULID as a `Long` ID?**

No. ULID is a 128-bit value, typically stored as a 26-character string or a `UUID`. If you need numeric IDs, use TSID (Time-Sorted Unique ID) — it fits in a `Long` and is sortable.
