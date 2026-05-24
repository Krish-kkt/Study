# 04_02 — Entity Relationships: Hibernate vs Database Responsibilities

---

## 1. Who Enforces What — `ddl-auto=validate` vs Runtime Behavior

A common point of confusion: **`ddl-auto` mode has zero effect on cascade behavior at runtime.**

`ddl-auto=validate` runs exactly once — at application startup. It checks that your entity mappings match the existing schema (column names, types, nullable constraints). If they don't match, the app refuses to start. After that, it never runs again.

All cascade operations — `CascadeType.ALL`, `orphanRemoval`, lifecycle hooks — are **Hibernate runtime behavior**, completely independent of the DDL mode.

### Two Separate Systems

| Layer | Who manages it | When it runs |
|---|---|---|
| Schema structure (columns, FKs, NOT NULL) | DB + DDL scripts (Flyway/Liquibase) | At migration time |
| Schema validation | Hibernate (`ddl-auto=validate`) | Once, at startup |
| `CascadeType` delete | Hibernate (issues SQL statements) | At runtime, per operation |
| `orphanRemoval` | Hibernate (dirty checking at commit) | At runtime, per transaction |
| `ON DELETE CASCADE` | Database engine | At runtime, when parent row is deleted |

---

## 2. What Relationship Annotations Actually Do

### The Core Problem They Solve

Your database has two tables connected by a foreign key:

```
orders table          order_item table
-----------           ----------------
id                    id
customer_id           order_id  ← FK pointing to orders.id
```

Without annotations, Hibernate treats these as two completely independent tables. `@OneToMany` / `@ManyToOne` is you telling Hibernate: *"these two classes are connected — here's how."*

### What They Enable

**1. Object graph navigation**

```java
// Without annotations — you write SQL yourself every time:
String sql = "SELECT * FROM order_item WHERE order_id = ?";
// ...manual JDBC boilerplate...

// With @OneToMany — Hibernate writes and executes that SQL for you:
order.getItems();  // Hibernate issues: SELECT * FROM order_item WHERE order_id = 1
```

**2. Declaring which side owns the FK column**

```java
// On Order — inverse side, does not own the FK:
@OneToMany(mappedBy = "order")  // "look at OrderItem.order for the FK"
private List<OrderItem> items;

// On OrderItem — owning side, holds the FK column:
@ManyToOne
@JoinColumn(name = "order_id")  // "order_id column lives in MY table"
private Order order;
```

Without `mappedBy`, Hibernate assumes a many-to-many and tries to create a third join table — which is wrong for this relationship.

