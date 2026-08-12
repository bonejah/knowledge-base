# 11 - Spring Transactional Annotation

## Table of Contents

1. Introduction
2. What is a Transaction?
3. ACID Principles
4. Transactional Context
5. Transaction Managers
6. Spring `@Transactional`
7. Rollback Behavior
8. Transaction Propagation
9. Isolation Levels
10. Transaction Timeouts
11. Read-Only Transactions
12. How Spring Implements Transactions
13. Limitations of `@Transactional`
14. Advantages and Disadvantages
15. Best Practices
16. Summary

---

# Introduction

In enterprise applications, data consistency is critical.

Consider the following operations:

- Money transfers
- Order creation
- Inventory updates
- Payment processing

If an application crashes in the middle of these operations, data can become inconsistent.

To solve this problem, relational databases implement **transactions**, and Spring provides the **`@Transactional`** annotation to manage them automatically.

---

# What is a Transaction?

A transaction is a logical unit of work that consists of one or more operations executed as a single indivisible unit.

A transaction has only two possible outcomes:

- **Commit** → permanently save all changes.
- **Rollback** → undo all changes.

## Example

Bank transfer:

```java
accountA.debit(100);
accountB.credit(100);
```

Both operations must succeed together.

If one operation fails:

```text
Rollback Everything
```

---

# ACID Principles

ACID is a set of properties that guarantee transaction reliability.

## A — Atomicity

A transaction is all-or-nothing.

### Example

```java
accountA.debit(100);
accountB.credit(100);
```

If the second operation fails:

```text
Rollback
```

Result:

```text
No changes persisted
```

### Benefit

Prevents partial updates.

---

## C — Consistency

A transaction must move the database from one valid state to another valid state.

### Example

Business rule:

```sql
balance >= 0
```

This operation is invalid:

```java
account.setBalance(-500);
```

Result:

```text
Transaction Rejected
```

### Benefit

Maintains business rules and constraints.

---

## I — Isolation

Concurrent transactions should not interfere with each other.

### Example

Inventory quantity:

```text
Stock = 1
```

Two users purchase simultaneously.

Without isolation:

```text
Stock = -1
```

With proper isolation:

```text
Stock = 0
```

### Benefit

Prevents concurrency issues.

---

## D — Durability

Once committed, data survives failures.

Examples:

- Server restart
- Power outage
- Application crash

```sql
COMMIT;
```

After commit:

```text
Data is permanently stored
```

### Benefit

Guarantees persistence.

---

# Transactional Context

## What is a Transactional Context?

A transactional context is the execution scope in which all database operations participate in the same transaction.

When Spring starts a transaction:

```text
Transaction Context Created
```

All repository operations executed inside that context share:

- Same transaction
- Same connection
- Same commit
- Same rollback

---

## Example

```java
@Service
public class UserService {

    @Transactional
    public void createUser() {

        userRepository.save(user);

        roleRepository.save(role);

        auditRepository.save(log);
    }
}
```

Execution:

```text
Transaction Context
    ├── userRepository.save()
    ├── roleRepository.save()
    └── auditRepository.save()
```

If any operation fails:

```text
Rollback Everything
```

---

## Why is it Important?

Without a transactional context:

```text
Operation 1 → Commit
Operation 2 → Fail
Operation 3 → Never Executes
```

Result:

```text
Inconsistent Data
```

---

# Transaction Managers

## What is a Transaction Manager?

The Transaction Manager is responsible for:

- Starting transactions
- Committing transactions
- Rolling back transactions

Spring delegates transaction management to a `PlatformTransactionManager`.

---

## PlatformTransactionManager

Core Spring transaction interface:

```java
public interface PlatformTransactionManager {
}
```

Spring automatically selects the appropriate implementation.

---

## JpaTransactionManager

Used with:

- JPA
- Hibernate
- Spring Data JPA

```java
@Bean
public PlatformTransactionManager transactionManager(
        EntityManagerFactory emf) {

    return new JpaTransactionManager(emf);
}
```

---

## DataSourceTransactionManager

Used with:

- JDBC
- JdbcTemplate

```java
@Bean
public PlatformTransactionManager transactionManager(
        DataSource ds) {

    return new DataSourceTransactionManager(ds);
}
```

---

## JtaTransactionManager

Used for distributed transactions.

Examples:

- Multiple databases
- Database + Messaging
- XA transactions

---

## Transaction Flow

```text
@Transactional
       ↓
Spring AOP Proxy
       ↓
PlatformTransactionManager
       ↓
Database
```

---

# Spring `@Transactional`

## What is `@Transactional`?

`@Transactional` is a Spring annotation that defines transactional boundaries.

