# 20 - Spring Interceptors

## What are Interceptors?

A **Spring Interceptor** is a mechanism provided by Spring MVC that allows us to execute code **before and/or after a controller method is executed**.

Interceptors are commonly used to implement **cross-cutting concerns**, such as:

- Logging
- Request tracking
- Auditing
- Authentication checks
- Authorization checks
- Measuring request execution time
- Adding information to the request
- Validating specific request conditions

Instead of putting this logic inside every controller:

```java
@RestController
public class UserController {

    @GetMapping("/users")
    public List<User> getUsers(HttpServletRequest request) {

        // Logging
        System.out.println("Request received: " + request.getRequestURI());

        // Business logic
        return userService.findAll();
    }
}
```

We can move the common logic into an Interceptor:

```java
public class LoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) {

        System.out.println(
                "Request received: " + request.getRequestURI()
        );

        return true;
    }
}
```

Now every controller request can be intercepted without modifying the controllers themselves.

---

# Spring MVC Request Lifecycle

To understand Interceptors, it is important to understand where they are executed.

A simplified Spring MVC request flow looks like this:

```text
Client
   |
   v
Servlet Container
   |
   v
Filter
   |
   v
DispatcherServlet
   |
   v
Handler Interceptor
   |
   v
Controller
   |
   v
Service
   |
   v
Repository
   |
   v
Controller
   |
   v
Handler Interceptor
   |
   v
Filter
   |
   v
Response
   |
   v
Client
```

The `DispatcherServlet` is the central component of Spring MVC.

It receives the HTTP request and determines which controller should handle it.

Before the controller is executed, Spring MVC gives Interceptors an opportunity to execute code.

---

# Handler Interceptor

Spring provides the `HandlerInterceptor` interface for implementing Interceptors.

```java
public interface HandlerInterceptor {

    default boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) throws Exception {
        return true;
    }

    default void postHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            ModelAndView modelAndView) throws Exception {
    }

    default void afterCompletion(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            Exception ex) throws Exception {
    }
}
```

The three most important methods are:

```text
preHandle()
postHandle()
afterCompletion()
```

Each one executes at a different point in the request lifecycle.

---

# preHandle()

`preHandle()` executes **before the controller method**.

```java
@Override
public boolean preHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler) {

    System.out.println("Before controller");

    return true;
}
```

The return value is important.

### Returning `true`

The request continues.

```java
return true;
```

The controller will be executed.

```text
Interceptor
     |
     | true
     v
Controller
```

### Returning `false`

The request is stopped.

```java
return false;
```

The controller will **not** be executed.

```text
Interceptor
     |
     | false
     X
Controller
```

This makes `preHandle()` useful for situations where we want to reject a request before it reaches the controller.

For example:

```java
@Override
public boolean preHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler) {

    String token = request.getHeader("Authorization");

    if (token == null) {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        return false;
    }

    return true;
}
```

However, authentication and authorization should generally be implemented using **Spring Security**, rather than manually building security logic inside an Interceptor.

---

# postHandle()

`postHandle()` executes **after the controller has executed**, but before the response is fully completed.

```java
@Override
public void postHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        ModelAndView modelAndView) {

    System.out.println("After controller");
}
```

The general flow is:

```text
preHandle()
     |
     v
Controller
     |
     v
postHandle()
```

`postHandle()` can be useful for MVC-related processing.

However, in modern Spring Boot applications that primarily expose REST APIs, `postHandle()` is less commonly needed.

---

# afterCompletion()

`afterCompletion()` executes after the request has been completely processed.

```java
@Override
public void afterCompletion(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        Exception ex) {

    System.out.println("Request completed");
}
```

The simplified lifecycle is:

```text
preHandle()
     |
     v
Controller
     |
     v
postHandle()
     |
     v
Response completed
     |
     v
afterCompletion()
```

The `Exception ex` parameter can contain an exception that occurred during request processing.

For example:

```java
@Override
public void afterCompletion(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        Exception ex) {

    if (ex != null) {
        System.out.println(
                "Request failed: " + ex.getMessage()
        );
    }

    System.out.println("Request completed");
}
```

---

# Interceptor Lifecycle

Putting everything together:

```text
HTTP Request
     |
     v
preHandle()
     |
     | true
     v
Controller
     |
     v
postHandle()
     |
     v
Response processing
     |
     v
afterCompletion()
     |
     v
HTTP Response
```

