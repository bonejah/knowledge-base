# 14 - Spring Transaction Isolation

## Introduction

When multiple users or processes access the same data concurrently, unexpected issues can occur. One transaction may read data that another transaction is modifying, resulting in inconsistent or incorrect results.

To address these concurrency problems, relational databases provide **Transaction Isolation Levels**.

In Spring, transaction isolation can be configured through the `@Transactional` annotation, allowing developers to control how transactions interact with one another.

Understanding isolation levels is essential for building reliable, scalable, and consistent applications.

---

# What is Transaction Isolation?

Transaction Isolation defines how and when changes made by one transaction become visible to other transactions.

It controls the degree of separation between concurrently executing transactions.

The goal is to balance:

* Data consistency
* Performance
* Concurrency

Higher isolation levels provide stronger consistency but may reduce performance and scalability.

Lower isolation levels improve concurrency but may allow inconsistent reads.

---

# Why Transaction Isolation Matters

Imagine a banking application.

Transaction A:

```sql
UPDATE account
SET balance = balance - 100
WHERE id = 1;
```

Transaction B:

```sql
SELECT balance
FROM account
WHERE id = 1;
```

If Transaction B reads the balance before Transaction A commits, it may see data that is not yet finalized.

This can lead to incorrect business decisions and inconsistent application behavior.

Isolation levels help prevent these situations.

---

# Common Concurrency Problems

Before understanding isolation levels, it is important to understand the problems they solve.

---

# Dirty Read

A transaction reads data modified by another transaction that has not yet committed.

## Example

Transaction A:

```sql
UPDATE account
SET balance = 500
WHERE id = 1;
```

Transaction B:

```sql
SELECT balance
FROM account
WHERE id = 1;
```

Transaction A later rolls back:

```sql
ROLLBACK;
```

Transaction B read data that never actually existed.

### Problem

```text
Original Balance = 1000
Temporary Balance = 500
Rollback
Actual Balance = 1000
```

Transaction B used invalid data.

---

# Non-Repeatable Read

A transaction reads the same row twice and gets different results.

## Example

Transaction A:

```sql
SELECT balance
FROM account
WHERE id = 1;
```

Result:

```text
1000
```

Transaction B:

```sql
UPDATE account
SET balance = 800
WHERE id = 1;

COMMIT;
```

Transaction A executes again:

```sql
SELECT balance
FROM account
WHERE id = 1;
```

Result:

```text
800
```

The same query returned different values within the same transaction.

---

# Phantom Read

A transaction executes the same query twice and receives different sets of rows.

## Example

Transaction A:

```sql
SELECT *
FROM orders
WHERE status = 'PENDING';
```

Returns:

```text
10 rows
```

Transaction B:

```sql
INSERT INTO orders(status)
VALUES('PENDING');

COMMIT;
```

Transaction A runs the query again:

```sql
SELECT *
FROM orders
WHERE status = 'PENDING';
```

Returns:

```text
11 rows
```

A new row appeared like a phantom.

---

# Lost Update

Two transactions update the same record simultaneously.

## Example

Initial value:

```text
Stock = 10
```

Transaction A:

```text
Reads 10
```

Transaction B:

```text
Reads 10
```

Transaction A:

```text
Updates to 9
```

Transaction B:

```text
Updates to 8
```

Expected:

```text
7
```

Actual:

```text
8
```

One update was lost.

---

# Isolation Levels in Spring

Spring supports the standard SQL isolation levels through:

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
```

Available levels:

* DEFAULT
* READ_UNCOMMITTED
* READ_COMMITTED
* REPEATABLE_READ
* SERIALIZABLE

---

# Isolation.DEFAULT

## What It Does

Uses the database's default isolation level.

```java
@Transactional(isolation = Isolation.DEFAULT)
public void processOrder() {
}
```

### Example Defaults

| Database     | Default Isolation |
| ------------ | ----------------- |
| PostgreSQL   | READ_COMMITTED    |
| Oracle       | READ_COMMITTED    |
| SQL Server   | READ_COMMITTED    |
| MySQL InnoDB | REPEATABLE_READ   |

### Advantages

* Uses database best practices
* Simple configuration

### Disadvantages

* Behavior may vary between databases

---

# Isolation.READ_UNCOMMITTED

## What It Does

Allows transactions to read uncommitted data from other transactions.

```java
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
public void generateReport() {
}
```

### Prevents

* Nothing

### Allows

* Dirty Reads
* Non-Repeatable Reads
* Phantom Reads

### Example

Transaction A:

```sql
UPDATE account
SET balance = 500;
```

Not committed yet.

Transaction B:

```sql
SELECT balance FROM account;
```

Result:

```text
500
```

If Transaction A rolls back, Transaction B used invalid data.

### Advantages

* Maximum performance
* Highest concurrency

### Disadvantages

* Lowest consistency
* Rarely recommended

---

# Isolation.READ_COMMITTED

## What It Does

Only allows reading committed data.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void processPayment() {
}
```

### Prevents

* Dirty Reads

### Allows

* Non-Repeatable Reads
* Phantom Reads

### Example

Transaction A updates data but does not commit.

Transaction B attempts to read.

Result:

```text
Transaction B cannot see uncommitted changes.
```

Only committed values are visible.

### Advantages

* Good balance between consistency and performance
* Most common database default

### Disadvantages

* Same row can return different values within a transaction

---

# Isolation.REPEATABLE_READ

## What It Does

