# 34 - Spring ConfigurationProperties

## 1. The Problem with Scattered Configuration

In a Spring Boot application, configuration values often start simple:

    server.port=8080
    app.name=my-app
    app.timeout=5000

As the application grows, configuration can become scattered across:

- `application.properties`
- `application.yml`
- `@Value`
- Environment Variables
- System Properties
- Kubernetes ConfigMaps
- Secrets
- Multiple configuration classes

A common approach is to inject individual properties using `@Value`:

    @Value("${app.name}")
    private String appName;

    @Value("${app.timeout}")
    private int timeout;

This works, but it becomes difficult to maintain when a feature has many related configuration properties.

For example:

    @Value("${payment.url}")
    private String paymentUrl;

    @Value("${payment.timeout}")
    private int paymentTimeout;

    @Value("${payment.retry.max-attempts}")
    private int maxAttempts;

    @Value("${payment.retry.backoff}")
    private long backoff;

The configuration becomes tightly coupled to individual fields throughout the application.

### Problems with Scattered Configuration

#### 1. Poor organization

Related properties can be spread across multiple classes.

#### 2. Difficult refactoring

Renaming a property may require searching through the entire codebase.

#### 3. Weak type safety

`@Value` performs individual property injection and can make complex configuration harder to model.

#### 4. Difficult validation

Validation becomes harder when configuration is injected into many unrelated classes.

#### 5. Poor discoverability

Developers have to search the codebase to understand which configuration properties exist.

#### 6. Configuration logic leaks into business classes

Business components may become responsible for knowing configuration property names.

---

# 2. What @ConfigurationProperties Actually Does

`@ConfigurationProperties` provides a way to bind a group of external configuration properties to a strongly typed Java object.

Instead of injecting individual properties:

    @Value("${payment.url}")
    private String url;

    @Value("${payment.timeout}")
    private int timeout;

    @Value("${payment.retry.max-attempts}")
    private int maxAttempts;

We can create a dedicated configuration object:

    @ConfigurationProperties(prefix = "payment")
    public class PaymentProperties {

        private String url;
        private int timeout;
        private Retry retry;

        // getters and setters
    }

Then configuration can be represented as:

    payment:
      url: https://payment.example.com
      timeout: 5000
      retry:
        max-attempts: 3

Spring Boot maps the external configuration into the Java object.

Conceptually:

    application.yml
          |
          v
    ConfigurationProperties Binder
          |
          v
    PaymentProperties
          |
          v
    Application Components

The important idea is:

> `@ConfigurationProperties` transforms external configuration into a typed configuration model.

---

# 3. A Simple Step-by-Step Example

## Step 1 - Define the configuration

Example `application.yml`:

    app:
      name: my-application
      timeout: 5000
      enabled: true

---

## Step 2 - Create the Properties class

    @ConfigurationProperties(prefix = "app")
    public class AppProperties {

        private String name;
        private Duration timeout;
        private boolean enabled;

        // getters and setters
    }

The prefix is:

    app

Therefore:

    app.name

maps to:

    AppProperties.name

And:

    app.timeout

maps to:

    AppProperties.timeout

---

# 4. Registering @ConfigurationProperties

There are several ways to register a `@ConfigurationProperties` class.

## Option 1 - @ConfigurationPropertiesScan

A modern approach is:

    @SpringBootApplication
    @ConfigurationPropertiesScan
    public class Application {

        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }

Spring Boot scans the application for classes annotated with:

    @ConfigurationProperties

and registers them as beans.

---

## Option 2 - @EnableConfigurationProperties

You can explicitly register the class:

    @SpringBootApplication
    @EnableConfigurationProperties(AppProperties.class)
    public class Application {
    }

This approach is useful when you want explicit registration.

---

# 5. Using the Configuration Bean

Once registered, the configuration object can be injected like any other Spring bean.

    @Service
    public class PaymentService {

        private final AppProperties properties;

        public PaymentService(AppProperties properties) {
            this.properties = properties;
        }

        public void execute() {

            if (properties.isEnabled()) {
                // execute operation
            }
        }
    }

The service no longer needs to know the actual configuration property names.

It only depends on:

    AppProperties

This creates a cleaner separation:

    External Configuration
            |
            v
    AppProperties
            |
            v
    Application Services

---

