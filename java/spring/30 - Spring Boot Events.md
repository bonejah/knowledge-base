# 30 - Spring Boot Events

## 1. What are Spring Boot Events?

Spring Boot Events are a mechanism provided by the Spring Framework that allows different components of an application to communicate through **events**.

Instead of one component directly calling another, a component can **publish an event**, and one or more other components can **listen and react**.

This is useful when an action needs to trigger multiple independent operations.

### Example

Imagine a user registers in your application.

After the registration, you may need to:

- Send a welcome email.
- Create an audit record.
- Notify another system.
- Update a cache.
- Publish a message to a message broker.

Without events, the registration service might need to call all these components directly.

With events, the registration service only publishes a `UserRegisteredEvent`.

The other components decide independently whether they need to react to that event.

---

## 2. Pub-Sub Design Pattern

Spring Events are based on the **Publish-Subscribe (Pub-Sub) design pattern**.

### Main participants

| Participant               | Responsibility                                  |
| ------------------------- | ----------------------------------------------- |
| **Publisher**             | Publishes an event when something happens.      |
| **Event**                 | Represents what happened.                       |
| **Subscriber / Listener** | Receives the event and executes a reaction.     |
| **Event Bus**             | Delivers the event to the registered listeners. |

### Traditional direct communication

    UserService
        |
        +----> EmailService
        |
        +----> AuditService
        |
        +----> NotificationService

The `UserService` knows about every service it needs to call.

### Communication using events

    UserService
        |
        v
    Event Bus
        |
        +----> WelcomeEmailListener
        |
        +----> AuditListener
        |
        +----> NotificationListener

The publisher does not need to know which listeners exist.

### Main advantage

The publisher and subscribers are **loosely coupled**.

This means that you can add a new listener without modifying the publisher.

---

## 3. What is an Event?

An event is an object that represents something that happened in the application.

For example:

- `UserRegisteredEvent`
- `OrderCreatedEvent`
- `PaymentCompletedEvent`
- `PasswordChangedEvent`
- `FileUploadedEvent`

An event usually contains the information that listeners need to process the event.

### Example

    public record UserRegisteredEvent(
            Long userId,
            String email
    ) {
    }

This event represents the fact that a user has been registered.

The event does not need to contain business logic.

It is simply a message describing an occurrence.

---

## 4. Event Publisher and @EventListener

Spring provides two important mechanisms for working with events:

- `ApplicationEventPublisher`
- `@EventListener`

### ApplicationEventPublisher

The `ApplicationEventPublisher` is responsible for publishing events.

You can inject it into a Spring bean and call:

    applicationEventPublisher.publishEvent(event);

### @EventListener

The `@EventListener` annotation marks a method as a listener.

Spring automatically detects these methods and invokes them when a matching event is published.

### Example

    @Component
    public class WelcomeEmailListener {

        @EventListener
        public void handleUserRegistered(UserRegisteredEvent event) {
            System.out.println(
                    "Sending welcome email to: " + event.email()
            );
        }
    }

When a `UserRegisteredEvent` is published, Spring calls the listener method.

---

## 5. Basic Architecture

The internal architecture can be represented as:

    Application Component
            |
            | publishEvent(...)
            v
    ApplicationEventPublisher
            |
            v
    ApplicationEventMulticaster
            |
            v
    Registered Listeners
            |
            v
    Listener Method
    @EventListener

### Main components

| Component                      | Responsibility                                   |
| ------------------------------ | ------------------------------------------------ |
| `ApplicationEventPublisher`    | Publishes the event.                             |
| `ApplicationEventMulticaster`  | Finds and invokes the appropriate listeners.     |
| `ApplicationListener`          | Represents a listener internally.                |
| `@EventListener`               | Registers a method as an event listener.         |
| `EventListenerMethodProcessor` | Detects methods annotated with `@EventListener`. |

---

## 6. Internal Working

When Spring starts, it scans the application beans.

It detects methods annotated with `@EventListener` and registers them as listeners.

When an event is published:

1. The publisher receives the event.
2. Spring delegates the event to the event multicaster.
3. The multicaster identifies the listeners that support that event type.
4. The listeners are invoked.
5. Each listener executes its business logic.