**3. Enabling cascade operations**

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items;
```

These only make sense because Hibernate knows a relationship exists. Without the annotation, there is nothing to cascade along.

**4. Controlling fetch strategy**

```java
@ManyToOne(fetch = FetchType.LAZY)
private Customer customer;
```

Hibernate knows: *"when I load an Order, don't load the Customer row yet — wait until someone calls `order.getCustomer()`."*

### What They Do NOT Do

- They do **not** create FK constraints in the DB — that's your DDL scripts
- They do **not** enforce referential integrity at the DB level
- They do **not** add `ON DELETE CASCADE` to the DB schema

### One-Line Mental Model

| Annotation | What it tells Hibernate |
|---|---|
| `@ManyToOne` | "This field is a FK column in my table pointing to another table" |
| `@OneToMany` | "This list is populated by querying another table's FK column" |
| `@OneToOne` | "This field maps 1:1 — FK is either here or on the other side" |
| `@ManyToMany` | "This relationship needs a join table with two FK columns" |

---

## 3. Hibernate Cascade vs Database `ON DELETE CASCADE`

### How Hibernate Cascade Works (Default)

When you delete a parent entity, Hibernate issues **multiple SQL statements** — one per child, then the parent:

```sql
DELETE FROM order_item WHERE id = ?   -- Hibernate does this for each item
DELETE FROM orders WHERE id = ?        -- then deletes the parent
```

This is purely a Hibernate operation. The database has no cascade constraint involved.

### Enabling DB-Level Cascade

If the database has `ON DELETE CASCADE` on the FK, deleting the parent row automatically removes all child rows at the DB engine level. You only need one query:

```sql
DELETE FROM orders WHERE id = ?   -- DB cascades the rest automatically
```

To tell Hibernate to rely on this instead of issuing its own child DELETEs:

```java
@Entity
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    @OnDelete(action = OnDeleteAction.CASCADE)  // skip child DELETEs, let DB handle it
    private Order order;
}
```

With `ddl-auto=create/update`, this also adds `ON DELETE CASCADE` to the FK DDL Hibernate generates. With `ddl-auto=validate`, you add it manually in your Flyway migration.

### Trade-offs of DB Cascade

**Why most teams still let Hibernate manage it:**

**1. First-level cache goes stale**

Hibernate's `EntityManager` caches loaded entities in memory for the duration of a transaction. If the DB silently deletes `order_item` rows, Hibernate doesn't know — those objects remain alive in the cache. Reading them after the parent delete returns phantom data until the transaction ends.

**2. Lifecycle hooks are bypassed**

If you have `@PreRemove` on `OrderItem` — for audit logging or firing domain events — DB cascade skips all of it. Hibernate-managed cascade fires your hooks; DB cascade does not.

```java
@PreRemove
public void onDelete() {
    auditLog.record("OrderItem deleted: " + this.id);  // never called with DB cascade
}
```

**3. `orphanRemoval` still needs Hibernate**

DB cascade only fires when the parent row is deleted. Removing a child from a Java collection while the parent still exists is invisible to the database (covered in detail below).

**When DB cascade is the right call:**

- Very large collections (thousands of child rows) where Hibernate issuing individual DELETEs per row is a performance bottleneck
- Bulk operations via JPQL or native SQL where Hibernate cascade won't fire anyway
- No lifecycle hooks, and cache consistency within the transaction is not a concern

---

## 4. `orphanRemoval` — Removing a Child While the Parent Survives

### The Scenario

`Order #5` has 3 items. A customer removes one item — the order still exists, you are just editing it.

```java
Order order = orderRepo.findById(5L);
OrderItem itemToRemove = order.getItems().get(0);

order.removeItem(itemToRemove);  // removes from the Java list
// transaction commits...
```

### What DB `ON DELETE CASCADE` Sees

DB cascade fires on one condition only — the parent row is deleted:

```sql
DELETE FROM orders WHERE id = ?
```

In this scenario, the `orders` row was never deleted. The order still exists. The DB cascade has no reason to fire. It does nothing.

### What `orphanRemoval` Sees

Hibernate watches your Java objects during the transaction. At commit time it runs **dirty checking** — comparing the current state of your entities against what it originally loaded from the DB.

It notices:
- When loaded: `order.items` had 3 elements
- At commit: `order.items` has 2 elements
- `itemToRemove` is no longer referenced by any entity in the session
- `orphanRemoval = true` is set on the collection

Hibernate concludes: *"this child has no parent anymore — it's an orphan. Delete it."*

```sql
DELETE FROM order_item WHERE id = ?
```

### Side by Side

| Action | DB `ON DELETE CASCADE` | `orphanRemoval` |
|---|---|---|
| Delete the parent `Order` | Fires — deletes all items | Also fires (via `CascadeType.REMOVE`) |
| Remove one item from `order.getItems()` while order still exists | Does nothing | Issues DELETE for that item |

### What Happens Without `orphanRemoval`

```java
order.removeItem(itemToRemove);  // removed from Java list
// transaction commits...
```

Hibernate tries to set the FK to null:

```sql
UPDATE order_item SET order_id = NULL WHERE id = ?
```

If `order_id` is `NOT NULL` (as declared with `nullable = false` on `@JoinColumn`), this throws a constraint violation at runtime. The app crashes.

This is why `orphanRemoval = true` is the correct contract for composition relationships: *"a child without a parent should not exist — delete it immediately."*