# 6. Why This Is Better Than @Value

Compare the two approaches.

## @Value

    @Value("${payment.url}")
    private String url;

    @Value("${payment.timeout}")
    private int timeout;

    @Value("${payment.retry.max-attempts}")
    private int maxAttempts;

## @ConfigurationProperties

    @ConfigurationProperties(prefix = "payment")
    public class PaymentProperties {

        private String url;
        private Duration timeout;
        private Retry retry;
    }

The second approach creates an explicit configuration model.

Instead of:

    String
    int
    int
    boolean
    String
    ...

you have:

    PaymentProperties

This makes configuration easier to understand and test.

---

# 7. Deep Benefits No One Talks About

## 7.1 Configuration becomes a domain model

A configuration class is more than a container for variables.

It represents a configuration domain.

For example:

    PaymentProperties
    DatabaseProperties
    KafkaProperties
    SecurityProperties
    StorageProperties

Each class represents a specific area of the application.

This creates a much clearer architecture.

---

## 7.2 Business classes become configuration-agnostic

Without `@ConfigurationProperties`:

    @Value("${payment.retry.max-attempts}")
    private int maxAttempts;

The service knows:

    payment.retry.max-attempts

With `@ConfigurationProperties`:

    private final PaymentProperties properties;

The service only knows the configuration model.

This means the service doesn't care whether the value came from:

- YAML
- Environment Variables
- Config Server
- Kubernetes
- Docker
- System Properties

The configuration source is an infrastructure concern.

---

# 8. Type Safety

One of the biggest advantages is strongly typed configuration.

Instead of:

    @Value("${app.timeout}")
    private String timeout;

you can use:

    private Duration timeout;

Configuration:

    app:
      timeout: 5s

Java:

    Duration timeout;

Now the application works with an actual duration instead of manually parsing strings.

Other useful types include:

    Duration
    DataSize
    URI
    URL
    InetAddress
    Enum
    boolean
    int
    long

Example:

    @ConfigurationProperties(prefix = "server")
    public class ServerProperties {

        private Duration timeout;
        private DataSize maxRequestSize;
        private URI baseUrl;
    }

This makes configuration much more expressive.

---

# 9. Nested Configuration

Complex configuration can be represented using nested objects.

Example:

    payment:
      url: https://payment.example.com
      timeout: 5s

      retry:
        enabled: true
        max-attempts: 3
        backoff: 2s

Java:

    @ConfigurationProperties(prefix = "payment")
    public class PaymentProperties {

        private URI url;
        private Duration timeout;
        private Retry retry;

        // getters and setters

        public static class Retry {

            private boolean enabled;
            private int maxAttempts;
            private Duration backoff;

            // getters and setters
        }
    }

Spring Boot maps:

    payment.retry.enabled

to:

    retry.enabled

and:

    payment.retry.max-attempts

to:

    retry.maxAttempts

Spring Boot's relaxed binding understands common naming conventions.

For example:

    max-attempts

can map to:

    maxAttempts

---

# 10. Lists

Configuration properties can contain lists.

Example:

    application:
      allowed-origins:
        - https://example.com
        - https://admin.example.com
        - https://app.example.com

Java:

    @ConfigurationProperties(prefix = "application")
    public class ApplicationProperties {

        private List<String> allowedOrigins;

        // getters and setters
    }

The resulting object contains:

    [
        "https://example.com",
        "https://admin.example.com",
        "https://app.example.com"
    ]

This is much cleaner than trying to manually parse comma-separated strings.

---

# 11. Maps

Maps are also useful for dynamic configuration.

Example:

    application:
      endpoints:
        users: https://users.example.com
        payments: https://payments.example.com
        notifications: https://notifications.example.com

Java:

    @ConfigurationProperties(prefix = "application")
    public class ApplicationProperties {

        private Map<String, URI> endpoints;

        // getters and setters
    }

Then:

    endpoints.get("users")

returns:

    https://users.example.com

This is useful when the configuration contains dynamic keys.

---

# 12. Lists of Objects

You can also represent more complex structures.

Configuration:

    clients:
      - name: payment
        url: https://payment.example.com
        timeout: 5s

      - name: notification
        url: https://notification.example.com
        timeout: 3s

