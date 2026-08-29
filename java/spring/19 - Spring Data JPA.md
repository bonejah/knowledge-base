# 19 - Spring Data JPA

## 1. What is JDBC?

**JDBC (Java Database Connectivity)** is the standard Java API for communicating with relational databases.

It provides a low-level API that allows a Java application to:

- Open database connections
- Execute SQL statements
- Read query results
- Insert, update, and delete records
- Manage transactions

### Basic JDBC Example

```java
String sql = "SELECT id, name, email FROM users";

try (Connection connection = dataSource.getConnection();
     PreparedStatement statement = connection.prepareStatement(sql);
     ResultSet resultSet = statement.executeQuery()) {

    while (resultSet.next()) {
        Long id = resultSet.getLong("id");
        String name = resultSet.getString("name");
        String email = resultSet.getString("email");

        System.out.println(id + " - " + name + " - " + email);
    }
}
```

With JDBC, the developer is responsible for many low-level details:

```text
Java Application
      |
      v
   JDBC API
      |
      v
Database Driver
      |
      v
   Database
```

### Problems with JDBC

JDBC is powerful, but applications can become repetitive:

```java
Connection
PreparedStatement
ResultSet
SQL
ResultSet mapping
Exception handling
Connection management
```

For example, every query requires manually converting database rows into Java objects.

This is one of the problems that ORM frameworks attempt to solve.

---

# 2. What is Spring JDBC?

**Spring JDBC** is a Spring abstraction over JDBC that simplifies database access.

The main component is `JdbcTemplate`.

Instead of manually managing connections and statements:

```java
jdbcTemplate.query(
    "SELECT id, name, email FROM users",
    (rs, rowNum) -> new User(
        rs.getLong("id"),
        rs.getString("name"),
        rs.getString("email")
    )
);
```

Spring manages much of the boilerplate.

### JdbcTemplate

Typical architecture:

```text
Application
     |
     v
JdbcTemplate
     |
     v
JDBC
     |
     v
Database Driver
     |
     v
Database
```

Spring JDBC is still fundamentally **SQL-based**.

You write SQL:

```sql
SELECT * FROM users WHERE email = ?
```

and Spring helps execute it.

---

# 3. What is JPA?

**JPA (Java Persistence API)** is a Java specification for object-relational persistence.

JPA defines concepts such as:

- Entities
- Entity lifecycle
- Relationships
- Persistence context
- EntityManager
- JPQL
- Transactions
- Mapping Java objects to database tables

JPA itself is **not an implementation**.

It defines a standard API and behavior.

Examples of JPA implementations include:

- Hibernate
- EclipseLink
- OpenJPA

A common misconception is:

> "JPA is Hibernate."

They are not the same thing.

A better analogy is:

```text
JPA
 |
 | specification
 v
Hibernate
 |
 | implementation
 v
Database
```

---

# 4. What is ORM?

**ORM (Object-Relational Mapping)** is a technique for mapping objects in an object-oriented programming language to relational database structures.

For example:

### Java

```java
public class User {

    private Long id;
    private String name;
    private String email;
}
```

### Database

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255)
);
```

ORM creates a conceptual mapping:

```text
Java Object              Database
-----------              --------
User              --->   users
id                --->   id
name              --->   name
email             --->   email
```

The goal is to reduce the amount of SQL and manual object mapping developers need to write.

---

# 5. What is Hibernate?

**Hibernate** is an ORM framework and one of the most popular implementations of JPA.

Hibernate is responsible for actually performing persistence operations.

For example:

```java
entityManager.persist(user);
```

The developer doesn't explicitly write:

```sql
INSERT INTO users ...
```

Hibernate can generate the SQL required to persist the entity.

Conceptually:

```text
Java Entity
    |
    v
JPA API
    |
    v
Hibernate
    |
    v
JDBC
    |
    v
Database
```

Spring Boot commonly uses Hibernate as the JPA provider.

---

# 6. Spring Data JPA

**Spring Data JPA** is a Spring project that makes it easier to build repositories using JPA.

It sits at a higher abstraction level than JPA.

```text
Spring Data JPA
       |
       v
      JPA
       |
       v
   Hibernate
       |
       v
      JDBC
       |
       v
    Database