### Example

    1. UserService publishes UserRegisteredEvent
                            |
                            v
    2. ApplicationEventPublisher
                            |
                            v
    3. ApplicationEventMulticaster
                            |
                            v
    4. Find listeners for UserRegisteredEvent
                            |
                  +---------+---------+
                  |                   |
                  v                   v
    5. WelcomeEmailListener   AuditListener
                  |                   |
                  v                   v
    6. Send email            Save audit record

### Important

By default, Spring application events are **synchronous**.

This means that the publisher waits for the listeners to finish.

    Publisher
        |
        | publishEvent()
        v
    Listener 1
        |
        | finishes
        v
    Listener 2
        |
        | finishes
        v
    Publisher continues

---

## 7. Synchronous Events

A synchronous listener executes in the same thread as the publisher.

### Example

    @Component
    public class AuditListener {

        @EventListener
        public void handleUserRegistered(UserRegisteredEvent event) {
            System.out.println("Saving audit record...");
        }
    }

### Flow

    UserService
        |
        | publishEvent()
        v
    AuditListener
        |
        | executes
        v
    UserService continues

### Characteristics

- The publisher waits for the listener.
- The listener runs in the same thread.
- Exceptions can propagate back to the publisher.
- The listener can affect the execution time of the publisher.

### When to use

Synchronous events are useful when the listener must complete before the publisher continues.

Examples:

- Updating in-memory state.
- Executing a required validation.
- Performing a local operation that must finish immediately.

---

## 8. Asynchronous Events

An asynchronous listener executes in a different thread.

This allows the publisher to continue without waiting for the listener to finish.

### Example

    @Component
    public class WelcomeEmailListener {

        @Async
        @EventListener
        public void handleUserRegistered(UserRegisteredEvent event) {
            System.out.println("Sending email asynchronously...");
        }
    }

### Required configuration

    @Configuration
    @EnableAsync
    public class AsyncConfig {
    }

### Flow

    UserService
        |
        | publishEvent()
        v
    Event Multicaster
        |
        +--------------------+
        |                    |
        v                    v
    Publisher continues   Async Executor
                               |
                               v
                        WelcomeEmailListener

### Characteristics

- The publisher does not wait for the listener to finish.
- The listener executes in another thread.
- The listener may continue after the publisher returns.
- Exceptions are handled differently from synchronous listeners.

### When to use

Asynchronous events are useful for operations that do not need to block the main request.

Examples:

- Sending emails.
- Sending notifications.
- Generating reports.
- Processing analytics.
- Calling external systems.

---

## 9. Synchronous vs Asynchronous

| Feature           | Synchronous          | Asynchronous            |
| ----------------- | -------------------- | ----------------------- |
| Execution thread  | Publisher thread     | Different thread        |
| Publisher waits?  | Yes                  | No                      |
| Default behavior  | Yes                  | No                      |
| Uses `@Async`?    | No                   | Yes                     |
| Error propagation | Can propagate        | Requires async handling |
| Useful for        | Immediate operations | Background operations   |

### Important distinction

Spring application events are **in-process events**.

They are not automatically distributed messages.

For example:

    Spring Application
        |
        +----> Event Listener

This is different from a message broker such as Kafka or RabbitMQ.

    Application A
        |
        v
    Message Broker
        |
        v
    Application B

---

## 10. Event Listeners: Conditional

Spring allows you to execute a listener only when a condition is satisfied.

You can use the `condition` attribute of `@EventListener`.

### Example

    @Component
    public class PremiumUserListener {

        @EventListener(condition = "#event.premium")
        public void handleUserRegistered(UserRegisteredEvent event) {
            System.out.println("Processing premium user...");
        }
    }

### Event

    public record UserRegisteredEvent(
            Long userId,
            String email,
            boolean premium
    ) {
    }

### How it works

    UserRegisteredEvent
            |
            v
    Is premium == true?
            |
       +----+----+
       |         |
      Yes        No
       |         |
       v         v
    Execute   Ignore

### Important

The condition is evaluated before the listener method executes.

If the condition is false, the listener is not invoked.

---

## 11. Event Listeners: Ordered

Sometimes multiple listeners need to execute in a specific order.

Spring provides the `@Order` annotation for this purpose.