Java:

    @ConfigurationProperties(prefix = "clients")
    public class ClientProperties {

        private List<Client> clients;

        public static class Client {

            private String name;
            private URI url;
            private Duration timeout;

            // getters and setters
        }
    }

This allows configuration to behave like structured data instead of a collection of unrelated strings.

---

# 13. Constructor Binding

Modern Spring Boot applications can use constructor-based configuration binding.

Example:

    @ConfigurationProperties(prefix = "app")
    public class AppProperties {

        private final String name;
        private final Duration timeout;

        public AppProperties(
                String name,
                Duration timeout) {

            this.name = name;
            this.timeout = timeout;
        }

        public String getName() {
            return name;
        }

        public Duration getTimeout() {
            return timeout;
        }
    }

This approach has an important advantage:

The configuration object can be immutable.

That means:

    AppProperties

cannot be modified after creation.

This is often preferable for configuration because configuration should generally be treated as application state established during startup.

---

# 14. Validation

Configuration validation is one of the most valuable features of `@ConfigurationProperties`.

Imagine:

    payment:
      url: ""
      timeout: -1s
      retry:
        max-attempts: 0

The application should fail fast instead of discovering these problems during runtime.

You can use Jakarta Bean Validation.

Example:

    @ConfigurationProperties(prefix = "payment")
    @Validated
    public class PaymentProperties {

        @NotBlank
        private String url;

        @NotNull
        private Duration timeout;

        @Min(1)
        private int maxAttempts;

        // getters and setters
    }

Now invalid configuration can prevent the application from starting.

This is called:

> Fail Fast

Instead of:

    Application starts
          |
          v
    Request arrives
          |
          v
    Invalid configuration discovered
          |
          v
    Runtime failure

You get:

    Application starts
          |
          v
    Configuration validation
          |
          v
    Invalid configuration
          |
          v
    Startup fails

This is much safer for production systems.

---

# 15. Nested Validation

Nested configuration can also be validated.

Example:

    @ConfigurationProperties(prefix = "payment")
    @Validated
    public class PaymentProperties {

        @Valid
        private Retry retry;

        public static class Retry {

            @Min(1)
            private int maxAttempts;

            @NotNull
            private Duration backoff;

            // getters and setters
        }
    }

The `@Valid` annotation tells Bean Validation to validate the nested object.

Without it, validation of nested properties may not happen as expected.

---

# 16. Configuration Metadata

Spring Boot can generate metadata for configuration properties.

This improves IDE support.

For example, when developers type:

    payment.

their IDE can potentially provide autocomplete for:

    payment.url
    payment.timeout
    payment.retry.max-attempts

The `spring-boot-configuration-processor` is commonly used for this purpose.

Maven dependency:

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>

This makes custom application configuration much easier to discover.

---

# 17. Relaxed Binding

Spring Boot supports relaxed binding between configuration names and Java property names.

For example:

    payment.max-attempts

can map to:

    maxAttempts

Environment variables can use:

    PAYMENT_MAX_ATTEMPTS

This is particularly useful when deploying applications through Docker or Kubernetes.

For example:

    PAYMENT_TIMEOUT=5s

can override a configuration property such as:

    payment.timeout

This allows the same application artifact to run in different environments without changing the application code.

---

# 18. Configuration Precedence

Spring Boot can obtain configuration from multiple sources.

Examples include:

- `application.properties`
- `application.yml`
- Profile-specific configuration
- Environment Variables
- System Properties
- Command-line arguments
- External configuration

Higher-precedence sources can override lower-precedence values.

For example:

    application.yml

might contain:

    payment:
      timeout: 5s

while an environment variable provides:

    PAYMENT_TIMEOUT=10s

The environment-specific value can override the default.

This is extremely useful in deployment environments.

---

# 19. Profiles

`@ConfigurationProperties` works naturally with Spring profiles.

Example:

    application.yml

    payment:
      timeout: 5s

Production:

    application-prod.yml

    payment:
      timeout: 10s

The same Java configuration class can be used:

    @ConfigurationProperties(prefix = "payment")
    public class PaymentProperties {
        ...
    }

The configuration source changes, not the Java code.

---

# 20. Best Practices

## 20.1 Group related properties

Prefer:

    PaymentProperties

containing:

    url
    timeout
    retry
    authentication

instead of scattering them across multiple services.

---

