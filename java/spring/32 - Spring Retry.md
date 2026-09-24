# 32 - Spring Retry

## 1. What is Retry?

**Retry** is a mechanism that automatically attempts an operation again when it fails due to a temporary or transient error.

In distributed systems, failures are common and can happen for reasons such as:

- Temporary network problems
- Database connection failures
- HTTP 5xx responses
- Timeout errors
- Temporary unavailability of another service
- Rate limiting
- Message broker interruptions
- Cloud service transient failures

Instead of immediately returning an error to the caller, the application can retry the operation after a configurable delay.

### Without Retry

    Application
        |
        v
    External Service
        |
        X
      Failure
        |
        v
      Error

### With Retry

    Application
        |
        v
    External Service
        |
        X
      Failure
        |
        v
      Wait
        |
        v
      Retry
        |
        v
    External Service
        |
        v
      Success

---

## 2. Retry Use Cases

Retry is useful when failures are expected to be **temporary**.

### Example: External REST API

Suppose your application calls a payment service:

    POST /payments

The payment service temporarily returns:

    HTTP 503 Service Unavailable

Instead of immediately failing the entire operation, the application can retry:

    Attempt 1 -> 503
    Wait 1 second
    Attempt 2 -> 503
    Wait 2 seconds
    Attempt 3 -> 200 OK

This can improve resilience without requiring the user to repeat the operation.

### Common Use Cases

- REST API calls
- Database operations
- Kafka operations
- Message processing
- Cloud API calls
- File operations
- External authentication services
- Identity verification services
- Payment services
- Temporary infrastructure failures

---

# 3. When NOT to Use Retry

Retry should not be used blindly.

Some errors are permanent and retrying will not solve the problem.

### Example

    POST /users

If the response is:

    HTTP 400 Bad Request

Retrying the same request will probably produce the same error.

Other examples:

- Invalid input
- Authentication failure
- Authorization failure
- Resource does not exist
- Business validation failure
- Invalid request format

A good retry strategy should distinguish between:

    Transient Error
        |
        +----> Retry

    Permanent Error
        |
        +----> Fail immediately

---

# 4. Spring Retry

**Spring Retry** provides support for automatically retrying operations that fail.

Instead of implementing retry logic manually:

    for (...)
        try operation
        catch exception
        wait
        try again

Spring Retry allows this behavior to be configured using annotations and policies.

A typical flow is:

    Client
      |
      v
    Service
      |
      v
    @Retryable Method
      |
      +---- Exception
      |
      v
    Retry Policy
      |
      +---- Retry
      |
      +---- Retry
      |
      +---- Retry
      |
      v
    Success / @Recover

---

# 5. Enabling Spring Retry

Add the Spring Retry dependency to the project.

For Maven:

    org.springframework.retry:spring-retry

Spring Retry also commonly uses Spring AOP to intercept method calls.

For Spring Boot applications, the AOP starter is typically included as well:

    org.springframework.boot:spring-boot-starter-aop

---

# 6. Enable Retry

Spring Retry must be enabled in the Spring application.

Use:

    @EnableRetry

Example:

    @SpringBootApplication
    @EnableRetry
    public class Application {

        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }

The important annotation is:

    @EnableRetry

It enables Spring's retry infrastructure and allows Spring to create proxies around retryable beans.

---

# 7. @Retryable

The `@Retryable` annotation marks a method that should be automatically retried when a configured exception occurs.

Example:

    @Service
    public class PaymentService {

        @Retryable(
            retryFor = PaymentServiceException.class,
            maxAttempts = 3
        )
        public void processPayment() {

            System.out.println("Processing payment...");

            callPaymentProvider();
        }
    }

If `PaymentServiceException` is thrown:

    Attempt 1
       |
       X PaymentServiceException
       |
    Attempt 2
       |
       X PaymentServiceException
       |
    Attempt 3
       |
       v
    Success

If all attempts fail, the exception is propagated unless a recovery method is configured.

---

# 8. maxAttempts

`maxAttempts` defines the maximum number of executions.

Example:

    @Retryable(
        retryFor = PaymentServiceException.class,
        maxAttempts = 3
    )

This means:

    Attempt 1
    Attempt 2
    Attempt 3

The initial invocation counts as one attempt.

Therefore:

    maxAttempts = 3

means:

    1 initial attempt + 2 retries

---

# 9. retryFor

`retryFor` defines which exceptions should trigger a retry.

Example:

    @Retryable(
        retryFor = IOException.class,
        maxAttempts = 3
    )

Only an `IOException` will trigger the configured retry behavior.

This is important because you normally don't want to retry every possible exception.

---

# 10. Example with Different Exceptions

Consider:

    @Retryable(
        retryFor = ExternalServiceException.class,
        maxAttempts = 3
    )
    public User getUser() {

        return externalService.getUser();
    }

