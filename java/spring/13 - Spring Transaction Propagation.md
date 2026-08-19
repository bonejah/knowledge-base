# 13 - Spring Transaction Propagation

## Introduction

Transaction management is one of the most important features provided by the Spring Framework. While the `@Transactional` annotation is commonly used to define transactional boundaries, understanding **Transaction Propagation** is essential when multiple transactional methods interact with each other.

Transaction propagation defines **how a transactional method should behave when it is called by another transactional method**. It determines whether a method should:

- Join an existing transaction
- Create a new transaction
- Execute without a transaction
- Throw an exception if a transaction exists or does not exist

Without proper propagation settings, applications may experience unexpected commits, rollbacks, data inconsistencies, or performance issues.

---

# What is Transaction Propagation?

Transaction Propagation specifies how Spring should manage transactions when one transactional method calls another transactional method.

Consider the following example:

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder() {
        paymentService.processPayment();
    }
}
```

```java
@Service
public class PaymentService {

    @Transactional
    public void processPayment() {
        // Payment logic
    }
}
```

When `placeOrder()` calls `processPayment()`, Spring must decide:

- Should `processPayment()` use the existing transaction?
- Should it create a new transaction?
- Should it execute without a transaction?

The answer depends on the configured propagation mode.

---

# Why Transaction Propagation Matters

Transaction propagation helps define business boundaries and control transactional behavior across services.

Common use cases include:

- Audit logging
- Payment processing
- Inventory updates
- Notification systems
- Batch processing
- Reporting operations

Proper propagation ensures:

- Data consistency
- Predictable rollback behavior
- Better separation of business concerns
- Improved system reliability

---

# How to Configure Propagation

Propagation is configured using the `propagation` attribute of `@Transactional`.

```java
@Transactional(propagation = Propagation.REQUIRED)
public void processOrder() {
    // business logic
}
```

Spring provides several propagation types.

---

# Propagation.REQUIRED (Default)

## What It Does

- Joins the existing transaction if one exists.
- Creates a new transaction if none exists.

This is the default propagation mode.

```java
@Transactional
public void processOrder() {
    paymentService.processPayment();
}
```

```java
@Transactional
public void processPayment() {
    // Payment logic
}
```

### Scenario

```
processOrder()
    └── processPayment()
```

Result:

- One transaction
- Both methods share the same transaction

### Rollback Behavior

If either method throws a runtime exception:

```java
throw new RuntimeException("Payment failed");
```

Entire transaction rolls back.

```
Order Insert     -> Rolled Back
Payment Insert   -> Rolled Back
```

### Advantages

- Simple
- Most common use case
- Maintains data consistency

### Disadvantages

- Large transactions may become expensive
- Failures affect all operations

---

# Propagation.REQUIRES_NEW

## What It Does

- Suspends the current transaction.
- Creates a completely new transaction.

```java
@Transactional
public void processOrder() {
    auditService.saveAudit();
}
```

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAudit() {
    // Save audit log
}
```

### Scenario

```
Transaction A
    └── Transaction B
```

Transaction B is independent.

### Example

```java
@Transactional
public void placeOrder() {

    orderRepository.save(order);

    auditService.saveAudit();

    throw new RuntimeException();
}
```

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAudit() {
    auditRepository.save(audit);
}
```

### Result

```
Order Insert -> Rolled Back
Audit Insert -> Committed
```

Because the audit transaction is independent.

### Common Use Cases

- Audit logs
- Error tracking
- Event logging
- Notifications

### Advantages

- Independent commit
- Independent rollback

### Disadvantages

- More database overhead
- Additional transaction creation

---

# Propagation.SUPPORTS

## What It Does

- Joins an existing transaction if one exists.
- Executes without a transaction if none exists.

```java
@Transactional(propagation = Propagation.SUPPORTS)
public User findUser(Long id) {
    return repository.findById(id);
}
```

### Scenario

Called from transactional method:

```
Transaction Exists
    └── Join Transaction
```

Called directly:

```
No Transaction
    └── Execute Normally
```

### Common Use Cases

- Read operations
- Utility methods

### Advantages

- Lightweight
- Flexible

### Disadvantages

- Transaction availability varies

---

# Propagation.NOT_SUPPORTED

## What It Does

- Suspends any existing transaction.
- Executes without a transaction.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void generateReport() {
    // Reporting logic
}
```

### Scenario

```
Transaction A
    └── Suspend A
          Execute without transaction
```

### Common Use Cases

- Large reports
- Read-heavy operations
- External service calls

### Advantages

- Better performance for long-running tasks

### Disadvantages

- No rollback support

---

# Propagation.MANDATORY

## What It Does

- Requires an existing transaction.
- Throws an exception if no transaction exists.

```java
@Transactional(propagation = Propagation.MANDATORY)
public void updateInventory() {
    // Logic
}
```

### Scenario