## 20.2 Prefer @ConfigurationProperties for groups of properties

Use `@Value` for very small, isolated values when appropriate.

Use `@ConfigurationProperties` when you have:

- Multiple related properties
- Nested configuration
- Lists
- Maps
- Validation
- Complex types
- Configuration reused by multiple components

---

## 20.3 Prefer meaningful prefixes

Good:

    payment
    security
    messaging
    storage
    application

Avoid vague prefixes such as:

    config
    settings
    values

The prefix should communicate the configuration domain.

---

## 20.4 Prefer immutable configuration when practical

Constructor binding can make configuration objects immutable.

This prevents accidental modification after startup.

Conceptually:

    Configuration
          |
          v
    Immutable Properties
          |
          v
    Application

rather than allowing application components to modify configuration state.

---

## 20.5 Use Duration instead of numeric time values

Avoid:

    timeout: 5000

when the unit is unclear.

Prefer:

    timeout: 5s

and:

    private Duration timeout;

This makes the configuration self-documenting.

---

## 20.6 Use DataSize for sizes

Instead of:

    max-file-size: 10485760

prefer:

    max-file-size: 10MB

and:

    private DataSize maxFileSize;

The configuration becomes easier to understand.

---

## 20.7 Validate configuration at startup

Use validation annotations such as:

    @NotBlank
    @NotNull
    @Min
    @Max
    @Positive
    @PositiveOrZero
    @Pattern
    @Valid

Invalid configuration should ideally prevent the application from starting.

---

## 20.8 Do not expose secrets unnecessarily

Avoid putting sensitive information directly into source-controlled configuration files.

Bad:

    database:
      password: my-secret-password

Prefer external secret management mechanisms appropriate to the deployment environment.

Examples:

- Kubernetes Secrets
- Cloud secret managers
- Vault
- Environment variables
- External configuration systems

`@ConfigurationProperties` is a binding mechanism; it is not itself a secret-management solution.

---

# 21. @ConfigurationProperties vs @Value

| Feature                     | @Value          | @ConfigurationProperties |
| --------------------------- | --------------- | ------------------------ |
| Single property             | Excellent       | Possible                 |
| Multiple related properties | Less convenient | Excellent                |
| Type-safe model             | Limited         | Excellent                |
| Nested configuration        | Awkward         | Excellent                |
| Lists                       | Possible        | Excellent                |
| Maps                        | Possible        | Excellent                |
| Validation                  | Less convenient | Excellent                |
| IDE metadata                | Limited         | Excellent                |
| Reusability                 | Moderate        | Excellent                |
| Immutable configuration     | Possible        | Excellent                |
| Configuration domain model  | No              | Yes                      |

A useful rule:

    One isolated value
        -> @Value can be enough

    Configuration group
        -> @ConfigurationProperties

---

# 22. Real-World Example

Imagine a microservice that communicates with a payment provider.

Configuration:

    payment:
      provider: stripe
      base-url: https://api.payment.example.com
      timeout: 5s

      retry:
        enabled: true
        max-attempts: 3
        backoff: 2s

      security:
        api-key: ${PAYMENT_API_KEY}

Java model:

    @ConfigurationProperties(prefix = "payment")
    @Validated
    public class PaymentProperties {

        @NotBlank
        private String provider;

        @NotNull
        private URI baseUrl;

        @NotNull
        private Duration timeout;

        @Valid
        private Retry retry;

        @Valid
        private Security security;

        // ...
    }

The service can now depend on:

    PaymentProperties

instead of knowing dozens of individual property names.

Architecture:

    application.yml
          |
          v
    Spring Boot Binder
          |
          v
    PaymentProperties
          |
          +----------------+
          |                |
          v                v
    PaymentService    PaymentClient
          |                |
          +--------+-------+
                   |
                   v
             Payment API

This creates a clear boundary between configuration and application logic.

---

# 23. ConfigurationProperties in a Microservices Architecture

In a microservices environment, each service may have configuration such as:

    database
    kafka
    redis
    security
    external APIs
    feature flags
    timeouts
    retry policies

Instead of distributing these values throughout the codebase:

    @Value(...)
    @Value(...)
    @Value(...)
    @Value(...)

create configuration models:

    DatabaseProperties
    KafkaProperties
    RedisProperties
    SecurityProperties
    PaymentProperties