If the method throws:

    ExternalServiceException

Spring Retry retries the operation.

But if it throws:

    IllegalArgumentException

the retry mechanism does not necessarily retry it because it was not configured as a retryable exception.

---

# 11. Retry and Backoff

Retry determines **how many times** an operation should be attempted.

Backoff determines **how long the application should wait between attempts**.

Without backoff:

    Attempt 1
       |
       v
    Attempt 2
       |
       v
    Attempt 3

The application may retry immediately.

With backoff:

    Attempt 1
       |
       v
    Wait
       |
       v
    Attempt 2
       |
       v
    Wait
       |
       v
    Attempt 3

Backoff is important because immediately retrying a failing dependency can make the problem worse.

---

# 12. @Backoff

The `@Backoff` annotation can be used together with `@Retryable`.

Example:

    @Retryable(
        retryFor = ExternalServiceException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000)
    )
    public void callExternalService() {

        externalService.call();
    }

This means:

    Attempt 1
       |
    Wait 1000 ms
       |
    Attempt 2
       |
    Wait 1000 ms
       |
    Attempt 3

The delay is specified in milliseconds.

---

# 13. Fixed Backoff

A fixed backoff uses the same delay between attempts.

Example:

    @Backoff(delay = 2000)

The sequence is approximately:

    Attempt 1
       |
    2 seconds
       |
    Attempt 2
       |
    2 seconds
       |
    Attempt 3

This is called **fixed backoff**.

---

# 14. Exponential Backoff

With exponential backoff, the delay increases after each failed attempt.

Example:

    @Retryable(
        retryFor = ExternalServiceException.class,
        maxAttempts = 4,
        backoff = @Backoff(
            delay = 1000,
            multiplier = 2
        )
    )

The delays can grow approximately like:

    Attempt 1
       |
    1 second
       |
    Attempt 2
       |
    2 seconds
       |
    Attempt 3
       |
    4 seconds
       |
    Attempt 4

Exponential backoff is often useful when the dependency may need time to recover.

---

# 15. Maximum Backoff Delay

Exponential backoff can also be limited.

Example:

    @Retryable(
        retryFor = ExternalServiceException.class,
        maxAttempts = 5,
        backoff = @Backoff(
            delay = 1000,
            multiplier = 2,
            maxDelay = 5000
        )
    )

The delay increases but does not exceed the configured maximum.

Conceptually:

    1s
    2s
    4s
    5s
    5s

This prevents retry delays from becoming excessively large.

---

# 16. Randomized Backoff

Randomization can help prevent many application instances from retrying at exactly the same time.

Imagine 100 instances fail at the same moment:

    Instance 1 -> retry at 10:00:01
    Instance 2 -> retry at 10:00:01
    Instance 3 -> retry at 10:00:01
    ...
    Instance 100 -> retry at 10:00:01

This can create another spike of traffic.

Randomized backoff introduces variation:

    Instance 1 -> retry at 10:00:01.2
    Instance 2 -> retry at 10:00:01.8
    Instance 3 -> retry at 10:00:02.1
    Instance 4 -> retry at 10:00:02.7

This concept is commonly called **jitter**.

---

# 17. @Recover

`@Recover` defines a recovery method that can be executed after all retry attempts have failed.

Example:

    @Retryable(
        retryFor = PaymentServiceException.class,
        maxAttempts = 3
    )
    public PaymentResult processPayment() {

        return paymentProvider.process();
    }

    @Recover
    public PaymentResult recover(
            PaymentServiceException exception) {

        System.out.println("Payment failed after retries");

        return PaymentResult.failed();
    }

The execution flow becomes:

    Attempt 1
       |
       X
       |
    Attempt 2
       |
       X
       |
    Attempt 3
       |
       X
       |
    @Recover
       |
       v
    Fallback Result

---

# 18. @Recover Method Signature

The recovery method should be compatible with the retryable method.

Example:

    @Retryable(
        retryFor = ExternalServiceException.class,
        maxAttempts = 3
    )
    public String getData() {

        return externalService.getData();
    }

    @Recover
    public String recover(
            ExternalServiceException exception) {

        return "Fallback data";
    }

The recovery method:

- Uses `@Recover`
- Returns the same compatible return type
- Receives the exception
- Can optionally receive the original method arguments

---

# 19. @Recover with Original Arguments

Suppose the retryable method receives an ID:

    @Retryable(
        retryFor = ExternalServiceException.class,
        maxAttempts = 3
    )
    public User getUser(String userId) {

        return externalService.getUser(userId);
    }

The recovery method can receive the exception and the original argument:

    @Recover
    public User recover(
            ExternalServiceException exception,
            String userId) {

        System.out.println(
            "Unable to retrieve user: " + userId
        );

        return User.fallback(userId);
    }

This is useful when the fallback behavior depends on the original request.

---