```

Spring Data JPA provides features such as:

- Repository interfaces
- CRUD operations
- Query methods
- Pagination
- Sorting
- `@Query`
- Integration with Spring transactions
- Reduced boilerplate

Instead of implementing:

```java
save()
findById()
findAll()
delete()
```

you can simply extend a repository interface.

```java
public interface UserRepository
        extends JpaRepository<User, Long> {
}
```

Spring Data generates the implementation automatically.

---

# 7. JDBC vs Spring JDBC vs JPA vs Spring Data JPA

| Technology      | Main Responsibility       | SQL Required? | Abstraction |
| --------------- | ------------------------- | ------------: | ----------- |
| JDBC            | Direct database access    |           Yes | Low         |
| Spring JDBC     | Simplify JDBC             |           Yes | Low/Medium  |
| JPA             | Persistence specification |    Usually no | High        |
| Hibernate       | JPA/ORM implementation    |    Usually no | High        |
| Spring Data JPA | Simplify JPA repositories |    Usually no | Very High   |

A useful way to remember it:

```text
JDBC
  ↓
Spring JDBC
  ↓
JPA
  ↓
Hibernate
  ↓
Spring Data JPA
```

However, this is not a strict replacement hierarchy.

Spring Data JPA uses JPA, and Hibernate is commonly the JPA provider underneath it.

---

# 8. JPA Architecture

A typical Spring Boot application using Spring Data JPA looks like this:

```text
+-----------------------------+
|       REST Controller       |
+-----------------------------+
              |
              v
+-----------------------------+
|       Service Layer         |
+-----------------------------+
              |
              v
+-----------------------------+
|    Spring Data Repository   |
+-----------------------------+
              |
              v
+-----------------------------+
|            JPA              |
|        EntityManager        |
+-----------------------------+
              |
              v
+-----------------------------+
|          Hibernate          |
|        JPA Provider         |
+-----------------------------+
              |
              v
+-----------------------------+
|            JDBC             |
+-----------------------------+
              |
              v
+-----------------------------+
|          Database           |
+-----------------------------+
```

### Important components

#### Entity

Represents a persistent object.

```java
@Entity
public class User {
}
```

#### EntityManager

JPA's main API for managing entities.

```java
@PersistenceContext
private EntityManager entityManager;
```

#### Persistence Context

A managed collection of entity instances associated with an `EntityManager`.

It is one of the most important concepts in JPA.

#### Hibernate

The implementation that performs ORM and generates SQL.

#### Database

The actual persistence layer.

---

# 9. Persistence Context

The **Persistence Context** is a central concept in JPA.

It acts as a first-level cache and keeps track of managed entities.

For example:

```java
User user = entityManager.find(User.class, 1L);
```

The entity becomes managed by the persistence context.

If we modify it:

```java
user.setName("John");
```

we don't necessarily need:

```java
entityManager.update(user);
```

Instead, Hibernate can detect the change and synchronize it with the database when the transaction is committed.

This mechanism is called:

**Dirty Checking**

```text
Database
   |
   v
EntityManager
   |
   v
Persistence Context
   |
   v
Managed Entity
   |
   | change
   v
Dirty Checking
   |
   v
SQL UPDATE
```

---

# 10. Entity Lifecycle

JPA entities have different lifecycle states.

The four main states are:

```text
Transient
    |
    | persist()
    v
Persistent / Managed
    |
    | detach()
    v
Detached
    |
    | remove()
    v
Removed
```

---

## 10.1 Transient

An object is transient when it has been created but is not associated with the persistence context.

```java
User user = new User();

user.setName("John");
user.setEmail("john@example.com");
```

At this point:

```text
Java Object
    |
    X
Database
```

Nothing has been persisted.

---

## 10.2 Managed

An entity becomes managed when it is associated with a persistence context.

For example:

```java
entityManager.persist(user);
```

Now:

```text
User
 |
 v
Persistence Context
```

Hibernate tracks changes to the entity.

---

## 10.3 Detached

An entity becomes detached when it is no longer associated with the persistence context.

For example:

```java
entityManager.detach(user);
```

Changes to the detached entity are not automatically synchronized with the database.

---

## 10.4 Removed

An entity is marked for deletion:

```java
entityManager.remove(user);
```

The actual SQL `DELETE` is normally executed when the persistence context is flushed.

---

# 11. Understanding JPA Internals

Consider:

```java
@Transactional
public void updateUser(Long id) {

    User user = userRepository.findById(id)
            .orElseThrow();

    user.setName("John");
}
```

There is no explicit:

```java
userRepository.save(user);
```

Yet the database can be updated.

Why?

### Step 1

The transaction starts.

```text
Transaction START
```

### Step 2

JPA loads the entity.

```text
Database
   |
   v