This creates explicit configuration boundaries.

It also makes configuration easier to:

- Test
- Validate
- Document
- Refactor
- Reuse
- Override per environment

---

# 24. Testing Configuration

Configuration classes can be tested independently.

Example test goals:

    Given valid configuration
        -> properties are correctly bound

    Given invalid timeout
        -> validation fails

    Given missing required URL
        -> application context fails

    Given environment variable
        -> environment value overrides default

This is particularly useful when configuration becomes complex.

---

# 25. Common Mistakes

## Mistake 1 - Using @Value everywhere

A large number of:

    @Value("${...}")

usually indicates that configuration should be grouped.

---

## Mistake 2 - Putting configuration inside business classes

Avoid:

    @Service
    public class PaymentService {

        @Value("${payment.timeout}")
        private Duration timeout;

        @Value("${payment.retry.max-attempts}")
        private int maxAttempts;
    }

Prefer:

    @Service
    public class PaymentService {

        private final PaymentProperties properties;

        public PaymentService(PaymentProperties properties) {
            this.properties = properties;
        }
    }

The service depends on a configuration model instead of raw configuration keys.

---

## Mistake 3 - Using unclear units

Avoid:

    timeout: 5000

Prefer:

    timeout: 5s

---

## Mistake 4 - No validation

If configuration is required for the application to work correctly, validate it.

---

## Mistake 5 - Creating one giant Properties class

Avoid:

    ApplicationProperties

containing hundreds of unrelated properties.

Prefer smaller configuration domains:

    DatabaseProperties
    PaymentProperties
    KafkaProperties
    SecurityProperties

---

# 26. Mental Model

Think about `@ConfigurationProperties` as a bridge:

    External Configuration
            |
            | Binding
            v
    Strongly Typed Java Object
            |
            | Dependency Injection
            v
    Application Components

The application should ideally interact with:

    PaymentProperties

instead of:

    "${payment.timeout}"

The property name belongs to the configuration layer.

The typed object belongs to the application layer.

---

# 27. Interview Questions

### Q1. What is @ConfigurationProperties?

It binds a group of external configuration properties to a strongly typed Java object.

---

### Q2. Why use @ConfigurationProperties instead of @Value?

Because it provides better organization, type safety, validation, nested structures, metadata, and reusability for groups of related configuration.

---

### Q3. How do you register a @ConfigurationProperties class?

Common approaches include:

    @ConfigurationPropertiesScan

or:

    @EnableConfigurationProperties

---

### Q4. Can @ConfigurationProperties contain nested objects?

Yes.

Nested objects are useful for modeling hierarchical configuration.

---

### Q5. Can it bind Lists and Maps?

Yes.

Spring Boot's configuration binder supports complex structures such as:

    List<T>

and:

    Map<K,V>

---

### Q6. Can configuration be validated?

Yes.

Use Jakarta Bean Validation annotations together with:

    @Validated

and:

    @Valid

for nested objects when needed.

---

### Q7. What is relaxed binding?

It allows configuration names using different naming conventions to map to Java property names.

For example:

    max-attempts

can map to:

    maxAttempts

---

### Q8. Can @ConfigurationProperties use Duration?

Yes.

Example:

    timeout: 5s

can bind to:

    private Duration timeout;

---

### Q9. What is the main architectural benefit?

It creates a clear boundary between external configuration and application logic.

Instead of spreading configuration keys throughout the application, configuration is modeled as a typed object.

---

# 28. Key Takeaways

`@ConfigurationProperties` is more than a convenient alternative to `@Value`.

It provides a structured configuration model for Spring Boot applications.

The key benefits are:

- Strongly typed configuration
- Centralized configuration
- Better separation of concerns
- Nested configuration
- Lists and Maps
- Validation
- IDE metadata
- Environment-specific overrides
- Better testability
- Better maintainability
- Support for immutable configuration
- Cleaner business services

The most important mental model is:

    Configuration keys
          |
          v
    Configuration Binding
          |
          v
    Typed Properties Object
          |
          v
    Application Components

For small, isolated values:

    @Value

can be sufficient.

For structured application configuration:

    @ConfigurationProperties

is usually the better abstraction.

The goal is not simply to inject configuration.

The goal is to make configuration a well-defined part of the application's architecture.
