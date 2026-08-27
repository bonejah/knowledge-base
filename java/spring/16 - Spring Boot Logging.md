# 16 - Spring Boot Logging

Logging is one of the most important mechanisms for understanding what is happening inside an application.

Logs help developers and operations teams:

* Understand application behavior
* Diagnose errors
* Troubleshoot production problems
* Monitor application health
* Investigate unexpected behavior
* Track important business operations
* Correlate events across distributed systems

In a Spring Boot application, logging is already configured by default. However, understanding how the logging ecosystem works is important when building production applications.

---

# 1. What Is Logging?

Logging is the process of recording information about an application's execution.

For example:

```java
public void createUser(User user) {
    log.info("Creating user {}", user.getEmail());

    // business logic
}
```

The application may produce:

```text
2026-08-26 21:30:15 INFO  UserService - Creating user john@example.com
```

A log entry normally contains information such as:

* Timestamp
* Log level
* Logger name
* Thread
* Message
* Exception information

Logging is different from simply printing information to the console.

For example:

```java
System.out.println("Creating user");
```

is generally not appropriate for production applications.

Instead:

```java
log.info("Creating user");
```

provides structured control over how the message is handled.

---

# 2. Why Should We Use Logging?

Logging provides visibility into the application.

Consider the following production problem:

```text
The API returned HTTP 500.
```

Without logs, it may be difficult to determine what happened.

With appropriate logging:

```text
INFO  UserService - Creating user john@example.com
INFO  UserRepository - Persisting user
ERROR UserService - Failed to create user
```

and:

```text
ERROR UserService - Failed to create user
java.sql.SQLException: Duplicate key
```

the problem becomes much easier to diagnose.

---

# 3. Logging Architecture

Java logging is commonly built around three concepts:

```text
Application
    |
    v
Logging API
    |
    v
Logging Implementation
    |
    v
Output
```

For example:

```text
Spring Application
       |
       v
     SLF4J
       |
       v
    Logback
       |
       v
 Console / File / External System
```

The important distinction is:

> SLF4J is an abstraction/API, while Logback and Log4j2 are logging implementations.

---

# 4. SLF4J

SLF4J stands for:

> Simple Logging Facade for Java

SLF4J provides a common API for logging.

Instead of writing application code directly against Logback:

```java
LogbackLogger logger = ...;
```

you write:

```java
private static final Logger log =
        LoggerFactory.getLogger(UserService.class);
```

Your application depends on the abstraction rather than the implementation.

This allows the logging implementation to be changed without rewriting application code.

For example:

```text
Application
     |
     v
   SLF4J
     |
     +----> Logback
     |
     +----> Log4j2
     |
     +----> Other implementation
```

This is one of the reasons SLF4J is widely used in Java applications.

---

# 5. Logback

Logback is a logging implementation for Java.

It is designed as a successor to the original Log4j implementation and integrates very well with Spring Boot.

A simplified architecture is:

```text
Application
     |
     v
   SLF4J
     |
     v
  Logback
     |
     +----> Console
     |
     +----> File
     |
     +----> External logging system
```

Logback provides features such as:

* Logging levels
* Appenders
* Formatting
* Rolling files
* Filtering
* Configuration
* Asynchronous logging

Spring Boot uses Logback as its default logging implementation when using the standard Spring Boot logging starter setup.

---

# 6. Log4j and Log4j2

Log4j is another Java logging framework.

There are two important versions to understand:

```text
Log4j
  |
  +----> Log4j 1.x
  |
  +----> Log4j2
```

Log4j 1.x is an older and obsolete implementation and should not be used for new applications.

Log4j2 is the modern version and provides features such as:

* High-performance logging
* Asynchronous logging
* Flexible configuration
* Multiple appenders
* Structured logging capabilities
* Advanced filtering

A modern application can therefore use:

```text
Application
     |
     v
   SLF4J
     |
     v
  Log4j2
```

or:

```text
Application
     |
     v
   SLF4J
     |
     v
  Logback
```

The application code can remain mostly unchanged because it uses SLF4J.

---

# 7. Logback vs Log4j2

Both are valid logging implementations.