Hibernate
   |
   v
Persistence Context
   |
   v
User
```

### Step 3

The entity is managed.

```java
user.setName("John");
```

### Step 4

Hibernate detects the change through dirty checking.

### Step 5

The transaction commits.

Hibernate generates:

```sql
UPDATE users
SET name = 'John'
WHERE id = ?;
```

### Step 6

Transaction completes.

```text
Transaction COMMIT
```

This is one of the most important differences between JPA and simple JDBC-style programming.

---

# 12. Setting Up a Spring Data JPA Application

For our example, imagine a simple application called:

**Online Store**

The application contains:

```text
Customer
Order
Product
```

We will use:

```text
Spring Boot
Spring Data JPA
Hibernate
PostgreSQL
```

A typical dependency is:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

And a PostgreSQL driver:

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

# 13. Creating Entities

An entity represents a persistent object.

```java
@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String email;

    // constructors
    // getters
    // setters
}
```

This creates a conceptual mapping:

```text
Customer
   |
   v
customers table
```

---

# 14. JPA Annotations

Some of the most common annotations are:

### `@Entity`

Marks a class as a JPA entity.

```java
@Entity
public class Customer {
}
```

---

### `@Table`

Defines the database table.

```java
@Table(name = "customers")
```

---

### `@Id`

Defines the primary key.

```java
@Id
private Long id;
```

---

### `@GeneratedValue`

Defines how the ID is generated.

```java
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

---

### `@Column`

Customizes column mapping.

```java
@Column(name = "customer_email", nullable = false)
private String email;
```

---

### `@Transient`

Marks a field that should not be persisted.

```java
@Transient
private String temporaryValue;
```

---

### `@Enumerated`

Maps Java enums to database values.

Prefer:

```java
@Enumerated(EnumType.STRING)
private CustomerStatus status;
```

instead of:

```java
@Enumerated(EnumType.ORDINAL)
```

`STRING` is generally safer because changing enum order won't change the stored meaning.

---

# 15. Spring Data JPA Repositories

Repositories provide the data-access abstraction.

```java
public interface CustomerRepository
        extends JpaRepository<Customer, Long> {
}
```

There is no implementation class.

Spring Data generates it.

Conceptually:

```text
CustomerRepository
        |
        v
Spring Data JPA
        |
        v
Generated Implementation
        |
        v
JPA / Hibernate
```

---

# 16. CrudRepository

`CrudRepository` provides basic CRUD operations.

```java
public interface CustomerRepository
        extends CrudRepository<Customer, Long> {
}
```

Common methods:

```java
save()
findById()
findAll()
existsById()
count()
deleteById()
delete()
deleteAll()
```

Example:

```java
Customer customer = new Customer();

customer.setName("John");
customer.setEmail("john@example.com");

customerRepository.save(customer);
```

---

# 17. PagingAndSortingRepository

`PagingAndSortingRepository` provides pagination and sorting capabilities.

For example:

```java
Page<Customer> customers =
        repository.findAll(
            PageRequest.of(0, 20)
        );
```

This means:

```text
Page 0
20 records
```

Sorting:

```java
PageRequest.of(
    0,
    20,
    Sort.by("name").ascending()
);
```

---

# 18. JpaRepository

`JpaRepository` provides JPA-specific repository functionality and is the most commonly used repository interface in Spring Boot applications.

```java
public interface CustomerRepository
        extends JpaRepository<Customer, Long> {
}
```

It provides CRUD functionality plus features such as:

- Pagination
- Sorting
- Batch-related operations
- JPA-specific behavior

In many applications:

```java
JpaRepository
```

is the default choice.

---

# 19. Repository Hierarchy

Conceptually:

```text
Repository
    |
    v
CrudRepository
    |
    v
PagingAndSortingRepository
    |
    v
JpaRepository
```

