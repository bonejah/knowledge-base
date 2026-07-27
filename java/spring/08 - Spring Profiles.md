# Spring Profiles

## What is Profiling?

Spring Profiles is a feature that allows you to define different configurations for different environments (or use cases) without changing your application code.

Instead of modifying configuration files every time you deploy your application, you can create environment-specific configurations and activate the appropriate profile.

Common environments include:

- Development (`dev`)
- Testing (`test`)
- Staging (`staging`)
- Production (`prod`)
- Local (`local`)

Each profile can have its own:

- Bean definitions
- Properties
- Database configuration
- External services
- Security settings
- Logging configuration

This helps keep the application clean, maintainable, and environment-independent.

---

## Why use Spring Profiles?

Without profiles, developers often need to:

- Comment/uncomment configuration code
- Change database URLs manually
- Modify API endpoints before deployment
- Risk deploying development configurations into production

Spring Profiles solve these problems by automatically loading the correct configuration based on the active environment.

For example:

| Environment | Database            | Logging |
| ----------- | ------------------- | ------- |
| dev         | Local PostgreSQL    | DEBUG   |
| test        | Test Database       | INFO    |
| prod        | Production Database | WARN    |

---

## How to use?

### 1. Create environment-specific configuration files

```
application.properties
application-dev.properties
application-test.properties
application-prod.properties
```

Example:

**application-dev.properties**

```properties
server.port=8080

spring.datasource.url=jdbc:postgresql://localhost/devdb

logging.level.root=DEBUG
```

**application-prod.properties**

```properties
server.port=8080

spring.datasource.url=jdbc:postgresql://prod-server/proddb

logging.level.root=WARN
```

---

### 2. Activate a profile

#### Using application.properties

```properties
spring.profiles.active=dev
```

---

#### Using environment variables

```
SPRING_PROFILES_ACTIVE=prod
```

---

#### Using command line

```bash
java -jar app.jar --spring.profiles.active=prod
```

or

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

---

### 3. Spring loads the matching configuration

If the active profile is:

```
dev
```

Spring loads:

```
application.properties
+
application-dev.properties
```

Properties in the profile-specific file override the default ones.

---

## @Profile Annotation

The `@Profile` annotation allows a Spring Bean to be created only when a specific profile is active.

### Example

```java
@Service
@Profile("dev")
public class MockEmailService implements EmailService {

    @Override
    public void send() {
        System.out.println("Sending fake email...");
    }
}
```

This bean exists only when the **dev** profile is active.

---

Another implementation:

```java
@Service
@Profile("prod")
public class SmtpEmailService implements EmailService {

    @Override
    public void send() {
        System.out.println("Sending real email...");
    }
}
```

When running with:

```properties
spring.profiles.active=prod
```

Spring creates:

```
SmtpEmailService
```

When running with:

```properties
spring.profiles.active=dev
```

Spring creates:

```
MockEmailService
```

No code changes are required.

---

## Multiple Profiles

A bean can belong to multiple profiles.

```java
@Service
@Profile({"dev", "test"})
public class MockNotificationService {
}
```

The bean will be created if either **dev** or **test** is active.

---

## Negating a Profile

You can activate a bean when a profile is **not** active.

```java
@Service
@Profile("!prod")
public class ConsoleLogger {
}
```

This bean is created for every profile except **prod**.

---

## Profile Expressions

Spring also supports profile expressions.

```java
@Profile("dev & cloud")
```

The bean is created only if **both** profiles are active.

```java
@Profile("dev | test")
```

The bean is created if **either** profile is active.

```java
@Profile("prod & !cloud")
```

The bean is created only when running in **prod** but **not** in the **cloud** profile.

---

## Typical Use Cases

Spring Profiles are commonly used for:

- Switching between development and production databases
- Using mock implementations during development
- Enabling test-only beans
- Configuring different security settings
- Connecting to different third-party services
- Enabling or disabling scheduled jobs
- Changing logging levels
- Environment-specific feature configuration

---

## Advantages

- Clean separation of environment configurations
- No code modifications between deployments
- Easy environment management
- Simplifies testing
- Supports different infrastructure configurations
- Works seamlessly with Spring Boot
- Easy integration with Docker, Kubernetes, and CI/CD pipelines

---

## Disadvantages

- Too many profiles can become difficult to manage
- Incorrect active profile may cause unexpected behavior
- Not intended for feature toggles (use feature flags or `@ConditionalOnProperty` instead)
- Profile-specific configurations may become duplicated if not organized carefully

---

## Best Practices

- Keep a default `application.properties` for common configuration.
- Use profiles only for environment-specific behavior.
- Prefer meaningful profile names (`dev`, `test`, `staging`, `prod`).
- Avoid embedding business logic in profile-specific beans.
- For enabling/disabling individual features, prefer `@ConditionalOnProperty` instead of creating many profiles.

---

## Summary

Spring Profiles provide a clean way to separate application configurations for different environments. By activating a specific profile, Spring loads the appropriate properties and creates only the beans associated with that profile through the `@Profile` annotation. This makes applications easier to configure, deploy, and maintain across development, testing, staging, and production environments.