| Feature                    | Logback                                      | Log4j2                         |
| -------------------------- | -------------------------------------------- | ------------------------------ |
| SLF4J support              | Yes                                          | Yes                            |
| Spring Boot integration    | Excellent                                    | Excellent                      |
| Performance                | Very good                                    | Very good                      |
| Async logging              | Supported                                    | Strong support                 |
| Configuration              | XML / Groovy / properties depending on setup | XML / JSON / YAML / properties |
| Common Spring Boot default | Yes                                          | No                             |
| Modern choice              | Yes                                          | Yes                            |

For a typical Spring Boot application, there is usually no need to replace Logback unless there is a specific technical requirement.

The important architectural principle is:

> Application code should normally depend on SLF4J rather than directly on Logback or Log4j2.

---

# 8. Logging Levels

Logging levels determine how important a log message is.

The most commonly used levels are:

```text
TRACE
DEBUG
INFO
WARN
ERROR
```

From the least severe to the most severe:

```text
TRACE
  ↓
DEBUG
  ↓
INFO
  ↓
WARN
  ↓
ERROR
```

---

# 9. TRACE

`TRACE` is the most detailed logging level.

It is generally used for extremely detailed diagnostic information.

Example:

```java
log.trace("Entering calculateTotal()");
log.trace("Cart contains {} items", items.size());
```

TRACE can generate a huge amount of data.

It is therefore usually disabled in production.

Typical usage:

```text
TRACE -> Extremely detailed debugging
```

---

# 10. DEBUG

`DEBUG` is used for information useful during development and troubleshooting.

Example:

```java
log.debug("Loading user with ID {}", userId);
```

Another example:

```java
log.debug("Received request with parameters: {}", parameters);
```

DEBUG logs are often enabled temporarily in production when investigating a problem.

Typical usage:

```text
DEBUG -> Developer troubleshooting
```

---

# 11. INFO

`INFO` represents normal application events.

Examples:

```java
log.info("Application started");
```

```java
log.info("User {} successfully created", userId);
```

```java
log.info("Payment {} completed", paymentId);
```

INFO should generally represent meaningful application events rather than every internal operation.

Good:

```java
log.info("Order {} successfully created", orderId);
```

Bad:

```java
log.info("Entering createOrder()");
log.info("Calling repository");
log.info("Repository returned");
log.info("Leaving createOrder()");
```

The second approach creates excessive noise.

Typical usage:

```text
INFO -> Important normal events
```

---

# 12. WARN

`WARN` indicates something unexpected or potentially problematic happened, but the application can continue operating.

Example:

```java
log.warn("User {} attempted to access an expired session", userId);
```

Another example:

```java
log.warn("External service response time exceeded {} ms", timeout);
```

WARN does not necessarily mean the operation failed.

Typical usage:

```text
WARN -> Potential problem / abnormal condition
```

---

# 13. ERROR

`ERROR` indicates that something failed or an operation could not be completed.

Example:

```java
log.error("Failed to create user {}", userId, exception);
```

When an exception is available, include it as the final argument:

```java
log.error("Failed to process payment {}", paymentId, exception);
```

This allows the logging framework to include the stack trace.

Typical usage:

```text
ERROR -> Operation failed / serious problem
```

---

# 14. Logging Level Filtering

Logging frameworks allow you to configure a minimum logging level.

For example:

```text
Configured level: INFO
```

Then:

```text
TRACE -> ignored
DEBUG -> ignored
INFO  -> logged
WARN  -> logged
ERROR -> logged
```

If configured as:

```text
DEBUG
```

then:

```text
TRACE -> ignored
DEBUG -> logged
INFO  -> logged
WARN  -> logged
ERROR -> logged
```

This is extremely useful in production because applications can generate a large amount of DEBUG information.

---

# 15. Logging in Spring Boot

Spring Boot provides logging configuration out of the box.

For example:

```properties
logging.level.root=INFO
```

This configures the root logger to use INFO as the minimum level.

You can configure a specific package:

```properties
logging.level.com.example.users=DEBUG
```

This means:

```text
Root application -> INFO
com.example.users -> DEBUG
```

This is useful when troubleshooting a specific part of the application.

---

# 16. Configuring Logging Levels

Example:

```properties
logging.level.root=INFO
logging.level.com.example=DEBUG
logging.level.org.springframework=WARN
```

This configuration means:

```text
Application
    |
    +---- com.example -> DEBUG
    |
    +---- org.springframework -> WARN
    |
    +---- everything else -> INFO
```

This approach can significantly reduce unnecessary logs.

---

# 17. Creating a Logger with SLF4J

Without Lombok, a logger can be created manually:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class UserService {

    private static final Logger log =
            LoggerFactory.getLogger(UserService.class);

    public void createUser(User user) {
        log.info("Creating user {}", user.getEmail());
    }
}
```

The important part is:

```java
private static final Logger log =
        LoggerFactory.getLogger(UserService.class);
```

---

# 18. SLF4J Parameterized Logging

Prefer:

```java
log.info("Creating user {}", userId);
```

instead of:

```java
log.info("Creating user " + userId);
```

SLF4J supports parameterized messages using `{}` placeholders.

Example:

```java
log.info(
    "Creating user {} with role {}",
    userId,
    role
);
```

Output:

```text
Creating user 123 with role ADMIN
```

This approach is preferable because it allows the logging framework to avoid unnecessary message construction when the corresponding log level is disabled.

---

# 19. Logging Exceptions

When logging an exception, pass the exception separately.

Prefer:

```java
try {
    userRepository.save(user);
} catch (Exception e) {
    log.error("Failed to save user {}", user.getId(), e);
}
```

This allows the logging framework to print the stack trace.

Avoid:

```java
log.error("Failed to save user: " + e.getMessage());
```

because this loses valuable diagnostic information such as the stack trace.

---

# 20. Lombok and @Slf4j

If the project uses Lombok, logging becomes much simpler.

Lombok provides the:

```java
@Slf4j
```

annotation.

Example:

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
public class UserService {

    public void createUser(User user) {
        log.info("Creating user {}", user.getEmail());
    }
}
```

Lombok automatically generates the logger.

Conceptually, Lombok generates something equivalent to:

```java
private static final Logger log =
        LoggerFactory.getLogger(UserService.class);
```

You therefore don't need to manually declare the logger.

---

# 21. @Slf4j

The most common Lombok annotation for SLF4J is:

```java
@Slf4j
```

Example:

```java
@Slf4j
@Service
public class UserService {

    public void createUser(User user) {
        log.info("Creating user {}", user.getEmail());

        userRepository.save(user);
    }
}
```

You can then use:

```java
log.trace(...)
log.debug(...)
log.info(...)
log.warn(...)
log.error(...)
```

---

# 22. Other Lombok Logging Annotations

Lombok provides several logging annotations.

For example:

```java
@Slf4j
```

creates an SLF4J logger.

Other annotations include:

```java
@Log
@Log4j
@Log4j2
```

For Spring Boot applications, `@Slf4j` is generally the preferred choice because it keeps application code dependent on SLF4J rather than directly on a specific logging implementation.

---

# 23. @Slf4j Does NOT Mean Logback

This is an important distinction.

When you write:

```java
@Slf4j
```

you are using SLF4J.

You are **not explicitly saying that the application uses Logback**.

The architecture is:

```text
@Slf4j
   |
   v
SLF4J API
   |
   v
Logging Implementation
   |
   +----> Logback
   |
   +----> Log4j2
```

Therefore:

```java
log.info("Hello");
```

uses the SLF4J API.

The underlying implementation determines how the message is actually processed.

---

# 24. Logging Example in a Spring Service

A typical Spring service could look like:

```java
@Slf4j
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User createUser(User user) {

        log.info("Creating user {}", user.getEmail());

        try {

            User savedUser = userRepository.save(user);

            log.info(
                "User {} successfully created",
                savedUser.getId()
            );

            return savedUser;

        } catch (Exception e) {

            log.error(
                "Failed to create user {}",
                user.getEmail(),
                e
            );

            throw e;
        }
    }
}
```

This provides useful information without exposing every internal implementation detail.

---

# 25. Logging in Controllers

Logging can also be used at the controller level.

Example:

```java
@Slf4j
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    public User createUser(@RequestBody User user) {

        log.info("Received request to create user");

        return userService.createUser(user);
    }
}
```

However, avoid logging sensitive request data.

For example, do not log:

```java
log.info("Login request: {}", loginRequest);
```

if the object contains:

```text
password
access token
refresh token
credit card information
personal information
```

---

# 26. What Should We Log?

