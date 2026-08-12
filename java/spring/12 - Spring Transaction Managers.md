# 12 - Spring Transaction Managers

## Introduction

In enterprise applications, transactions are essential to ensure that a group of operations either succeeds completely or fails completely.

Spring provides a powerful transaction management abstraction that separates transaction handling from business logic, allowing developers to focus on application functionality rather than transaction boilerplate code.

Spring supports two main approaches:

- Programmatic Transaction Management
- Declarative Transaction Management

It also provides different transaction manager implementations depending on the technology being used.

---

# Why Do We Need Transaction Managers?

Without a transaction manager, developers would need to manually:

1. Open a transaction
2. Execute business logic
3. Commit if successful
4. Rollback if an error occurs
5. Close resources

Example without Spring:

    Connection connection = dataSource.getConnection();

    try {
        connection.setAutoCommit(false);

        // Business logic

        connection.commit();
    } catch (Exception ex) {
        connection.rollback();
    } finally {
        connection.close();
    }

Problems with this approach:

- Repetitive code
- Hard to maintain
- Error-prone
- Business logic becomes coupled to transaction management
- Difficult to handle complex transaction scenarios

Spring solves these problems by providing a transaction abstraction.

---

# What is a Transaction Manager?

A Transaction Manager is a Spring component responsible for controlling transactions.

Its main responsibilities include:

- Starting transactions
- Committing transactions
- Rolling back transactions
- Managing transaction boundaries
- Coordinating transactional resources
- Integrating transactions with persistence technologies

The central Spring abstraction is:

    PlatformTransactionManager

Conceptually:

    Application
        |
        v
    Transaction Manager
        |
        +---- Begin Transaction
        |
        +---- Execute Business Logic
        |
        +---- Commit
        |       or
        +---- Rollback

The `PlatformTransactionManager` interface provides the fundamental transaction operations:

    public interface PlatformTransactionManager {

        TransactionStatus getTransaction(
            TransactionDefinition definition
        );

        void commit(TransactionStatus status);

        void rollback(TransactionStatus status);
    }

---

# Transaction Manager Implementations

Spring provides different transaction manager implementations depending on the technology.

Common examples include:

- `DataSourceTransactionManager`
- `JpaTransactionManager`
- `JtaTransactionManager`
- `R2dbcTransactionManager`

---

# DataSourceTransactionManager

`DataSourceTransactionManager` is used for applications that work directly with JDBC.

Example:

    @Bean
    public PlatformTransactionManager transactionManager(
            DataSource dataSource) {

        return new DataSourceTransactionManager(dataSource);
    }

Typical architecture:

    Spring
       |
       v
    DataSourceTransactionManager
       |
       v
    JDBC
       |
       v
    Database

## When to Use

Use it when:

- Working directly with JDBC
- Using `JdbcTemplate`
- Not using JPA/Hibernate

## Advantages

- Lightweight
- Simple
- Good control over JDBC transactions

## Disadvantages

- JDBC-specific
- Not intended for JPA entity management

---

# JpaTransactionManager

`JpaTransactionManager` is commonly used with:

- Spring Data JPA
- JPA
- Hibernate

Example:

    @Bean
    public PlatformTransactionManager transactionManager(
            EntityManagerFactory entityManagerFactory) {

        return new JpaTransactionManager(entityManagerFactory);
    }

Typical architecture:

    Spring
       |
       v
    JpaTransactionManager
       |
       v
    JPA / Hibernate
       |
       v
    Database

In Spring Boot applications using Spring Data JPA, Spring Boot generally configures the necessary transaction infrastructure automatically.

## Advantages

- Integrates with JPA
- Integrates with Hibernate
- Manages `EntityManager`
- Supports transaction synchronization
- Works naturally with `@Transactional`

## When to Use

Use it when your application uses:

    Spring Data JPA
            +
    Hibernate
            +
    Relational Database

This is one of the most common transaction manager configurations in Spring applications.

---

# JtaTransactionManager

`JtaTransactionManager` is designed for JTA-based transactions and distributed transaction scenarios.

It can coordinate transactions involving multiple resources.