The exact inheritance structure can vary between Spring Data versions, but the important idea is that more specialized repositories provide additional capabilities.

---

# 20. Inbuilt CRUD Methods

Spring Data provides many methods without requiring implementation.

### Create

```java
customerRepository.save(customer);
```

### Read

```java
customerRepository.findById(id);
```

```java
customerRepository.findAll();
```

### Update

```java
customer.setName("New Name");

customerRepository.save(customer);
```

### Delete

```java
customerRepository.deleteById(id);
```

---

# 21. Query Methods

One of the most useful Spring Data JPA features is **derived query methods**.

Spring Data can create queries based on method names.

Example:

```java
List<Customer> findByName(String name);
```

Spring Data interprets the method name and generates the appropriate query.

Another example:

```java
Optional<Customer> findByEmail(String email);
```

Conceptually:

```sql
SELECT *
FROM customers
WHERE email = ?;
```

---

# 22. More Query Method Examples

### Find by two fields

```java
Optional<Customer> findByNameAndEmail(
    String name,
    String email
);
```

### Find by partial text

```java
List<Customer> findByNameContaining(String name);
```

### Find by starting text

```java
List<Customer> findByNameStartingWith(String prefix);
```

### Greater than

```java
List<Product> findByPriceGreaterThan(BigDecimal price);
```

### Less than

```java
List<Product> findByPriceLessThan(BigDecimal price);
```

### Ordering

```java
List<Customer> findByNameOrderByNameAsc();
```

### Multiple conditions

```java
List<Customer> findByNameContainingAndEmailContaining(
    String name,
    String email
);
```

---

# 23. When Query Methods Become a Problem

Derived query methods are convenient:

```java
findByNameAndEmailAndStatusAndCreatedAtAfter(...)
```

But extremely complex method names can become difficult to read and maintain.

For example:

```java
findByNameContainingAndStatusAndCreatedAtBetweenAndEmailContainingOrderByNameAsc(...)
```

At some point, `@Query` or a more advanced query mechanism becomes more appropriate.

---

# 24. @Query

The `@Query` annotation allows us to define queries explicitly.

Example:

```java
@Query("""
    SELECT c
    FROM Customer c
    WHERE c.email = :email
""")
Optional<Customer> findCustomerByEmail(
    @Param("email") String email
);
```

This uses **JPQL**, not SQL.

---

# 25. JPQL vs SQL

SQL operates primarily on tables:

```sql
SELECT *
FROM customers
WHERE email = ?;
```

JPQL operates on entities:

```java
SELECT c
FROM Customer c
WHERE c.email = :email
```

The important difference is:

```text
SQL  -> Database tables and columns

JPQL -> Entities and their fields
```

Hibernate translates JPQL into database-specific SQL.

---

# 26. Native Queries

Sometimes you need database-specific SQL.

You can use:

```java
@Query(
    value = """
        SELECT *
        FROM customers
        WHERE email = :email
    """,
    nativeQuery = true
)
Optional<Customer> findByEmailNative(
    @Param("email") String email
);
```

This executes SQL directly.

Use native queries carefully because they reduce database portability.

---

# 27. Modifying Queries

For `UPDATE` and `DELETE` queries, use `@Modifying`.

Example:

```java
@Modifying
@Query("""
    UPDATE Customer c
    SET c.status = :status
    WHERE c.id = :id
""")
int updateStatus(
    @Param("id") Long id,
    @Param("status") CustomerStatus status
);
```

Usually this operation should execute inside a transaction:

```java
@Transactional
```

---

# 28. JPA Relationships

Real-world applications rarely consist of isolated entities.

Entities usually have relationships.

The four major JPA relationships are:

```text
One-to-One
One-to-Many
Many-to-One
Many-to-Many
```

---

# 29. One-to-One

A One-to-One relationship means:

> One entity is associated with one other entity.

Example:

```text
Customer
   |
   | 1
   |
   | 1
   v
CustomerProfile
```

Example:

```java
@Entity
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    @OneToOne
    private CustomerProfile profile;
}
```

---

# 30. One-to-Many

One entity is associated with multiple entities.

Example:

```text
Customer
   |
   | 1
   |
   +---- Order
   |
   +---- Order
   |
   +---- Order
```

```java
@OneToMany
private List<Order> orders;
```

A customer can have many orders.

---

