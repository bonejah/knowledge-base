# Spring Bean Conditional Annotations

## Pre-requisites

Before using Spring Boot Conditional Annotations, you should be familiar with:

- Spring Boot Auto Configuration
- Spring Beans
- Dependency Injection (DI)
- `application.properties` or `application.yml`
- `@Configuration` and `@Bean` annotations
- Spring Application Context

---

## Why Conditional Bean Creation?

By default, Spring creates every Bean found during component scanning or declared in configuration classes.

However, in real-world applications, some Beans should only be created when a specific condition is satisfied.

Examples:

- Enable or disable a feature
- Enable integrations with external services
- Create a Bean only when another Bean does not exist
- Create a Bean only when another Bean already exists
- Create functionality only when a specific library is available
- Create different implementations depending on configuration
- Implement Spring Boot Auto Configuration

Spring Boot provides several conditional annotations to solve these problems.

The most important ones are:

- `@ConditionalOnProperty`
- `@ConditionalOnMissingBean`
- `@ConditionalOnBean`
- `@ConditionalOnClass`
- Custom Conditions

---

## 1. `@ConditionalOnProperty`

`@ConditionalOnProperty` creates a Bean based on the value of a configuration property.

Example:

```java
@Service
@ConditionalOnProperty(
    name = "feature.cache.enabled",
    havingValue = "true"
)
public class CacheService {

}
```

Configuration:

```properties
feature.cache.enabled=true
```

When the property is `true`, the Bean is created.

When the property is `false`, the Bean is not created.

### Main attributes

| Attribute        | Description                                                     |
| ---------------- | --------------------------------------------------------------- |
| `name`           | Property name to evaluate                                       |
| `havingValue`    | Expected property value                                         |
| `matchIfMissing` | Whether the condition should match when the property is missing |
| `prefix`         | Prefix used to avoid repeating property names                   |

---

## 2. `@ConditionalOnMissingBean`

`@ConditionalOnMissingBean` creates a Bean only when another matching Bean does not already exist.

This annotation is extremely important in Spring Boot Auto Configuration.

For example, imagine that Spring Boot provides a default implementation:

```java
@Configuration
public class PaymentAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public PaymentService paymentService() {
        return new DefaultPaymentService();
    }
}
```

If the application does not define a `PaymentService`, Spring Boot creates the default implementation.

But if the application provides its own:

```java
@Bean
public PaymentService paymentService() {
    return new CustomPaymentService();
}
```

Then the auto-configured Bean is not created.

### Why is this useful?

It allows Spring Boot to provide sensible defaults while still allowing developers to override them.

This is one of the fundamental mechanisms behind Spring Boot Auto Configuration.

### Concept

```text
Application provides Bean?
        |
        +-- YES --> Use application's Bean
        |
        +-- NO --> Create default Bean
```

### Example

```java
@Bean
@ConditionalOnMissingBean
public ObjectMapper objectMapper() {
    return new ObjectMapper();
}
```

The default Bean is created only when an appropriate `ObjectMapper` Bean is not already available.

---

## 3. `@ConditionalOnBean`

`@ConditionalOnBean` does the opposite.

It creates a Bean only when another matching Bean already exists.

Example:

```java
@Configuration
public class MetricsConfiguration {

    @Bean
    @ConditionalOnBean(MeterRegistry.class)
    public MetricsService metricsService() {
        return new MetricsService();
    }
}
```

The `MetricsService` will only be created if a `MeterRegistry` Bean exists in the Application Context.

### Concept

```text
Required Bean exists?
        |
        +-- YES --> Create dependent Bean
        |
        +-- NO --> Do not create Bean
```

### When is it useful?

It is useful when one component depends on the existence of another component at configuration level.

Examples:

- Create monitoring functionality only when a metrics system exists
- Create Kafka-related functionality only when Kafka infrastructure is configured
- Create integration components only when another infrastructure Bean exists

---

## `@ConditionalOnBean` vs `@ConditionalOnMissingBean`

These annotations are opposites in terms of their basic condition.

| Annotation                  | Bean creation condition                     |
| --------------------------- | ------------------------------------------- |
| `@ConditionalOnBean`        | Create if the specified Bean exists         |
| `@ConditionalOnMissingBean` | Create if the specified Bean does not exist |

Example:

```java
@ConditionalOnBean(DataSource.class)
```

Means:

```text
DataSource exists
        ↓
Create this Bean
```

While:

```java
@ConditionalOnMissingBean(DataSource.class)
```

Means:

```text
DataSource does NOT exist
        ↓
Create this Bean
```

---

## 4. `@ConditionalOnClass`

`@ConditionalOnClass` creates a Bean only when a specific Java class is available on the application's classpath.

Example:

```java
@Configuration
@ConditionalOnClass(KafkaTemplate.class)
public class KafkaConfiguration {

}
```

Spring checks whether:

```java
KafkaTemplate.class
```

is available.

If the Kafka dependency is present, the condition matches.

If the Kafka dependency is not present, the configuration is not applied.

### Why is this useful?

This is particularly important for Spring Boot Auto Configuration.

Imagine a library that supports several optional technologies:

```text
Application
   |
   +-- Kafka dependency
   |       ↓
   |   Kafka configuration enabled
   |
   +-- Redis dependency
   |       ↓
   |   Redis configuration enabled
   |
   +-- Neither
           ↓
      Neither configuration enabled
```

This allows Spring Boot to automatically configure features based on which libraries are available.

### Example

```java
@Configuration
@ConditionalOnClass(RedisTemplate.class)
public class RedisConfiguration {

    @Bean
    public CacheService cacheService() {
        return new RedisCacheService();
    }
}
```

If `RedisTemplate` is available on the classpath, the configuration can be activated.

---

## `@ConditionalOnClass` vs `@ConditionalOnBean`

These annotations operate at different levels.

| Annotation            | Checks                                           |
| --------------------- | ------------------------------------------------ |
| `@ConditionalOnClass` | Whether a class exists on the classpath          |
| `@ConditionalOnBean`  | Whether a Bean exists in the Application Context |

Example:

```java
@ConditionalOnClass(RedisTemplate.class)
```

Means:

> Is the Redis class available?

While:

```java
@ConditionalOnBean(RedisTemplate.class)
```

Means:

> Has Spring already created a RedisTemplate Bean?

This distinction is important.

A class can exist on the classpath without a corresponding Spring Bean existing in the Application Context.

---

## 5. Create Custom Conditions

Spring Boot provides many predefined conditions, but sometimes the application requires a condition that does not exist out of the box.

In that case, we can create a custom condition using:

```java
Condition
```

and:

```java
@Conditional
```

---

### Step 1 — Create a Custom Condition

```java
public class CustomFeatureCondition
        implements Condition {

    @Override
    public boolean matches(
            ConditionContext context,
            AnnotatedTypeMetadata metadata) {

        String value = context
                .getEnvironment()
                .getProperty("feature.custom.enabled");

        return "true".equalsIgnoreCase(value);
    }
}
```

The `matches()` method determines whether the condition should match.

If it returns:

```text
true
```

Spring allows the Bean or configuration to be created.

If it returns:

```text
false
```

Spring does not create it.

---

### Step 2 — Use `@Conditional`

```java
@Configuration
@Conditional(CustomFeatureCondition.class)
public class CustomFeatureConfiguration {

    @Bean
    public CustomFeatureService customFeatureService() {
        return new CustomFeatureService();
    }
}
```

Configuration:

```properties
feature.custom.enabled=true
```

When the property is `true`, the configuration is activated.

---

## How Custom Conditions Work

The basic flow is:

```text
Spring starts
     ↓
Reads configuration
     ↓
Evaluates Condition
     ↓
matches() called
     ↓
     ┌───────────────┐
     │               │
   true            false
     │               │
     ↓               ↓
Create Bean     Skip Bean
```

This provides much more flexibility than predefined annotations.

---

## When Should You Create a Custom Condition?

Custom conditions are useful when the condition depends on logic that cannot be expressed easily with existing annotations.

Examples:

- Multiple configuration properties
- Complex environment rules
- Custom system properties
- Specific runtime configuration
- Combination of several conditions
- Company-specific infrastructure rules
- Custom Spring Boot starters

However, custom conditions should not be created unnecessarily.

If an existing annotation solves the problem, prefer the existing annotation.

For example, do not create a custom `Condition` just to check:

```properties
feature.enabled=true
```

Use:

```java
@ConditionalOnProperty(
    name = "feature.enabled",
    havingValue = "true"
)
```

instead.

---

## Combining Conditions

Spring Boot allows multiple conditions to be combined.

For example:

```java
@Configuration
@ConditionalOnClass(KafkaTemplate.class)
@ConditionalOnProperty(
    name = "kafka.enabled",
    havingValue = "true"
)
public class KafkaConfiguration {

}
```

The configuration is activated only when both conditions are satisfied.

Conceptually:

```text
Kafka class exists
        AND
kafka.enabled=true
        ↓
Create configuration
```

This is useful for optional integrations.

---

## Real-World Example — Optional Kafka Integration

Imagine an application that supports Kafka but does not require Kafka.

Configuration:

```properties
kafka.enabled=true
```

Auto Configuration:

```java
@Configuration
@ConditionalOnClass(KafkaTemplate.class)
@ConditionalOnProperty(
    name = "kafka.enabled",
    havingValue = "true"
)
public class KafkaAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public KafkaService kafkaService() {
        return new DefaultKafkaService();
    }
}
```

Several conditions are working together.

### Condition 1

```java
@ConditionalOnClass(KafkaTemplate.class)
```