Guarantees that repeated reads of the same row return the same result.

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void generateStatement() {
}
```

### Prevents

* Dirty Reads
* Non-Repeatable Reads

### Allows

* Phantom Reads (depending on database implementation)

### Example

Transaction A:

```sql
SELECT balance
FROM account
WHERE id = 1;
```

Transaction B:

```sql
UPDATE account
SET balance = 500
WHERE id = 1;
```

Transaction A reads again:

```sql
SELECT balance
FROM account
WHERE id = 1;
```

Still sees:

```text
Original Value
```

### Advantages

* Stronger consistency
* Good for financial operations

### Disadvantages

* More locking
* Reduced concurrency

---

# Isolation.SERIALIZABLE

## What It Does

Provides the highest level of isolation.

Transactions behave as if they are executed one at a time.

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferMoney() {
}
```

### Prevents

* Dirty Reads
* Non-Repeatable Reads
* Phantom Reads

### Example

Transaction A:

```sql
SELECT *
FROM orders
WHERE status = 'PENDING';
```

While Transaction A is running:

Transaction B:

```sql
INSERT INTO orders(status)
VALUES('PENDING');
```

Transaction B must wait.

### Advantages

* Maximum consistency
* Prevents all standard concurrency issues

### Disadvantages

* Lowest concurrency
* More locking
* Reduced throughput

---

# Isolation Level Comparison

| Isolation Level  | Dirty Read | Non-Repeatable Read | Phantom Read |
| ---------------- | ---------- | ------------------- | ------------ |
| READ_UNCOMMITTED | Possible   | Possible            | Possible     |
| READ_COMMITTED   | Prevented  | Possible            | Possible     |
| REPEATABLE_READ  | Prevented  | Prevented           | Possible*    |
| SERIALIZABLE     | Prevented  | Prevented           | Prevented    |

* Database dependent.

---

# Configuring Isolation in Spring

## Method Level

```java
@Transactional(
    isolation = Isolation.REPEATABLE_READ
)
public void processPayment() {
}
```

---

## Combined with Propagation

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.READ_COMMITTED
)
public void createOrder() {
}
```

---

## Combined with Timeout

```java
@Transactional(
    isolation = Isolation.SERIALIZABLE,
    timeout = 30
)
public void transferFunds() {
}
```

---

# Real-World Examples

## Banking Transfers

Recommended:

```java
@Transactional(
    isolation = Isolation.SERIALIZABLE
)
public void transferMoney() {
}
```

Reason:

* Maximum consistency
* No lost updates

---

## Order Processing

Recommended:

```java
@Transactional(
    isolation = Isolation.READ_COMMITTED
)
public void placeOrder() {
}
```

Reason:

* Good performance
* Prevents dirty reads

---

## Reporting Systems

Recommended:

```java
@Transactional(
    readOnly = true,
    isolation = Isolation.READ_COMMITTED
)
public Report generateReport() {
}
```

Reason:

* Faster execution
* Consistent enough for reporting

---

## Inventory Management

Recommended:

```java
@Transactional(
    isolation = Isolation.REPEATABLE_READ
)
public void reserveInventory() {
}
```

Reason:

* Prevents inconsistent stock calculations

---

# Common Mistakes

## Always Using SERIALIZABLE

Bad practice:

```java
@Transactional(
    isolation = Isolation.SERIALIZABLE
)
public void findUser() {
}
```

Problems:

* Excessive locking
* Poor scalability
* Reduced performance

Use only when absolutely necessary.

---

## Ignoring Database Defaults

Developers often assume all databases behave the same.

Example:

```text
PostgreSQL -> READ_COMMITTED
MySQL -> REPEATABLE_READ
```

Application behavior may change after migration.

Always verify database defaults.

---

## Using High Isolation for Read-Only Queries

Bad:

```java
@Transactional(
    isolation = Isolation.SERIALIZABLE,
    readOnly = true
)
public List<User> findUsers() {
}
```

This provides little benefit while increasing resource usage.

---

# Best Practices

## Start with READ_COMMITTED

For most applications:

```java
@Transactional(
    isolation = Isolation.READ_COMMITTED
)
public void processOrder() {
}
```

This is usually the best balance.

---

## Use REPEATABLE_READ for Financial Calculations

When consistency matters more than throughput:

```java
@Transactional(
    isolation = Isolation.REPEATABLE_READ
)
public void calculateInterest() {
}
```

---

## Reserve SERIALIZABLE for Critical Business Operations

Examples:

* Banking transfers
* High-value financial transactions
* Critical inventory reservations

---

## Keep Transactions Short

Good:

```java
@Transactional
public void saveOrder() {
    repository.save(order);
}
```

Avoid:

```java
@Transactional
public void saveOrder() {

    repository.save(order);

    externalApi.call();

    Thread.sleep(30000);
}
```

Long-running transactions increase locking and contention.

---

## Understand Your Database

Different databases implement isolation differently.

Always verify:

* Default isolation level
* Locking behavior
* MVCC implementation
* Performance implications

---

# Summary

Transaction Isolation controls how concurrent transactions interact with one another and determines what data is visible during transaction execution.

Spring supports five isolation levels:

* `DEFAULT`
* `READ_UNCOMMITTED`
* `READ_COMMITTED`
* `REPEATABLE_READ`
* `SERIALIZABLE`

As isolation increases, data consistency improves, but concurrency and performance typically decrease.

For most enterprise applications:

* Use `READ_COMMITTED` as the default choice.
* Use `REPEATABLE_READ` when consistent repeated reads are required.
* Use `SERIALIZABLE` only for critical operations that require maximum consistency.

Choosing the correct isolation level is one of the most important decisions in designing reliable and scalable transactional systems.