### Example

    @Component
    public class FirstListener {

        @Order(1)
        @EventListener
        public void handle(UserRegisteredEvent event) {
            System.out.println("First listener");
        }
    }

    @Component
    public class SecondListener {

        @Order(2)
        @EventListener
        public void handle(UserRegisteredEvent event) {
            System.out.println("Second listener");
        }
    }

### Execution order

    UserRegisteredEvent
            |
            v
    FirstListener
        @Order(1)
            |
            v
    SecondListener
        @Order(2)

### Important

Ordering is useful when multiple listeners need to execute sequentially.

However, you should avoid creating unnecessary dependencies between listeners.

If the business process requires a strict sequence, consider whether a direct service call or an explicit workflow would be clearer.

---

## 12. Event Listeners: Async and Ordered

You can combine `@Async` and `@Order`.

However, ordering behavior is different when listeners execute asynchronously.

### Example

    @Component
    public class FirstListener {

        @Async
        @Order(1)
        @EventListener
        public void handle(UserRegisteredEvent event) {
            System.out.println("First async listener");
        }
    }

    @Component
    public class SecondListener {

        @Async
        @Order(2)
        @EventListener
        public void handle(UserRegisteredEvent event) {
            System.out.println("Second async listener");
        }
    }

### Important

`@Order` determines the order in which listeners are submitted to the executor.

It does **not** guarantee that the asynchronous business logic will finish in that order.

    Publisher
        |
        v
    Submit FirstListener
        |
        v
    Submit SecondListener
        |
        +----------------------+
        |                      |
        v                      v
    Async execution       Async execution
    FirstListener         SecondListener

If strict execution order is required, asynchronous listeners may not be the right solution.

---

## 13. Global Error Handling in Listeners

Error handling is important because listeners may fail while processing an event.

### Synchronous listeners

For synchronous listeners, an exception can propagate back to the publisher.

### Example

    @Component
    public class AuditListener {

        @EventListener
        public void handle(UserRegisteredEvent event) {
            throw new RuntimeException(
                    "Failed to save audit record"
            );
        }
    }

If this listener is executed synchronously, the exception can affect the method that published the event.

### Flow

    UserService
        |
        | publishEvent()
        v
    AuditListener
        |
        | throws exception
        v
    Exception propagates
        |
        v
    UserService

### Asynchronous listeners

For asynchronous listeners, the publisher does not receive the exception directly.

The exception happens in the async thread.

Therefore, you need a strategy to handle it.

---

## 14. AsyncUncaughtExceptionHandler

Spring provides `AsyncUncaughtExceptionHandler` for handling uncaught exceptions from asynchronous methods that return `void`.

### Example

    @Configuration
    @EnableAsync
    public class AsyncConfig implements AsyncConfigurer {

        @Override
        public AsyncUncaughtExceptionHandler
        getAsyncUncaughtExceptionHandler() {

            return new AsyncUncaughtExceptionHandler() {

                @Override
                public void handleUncaughtException(
                        Throwable ex,
                        Method method,
                        Object... params
                ) {
                    System.out.println(
                            "Async error: " + ex.getMessage()
                    );
                }
            };
        }
    }

### Listener

    @Component
    public class WelcomeEmailListener {

        @Async
        @EventListener
        public void handleUserRegistered(UserRegisteredEvent event) {
            throw new RuntimeException(
                    "Email service unavailable"
            );
        }
    }

### Flow

    Async Listener
          |
          | throws exception
          v
    AsyncUncaughtExceptionHandler
          |
          v
    Log / Alert / Handle error

### Important

This handler is useful for logging or handling uncaught exceptions in asynchronous methods.

For more complex error handling, you may want to use a dedicated error-handling strategy.

---

## 15. Global Error Handling with @ControllerAdvice

`@ControllerAdvice` is commonly used for handling exceptions thrown by controllers.

However, it is important to understand that it is **not a general-purpose handler for all asynchronous listener exceptions**.

### Example

    @RestControllerAdvice
    public class GlobalExceptionHandler {

        @ExceptionHandler(RuntimeException.class)
        public ResponseEntity<String> handleRuntimeException(
                RuntimeException ex
        ) {
            return ResponseEntity
                    .status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(ex.getMessage());
        }
    }

This handles exceptions that reach the controller layer.

It does not automatically handle exceptions thrown in a background async listener after the HTTP request has returned.

### Recommended approach

For event listeners, consider:

- Handling exceptions inside the listener.
- Using `AsyncUncaughtExceptionHandler`.
- Logging failures.
- Sending alerts.
- Persisting failed events.
- Retrying the operation.
- Using a message broker when reliable delivery is required.

---

## 16. Example: Spring Boot Application

Let's create a simple application that publishes a `UserRegisteredEvent`.

The application will contain:

- A REST controller.
- A service that registers users.
- A synchronous audit listener.
- An asynchronous email listener.
- Global async error handling.

### Project structure

    src/main/java/com/example/events
        |
        +---- EventsApplication.java
        |
        +---- config
        |       |
        |       +---- AsyncConfig.java
        |
        +---- controller
        |       |
        |       +---- UserController.java
        |
        +---- service
        |       |
        |       +---- UserService.java
        |
        +---- event
        |       |
        |       +---- UserRegisteredEvent.java
        |
        +---- listener
                |
                +---- AuditListener.java
                +---- WelcomeEmailListener.java

---

## 17. Event Class

    package com.example.events.event;

    public record UserRegisteredEvent(
            Long userId,
            String email
    ) {
    }

This record represents the event that a user has been registered.

---

## 18. UserService

The service publishes the event after registering the user.

    package com.example.events.service;

    import com.example.events.event.UserRegisteredEvent;
    import org.springframework.context.ApplicationEventPublisher;
    import org.springframework.stereotype.Service;

    @Service
    public class UserService {

        private final ApplicationEventPublisher eventPublisher;

        public UserService(ApplicationEventPublisher eventPublisher) {
            this.eventPublisher = eventPublisher;
        }

        public void registerUser(Long userId, String email) {

            System.out.println("Registering user...");

            // Simulate saving the user
            System.out.println("User saved successfully");

            eventPublisher.publishEvent(
                    new UserRegisteredEvent(userId, email)
            );

            System.out.println("Registration flow finished");
        }
    }

### What happens here?

1. The user is registered.
2. The event is published.
3. Spring notifies the listeners.
4. The listeners execute.
5. The service continues.

---

## 19. Synchronous Audit Listener

    package com.example.events.listener;

    import com.example.events.event.UserRegisteredEvent;
    import org.springframework.context.event.EventListener;
    import org.springframework.stereotype.Component;

    @Component
    public class AuditListener {

        @EventListener
        public void handleUserRegistered(UserRegisteredEvent event) {

            System.out.println(
                    "Saving audit record for user: " + event.userId()
            );
        }
    }

This listener executes synchronously.

### Execution

    UserService
        |
        v
    publishEvent()
        |
        v
    AuditListener
        |
        v
    Save audit record
        |
        v
    UserService continues

---

## 20. Asynchronous Email Listener

    package com.example.events.listener;

    import com.example.events.event.UserRegisteredEvent;
    import org.springframework.scheduling.annotation.Async;
    import org.springframework.context.event.EventListener;
    import org.springframework.stereotype.Component;

    @Component
    public class WelcomeEmailListener {

        @Async
        @EventListener
        public void handleUserRegistered(UserRegisteredEvent event) {

            System.out.println(
                    "Sending welcome email to: " + event.email()
            );
        }
    }

This listener executes asynchronously.

The publisher does not wait for the email operation to finish.

---

## 21. Enable Async Processing

    package com.example.events.config;

    import org.springframework.context.annotation.Configuration;
    import org.springframework.scheduling.annotation.EnableAsync;

    @Configuration
    @EnableAsync
    public class AsyncConfig {
    }

The `@EnableAsync` annotation enables asynchronous method execution.

---

## 22. REST Controller

    package com.example.events.controller;

    import com.example.events.service.UserService;
    import org.springframework.web.bind.annotation.*;

    @RestController
    @RequestMapping("/users")
    public class UserController {

        private final UserService userService;

        public UserController(UserService userService) {
            this.userService = userService;
        }

        @PostMapping
        public String registerUser() {

            userService.registerUser(
                    1L,
                    "user@example.com"
            );

            return "User registered";
        }
    }

---

## 23. Application Flow

When the endpoint is called:

    POST /users
            |
            v
    UserController
            |
            v
    UserService
            |
            v
    publishEvent(UserRegisteredEvent)
            |
            +-----------------------------+
            |                             |
            v                             v
    AuditListener                 WelcomeEmailListener
            |                             |
            v                             v
    Synchronous                  Asynchronous
            |                             |
            v                             v
    Save audit record            Send welcome email