Spring automatically:

1. Opens a transaction.
2. Executes the method.
3. Commits on success.
4. Rolls back on failure.

---

## Example

```java
@Service
public class TransferService {

    @Transactional
    public void transfer() {

        accountA.debit(100);

        accountB.credit(100);
    }
}
```

Execution:

```text
Open Transaction
        ↓
Debit
        ↓
Credit
        ↓
Commit
```

Failure:

```text
Open Transaction
        ↓
Debit
        ↓
Exception
        ↓
Rollback
```

---

## Class-Level Transaction

```java
@Service
@Transactional
public class OrderService {

    public void createOrder() {}

    public void cancelOrder() {}
}
```

All public methods become transactional.

---

# Rollback Behavior

## Default Rollback Rules

Spring rolls back automatically for:

```java
RuntimeException
Error
```

Example:

```java
@Transactional
public void process() {

    throw new RuntimeException();
}
```

Result:

```text
Rollback
```

---

## Checked Exceptions

Checked exceptions require explicit configuration.

```java
@Transactional(
    rollbackFor = Exception.class
)
public void process() throws Exception {

    throw new Exception();
}
```

Result:

```text
Rollback
```

---

# Transaction Propagation

Propagation defines how a transactional method behaves when another transaction already exists.

---

## REQUIRED (Default)

```java
@Transactional(
    propagation = Propagation.REQUIRED
)
```

Behavior:

```text
Existing Transaction?
    Yes → Join
    No → Create New
```

Most commonly used.

---

## REQUIRES_NEW

```java
@Transactional(
    propagation = Propagation.REQUIRES_NEW
)
```

Behavior:

```text
Suspend Current Transaction
Create New Transaction
Execute
Commit
Resume Original Transaction
```

Common use cases:

- Audit logs
- Notifications
- Event history

---

## SUPPORTS

```java
@Transactional(
    propagation = Propagation.SUPPORTS
)
```

Behavior:

```text
Transaction Exists?
    Yes → Join
    No → Execute Without Transaction
```

Common for read operations.

---

## NOT_SUPPORTED

```java
@Transactional(
    propagation = Propagation.NOT_SUPPORTED
)
```

Behavior:

```text
Suspend Existing Transaction
Execute Without Transaction
Resume Transaction
```

Useful for:

- External API calls
- Long-running tasks

---

## MANDATORY

```java
@Transactional(
    propagation = Propagation.MANDATORY
)
```

Requires an existing transaction.

Otherwise:

```text
IllegalTransactionStateException
```

---

## NEVER

```java
@Transactional(
    propagation = Propagation.NEVER
)
```

Fails if a transaction exists.

---

## NESTED

```java
@Transactional(
    propagation = Propagation.NESTED
)
```

Creates savepoints.

```text
Parent Transaction
      │
      ├── Savepoint
      │
      └── Nested Transaction
```

Allows partial rollback.

Database support varies.

---

# Isolation Levels

Isolation levels determine how concurrent transactions interact.

---

## Concurrency Problems

### Dirty Read

Reading uncommitted data.

```text
T1 Updates Value
T2 Reads Value
T1 Rollbacks
```

T2 read invalid data.

---

### Non-Repeatable Read

Reading the same row twice returns different values.

```text
T1 Reads Salary = 1000

T2 Updates Salary = 2000

T1 Reads Again = 2000
```

---

### Phantom Read

A query returns additional rows.

```text
SELECT * FROM employees
```

Another transaction inserts new rows.

Second query returns different results.

---

## READ_UNCOMMITTED

Allows:

- Dirty Reads
- Non-repeatable Reads
- Phantom Reads

```java
@Transactional(
    isolation = Isolation.READ_UNCOMMITTED
)
```

Highest performance, lowest consistency.

---

## READ_COMMITTED

Prevents:

- Dirty Reads

Allows:

- Non-repeatable Reads
- Phantom Reads

```java
@Transactional(
    isolation = Isolation.READ_COMMITTED
)
```

Most commonly used.

---

## REPEATABLE_READ

Prevents:

- Dirty Reads
- Non-repeatable Reads

Allows:

- Phantom Reads

```java
@Transactional(
    isolation = Isolation.REPEATABLE_READ
)
```

Default in MySQL InnoDB.

---

## SERIALIZABLE

Prevents:

- Dirty Reads
- Non-repeatable Reads
- Phantom Reads

```java
@Transactional(
    isolation = Isolation.SERIALIZABLE
)
```

Highest consistency.

Trade-offs:

- More locks
- Lower throughput

---

## Isolation Comparison Table