For example:

    Application
         |
         v
    JtaTransactionManager
         |
         +---- Database A
         |
         +---- Database B
         |
         +---- Message Broker

This type of transaction is sometimes called a distributed transaction.

## When to Use

Typical use cases include:

- Multiple databases
- Multiple transactional resources
- Legacy enterprise applications
- XA transactions

## Advantages

- Can coordinate multiple transactional resources
- Supports distributed transactions

## Disadvantages

- More complex
- Higher operational overhead
- Can negatively affect performance
- Usually unnecessary for modern microservice architectures

---

# R2dbcTransactionManager

For reactive applications using R2DBC, Spring provides:

    R2dbcTransactionManager

Example:

    @Bean
    ReactiveTransactionManager transactionManager(
            ConnectionFactory connectionFactory) {

        return new R2dbcTransactionManager(connectionFactory);
    }

Typical architecture:

    Spring WebFlux
          |
          v
    Reactive Transaction Manager
          |
          v
    R2DBC
          |
          v
    Database

This is used with reactive types such as:

    Mono<T>
    Flux<T>

---

# Programmatic Transaction Management

## What is Programmatic Transaction Management?

Programmatic transaction management means that the developer explicitly controls transaction behavior through code.

The developer decides:

- When the transaction starts
- When it commits
- When it rolls back
- Which transaction configuration is used

Spring provides two important mechanisms:

- `PlatformTransactionManager`
- `TransactionTemplate`

---

# Programmatic Transactions with TransactionTemplate

`TransactionTemplate` is one of the easiest ways to implement programmatic transactions.

Example:

    @Service
    public class PaymentService {

        private final TransactionTemplate transactionTemplate;

        public PaymentService(
                PlatformTransactionManager transactionManager) {

            this.transactionTemplate =
                    new TransactionTemplate(transactionManager);
        }

        public void processPayment() {

            transactionTemplate.execute(status -> {

                // Business logic

                return null;
            });
        }
    }

Spring manages:

    Begin
      |
      v
    Business Logic
      |
      +---- Success ---> Commit
      |
      +---- Exception --> Rollback

---

# Programmatic Rollback

You can explicitly mark a transaction for rollback:

    transactionTemplate.execute(status -> {

        try {

            // Business logic

        } catch (Exception ex) {

            status.setRollbackOnly();
        }

        return null;
    });

The transaction will be rolled back when the transaction callback finishes.

---

# Programmatic Transaction Configuration

You can configure transaction properties using `TransactionTemplate`.

For example:

    transactionTemplate.setTimeout(30);
    transactionTemplate.setReadOnly(true);

You can also configure propagation:

    transactionTemplate.setPropagationBehavior(
            TransactionDefinition.PROPAGATION_REQUIRES_NEW
    );

And isolation:

    transactionTemplate.setIsolationLevel(
            TransactionDefinition.ISOLATION_READ_COMMITTED
    );

This gives programmatic transactions a high level of control.

---

# Advantages of Programmatic Transactions

## Fine-Grained Control

You decide exactly where the transaction begins and ends.

    transactionTemplate.execute(status -> {

        // Transaction boundary is explicit

        return null;
    });

---

## Dynamic Transaction Behavior

Transaction settings can be configured programmatically.

For example:

    transactionTemplate.setTimeout(30);

or:

    transactionTemplate.setReadOnly(true);

---

## Useful for Complex Workflows

Programmatic transactions can be useful when transaction boundaries depend on runtime conditions.

Example:

    if condition A
        Transaction A

    else
        Transaction B

---

# Disadvantages of Programmatic Transactions

## More Boilerplate

Transaction logic is mixed with application code.

    transactionTemplate.execute(status -> {

        // Transaction-related code

        // Business logic

        return null;
    });

---

## Harder to Maintain

If many methods use programmatic transactions, the application can become repetitive.

---

## Business Logic Can Become Coupled to Transactions

Instead of:

    @Transactional
    public void processOrder() {
    }

you may have:

    transactionTemplate.execute(status -> {

        processOrder();

        return null;
    });

The transaction infrastructure becomes more visible in the business code.

---

# When Should You Use Programmatic Transactions?

Programmatic transaction management is useful when:

- Transaction boundaries are dynamic
- Advanced transaction control is required
- Different transaction behaviors are required within one method
- Conditional rollback logic is necessary
- Declarative transactions are not flexible enough

For most standard service-layer operations, declarative transactions are usually preferable.

---

# Declarative Transaction Management

## What is Declarative Transaction Management?

Declarative transaction management allows developers to define transaction behavior without manually controlling the transaction lifecycle.

The most common approach is:

    @Transactional

Example:

    @Service
    public class AccountService {

        @Transactional
        public void transferMoney(
                Long from,
                Long to,
                BigDecimal amount) {

            withdraw(from, amount);
            deposit(to, amount);
        }
    }

Spring automatically manages:

- Transaction creation
- Commit
- Rollback
- Transaction synchronization
- Transaction propagation
- Isolation
- Transaction lifecycle

---

# How Declarative Transactions Work

Declarative transaction management relies heavily on Spring AOP proxies.

Consider:

    @Transactional
    public void transferMoney() {
    }

Spring creates a proxy around the bean.

Conceptually:

    Client
       |
       v
    Spring Proxy
       |
       +---- Begin Transaction
       |
       v
    Business Method
       |
       +---- Success ---> Commit
       |
       +---- Exception --> Rollback

The proxy interacts with the configured transaction manager.

For example:

    @Transactional
          |
          v
    Spring AOP Proxy
          |
          v
    Transaction Manager
          |
          v
    Database

---

# Declarative Transaction Example

    @Service
    public class OrderService {

        private final OrderRepository orderRepository;

        public OrderService(OrderRepository orderRepository) {
            this.orderRepository = orderRepository;
        }

        @Transactional
        public void createOrder(Order order) {

            orderRepository.save(order);

            // Additional database operations
        }
    }

Spring automatically starts the transaction before executing the method.

If the method finishes successfully:

    Commit

If an appropriate exception causes rollback:

    Rollback

---

# Declarative Transaction Configuration

In modern Spring Boot applications, transaction management is often configured automatically.

For example, when using:

    Spring Boot
        +
    Spring Data JPA
        +
    Hibernate

Spring Boot can automatically configure the necessary transaction infrastructure.

Explicit transaction management can also be enabled using:

    @Configuration
    @EnableTransactionManagement
    public class TransactionConfig {
    }

In many Spring Boot applications, however, explicitly adding `@EnableTransactionManagement` is unnecessary because Boot provides the required auto-configuration.

---

# Advantages of Declarative Transactions

## Cleaner Code

Business logic remains focused on the business operation.

    @Transactional
    public void processOrder() {
        // Business logic
    }

---

## Less Boilerplate

You do not need to manually:

    begin
    commit
    rollback
    close

---

## Easier Maintenance

Transaction behavior is declared directly on the method.

---

## Better Readability

Developers can immediately see that a method is transactional:

    @Transactional
    public void transferMoney() {
    }

---

# Disadvantages of Declarative Transactions

## Proxy Limitations

Declarative transactions are usually implemented through Spring proxies.

This creates an important limitation called self-invocation.

Example:

    @Service
    public class UserService {

        public void methodA() {
            methodB();
        }

        @Transactional
        public void methodB() {
            // Transaction
        }
    }

Calling:

    methodA();

does not necessarily activate the transactional proxy for `methodB()`.

The reason is that the call happens internally within the same object.

Conceptually:

    External Caller
          |
          v
    Spring Proxy
          |
          v
    methodA()
          |
          v
    methodB()

The call from `methodA()` to `methodB()` does not pass through the proxy again.

---

# Programmatic vs Declarative Transactions

| Feature             | Programmatic  | Declarative       |
| ------------------- | ------------- | ----------------- |
| Ease of Use         | Low           | High              |
| Boilerplate         | High          | Low               |
| Readability         | Medium        | High              |
| Flexibility         | Very High     | High              |
| Maintenance         | Harder        | Easier            |
| Transaction Control | Explicit      | Implicit          |
| Recommended         | Special Cases | Most Applications |

---

# Multiple Transaction Managers

An application can have more than one transaction manager.