If `preHandle()` returns `false`:

```text
HTTP Request
     |
     v
preHandle()
     |
     | false
     X
Controller is NOT executed
```

---

# Implementation

Let's create a simple Spring Boot application.

Suppose we have this controller:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping
    public List<String> getUsers() {

        return List.of(
                "John",
                "Mary",
                "Peter"
        );
    }

    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id) {

        return "User " + id;
    }
}
```

We want to log every request.

Instead of doing this inside every controller:

```java
System.out.println("Request received");
```

we create an Interceptor.

---

# Creating an Interceptor

Create a class:

```java
@Component
public class LoggingInterceptor
        implements HandlerInterceptor {

    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) {

        System.out.println(
                "Request: "
                        + request.getMethod()
                        + " "
                        + request.getRequestURI()
        );

        return true;
    }

    @Override
    public void postHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            ModelAndView modelAndView) {

        System.out.println("Controller completed");
    }

    @Override
    public void afterCompletion(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            Exception ex) {

        System.out.println(
                "Response status: "
                        + response.getStatus()
        );
    }
}
```

Now we need to register the Interceptor.

---

# Registering the Interceptor

Spring MVC allows us to register Interceptors using `WebMvcConfigurer`.

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    private final LoggingInterceptor loggingInterceptor;

    public WebMvcConfig(
            LoggingInterceptor loggingInterceptor) {

        this.loggingInterceptor = loggingInterceptor;
    }

    @Override
    public void addInterceptors(
            InterceptorRegistry registry) {

        registry
                .addInterceptor(loggingInterceptor);
    }
}
```

Now the Interceptor will be executed for Spring MVC requests.

---

# Including and Excluding Endpoints

We can control which endpoints are intercepted.

For example:

```java
@Override
public void addInterceptors(
        InterceptorRegistry registry) {

    registry
        .addInterceptor(loggingInterceptor)
        .addPathPatterns("/api/**")
        .excludePathPatterns(
                "/api/public/**"
        );
}
```

This means:

```text
/api/users          -> intercepted
/api/orders         -> intercepted
/api/products       -> intercepted

/api/public/login   -> NOT intercepted
/api/public/signup  -> NOT intercepted
```

This is useful when we have endpoints that don't need a particular Interceptor.

---

# Measuring Request Execution Time

A very common use case for an Interceptor is measuring how long a request takes.

We can store the start time in `preHandle()`:

```java
@Component
public class PerformanceInterceptor
        implements HandlerInterceptor {

    private static final String START_TIME =
            "requestStartTime";

    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) {

        request.setAttribute(
                START_TIME,
                System.currentTimeMillis()
        );

        return true;
    }

    @Override
    public void afterCompletion(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            Exception ex) {

        Long startTime =
                (Long) request.getAttribute(START_TIME);

        long executionTime =
                System.currentTimeMillis() - startTime;

        System.out.println(
                request.getRequestURI()
                        + " took "
                        + executionTime
                        + " ms"
        );
    }
}
```

The request attribute allows us to share information between the different stages of the request.

---

# Interceptor with Multiple Requests

Suppose we make this request:

```http
GET /users/10
```

The execution could look like:

```text
LoggingInterceptor.preHandle()
        |
        v
UserController.getUser(10)
        |
        v
LoggingInterceptor.postHandle()
        |
        v
LoggingInterceptor.afterCompletion()
```

The controller itself doesn't need to know that the logging Interceptor exists.

That's one of the main benefits of Interceptors.

---

# Multiple Interceptors

We can register multiple Interceptors.

```java
@Override
public void addInterceptors(
        InterceptorRegistry registry) {

    registry.addInterceptor(loggingInterceptor);

    registry.addInterceptor(
            performanceInterceptor
    );
}
```

They execute according to their order.

For example:

```text
LoggingInterceptor.preHandle()
        |
        v
PerformanceInterceptor.preHandle()
        |
        v
Controller
        |
        v
PerformanceInterceptor.postHandle()
        |
        v
LoggingInterceptor.postHandle()
```

For completion callbacks, the order is reversed.

This is important when multiple Interceptors depend on each other.

---

# Interceptor vs Filter

One of the most common questions is:

> What is the difference between a Filter and an Interceptor?