```
Transaction Exists -> OK
No Transaction     -> Exception
```

### Exception

```java
IllegalTransactionStateException
```

### Common Use Cases

- Methods that must always participate in a business transaction

### Advantages

- Enforces transactional discipline

### Disadvantages

- Less flexible

---

# Propagation.NEVER

## What It Does

- Must execute without a transaction.
- Throws an exception if a transaction exists.

```java
@Transactional(propagation = Propagation.NEVER)
public void externalApiCall() {
    // Call external system
}
```

### Scenario

```
Transaction Exists -> Exception
No Transaction     -> OK
```

### Exception

```java
IllegalTransactionStateException
```

### Common Use Cases

- Non-transactional integrations
- External APIs

### Advantages

- Prevents long-running transactions

### Disadvantages

- Very specialized use case

---

# Propagation.NESTED

## What It Does

Creates a nested transaction inside an existing transaction using savepoints.

```java
@Transactional
public void processOrder() {
    inventoryService.reserveStock();
}
```

```java
@Transactional(propagation = Propagation.NESTED)
public void reserveStock() {
    // Logic
}
```

### Scenario

```
Main Transaction
    └── Savepoint
           Nested Transaction
```

### Rollback Behavior

Nested transaction fails:

```java
throw new RuntimeException();
```

Spring can roll back to the savepoint while keeping the outer transaction active.

### Example

```
Order Insert       -> Committed
Inventory Reserve  -> Rolled Back
```

### Advantages

- Partial rollback capability

### Disadvantages

- Database support required
- Less commonly used

---

# Propagation Comparison Table

| Propagation   | Existing Transaction         | No Existing Transaction     |
| ------------- | ---------------------------- | --------------------------- |
| REQUIRED      | Join                         | Create New                  |
| REQUIRES_NEW  | Suspend Current + Create New | Create New                  |
| SUPPORTS      | Join                         | Execute Without Transaction |
| NOT_SUPPORTED | Suspend Current              | Execute Without Transaction |
| MANDATORY     | Join                         | Exception                   |
| NEVER         | Exception                    | Execute Without Transaction |
| NESTED        | Nested Transaction           | Create New Transaction      |

---

# Real-World Example

Imagine an e-commerce system.

```java
@Transactional
public void checkout() {

    orderService.createOrder();

    paymentService.processPayment();

    auditService.saveAudit();
}
```

### Recommended Configuration

```java
@Transactional
public void createOrder() {
}
```

```java
@Transactional
public void processPayment() {
}
```

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAudit() {
}
```

### Behavior

If payment fails:

```
Order -> Rollback
Payment -> Rollback
Audit -> Commit
```

Audit logs remain available for troubleshooting.

---

# Common Mistakes

## Using REQUIRES_NEW Everywhere

Bad practice:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveSomething() {
}
```

Creating many independent transactions increases:

- Database load
- Locking complexity
- Resource consumption

Use it only when truly needed.

---

## Expecting NESTED Support Everywhere

Not all databases and transaction managers support nested transactions.

Always verify database compatibility.

---

## Long Running Transactions

Avoid:

```java
@Transactional
public void processOrder() {

    repository.save(order);

    externalApi.call();

    Thread.sleep(30000);
}
```

Problems:

- Locks remain active
- Connection stays occupied
- Performance degrades

---

# Best Practices

## Use REQUIRED as the Default

For most business operations:

```java
@Transactional
public void createOrder() {
}
```

Spring automatically uses `REQUIRED`.

---

## Use REQUIRES_NEW for Independent Operations

Examples:

- Audit logs
- Error logs
- Event tracking

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAudit() {
}
```

---

## Keep Transactions Short

Good:

```java
@Transactional
public void createOrder() {
    repository.save(order);
}
```

Avoid external calls inside transactions whenever possible.

---

## Separate Read and Write Operations

Read-only operations:

```java
@Transactional(readOnly = true)
public User findUser(Long id) {
    return repository.findById(id);
}
```

This can improve performance and reduce unnecessary locking.

---

## Understand Rollback Boundaries

Always know:

- Which transaction owns the commit
- Which transaction owns the rollback
- Whether child methods share or create new transactions

This prevents unexpected behavior in production systems.

---

# Summary

Transaction Propagation determines how transactional methods interact when one method calls another. It is a critical concept for designing reliable and maintainable enterprise applications.

The most commonly used propagation modes are:

- `REQUIRED` (default)
- `REQUIRES_NEW`
- `SUPPORTS`

For most business logic, `REQUIRED` is sufficient. Use `REQUIRES_NEW` when an operation must succeed independently of the main transaction, such as audit logging. More specialized modes like `MANDATORY`, `NEVER`, `NOT_SUPPORTED`, and `NESTED` should be used only when their specific transactional semantics are required.

A solid understanding of transaction propagation helps developers build systems that are more predictable, resilient, and easier to maintain.