# 20. Retry + Backoff + Recover

A real-world example can combine all three mechanisms:

    @Retryable(
        retryFor = ExternalServiceException.class,
        maxAttempts = 4,
        backoff = @Backoff(
            delay = 1000,
            multiplier = 2,
            maxDelay = 5000
        )
    )
    public User getUser(String userId) {

        return identityService.getUser(userId);
    }

    @Recover
    public User recover(
            ExternalServiceException exception,
            String userId) {

        return User.unavailable(userId);
    }

Conceptually:

    Request
       |
       v
    getUser()
       |
       X
    1 second
       |
       v
    Retry
       |
       X
    2 seconds
       |
       v
    Retry
       |
       X
    4 seconds
       |
       v
    Retry
       |
       X
       |
       v
    @Recover
       |
       v
    Fallback

---

# 21. Internal Working of Spring Retry

The most important concept is that Spring Retry does not simply modify the method itself.

Spring uses **proxies and interceptors** to intercept calls to retryable methods.

Conceptually:

    Client
      |
      v
    Spring Proxy
      |
      v
    Retry Interceptor
      |
      v
    Target Method
      |
      +---- Success
      |
      +---- Exception
                |
                v
          Retry Policy
                |
          +-----+-----+
          |           |
        Retry       Stop
          |           |
          v           v
       Backoff     @Recover /
          |         Exception
          v
       Target
       Method

---

# 22. Spring AOP Proxy

Consider:

    @Service
    public class OrderService {

        @Retryable(
            retryFor = OrderServiceException.class,
            maxAttempts = 3
        )
        public void createOrder() {

            // business logic
        }
    }

When Spring creates the bean, the object exposed to other Spring beans can be a proxy.

Conceptually:

    OrderService Proxy
           |
           v
    Retry Interceptor
           |
           v
    Real OrderService
           |
           v
    createOrder()

The proxy is responsible for executing the retry behavior.

---

# 23. What Happens When a Method Is Called?

Suppose another service executes:

    orderService.createOrder();

The call goes approximately through:

    1. Caller invokes createOrder()
                 |
                 v
    2. Spring Proxy intercepts call
                 |
                 v
    3. RetryInterceptor starts
                 |
                 v
    4. Target method executes
                 |
                 v
    5. Method throws exception
                 |
                 v
    6. Retry policy evaluates exception
                 |
                 v
    7. Backoff is applied
                 |
                 v
    8. Method executes again
                 |
                 v
    9. Eventually success or final failure
                 |
                 v
    10. @Recover or exception

---

# 24. Retry Policy

The retry policy determines whether another attempt should be performed.

Conceptually:

    Exception
       |
       v
    Retry Policy
       |
       +---- Retry available?
       |         |
       |        YES
       |         |
       |         v
       |      Retry
       |
       +---- NO
                 |
                 v
              Recover /
              Exception

The policy can consider things such as:

- Maximum attempts
- Exception type
- Retry context
- Backoff configuration

---

# 25. RetryContext

Spring Retry maintains information about the current retry operation using a retry context.

Conceptually:

    RetryContext
        |
        +-- Retry count
        |
        +-- Last exception
        |
        +-- Retry state
        |
        +-- Other retry metadata

For example:

    RetryContext

    retryCount = 2
    lastException = ExternalServiceException
    exhausted = false

This allows the retry infrastructure to keep track of the current operation.

---

# 26. Important Proxy Limitation

Because retry behavior is implemented through a Spring proxy, **self-invocation** can be a problem.

Example:

    @Service
    public class OrderService {

        public void process() {

            createOrder();
        }

        @Retryable(
            retryFor = OrderServiceException.class,
            maxAttempts = 3
        )
        public void createOrder() {

            // retryable operation
        }
    }

Calling:

    process()

causes:

    process()
       |
       v
    this.createOrder()

The internal call does not necessarily pass through the Spring proxy.

Therefore, the `@Retryable` interception may not happen.

---

# 27. Better Design for Retry

Separate the retryable operation into another Spring bean.

Example:

    @Service
    public class OrderService {

        private final OrderClient orderClient;

        public void process() {

            orderClient.createOrder();
        }
    }

And:

    @Service
    public class OrderClient {

        @Retryable(
            retryFor = OrderServiceException.class,
            maxAttempts = 3
        )
        public void createOrder() {

            // retryable operation
        }
    }

Now the call goes through the Spring proxy:

    OrderService
         |
         v
    OrderClient Proxy
         |
         v
    Retry Interceptor
         |
         v
    createOrder()

This makes the retry behavior explicit and easier to reason about.

---

# 28. Retry and Idempotency

Retry introduces an important distributed-systems concern:

**Idempotency.**

Suppose:

    POST /payments

The first request reaches the payment provider successfully, but the response is lost because of a network failure.

Your application sees:

    Timeout