# 31. Many-to-One

Many entities belong to one entity.

For example:

```text
Order
   |
   +---- Customer
   |
   +---- Customer
   |
   +---- Customer
```

Actually, the relationship is:

```text
Customer 1 <------ N Orders
```

The `Order` entity commonly owns the foreign key:

```java
@ManyToOne
@JoinColumn(name = "customer_id")
private Customer customer;
```

This is one of the most common relationships in real applications.

---

# 32. One-to-Many + Many-to-One

These two annotations are frequently used together.

### Customer

```java
@OneToMany(mappedBy = "customer")
private List<Order> orders;
```

### Order

```java
@ManyToOne
@JoinColumn(name = "customer_id")
private Customer customer;
```

Database:

```text
customers
----------------
id
name
email


orders
----------------
id
customer_id
total
```

The foreign key is stored in:

```text
orders.customer_id
```

---

# 33. Many-to-Many

A Many-to-Many relationship means:

> Many records can be associated with many other records.

Example:

```text
Student              Course

Student A ---------- Course A
     |  \             |
     |   \------------ Course B
     |
Student B ---------- Course B
```

A common implementation uses a join table:

```text
students
courses
student_courses
```

Example:

```java
@ManyToMany
@JoinTable(
    name = "student_courses",
    joinColumns = @JoinColumn(name = "student_id"),
    inverseJoinColumns = @JoinColumn(name = "course_id")
)
private Set<Course> courses;
```

Database:

```text
students
---------
id
name


courses
---------
id
name


student_courses
---------------
student_id
course_id
```

---

# 34. Owning Side of a Relationship

One of the most important concepts in JPA relationships is the **owning side**.

The owning side is responsible for managing the relationship mapping, particularly the foreign key or join table.

For example:

```java
@ManyToOne
@JoinColumn(name = "customer_id")
private Customer customer;
```

Here, `Order` is the owning side.

The `Customer` side can use:

```java
@OneToMany(mappedBy = "customer")
private List<Order> orders;
```

The `mappedBy` tells JPA:

> The other entity owns this relationship.

---

# 35. Cascade Operations

Cascade controls whether operations performed on one entity should propagate to related entities.

Example:

```java
@OneToMany(
    mappedBy = "customer",
    cascade = CascadeType.ALL
)
private List<Order> orders;
```

Possible cascade types include:

```text
PERSIST
MERGE
REMOVE
REFRESH
DETACH
ALL
```

Be careful with:

```java
CascadeType.ALL
```

especially when it includes:

```text
REMOVE
```

For example, deleting a customer could potentially delete all associated orders.

Cascade should be chosen based on the actual ownership/lifecycle relationship between entities.

---

# 36. Fetch Types

JPA relationships can use different fetching strategies.

### EAGER

Load the relationship immediately.

```java
@ManyToOne(fetch = FetchType.EAGER)
```

### LAZY

Load the relationship only when needed.

```java
@ManyToOne(fetch = FetchType.LAZY)
```

In general, prefer **LAZY loading** where practical and explicitly fetch related data when you actually need it.

This helps avoid unnecessarily loading large object graphs.

---

# 37. The N+1 Query Problem

One of the most common JPA performance problems is the **N+1 query problem**.

Suppose:

```java
List<Customer> customers =
    customerRepository.findAll();
```

Then:

```java
for (Customer customer : customers) {
    customer.getOrders().size();
}
```

Depending on the mapping and query strategy, this can result in:

```text
1 query -> load customers

N queries -> load orders for each customer
```

For 100 customers:

```text
1 + 100 = 101 queries
```

This can seriously affect performance.

Solutions can include:

- `JOIN FETCH`
- Entity graphs
- DTO projections
- Carefully designed queries
- Batch fetching

---

# 38. JOIN FETCH

Example:

```java
@Query("""
    SELECT c
    FROM Customer c
    JOIN FETCH c.orders
    WHERE c.id = :id
""")
Optional<Customer> findCustomerWithOrders(
    @Param("id") Long id
);
```

This tells Hibernate to retrieve the customer and its orders in the same query.

The goal is to avoid unnecessary additional queries.

---

# 39. Pagination

Never assume that loading thousands or millions of database records into memory is acceptable.

Instead of:

```java
List<Customer> customers =
    customerRepository.findAll();
```