Checks whether Kafka is available.

### Condition 2

```java
@ConditionalOnProperty(
    name = "kafka.enabled",
    havingValue = "true"
)
```

Checks whether Kafka functionality is enabled.

### Condition 3

```java
@ConditionalOnMissingBean
```

Checks whether the application already provides its own `KafkaService`.

The resulting logic is:

```text
Kafka library available?
        |
        +-- NO --> Do nothing
        |
       YES
        ↓
kafka.enabled=true?
        |
        +-- NO --> Do nothing
        |
       YES
        ↓
KafkaService already exists?
        |
        +-- YES --> Use application's Bean
        |
        +-- NO --> Create default KafkaService
```

This is very similar to the design used by Spring Boot Auto Configuration.

---

## Conditional Annotations — Quick Comparison

| Annotation                  | Condition                            |
| --------------------------- | ------------------------------------ |
| `@ConditionalOnProperty`    | Property has the expected value      |
| `@ConditionalOnMissingBean` | Matching Bean does not exist         |
| `@ConditionalOnBean`        | Matching Bean exists                 |
| `@ConditionalOnClass`       | Class exists on the classpath        |
| `@Conditional`              | Custom condition evaluates to `true` |

---

## Common Interview Question

What is the difference between `@ConditionalOnClass`, `@ConditionalOnBean`, and `@ConditionalOnProperty`?

A good answer:

> `@ConditionalOnClass` checks whether a class is available on the classpath. `@ConditionalOnBean` checks whether a Spring Bean already exists in the Application Context. `@ConditionalOnProperty` checks the value of a configuration property. These conditions are commonly used by Spring Boot Auto Configuration to decide which configuration and Beans should be created.

---

## Common Interview Question

Why is `@ConditionalOnMissingBean` important in Spring Boot?

A good answer:

> It allows Spring Boot to provide default Beans while still allowing applications to override those defaults. If the application defines its own Bean of the required type, the auto-configured Bean is not created.

---

## Common Interview Question

What happens when `@ConditionalOnProperty` changes at runtime?

By default, the condition is evaluated during application context initialization.

Changing the property afterward does not automatically create or destroy the Bean.

For dynamic runtime configuration, additional mechanisms are required.

---

## Advantages

- Declarative configuration.
- Reduces `if/else` configuration logic.
- Enables modular application design.
- Supports Spring Boot Auto Configuration.
- Allows sensible default Beans.
- Allows application-specific Bean overrides.
- Supports optional dependencies.
- Enables feature flags.
- Makes integrations easier to activate or deactivate.
- Supports custom configuration rules.

---

## Disadvantages

- Conditions can make application startup behavior harder to understand.
- Incorrect property names can prevent Bean creation.
- Conditions are normally evaluated during application startup.
- Excessive use can make configuration complex.
- Custom Conditions can become difficult to maintain.
- Debugging conditional configuration can require understanding Spring Boot's condition evaluation.

---

## Important Concept — Condition Evaluation

When Spring Boot starts, it evaluates conditions while building the Application Context.

For example:

```text
Spring Boot starts
        ↓
Auto Configuration discovered
        ↓
Conditions evaluated
        ↓
Configuration accepted or rejected
        ↓
Beans registered
        ↓
Application Context created
```

This means conditional annotations are fundamentally part of the Bean registration and configuration phase, not normal business logic execution.

---

## Debugging Conditional Configuration

When a conditional Bean is unexpectedly missing, Spring Boot can provide information about why a condition matched or did not match.

One useful option is enabling the condition evaluation report with debug logging:

```properties
debug=true
```

This can help identify:

- Which condition matched
- Which condition did not match
- Why an Auto Configuration was applied
- Why an Auto Configuration was skipped
- Which Bean caused `@ConditionalOnMissingBean` not to match

This is particularly useful when troubleshooting Spring Boot Auto Configuration.

---

## Summary

Spring Boot Conditional Annotations allow the Application Context to be built dynamically based on the application's environment, configuration, classpath, and existing Beans.

The most important annotations are:

```text
@ConditionalOnProperty
        ↓
Configuration property

@ConditionalOnMissingBean
        ↓
Bean does NOT exist

@ConditionalOnBean
        ↓
Bean exists

@ConditionalOnClass
        ↓
Class exists on classpath

@Conditional
        ↓
Custom condition
```

A useful way to remember them is:

```text
PROPERTY
    ↓
@ConditionalOnProperty

BEAN EXISTS
    ↓
@ConditionalOnBean

BEAN DOES NOT EXIST
    ↓
@ConditionalOnMissingBean

CLASS EXISTS
    ↓
@ConditionalOnClass

CUSTOM LOGIC
    ↓
@Conditional
```

Together, these mechanisms form an important part of Spring Boot's Auto Configuration model and allow applications and libraries to create Beans only when the required conditions are satisfied.