It retries.

The payment provider may receive:

    Payment #123
    Payment #123

This could result in duplicate processing.

Therefore, retrying operations that change state should consider idempotency.

Common solutions include:

- Idempotency keys
- Unique request IDs
- Database constraints
- Deduplication
- Transactional processing
- Provider-supported idempotency mechanisms

---

# 29. Retry vs Circuit Breaker

Retry and Circuit Breaker solve different problems.

### Retry

Answers:

    "Should I try this operation again?"

### Circuit Breaker

Answers:

    "Should I stop calling this dependency because it is currently failing?"

A common architecture combines them:

    Application
        |
        v
    Circuit Breaker
        |
        v
      Retry
        |
        v
    External Service

However, the exact ordering and configuration should be chosen carefully because aggressive retry combined with a circuit breaker can create unexpected traffic patterns.

---

# 30. Retry Best Practices

### 1. Retry only transient failures

Good candidates:

    Timeout
    Temporary network failure
    HTTP 503
    Temporary infrastructure failure

Avoid retrying:

    Invalid input
    HTTP 400
    Authentication failure
    Authorization failure
    Business validation errors

### 2. Use backoff

Avoid:

    Retry -> Retry -> Retry -> Retry

Prefer:

    Retry
      |
    Wait
      |
    Retry
      |
    Wait
      |
    Retry

### 3. Consider exponential backoff

For distributed systems, increasing the delay can reduce pressure on an unhealthy dependency.

### 4. Limit the number of attempts

Avoid infinite retries.

### 5. Consider jitter

Jitter can reduce synchronized retries when many application instances fail simultaneously.

### 6. Think about idempotency

Retries can duplicate operations.

### 7. Monitor retries

Track metrics such as:

    retry_attempts
    retry_success
    retry_exhausted
    recovery_count

### 8. Keep retry scope small

Retry only the operation that is actually expected to fail transiently.

---

# 31. Real-World Example

Imagine an Identity Service calling an external Identity Verification provider.

    User
      |
      v
    Identity API
      |
      v
    Identity Verification Client
      |
      v
    External IDV Provider

The provider temporarily returns:

    HTTP 503

The application can use:

    @Retryable(
        retryFor = IdvServiceException.class,
        maxAttempts = 3,
        backoff = @Backoff(
            delay = 1000,
            multiplier = 2
        )
    )

The flow becomes:

    Request
       |
       v
    IDV Provider
       |
       X 503
       |
    Wait 1s
       |
       v
    IDV Provider
       |
       X 503
       |
    Wait 2s
       |
       v
    IDV Provider
       |
       v
      200
       |
       v
    Return result

If the provider continues failing:

    Attempt 1
       |
       X
       |
    Attempt 2
       |
       X
       |
    Attempt 3
       |
       X
       |
       v
    @Recover
       |
       v
    Fallback / Error Response

---

# 32. Interview Perspective

When discussing Spring Retry in an interview, be prepared to explain:

### What is Retry?

A resilience mechanism that automatically repeats an operation after transient failures.

### Why use Retry?

To tolerate temporary failures in external dependencies and distributed systems.

### What does @Retryable do?

It tells Spring to intercept a method and retry it when configured exceptions occur.

### What does @Backoff do?

It controls the delay between retry attempts.

### What is exponential backoff?

A strategy where the delay increases between attempts.

### What is @Recover?

A fallback method invoked after retry attempts are exhausted.

### How does Spring Retry work internally?

Spring uses AOP proxies and retry interceptors to intercept calls to retryable methods, execute the target method, evaluate exceptions against a retry policy, apply backoff, and either retry, recover, or propagate the exception.

### What is the self-invocation problem?

An internal call such as `this.retryableMethod()` may bypass the Spring proxy, preventing the retry interceptor from being invoked.

### What is the biggest concern when retrying write operations?

Idempotency. A retry can cause the same operation to be processed more than once.

---

# 33. Summary

Spring Retry provides a declarative way to implement retry behavior in Spring applications.

The main concepts are:

    @EnableRetry
        |
        v
    Enables retry infrastructure

    @Retryable
        |
        v
    Defines retry behavior

    @Backoff
        |
        v
    Defines delay between attempts

    Retry Policy
        |
        v
    Decides whether another attempt is allowed

    @Recover
        |
        v
    Handles final failure

The overall architecture is:

    Client
       |
       v
    Spring Proxy
       |
       v
    Retry Interceptor
       |
       v
    Target Method
       |
       +---- Success ------> Return
       |
       +---- Failure
                |
                v
          Retry Policy
                |
          +-----+-----+
          |           |
        Retry        Stop
          |           |
          v           v
       Backoff     @Recover
          |
          v
       Retry

The key idea is:

**Retry handles temporary failures, Backoff controls when to try again, and Recover defines what happens when all attempts are exhausted.**
