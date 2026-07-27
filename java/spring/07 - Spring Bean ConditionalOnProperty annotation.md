# Spring Bean `@ConditionalOnProperty` Annotation

## Pre-requisites

Before using `@ConditionalOnProperty`, you should be familiar with:

- Spring Boot Auto Configuration
- Spring Beans
- Dependency Injection (DI)
- `application.properties` or `application.yml`
- `@Configuration` and `@Bean` annotations (optional but recommended)

---

## Why `@ConditionalOnProperty`?

By default, Spring creates every Bean found during component scanning or declared in configuration classes.

However, in real-world applications, some Beans should only be created when a specific feature or configuration is enabled.

For example:

- Enable or disable a feature (Feature Flag)
- Enable integrations with external services
- Create different implementations depending on configuration
- Enable caching only in production
- Enable scheduled jobs only when required

Without `@ConditionalOnProperty`, developers often need to write conditional logic inside configuration classes, making the code harder to maintain.

`@ConditionalOnProperty` allows Spring Boot to decide whether a Bean should exist based on configuration properties.

---

## What is `@ConditionalOnProperty`?

`@ConditionalOnProperty` is a Spring Boot annotation that conditionally registers a Bean in the Spring Application Context based on the value of one or more configuration properties.

If the configured condition is satisfied, the Bean is created.

Otherwise, the Bean is ignored during application startup.

It belongs to the Spring Boot package:

```java
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
```

### Main attributes

| Attribute        | Description                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------ |
| `name`           | Property name to evaluate.                                                                       |
| `havingValue`    | Expected value of the property.                                                                  |
| `matchIfMissing` | Defines whether the Bean should be created when the property does not exist. Default is `false`. |
| `prefix`         | Optional prefix used to avoid repeating property names.                                          |

---

## Use cases?

### Feature Flags

Enable or disable new features without changing the code.

```properties
feature.checkout.enabled=true
```

---

### External Integrations

Enable integrations only when configured.

Example:

- Stripe
- PayPal
- Kafka
- RabbitMQ

---

### Environment-specific Components

Create Beans only for certain configurations instead of creating them in every environment.

---

### Optional Modules

Applications with optional modules can create Beans only when that module is enabled.

---

### Multiple Implementations

Choose one implementation based on configuration.

Example:

```text
notification.provider=email
```

or

```text
notification.provider=sms
```

---

## Implementation?

### Step 1 — Configure a property

```properties
notification.email.enabled=true
```

---

### Step 2 — Create the Bean

```java
@Service
@ConditionalOnProperty(
    name = "notification.email.enabled",
    havingValue = "true"
)
public class EmailNotificationService implements NotificationService {

}
```

If the property is:

```properties
notification.email.enabled=true
```

Spring creates the Bean.

If:

```properties
notification.email.enabled=false
```

Spring ignores the Bean.

---

### Using `matchIfMissing`

```java
@Service
@ConditionalOnProperty(
    name = "notification.email.enabled",
    havingValue = "true",
    matchIfMissing = true
)
public class EmailNotificationService {

}
```

If the property is missing completely, Spring will still create the Bean.

---

### Using `prefix`

Instead of writing:

```java
@ConditionalOnProperty(
    name = "notification.email.enabled"
)
```

You can write:

```java
@ConditionalOnProperty(
    prefix = "notification.email",
    name = "enabled",
    havingValue = "true"
)
```

Which checks:

```properties
notification.email.enabled=true
```

---

## Advantages x Disadvantages?

### Advantages

- Clean and declarative configuration.
- No `if/else` statements inside configuration classes.
- Easy to implement Feature Flags.
- Beans are only created when needed.
- Improves application modularity.
- Makes applications easier to configure across different environments.
- Integrates seamlessly with Spring Boot Auto Configuration.

### Disadvantages

- Conditions are evaluated only during application startup.
- Changing a property at runtime does not automatically create or destroy Beans.
- Incorrect property names may silently prevent Bean creation.
- Excessive use can make application configuration harder to understand.
- Should not replace `@Profile` when the goal is environment separation.

---

## Example

```properties
feature.cache.enabled=true
```

```java
@Service
@ConditionalOnProperty(
    name = "feature.cache.enabled",
    havingValue = "true"
)
public class CacheService {

}
```

Application behavior:

| Property Value                  | Bean Created             |
| ------------------------------- | ------------------------ |
| `true`                          | ✅ Yes                   |
| `false`                         | ❌ No                    |
| Missing                         | ❌ No (default behavior) |
| Missing + `matchIfMissing=true` | ✅ Yes                   |

---

## Summary

`@ConditionalOnProperty` is a Spring Boot annotation used to conditionally create Beans based on configuration properties.

It is commonly used for feature flags, optional modules, external integrations, and configurable application behavior, allowing developers to enable or disable functionality without modifying the application code.