They look similar because both can execute code before and after a request.

But they operate at different levels.

---

# Servlet Filter

A Filter belongs to the **Servlet specification**.

It operates at the Servlet container level.

For example:

```java
@Component
public class LoggingFilter
        implements Filter {

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain)
            throws IOException, ServletException {

        System.out.println("Before request");

        chain.doFilter(request, response);

        System.out.println("After request");
    }
}
```

The important part is:

```java
chain.doFilter(request, response);
```

If we don't call it, the request doesn't continue through the Filter chain.

---

# Filter Request Flow

A simplified flow is:

```text
HTTP Request
     |
     v
Filter
     |
     v
DispatcherServlet
     |
     v
Interceptor
     |
     v
Controller
```

Therefore, a Filter is executed **before Spring MVC Interceptors**.

---

# Main Difference

The easiest way to remember the difference is:

```text
Filter
   ↓
Servlet level

Interceptor
   ↓
Spring MVC level
```

A Filter doesn't specifically care which Spring controller will process the request.

An Interceptor is integrated with Spring MVC and has access to the handler/controller information.

---

# Filter vs Interceptor

| Feature                       | Filter                | Interceptor |
| ----------------------------- | --------------------- | ----------- |
| Part of                       | Servlet specification | Spring MVC  |
| Runs before DispatcherServlet | Yes                   | No          |
| Runs before Controller        | Yes                   | Yes         |
| Knows the MVC Handler         | No                    | Yes         |
| `preHandle()`                 | No                    | Yes         |
| `postHandle()`                | No                    | Yes         |
| `afterCompletion()`           | No                    | Yes         |
| Can stop request              | Yes                   | Yes         |
| Spring MVC specific           | No                    | Yes         |
| Servlet-level concerns        | Excellent             | Not ideal   |
| Controller-level concerns     | Less suitable         | Excellent   |

---

# When Should I Use a Filter?

Filters are a good choice when the requirement belongs to the **HTTP/Servlet layer**.

Examples:

- CORS
- Request/response wrapping
- Low-level HTTP logging
- Encoding
- Servlet-level security concerns
- Manipulating raw request/response data

For example:

```text
HTTP Request
     |
     v
Filter
     |
     v
Spring MVC
```

The Filter doesn't need to know which controller will eventually execute.

---

# When Should I Use an Interceptor?

Interceptors are a good choice when the requirement is related to **Spring MVC request handling**.

Examples:

- Logging controller requests
- Measuring controller execution time
- Auditing controller calls
- Checking information before a controller executes
- Working with the selected handler
- Adding MVC-specific request information

For example:

```text
HTTP Request
     |
     v
DispatcherServlet
     |
     v
Interceptor
     |
     v
Specific Controller
```

The Interceptor knows that Spring MVC has selected a particular handler.

---

# Authentication: Filter or Interceptor?

It is technically possible to implement authentication using an Interceptor:

```java
@Override
public boolean preHandle(...) {

    // validate token

    return true;
}
```

But for a real Spring Boot application, **Spring Security should normally be used**.

Spring Security integrates with the security filter chain and provides established mechanisms for:

- Authentication
- Authorization
- JWT
- OAuth2
- Session security
- CSRF
- Security context
- Method-level security

Therefore:

```text
Authentication / Authorization
            |
            v
      Spring Security
```

rather than building your own authentication Interceptor.

---

# Practical Example: Request Logger

Let's build a more realistic logging Interceptor.

```java
@Component
public class RequestLoggingInterceptor
        implements HandlerInterceptor {

    private static final String START_TIME =
            "startTime";

    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) {

        request.setAttribute(
                START_TIME,
                System.currentTimeMillis()
        );

        System.out.println(
                "Incoming request: "
                        + request.getMethod()
                        + " "
                        + request.getRequestURI()
        );

        return true;
    }

    @Override
    public void afterCompletion(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            Exception ex) {

        long startTime =
                (Long) request.getAttribute(
                        START_TIME
                );

        long duration =
                System.currentTimeMillis()
                        - startTime;

        System.out.println(
                "Completed request: "
                        + request.getMethod()
                        + " "
                        + request.getRequestURI()
                        + " | status="
                        + response.getStatus()
                        + " | duration="
                        + duration
                        + "ms"
        );
    }
}
```

A request could produce:

```text
Incoming request: GET /users/10

Completed request: GET /users/10
| status=200
| duration=42ms
```

This allows us to centralize request logging instead of duplicating it across controllers.

---

# Best Practices

## 1. Keep Interceptors focused

An Interceptor should generally handle a **cross-cutting concern**.

Good:

```text
Logging
Auditing
Request timing
Request metadata
```

Bad:

```text
User creation
Payment processing
Order calculation
Business rules
```

Business logic belongs in the appropriate application/service layer.

---

## 2. Don't put business logic inside Interceptors

Avoid:

```java
@Override
public boolean preHandle(...) {

    userService.createUser(...);

    return true;
}
```

The Interceptor shouldn't become another business layer.

Instead:

```text
Controller
    |
    v
Service
    |
    v
Repository
```

The Interceptor should remain focused on the cross-cutting concern.

---

## 3. Don't manually implement authentication when Spring Security is appropriate

Avoid creating a custom token authentication system inside:

```java
preHandle()
```

For production applications, prefer Spring Security.

---

## 4. Avoid expensive operations

Remember that an Interceptor may execute for **many requests**.

Avoid unnecessary:

```text
Database queries
External API calls
Expensive calculations
```

inside `preHandle()`.

A poorly designed Interceptor can negatively impact the performance of the entire application.

---

## 5. Use logging frameworks

Avoid:

```java
System.out.println("Request received");
```

Use a logging framework instead:

```java
private static final Logger log =
        LoggerFactory.getLogger(
                RequestLoggingInterceptor.class
        );
```

Then:

```java
log.info(
        "Incoming request: {} {}",
        request.getMethod(),
        request.getRequestURI()
);
```

---

# Important Limitation

Interceptors are part of **Spring MVC**.

That means they are not the correct tool for every type of request.

For example, if a requirement needs to operate at the Servlet level before Spring MVC processes the request, a Filter may be more appropriate.

Think about the architecture:

```text
             HTTP Request
                   |
                   v
              Servlet Filter
                   |
                   v
             DispatcherServlet
                   |
                   v
          Handler Interceptor
                   |
                   v
               Controller
                   |
                   v
                Service
                   |
                   v
              Repository
```

The further up the request pipeline your requirement belongs, the more likely a Filter is appropriate.

The closer it is to Spring MVC's controller handling, the more likely an Interceptor is appropriate.

---

# Summary

A **Spring MVC Interceptor** allows us to execute code around controller execution without modifying the controller itself.

The three main lifecycle methods are:

```text
preHandle()
    ↓
Controller
    ↓
postHandle()
    ↓
afterCompletion()
```

The most important method for controlling whether the request continues is:

```java
preHandle()
```

because:

```java
return true;
```

allows the request to continue, while:

```java
return false;
```

stops the request from reaching the controller.

The key difference between Filters and Interceptors is their level of integration:

```text
Filter
  ↓
Servlet level
  ↓
DispatcherServlet
  ↓
Interceptor
  ↓
Spring MVC / Controller
```

A useful mental model is:

> **Filter = HTTP/Servlet-level concern**

> **Interceptor = Spring MVC/controller-level concern**

And for authentication and authorization:

> **Prefer Spring Security instead of implementing custom security logic inside an Interceptor.**

---

# Quick Interview Questions

### What is a Spring Interceptor?

A Spring MVC mechanism that allows us to execute logic before and after controller execution.

### What is `preHandle()`?

A method executed before the controller. Returning `true` allows execution to continue; returning `false` stops the request.

### What is `postHandle()`?

A method executed after the controller but before the request is completely completed.

### What is `afterCompletion()`?

A method executed after the request has been completely processed.

### What is the difference between Filter and Interceptor?

A Filter operates at the Servlet level, while an Interceptor operates within Spring MVC and is associated with the selected handler/controller.

### Should authentication be implemented using an Interceptor?

Usually no. For Spring Boot applications, Spring Security should generally handle authentication and authorization.

### Can an Interceptor stop a request?

Yes. `preHandle()` can return `false`, preventing the controller from being executed.

### Can we have multiple Interceptors?

Yes. Multiple Interceptors can be registered and executed according to their configured order.

### When should I use an Interceptor?

When you need cross-cutting logic associated with Spring MVC request handling, such as logging, auditing, request timing, or controller-level checks.