use pagination:

```java
Page<Customer> customers =
    customerRepository.findAll(
        PageRequest.of(0, 50)
    );
```

This requests:

```text
Page: 0
Size: 50
```

Pagination is especially important for:

- REST APIs
- Admin screens
- Search endpoints
- Large tables
- Reporting systems

---

# 40. Transactions and Spring Data JPA

Spring Data JPA works closely with Spring's transaction management.

Example:

```java
@Transactional
public void createOrder(Order order) {

    orderRepository.save(order);

    // additional database operations
}
```

The transaction ensures that the operations participate in the same transactional context.

For example:

```text
BEGIN TRANSACTION

Save Order
Save Order Items
Update Inventory

COMMIT
```

If an appropriate exception causes rollback:

```text
BEGIN TRANSACTION

Save Order
Save Order Items
Update Inventory

ERROR

ROLLBACK
```

---

# 41. Read-Only Transactions

For operations that only read data, you can use:

```java
@Transactional(readOnly = true)
```

Example:

```java
@Transactional(readOnly = true)
public Customer getCustomer(Long id) {
    return customerRepository.findById(id)
            .orElseThrow();
}
```

This communicates the intent that the transaction is read-only.

However, `readOnly = true` is primarily an optimization/hint rather than an absolute guarantee that no write can ever occur.

---

# 42. Entity Design Best Practices

### Use a generated ID when appropriate

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

### Avoid exposing entities directly from APIs

Prefer:

```text
Entity
  |
  v
Service
  |
  v
DTO
  |
  v
Controller
```

instead of returning JPA entities directly.

---

### Be careful with Lombok

Avoid blindly using:

```java
@Data
```

on entities.

Generated `equals()`, `hashCode()`, and `toString()` can cause problems with:

- Lazy relationships
- Bidirectional relationships
- Proxy objects
- Recursive calls
- Performance

---

# 43. Avoid Bidirectional Relationships Unless Needed

It is tempting to create:

```java
Customer -> orders
Order -> customer
```

This can be useful, but bidirectional relationships increase complexity.

For example:

```text
Customer
   |
   v
Orders
   |
   v
Customer
   |
   v
Orders
   |
   ...
```

This can cause problems with:

- JSON serialization
- `toString()`
- `equals()`
- `hashCode()`
- Unexpected lazy loading

Use bidirectional relationships when they provide real value.

---

# 44. Keep the Service Layer in Control

A common architecture is:

```text
Controller
    |
    v
Service
    |
    v
Repository
    |
    v
Database
```

Example:

```java
@Service
public class CustomerService {

    private final CustomerRepository repository;

    public CustomerService(CustomerRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public Customer create(Customer customer) {
        return repository.save(customer);
    }

    @Transactional(readOnly = true)
    public Customer findById(Long id) {
        return repository.findById(id)
                .orElseThrow();
    }
}
```

The service layer is a good place to define business transactions.

---

# 45. Don't Treat JPA as Magic

JPA reduces boilerplate, but SQL still exists underneath.

When you write:

```java
customerRepository.findByEmail(email);
```

the database still executes SQL.

Conceptually:

```text
Repository
    |
    v
Spring Data
    |
    v
JPA
    |
    v
Hibernate
    |
    v
SQL
    |
    v
Database
```

Understanding the generated SQL is extremely important for performance troubleshooting.

---

# 46. Common JPA Problems

## 46.1 N+1 Queries

```text
1 query + N queries
```

Solution:

- Fetch joins
- Entity graphs
- DTO projections
- Batch fetching

---

## 46.2 Loading Too Much Data

Avoid:

```java
findAll()
```

when the table contains millions of rows.

Use:

```java
Page<T>
```

or projections.

---

## 46.3 EAGER Relationships

Large object graphs can be loaded unintentionally.

Prefer lazy relationships where appropriate.

---

## 46.4 LazyInitializationException

This can happen when accessing a lazy relationship after the persistence context is no longer available.

Example:

```java
customer.getOrders();
```

outside the appropriate persistence context.

The solution is not simply:

```java
FetchType.EAGER
```

Instead, design the query/service boundary so that the required data is fetched intentionally.

---

# 47. Entity vs DTO

A JPA entity represents persistence.

A DTO represents data exchanged between application boundaries.