Good candidates include:

### Application lifecycle

```java
log.info("Application started");
```

### Important business operations

```java
log.info("Order {} created", orderId);
```

### External service interactions

```java
log.debug("Calling payment provider for order {}", orderId);
```

### Unexpected situations

```java
log.warn("Retrying payment request for order {}", orderId);
```

### Errors

```java
log.error("Payment processing failed for order {}", orderId, exception);
```

---

# 27. What Should We NOT Log?

Avoid logging sensitive information.

Never log things such as:

```text
Passwords
Access tokens
Refresh tokens
API keys
Client secrets
Credit card numbers
Private keys
Session credentials
```

Bad:

```java
log.info("User login: {} / {}", username, password);
```

Bad:

```java
log.debug("Authorization header: {}", authorizationHeader);
```

Bad:

```java
log.debug("Refresh token: {}", refreshToken);
```

Even DEBUG logs can end up in production systems.

---

# 28. Logging and Personal Data

Be careful with personally identifiable information (PII).

Examples include:

```text
Email addresses
Phone numbers
Addresses
Government IDs
Financial information
Health information
```

Whether specific data can be logged depends on the application's requirements, jurisdiction, and organizational policies.

A good rule is:

> Log only what is necessary to understand and operate the system.

---

# 29. Logging at Different Layers

A common Spring application might look like:

```text
Controller
    |
    v
Service
    |
    v
Repository
```

Logging should generally focus on meaningful events.

For example:

```text
INFO  Controller - Request received
INFO  Service    - Creating order 123
DEBUG Repository - Executing query
INFO  Service    - Order 123 created
```

Avoid logging the same event repeatedly at every layer.

For example, this is noisy:

```text
Controller -> Creating order
Service    -> Creating order
Repository -> Creating order
Database   -> Creating order
```

The service may be the appropriate place to log the business event, while lower layers can use DEBUG for technical troubleshooting.

---

# 30. Logging and Exceptions

Logging and exception handling are related but not the same thing.

Consider:

```java
try {
    processPayment();
} catch (PaymentException e) {

    log.error("Payment failed", e);

    throw e;
}
```

The exception is still propagated.

Logging does not replace exception handling.

Also avoid logging an exception multiple times unnecessarily:

```text
Repository -> ERROR
Service    -> ERROR
Controller -> ERROR
```

This can produce three identical stack traces.

A better architecture is often to log the exception at the boundary where it is handled, while allowing lower layers to propagate it.

---

# 31. Structured Logging

Modern applications increasingly use structured logs.

Instead of:

```text
User 123 created successfully
```

you may produce structured information such as:

```json
{
  "event": "user_created",
  "userId": "123",
  "service": "user-service"
}
```

Structured logs are easier for centralized logging systems to search and analyze.

They are especially useful in:

* Microservices
* Kubernetes
* Cloud applications
* Distributed systems
* Observability platforms

---

# 32. Correlation IDs

In distributed systems, a single request may travel through multiple services.

For example:

```text
Client
  |
  v
API Gateway
  |
  v
User Service
  |
  v
Payment Service
  |
  v
Notification Service
```

A correlation ID can identify the same request across all services.

Example:

```text
correlationId=abc-123
```

Logs might look like:

```text
INFO UserService    correlationId=abc-123 Creating user
INFO PaymentService correlationId=abc-123 Processing payment
INFO Notification   correlationId=abc-123 Sending email
```

This makes troubleshooting distributed systems significantly easier.

---

# 33. Logging vs Monitoring vs Tracing

Logging is only one part of observability.

A modern application commonly uses:

```text
Observability
    |
    +---- Logs
    |
    +---- Metrics
    |
    +---- Traces
```

### Logs

Tell you:

> What happened?

Example:

```text
Payment failed for order 123
```

### Metrics

Tell you:

> How much / how often?

Example:

```text
HTTP 500 rate = 2.4%
```

### Traces

Tell you:

> Where did the request spend time?

Example:

```text
API Gateway
    |
    +-- User Service 120ms
    |
    +-- Payment Service 850ms
```

These three signals complement each other.

---

# 34. Best Practices

## 34.1 Use SLF4J

Prefer:

```java
@Slf4j
```

or:

```java
private static final Logger log =
        LoggerFactory.getLogger(MyClass.class);
```

