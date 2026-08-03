# 10 - Aspect Oriented Programming (Spring AOP)

## What is Aspect Oriented Programming (AOP)?

Aspect-Oriented Programming (AOP) is a programming paradigm that helps separate **cross-cutting concerns** from the main business logic of an application.

Cross-cutting concerns are functionalities that affect multiple parts of the application, such as:

- Logging
- Security
- Transaction Management
- Performance Monitoring
- Auditing
- Exception Handling

Without AOP, the same code would need to be repeated across multiple classes, making the application harder to maintain.

Spring AOP allows developers to add these functionalities without modifying the business code directly.

---

## Why use AOP?

### Without AOP

```java
@Service
public class PaymentService {

    public void processPayment() {
        System.out.println("Starting method...");

        // Business Logic

        System.out.println("Ending method...");
    }
}
```

Logging code is mixed with business logic.

### With AOP

```java
@Service
public class PaymentService {

    public void processPayment() {
        // Business Logic
    }
}
```

```java
@Aspect
@Component
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore() {
        System.out.println("Starting method...");
    }
}
```

Business logic remains clean while logging is handled separately.

---

# Terminologies in AOP

Understanding the main AOP concepts is essential before implementing aspects.

## Aspect

An **Aspect** is a module that encapsulates a cross-cutting concern.

Examples:

- LoggingAspect
- SecurityAspect
- MonitoringAspect
- AuditAspect

```java
@Aspect
@Component
public class LoggingAspect {
}
```

---

## Join Point

A **Join Point** represents a specific point during the execution of a program where an aspect can be applied.

In Spring AOP, join points are typically:

- Method execution
- Constructor execution (limited support)
- Exception handling

Example:

```java
paymentService.processPayment();
```

The execution of `processPayment()` is a join point.

---

## Advice

An **Advice** defines what action should be executed and when.

Spring provides several advice types:

| Advice          | Description                         |
| --------------- | ----------------------------------- |
| @Before         | Executes before a method            |
| @After          | Executes after a method             |
| @AfterReturning | Executes after successful execution |
| @AfterThrowing  | Executes when an exception occurs   |
| @Around         | Executes before and after a method  |

### Example

```java
@Before("execution(* com.example.service.*.*(..))")
public void logBefore() {
    System.out.println("Method execution started");
}
```

---

## Pointcut

A **Pointcut** defines where an advice should be applied.

Example:

```java
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceMethods() {}
```

This pointcut selects all methods inside the service package.

---

## Target Object

The **Target Object** is the object whose method execution is being intercepted.

Example:

```java
@Service
public class PaymentService {
}
```

`PaymentService` is the target object.

---

## Weaving

**Weaving** is the process of connecting aspects to target objects to create an advised object.

Spring performs weaving at runtime using proxies.

---

# AOP Proxies

## What is a Proxy?

A proxy is an object created by Spring that sits between the client and the target object.

Instead of calling the target directly:

```text
Client → PaymentService
```

The call goes through a proxy:

```text
Client → Proxy → PaymentService
```

The proxy executes any configured aspects before and/or after calling the target method.

---

## Why are Proxies Needed?

Spring AOP is proxy-based.

The proxy is responsible for:

- Intercepting method calls
- Applying advice
- Invoking the target method
- Returning results

Without proxies, Spring would not be able to inject cross-cutting behavior dynamically.

---

## Types of Proxies in Spring

### JDK Dynamic Proxy

Used when the target class implements an interface.

```java
public interface PaymentService {
    void processPayment();
}
```

```java
@Service
public class PaymentServiceImpl implements PaymentService {
}
```

Spring creates a proxy based on the interface.

---

### CGLIB Proxy

Used when no interface exists.

```java
@Service
public class PaymentService {

    public void processPayment() {
    }
}
```

Spring generates a subclass dynamically at runtime.

---

## Proxy Selection Rules

| Scenario                    | Proxy Type        |
| --------------------------- | ----------------- |
| Class implements interface  | JDK Dynamic Proxy |
| Class without interface     | CGLIB             |
| force proxyTargetClass=true | CGLIB             |

Example:

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
```

---

## Limitation of Spring Proxies

Spring AOP only intercepts method calls that pass through the proxy.

Example:

```java
@Service
public class UserService {

    public void methodA() {
        methodB();
    }

    public void methodB() {
        System.out.println("Method B");
    }
}
```

If an aspect is applied to `methodB()`, it will not be triggered when `methodA()` calls it internally because the call does not pass through the proxy.

This is known as the **Self-Invocation Problem**.

---

# Pointcut Expressions

## What is a Pointcut Expression?

A Pointcut Expression is a rule used to select which methods should receive advice.

Spring uses AspectJ expression syntax.

---

## Basic Syntax

```java
execution(modifiers-pattern?
          return-type-pattern
          declaring-type-pattern?
          method-name-pattern
          parameter-pattern
          throws-pattern?)
```

Example:

```java
execution(* com.example.service.*.*(..))
```

---

## Wildcards

### Match Any Return Type

```java
execution(* com.example.service.*.*(..))
```

`*` means any return type.

---

### Match Any Method Name

```java
execution(* com.example.service.*.*(..))
```

Matches all methods.

---

### Match Any Parameters

```java
execution(* com.example.service.*.*(..))
```

`(..)` means zero or more parameters.

---

## Common Pointcut Expressions

### All Methods in a Package

```java
execution(* com.example.service.*.*(..))
```

---

### All Methods in Subpackages

```java
execution(* com.example.service..*.*(..))
```

The double dot (`..`) includes subpackages.

---

### Specific Method

```java
execution(* com.example.service.UserService.findUser(..))
```

---

### Methods Starting with "find"

```java
execution(* com.example.service.*.find*(..))
```

Examples:

```java
findUser()
findById()
findAll()
```

---

### Methods with Specific Return Type

```java
execution(List com.example.service.*.*(..))
```

Matches only methods returning `List`.

---

### Methods with One String Parameter

```java
execution(* *(String))
```

---

### Methods with Any Number of Parameters

```java
execution(* *(..))
```

---

## Using Named Pointcuts

Instead of repeating expressions:

```java
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceLayer() {}
```

Reuse them:

```java
@Before("serviceLayer()")
public void logBefore() {
}
```

This improves readability and maintainability.

---

# Complete Example

```java
@Aspect
@Component
public class LoggingAspect {

    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceMethods() {}

    @Before("serviceMethods()")
    public void beforeMethod() {
        System.out.println("Method started");
    }

    @AfterReturning("serviceMethods()")
    public void afterMethod() {
        System.out.println("Method finished");
    }
}
```

---

# Advantages of Spring AOP

- Separates cross-cutting concerns from business logic.
- Improves code readability.
- Reduces code duplication.
- Easier maintenance.
- Centralizes logging, security, auditing, and monitoring.
- Integrates seamlessly with Spring Framework.

---

# Disadvantages of Spring AOP

- Can make application flow harder to follow.
- Debugging may become more complex.
- Self-invocation limitation due to proxy-based implementation.
- Runtime proxy creation introduces a small overhead.
- Supports only method execution join points (unlike full AspectJ).

---

# Summary

Spring AOP is a powerful feature that enables developers to modularize cross-cutting concerns such as logging, security, monitoring, and transaction management. It works through runtime-generated proxies that intercept method calls and apply advice based on pointcut expressions. Understanding the core concepts—Aspect, Advice, Join Point, Pointcut, Proxy, and Weaving—is essential for effectively using AOP in Spring applications.