### Example output

    Registering user...
    User saved successfully
    Saving audit record for user: 1
    Registration flow finished
    Sending welcome email to: user@example.com

The exact order of asynchronous output may vary.

---

## 24. Adding Conditional Listeners

Suppose we want to send a special email only to premium users.

### Event

    public record UserRegisteredEvent(
            Long userId,
            String email,
            boolean premium
    ) {
    }

### Listener

    @Component
    public class PremiumUserListener {

        @EventListener(condition = "#event.premium")
        public void handlePremiumUser(UserRegisteredEvent event) {

            System.out.println(
                    "Sending premium welcome email to: " + event.email()
            );
        }
    }

### Publisher

    eventPublisher.publishEvent(
            new UserRegisteredEvent(
                    userId,
                    email,
                    true
            )
    );

### Result

    UserRegisteredEvent
            |
            v
    premium == true?
            |
            v
    PremiumUserListener
            |
            v
    Send premium email

---

## 25. Adding Ordered Listeners

Suppose we want to execute the audit listener before another listener.

### First listener

    @Component
    public class AuditListener {

        @Order(1)
        @EventListener
        public void handleUserRegistered(UserRegisteredEvent event) {

            System.out.println("Saving audit record...");
        }
    }

### Second listener

    @Component
    public class NotificationListener {

        @Order(2)
        @EventListener
        public void handleUserRegistered(UserRegisteredEvent event) {

            System.out.println("Sending notification...");
        }
    }

### Execution

    UserRegisteredEvent
            |
            v
    AuditListener
        @Order(1)
            |
            v
    NotificationListener
        @Order(2)

---

## 26. Global Async Error Handling Example

Let's configure a custom async error handler.

    package com.example.events.config;

    import org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.scheduling.annotation.AsyncConfigurer;
    import org.springframework.scheduling.annotation.EnableAsync;

    @Configuration
    @EnableAsync
    public class AsyncConfig implements AsyncConfigurer {

        @Override
        public AsyncUncaughtExceptionHandler
        getAsyncUncaughtExceptionHandler() {

            return (ex, method, params) -> {

                System.out.println(
                        "Async listener failed in method: "
                                + method.getName()
                );

                System.out.println(
                        "Error: " + ex.getMessage()
                );
            };
        }
    }

### Listener that fails

    @Component
    public class WelcomeEmailListener {

        @Async
        @EventListener
        public void handleUserRegistered(UserRegisteredEvent event) {

            throw new RuntimeException(
                    "Email service unavailable"
            );
        }
    }

### Flow

    UserService
        |
        v
    publishEvent()
        |
        v
    Async Listener
        |
        v
    Exception
        |
        v
    AsyncUncaughtExceptionHandler
        |
        v
    Log error

---

## 27. Important: Events and Transactions

One of the most important concepts in Spring Events is the relationship between events and database transactions.

Suppose you publish an event inside a transaction.

    @Transactional
    public void registerUser() {

        userRepository.save(user);

        eventPublisher.publishEvent(
                new UserRegisteredEvent(
                        user.getId(),
                        user.getEmail()
                )
        );
    }

The event may be delivered **before the transaction commits**.

This means that a listener could execute even if the transaction later rolls back.

### Example problem

    Transaction starts
            |
            v
    Save user
            |
            v
    Publish event
            |
            v
    Listener sends email
            |
            v
    Transaction rolls back

The email may have already been sent even though the user was not saved.

---

## 28. @TransactionalEventListener

Spring provides `@TransactionalEventListener` for transaction-aware event processing.

It allows a listener to execute at a specific transaction phase.

### Example

    @Component
    public class WelcomeEmailListener {

        @TransactionalEventListener
        public void handleUserRegistered(UserRegisteredEvent event) {

            System.out.println(
                    "Sending email after transaction commit"
            );
        }
    }

By default, `@TransactionalEventListener` executes after the transaction commits.

### Flow

    Transaction starts
            |
            v
    Save user
            |
            v
    Publish event
            |
            v
    Transaction commits
            |
            v
    TransactionalEventListener
            |
            v
    Send email

### Important