For example, an application may use two databases:

    Application
        |
        +---- Database A
        |
        +---- Database B

Each database can have its own transaction infrastructure.

Example:

    @Bean
    public PlatformTransactionManager userTransactionManager(
            EntityManagerFactory userEntityManagerFactory) {

        return new JpaTransactionManager(
                userEntityManagerFactory
        );
    }

And:

    @Bean
    public PlatformTransactionManager auditTransactionManager(
            EntityManagerFactory auditEntityManagerFactory) {

        return new JpaTransactionManager(
                auditEntityManagerFactory
        );
    }

You can explicitly select a transaction manager:

    @Transactional("auditTransactionManager")
    public void saveAudit() {
    }

This tells Spring which transaction manager should control the transaction.

---

# TransactionSynchronizationManager

One important internal Spring component is:

    TransactionSynchronizationManager

It helps Spring associate transactional resources with the current execution context.

It can manage resources such as:

- JDBC connections
- EntityManagers
- Transaction synchronization callbacks
- Transaction-related state

Conceptually:

    Application Thread
           |
           v
    TransactionSynchronizationManager
           |
           +---- EntityManager
           |
           +---- JDBC Connection
           |
           +---- Transaction State

This mechanism is important for allowing different Spring components to participate in the same transaction.

---

# Transaction Manager and @Transactional

It is important to understand the relationship between these two concepts.

They are not the same thing.

`@Transactional` declares transaction behavior:

    @Transactional
    public void saveOrder() {
    }

The Transaction Manager performs the actual transaction management.

    @Transactional
          |
          v
    Spring AOP Proxy
          |
          v
    Transaction Manager
          |
          v
    Database

So:

    @Transactional
            =
    Transaction configuration/declaration

    Transaction Manager
            =
    Transaction execution/infrastructure

---

# Transaction Manager and Propagation

Transaction managers also participate in transaction propagation.

For example:

    @Transactional(
        propagation = Propagation.REQUIRES_NEW
    )

The transaction manager determines how the current transaction should be handled.

For example:

    Transaction A
          |
          v
    Service A
          |
          v
    REQUIRES_NEW
          |
          v
    Transaction B

Transaction B runs independently from Transaction A.

This is one of the reasons understanding transaction managers is important before studying transaction propagation in depth.

---

# Transaction Manager and Isolation

Transaction managers also apply isolation settings.

Example:

    @Transactional(
        isolation = Isolation.READ_COMMITTED
    )
    public void processOrder() {
    }

The transaction manager communicates the requested transaction configuration to the underlying transactional resource.

Common isolation levels include:

    READ_UNCOMMITTED
    READ_COMMITTED
    REPEATABLE_READ
    SERIALIZABLE

The actual behavior depends on the underlying database and transaction implementation.

---

# Transaction Manager and Read-Only Transactions

A transaction can be declared as read-only:

    @Transactional(readOnly = true)
    public List<User> findUsers() {
        return userRepository.findAll();
    }

This communicates that the transaction is intended for reading data.

Depending on the persistence technology, database, and configuration, read-only transactions can provide optimizations and help prevent unintended writes.

---

# Transaction Manager and Timeouts

Transaction managers can also enforce transaction timeouts.

Using declarative transactions:

    @Transactional(timeout = 30)
    public void processPayment() {
    }

This means the transaction should not remain active indefinitely.

Timeouts are particularly important for preventing long-running transactions from holding:

- Database connections
- Locks
- Resources

for excessive periods.

---

# Best Practices

## Prefer Declarative Transactions

For most service-layer operations, use:

    @Transactional

instead of manually managing transactions.

---

## Put Transaction Boundaries in the Service Layer

A common pattern is:

    Controller
        |
        v
    Service
        |
        +---- @Transactional
        |
        v
    Repository

Example:

    @Service
    public class OrderService {

        @Transactional
        public void createOrder(Order order) {

            orderRepository.save(order);
            paymentRepository.save(order.getPayment());
        }
    }

The service layer usually represents the business operation that needs to be atomic.

---

## Keep Transactions Small

Avoid unnecessarily long transactions.

Good:

    @Transactional
    public void saveOrder() {
        orderRepository.save(order);
    }