Example:

```java
public record CustomerResponse(
    Long id,
    String name,
    String email
) {
}
```

A controller can return:

```text
Customer Entity
      |
      v
CustomerResponse DTO
      |
      v
HTTP Response
```

This prevents the API contract from being tightly coupled to the database model.

---

# 48. A Complete Example

### Entity

```java
@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    // getters and setters
}
```

### Repository

```java
public interface CustomerRepository
        extends JpaRepository<Customer, Long> {

    Optional<Customer> findByEmail(String email);

    List<Customer> findByNameContainingIgnoreCase(String name);
}
```

### Service

```java
@Service
public class CustomerService {

    private final CustomerRepository repository;

    public CustomerService(CustomerRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public Customer create(Customer customer) {
        return repository.save(customer);
    }

    @Transactional(readOnly = true)
    public Customer findById(Long id) {
        return repository.findById(id)
                .orElseThrow();
    }
}
```

### Controller

```java
@RestController
@RequestMapping("/customers")
public class CustomerController {

    private final CustomerService service;

    public CustomerController(CustomerService service) {
        this.service = service;
    }

    @GetMapping("/{id}")
    public Customer findById(@PathVariable Long id) {
        return service.findById(id);
    }
}
```

The complete flow is:

```text
HTTP Request
     |
     v
Controller
     |
     v
Service
     |
     v
Repository
     |
     v
Spring Data JPA
     |
     v
JPA
     |
     v
Hibernate
     |
     v
JDBC
     |
     v
PostgreSQL
```

---

# 49. Spring Data JPA Mental Model

A useful mental model is:

```text
              YOUR CODE
                  |
                  v
        Spring Data Repository
                  |
                  v
                 JPA
                  |
          +-------+-------+
          |               |
          v               v
   EntityManager      Persistence
                       Context
          |
          v
       Hibernate
          |
          v
         JDBC
          |
          v
       Database
```

And for an entity:

```text
Java Object
     |
     | @Entity
     v
JPA Entity
     |
     v
Persistence Context
     |
     | dirty checking
     v
Hibernate
     |
     | generated SQL
     v
Database
```

---

# 50. Best Practices Summary

### Repository

- Prefer `JpaRepository` for most Spring Data JPA applications.
- Keep repositories focused on persistence operations.
- Use derived query methods for simple queries.
- Use `@Query` for more complex queries.
- Use native SQL only when there is a clear reason.

### Transactions

- Define transaction boundaries at the service layer.
- Use `@Transactional` for business operations requiring atomicity.
- Consider `readOnly = true` for read operations.
- Understand rollback behavior.

### Relationships

- Understand the owning side.
- Use `mappedBy` correctly.
- Avoid unnecessary bidirectional relationships.
- Be careful with cascading deletes.
- Prefer LAZY loading where appropriate.

### Performance

- Watch for N+1 queries.
- Use pagination for large datasets.
- Don't blindly use `findAll()`.
- Understand generated SQL.
- Use fetch joins, projections, or entity graphs when appropriate.

### Entities

- Keep entities focused on persistence/domain behavior.
- Avoid exposing entities directly as API contracts.
- Use DTOs for external boundaries.
- Be careful with `equals()`, `hashCode()`, and `toString()`.
- Avoid blindly using Lombok `@Data` on entities.

---

# 51. Final Comparison

```text
JDBC
 |
 | Direct database communication
 v
Spring JDBC
 |
 | JDBC with Spring abstractions
 v
JPA
 |
 | Persistence specification
 v
Hibernate
 |
 | ORM implementation
 v
Spring Data JPA
 |
 | Repository abstraction
 v
Your Application
```

The key idea is:

> **Spring Data JPA does not replace the database, SQL, JDBC, JPA, or Hibernate. It provides a higher-level repository abstraction that makes working with JPA much easier.**

Understanding what happens underneath the repository is essential.

When you write:

```java
customerRepository.findById(id);
```

you should mentally understand that there is a chain underneath:

```text
Repository
    ↓
Spring Data JPA
    ↓
JPA / EntityManager
    ↓
Persistence Context
    ↓
Hibernate
    ↓
JDBC
    ↓
SQL
    ↓
Database
```

That mental model is what allows you to use Spring Data JPA effectively rather than simply treating it as "magic CRUD."