rather than directly coupling application code to Logback or Log4j2.

---

## 34.2 Use Appropriate Log Levels

Use:

```text
TRACE -> Extremely detailed diagnostics
DEBUG -> Development/troubleshooting
INFO  -> Normal important events
WARN  -> Potential problems
ERROR -> Failures
```

Do not use ERROR for everything.

---

## 34.3 Use Parameterized Logging

Prefer:

```java
log.info("User {} created", userId);
```

over:

```java
log.info("User " + userId + " created");
```

---

## 34.4 Do Not Log Secrets

Never log:

```text
Passwords
Tokens
API keys
Secrets
Private keys
```

---

## 34.5 Do Not Overuse INFO

Avoid:

```java
log.info("Entering method");
log.info("Calling repository");
log.info("Repository returned");
log.info("Leaving method");
```

Use DEBUG for low-level troubleshooting instead.

---

## 34.6 Include Useful Context

Prefer:

```java
log.error(
    "Failed to process order {} for customer {}",
    orderId,
    customerId,
    exception
);
```

over:

```java
log.error("Something went wrong");
```

The first message provides useful diagnostic context.

---

## 34.7 Include the Exception

Prefer:

```java
log.error("Failed to process payment {}", paymentId, exception);
```

instead of:

```java
log.error("Failed to process payment: {}", exception.getMessage());
```

The stack trace is usually critical for troubleshooting.

---

## 34.8 Avoid Duplicate Logging

Do not automatically log the same exception at every layer.

Instead, decide where the exception is actually handled and log it there.

---

## 34.9 Keep Production Logging Manageable

A common production configuration is:

```text
INFO
```

with specific packages temporarily changed to:

```text
DEBUG
```

when troubleshooting.

---

# 35. Example Architecture

A typical Spring Boot application might use:

```text
                    Spring Boot Application
                              |
                              v
                           @Slf4j
                              |
                              v
                            SLF4J
                              |
                              v
                           Logback
                              |
                 +------------+------------+
                 |            |            |
                 v            v            v
              Console       File       Log Platform
```

For example:

```java
@Slf4j
@Service
public class PaymentService {

    public void processPayment(String paymentId) {

        log.info(
            "Starting payment processing: {}",
            paymentId
        );

        try {

            // Payment processing

            log.info(
                "Payment {} successfully processed",
                paymentId
            );

        } catch (Exception e) {

            log.error(
                "Payment {} processing failed",
                paymentId,
                e
            );

            throw e;
        }
    }
}
```

---

# 36. Key Takeaways

The Java logging ecosystem can be summarized as:

```text
SLF4J
  |
  |-- Logging abstraction/API
  |
  v
Logback / Log4j2
  |
  |-- Logging implementation
  |
  v
Console / File / Centralized Logging System
```

The most important concepts are:

1. **SLF4J** is the logging abstraction.
2. **Logback** and **Log4j2** are logging implementations.
3. **Log4j 1.x** is obsolete and should not be used for new applications.
4. Spring Boot commonly uses **Logback by default**.
5. `@Slf4j` from Lombok automatically creates an SLF4J logger.
6. Use appropriate logging levels:

   * `TRACE`
   * `DEBUG`
   * `INFO`
   * `WARN`
   * `ERROR`
7. Prefer parameterized logging with `{}`.
8. Include exceptions as separate arguments to preserve stack traces.
9. Never log secrets or sensitive information.
10. Avoid excessive and duplicated logging.
11. Use structured logs and correlation IDs in distributed systems.
12. Logging is one of the three major observability signals alongside metrics and traces.

The most common Spring Boot pattern is therefore:

```java
@Slf4j
@Service
public class UserService {

    public void createUser(User user) {

        log.info("Creating user {}", user.getId());

        try {

            // Business logic

            log.info(
                "User {} successfully created",
                user.getId()
            );

        } catch (Exception e) {

            log.error(
                "Failed to create user {}",
                user.getId(),
                e
            );

            throw e;
        }
    }
}
```

This gives you a clean separation:

```text
Your Code
    |
    v
  @Slf4j
    |
    v
  SLF4J
    |
    v
 Logback / Log4j2
    |
    v
 Log Destination
```

That separation is one of the most important concepts to understand when working with logging in Spring Boot.