`@TransactionalEventListener` is useful when the listener should react only after the transaction has successfully completed.

---

## 29. Transaction Phases

`@TransactionalEventListener` supports different transaction phases.

    @TransactionalEventListener(
            phase = TransactionPhase.AFTER_COMMIT
    )
    public void handle(UserRegisteredEvent event) {
    }

### Available phases

| Phase              | Description                                                                 |
| ------------------ | --------------------------------------------------------------------------- |
| `BEFORE_COMMIT`    | Executes before the transaction commits.                                    |
| `AFTER_COMMIT`     | Executes after the transaction commits.                                     |
| `AFTER_ROLLBACK`   | Executes after the transaction rolls back.                                  |
| `AFTER_COMPLETION` | Executes after the transaction completes, regardless of commit or rollback. |

### Example

    @Component
    public class UserEventListener {

        @TransactionalEventListener(
                phase = TransactionPhase.AFTER_ROLLBACK
        )
        public void handleRollback(UserRegisteredEvent event) {

            System.out.println(
                    "Transaction rolled back for user: "
                            + event.userId()
            );
        }
    }

---

## 30. Best Practices

### 1. Use meaningful event names

Prefer:

    UserRegisteredEvent

Instead of:

    UserEvent

The event should clearly communicate what happened.

### 2. Keep events simple

Events should usually contain data, not complex business logic.

### 3. Avoid unnecessary coupling

The publisher should not depend on the implementation details of listeners.

### 4. Use synchronous events when completion matters

If the publisher needs the listener to finish before continuing, synchronous execution may be appropriate.

### 5. Use asynchronous events for background work

For example:

- Emails.
- Notifications.
- Reports.
- Analytics.

### 6. Handle async errors

Do not assume that exceptions from asynchronous listeners will automatically reach the controller.

### 7. Be careful with transactions

If a listener depends on committed data, consider using `@TransactionalEventListener`.

### 8. Do not use events for everything

Events are useful for decoupling, but excessive use can make the application difficult to understand.

If one service must always call another service in a specific order, a direct service call may be clearer.

---

## 31. Spring Events vs Message Brokers

Spring Events are useful for communication **inside the same application**.

Message brokers are useful for communication **between applications or services**.

| Feature               | Spring Events               | Message Broker      |
| --------------------- | --------------------------- | ------------------- |
| Communication         | In-process                  | Usually distributed |
| Delivery              | In-memory                   | Broker-managed      |
| Persistence           | Not automatic               | Often supported     |
| Retry                 | Not automatic               | Often supported     |
| Multiple applications | No                          | Yes                 |
| Example               | `ApplicationEventPublisher` | Kafka / RabbitMQ    |

### Example

    Spring Events

    UserService
        |
        v
    ApplicationEventPublisher
        |
        v
    EmailListener

    Message Broker

    UserService
        |
        v
    Kafka / RabbitMQ
        |
        v
    Email Service

### Important

Spring Events do not automatically provide:

- Durable event storage.
- Guaranteed delivery.
- Automatic retries.
- Cross-application communication.

If these features are required, a message broker or another reliable messaging solution may be more appropriate.

---

## 32. Summary

Spring Boot Events provide a simple way to implement the **Publish-Subscribe design pattern**.

The main idea is:

> A component publishes an event, and other components listen and react to it.

### Key concepts

- `ApplicationEventPublisher` publishes events.
- `@EventListener` defines event listeners.
- Spring uses an event multicaster to deliver events.
- Events are synchronous by default.
- `@Async` enables asynchronous listener execution.
- `condition` allows conditional listeners.
- `@Order` controls listener ordering.
- `AsyncUncaughtExceptionHandler` handles uncaught async exceptions.
- `@TransactionalEventListener` allows transaction-aware event processing.

### Final architecture

    Controller
        |
        v
    Service
        |
        | publishEvent(...)
        v
    ApplicationEventPublisher
        |
        v
    Event Multicaster
        |
        +------------------+
        |                  |
        v                  v
    Sync Listener      Async Listener
        |                  |
        v                  v
    Execute            Executor
    immediately           |
                          v
                      Background
                      execution

### Main takeaway

Spring Events are a powerful way to decouple components inside a Spring Boot application.

They are especially useful when one action needs to trigger multiple independent reactions without making the publisher responsible for every operation.