Potentially problematic:

    @Transactional
    public void processLargeBatch() {

        // Thousands of database operations
        // External API calls
        // Long calculations
        // File processing
    }

Long transactions can hold database resources and locks for too long.

---

## Avoid External API Calls Inside Transactions

Avoid:

    @Transactional
    public void processOrder() {

        paymentGateway.call();

        orderRepository.save(order);
    }

If the external API takes a long time to respond, the database transaction may remain open unnecessarily.

A better architecture may separate the external communication from the database transaction or use asynchronous/event-driven processing when appropriate.

---

## Understand the Underlying Transaction Manager

When debugging transaction problems, it is important to know which transaction manager is being used.

For example:

    Spring Data JPA
           |
           v
    JpaTransactionManager
           |
           v
    Hibernate
           |
           v
    Database

Understanding this chain makes it easier to diagnose problems involving:

- Rollbacks
- Propagation
- Isolation
- Connections
- EntityManager
- Persistence context
- Transaction boundaries

---

# Common Mistakes

## Mistake 1: Assuming @Transactional Creates the Transaction by Itself

`@Transactional` is a declaration.

The transaction manager is responsible for managing the actual transaction.

    @Transactional
          |
          v
    Proxy
          |
          v
    Transaction Manager
          |
          v
    Transaction

---

## Mistake 2: Using Programmatic Transactions Everywhere

Programmatic transactions provide control, but excessive use creates unnecessary complexity.

Prefer:

    @Transactional

when the transaction behavior is straightforward.

---

## Mistake 3: Ignoring Proxy Behavior

Remember that Spring's declarative transactions generally rely on proxies.

Therefore, self-invocation can prevent the transactional interceptor from being triggered.

---

## Mistake 4: Making Transactions Too Large

Large transactions can lead to:

- Long-running database locks
- Connection exhaustion
- Increased contention
- Poor performance
- Difficult rollback scenarios

---

# Recommended Architecture

A common Spring Boot architecture looks like:

    Client
      |
      v
    REST Controller
      |
      v
    Service Layer
      |
      +---- @Transactional
      |
      v
    Transaction Manager
      |
      v
    Repository Layer
      |
      v
    JPA / Hibernate
      |
      v
    Database

Example:

    @RestController
    public class OrderController {

        private final OrderService orderService;

        public OrderController(OrderService orderService) {
            this.orderService = orderService;
        }

        @PostMapping("/orders")
        public void createOrder(@RequestBody Order order) {
            orderService.createOrder(order);
        }
    }

Service:

    @Service
    public class OrderService {

        private final OrderRepository orderRepository;

        public OrderService(OrderRepository orderRepository) {
            this.orderRepository = orderRepository;
        }

        @Transactional
        public void createOrder(Order order) {

            orderRepository.save(order);

            // Other database operations
        }
    }

The controller does not need to know how the transaction works.

The service defines the business transaction boundary.

---

# Summary

Spring Transaction Managers provide the infrastructure required to manage transactions in Spring applications.

The most important concepts are:

- `PlatformTransactionManager` is the main Spring transaction abstraction.
- `JpaTransactionManager` is commonly used with JPA and Hibernate.
- `DataSourceTransactionManager` is used for JDBC-based applications.
- `JtaTransactionManager` supports distributed/JTA transactions.
- `R2dbcTransactionManager` is used for reactive database transactions.
- Programmatic transaction management provides explicit control.
- Declarative transaction management uses `@Transactional` and is the preferred approach for most applications.
- Spring uses AOP proxies to implement declarative transactions.
- Multiple transaction managers can exist in the same application.
- `TransactionSynchronizationManager` helps coordinate transactional resources.
- Transaction managers work together with propagation, isolation, timeouts, and read-only settings.
- Transaction boundaries are commonly placed in the service layer.

The key relationship to remember is:

    @Transactional
          |
          v
    Spring AOP Proxy
          |
          v
    Transaction Manager
          |
          v
    Transaction
          |
          v
    Database

In most Spring Boot applications, the developer should focus on defining the correct transaction boundary with `@Transactional`, while Spring and the configured Transaction Manager handle the transaction lifecycle.