| Isolation Level  | Dirty Read   | Non-Repeatable Read | Phantom Read |
| ---------------- | ------------ | ------------------- | ------------ |
| READ_UNCOMMITTED | ❌ Allowed   | ❌ Allowed          | ❌ Allowed   |
| READ_COMMITTED   | ✅ Prevented | ❌ Allowed          | ❌ Allowed   |
| REPEATABLE_READ  | ✅ Prevented | ✅ Prevented        | ❌ Allowed   |
| SERIALIZABLE     | ✅ Prevented | ✅ Prevented        | ✅ Prevented |

---

# Transaction Timeouts

## What is a Timeout?

Defines the maximum duration of a transaction.

---

## Example

```java
@Transactional(timeout = 10)
public void processOrder() {
}
```

Meaning:

```text
Maximum Duration = 10 Seconds
```

If exceeded:

```text
TransactionTimedOutException
Rollback
```

---

## Why Use Timeouts?

Prevents:

- Long-running transactions
- Resource exhaustion
- Lock contention
- Deadlocks

---

# Read-Only Transactions

## What is a Read-Only Transaction?

Used when no data modifications are expected.

```java
@Transactional(readOnly = true)
public User findUser(Long id) {
    return repository.findById(id).orElseThrow();
}
```

---

## Benefits

### Hibernate Optimization

Normal flow:

```text
Load Entity
↓
Track Changes
↓
Flush
```

Read-only:

```text
Load Entity
↓
No Dirty Checking
```

---

### Better Performance

Useful for:

```java
@Transactional(readOnly = true)
public List<User> findAllUsers() {
    return repository.findAll();
}
```

Benefits:

- Less memory
- Less CPU
- Faster execution

---

## Important Note

Read-only does not always physically prevent writes.

Behavior depends on:

- Database
- JDBC Driver
- Transaction Manager

---

# How Spring Implements Transactions

Spring uses:

- AOP (Aspect-Oriented Programming)
- Dynamic Proxies

---

## Internal Flow

```text
Client
  ↓
Spring Proxy
  ↓
Open Transaction
  ↓
Execute Method
  ↓
Commit / Rollback
```

Equivalent logic:

```java
beginTransaction();

try {
    businessMethod();
    commit();
}
catch(Exception ex) {
    rollback();
}
```

---

# Limitations of `@Transactional`

## Private Methods

Does not work.

```java
@Transactional
private void save() {}
```

Reason:

```text
Proxy Cannot Intercept Private Methods
```

---

## Self Invocation

Does not work.

```java
@Service
public class UserService {

    public void methodA() {
        methodB();
    }

    @Transactional
    public void methodB() {
    }
}
```

Reason:

```text
Call Bypasses Spring Proxy
```

---

# Advantages

## Simplicity

```java
@Transactional
public void save() {}
```

---

## Less Boilerplate

No need for:

```java
beginTransaction();
commit();
rollback();
```

---

## Seamless Integration

Works with:

- JPA
- Hibernate
- JDBC
- Spring Data

---

## Better Consistency

Helps enforce ACID principles.

---

# Disadvantages

## Long Transactions Can Be Dangerous

May cause:

- Locks
- Deadlocks
- Performance degradation

---

## External Calls Inside Transactions

Bad example:

```java
@Transactional
public void process() {

    externalApi.call();

    repository.save(entity);
}
```

The transaction remains open during the external call.

---

## Learning Curve

Developers must understand:

- Propagation
- Isolation
- Rollback rules
- Proxy behavior

---

# Best Practices

✅ Place `@Transactional` in the Service layer.

✅ Keep transactions short.

✅ Use `readOnly = true` whenever possible.

✅ Avoid remote calls inside transactions.

✅ Understand propagation before using `REQUIRES_NEW`.

✅ Use timeouts for long-running processes.

❌ Do not place `@Transactional` on Controllers.

❌ Do not create large transactional scopes.

❌ Do not ignore isolation-level implications.

---

# Summary

| Topic                 | Purpose                            |
| --------------------- | ---------------------------------- |
| Transaction           | Unit of work                       |
| ACID                  | Reliability guarantees             |
| Transactional Context | Shared transactional scope         |
| Transaction Manager   | Controls transaction lifecycle     |
| `@Transactional`      | Declarative transaction management |
| Propagation           | Transaction interaction rules      |
| Isolation             | Concurrency control                |
| Timeout               | Maximum transaction duration       |
| Read-Only             | Query optimization                 |
| AOP Proxy             | Spring transaction mechanism       |

Understanding these concepts is essential for building reliable, scalable, and consistent Spring applications. Together, ACID principles, transaction managers, propagation rules, isolation levels, and the `@Transactional` annotation form the foundation of transaction management in Spring Framework and Spring Boot.
