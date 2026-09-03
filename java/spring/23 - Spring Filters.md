# 23 - Spring Filters

## What are Filters?

A **Filter** is a component that intercepts HTTP requests and responses before they reach the controller or after the controller finishes processing.

Filters are part of the **Servlet specification**, so they operate at a lower level than Spring MVC controllers.

Filters are commonly used for:

- Request logging
- Authentication
- Authorization
- CORS
- Adding or modifying HTTP headers
- Request validation
- Measuring request execution time
- Modifying HTTP responses
- Blocking requests
- Security-related processing

### Request Flow

A simplified Spring Boot request flow looks like this:

    Client
       |
       v
    HTTP Request
       |
       v
    Servlet Container (Tomcat)
       |
       v
    Filter Chain
       |
       +--> Filter 1
       |
       +--> Filter 2
       |
       +--> Filter 3
       |
       v
    DispatcherServlet
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
    HTTP Response

A filter can execute logic **before and after** the next component in the chain.

For example:

    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain) {

        // Before the request continues
        System.out.println("Request received");

        chain.doFilter(request, response);

        // After the request has been processed
        System.out.println("Response generated");
    }

The most important line is:

    chain.doFilter(request, response);

This tells the application to continue processing the request and execute the next element in the chain.

If the filter does not call `chain.doFilter()`, the request can be stopped at that point.

---

# Understanding the Filter Chain

A **Filter Chain** is an ordered sequence of filters through which an HTTP request passes.

For example:

    Request
       |
       v
    +----------------+
    | Logging Filter |
    +----------------+
            |
            v
    +----------------+
    | Auth Filter    |
    +----------------+
            |
            v
    +----------------+
    | CORS Filter    |
    +----------------+
            |
            v
    +----------------+
    | Dispatcher     |
    | Servlet        |
    +----------------+
            |
            v
       Controller

Each filter receives the request and can:

1. Process the request
2. Modify the request
3. Stop the request
4. Pass the request to the next filter
5. Process the response after the next component finishes

### The `FilterChain` Interface

The Servlet API provides the `FilterChain` interface.

The most important method is:

    chain.doFilter(request, response);

It passes the request to the next element in the chain.

A filter usually follows this pattern:

    public void doFilter(...) {

        // Logic before the next filter

        chain.doFilter(request, response);

        // Logic after the next filter
    }

### Before and After Execution

Imagine three filters:

    Filter A
       |
       v
    Filter B
       |
       v
    Filter C
       |
       v
    Controller

The execution order is similar to nested method calls:

    Filter A - BEFORE
        |
    Filter B - BEFORE
        |
    Filter C - BEFORE
        |
    Controller
        |
    Filter C - AFTER
        |
    Filter B - AFTER
        |
    Filter A - AFTER

This happens because each filter calls `chain.doFilter()` and waits for the next component to finish.

---

## Why Filter Order Matters

The order of filters can be very important.

For example:

    Authentication Filter
            |
            v
    Authorization Filter
            |
            v
        Controller

Authentication normally needs to happen before authorization.

Why?

Because authorization needs to know:

- Who is the user?
- Is the user authenticated?
- What roles does the user have?
- What authorities does the user have?

Only after authentication can the application properly determine whether the user is authorized to perform an operation.

---

# Implementing a Custom Filter

There are several ways to create a custom filter in Spring Boot.

A common approach is to extend:

    OncePerRequestFilter

`OncePerRequestFilter` is provided by Spring and is commonly used when implementing custom HTTP filters.

It guarantees that the filter is invoked once per request dispatch.

## Example: Request Logging Filter

Suppose we want to create a filter that logs every HTTP request.

### Step 1 — Create the Filter

    @Component
    public class RequestLoggingFilter extends OncePerRequestFilter {

        @Override
        protected void doFilterInternal(
                HttpServletRequest request,
                HttpServletResponse response,
                FilterChain filterChain)
                throws ServletException, IOException {

            System.out.println(
                    "Request: " +
                    request.getMethod() +
                    " " +
                    request.getRequestURI()
            );

            filterChain.doFilter(request, response);
        }
    }

Because the class is annotated with:

    @Component

Spring automatically detects the class as a Spring bean and registers the filter.

### Step 2 — Continue the Filter Chain

The most important line is:

    filterChain.doFilter(request, response);

Without this call, the request will not continue to the next filter or controller.

For example:

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain)
            throws ServletException, IOException {

        System.out.println("Before controller");

        filterChain.doFilter(request, response);

        System.out.println("After controller");
    }

The execution can be visualized as:

    Request
       |
       v
    Filter
       |
       | "Before controller"
       |
       v
    Controller
       |
       v
    Filter
       |
       | "After controller"
       |
       v
    Response

---

# Blocking a Request

A filter can also stop a request.

For example, suppose an application expects an API key.

    @Component
    public class ApiKeyFilter extends OncePerRequestFilter {

        @Override
        protected void doFilterInternal(
                HttpServletRequest request,
                HttpServletResponse response,
                FilterChain filterChain)
                throws ServletException, IOException {

            String apiKey = request.getHeader("X-API-KEY");

            if (!"my-secret-key".equals(apiKey)) {
                response.setStatus(
                        HttpServletResponse.SC_UNAUTHORIZED
                );

                return;
            }

            filterChain.doFilter(request, response);
        }
    }

If the API key is invalid:

    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    return;

The filter does not call:

    filterChain.doFilter(request, response);

Therefore, the request stops at the filter.

The flow becomes:

    Request
       |
       v
    API Key Filter
       |
       +---- Invalid ----> 401 Unauthorized
       |
       +---- Valid ------> Controller

This is one of the important characteristics of filters:

> A filter can decide whether a request should continue.

---

# Registering and Ordering Filters

If you need more control over how a filter is registered, you can use `FilterRegistrationBean`.

For example:

    @Configuration
    public class FilterConfig {

        @Bean
        public FilterRegistrationBean<RequestLoggingFilter>
        requestLoggingFilter(RequestLoggingFilter filter) {

            FilterRegistrationBean<RequestLoggingFilter> registration =
                    new FilterRegistrationBean<>();

            registration.setFilter(filter);
            registration.addUrlPatterns("/api/*");
            registration.setOrder(1);

            return registration;
        }
    }

The following configuration:

    registration.addUrlPatterns("/api/*");

means that the filter applies to URLs such as:

    /api/users
    /api/products
    /api/orders

but not necessarily to endpoints outside `/api`.

The following configuration:

    registration.setOrder(1);

controls the filter's position relative to other servlet filters.

---

# Filters vs Interceptors

Filters and Spring MVC Interceptors are similar, but they operate at different levels.

The general request flow is:

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

## Filter

A Filter belongs to the **Servlet layer**.

It can execute before Spring MVC processes the request.

    HTTP Request
         |
         v
       Filter
         |
         v
    Spring MVC

Typical use cases:

- Request logging
- Authentication
- Security
- CORS
- Request/response manipulation
- HTTP-level processing

## Interceptor

An Interceptor belongs to the **Spring MVC layer**.

It operates around controller execution.

    HTTP Request
         |
         v
    Spring MVC
         |
         v
     Interceptor
         |
         v
     Controller

Typical use cases:

- Controller-specific validation
- Auditing controller execution
- Measuring controller execution time
- Adding model attributes
- Executing logic before or after controllers

### General Rule

Use a **Filter** when you need to work at the HTTP/Servlet level.

Use an **Interceptor** when your logic is closely related to Spring MVC controllers.

---

# A Glimpse at Spring Security Filters

Spring Security heavily relies on filters.

When Spring Security is enabled, HTTP requests pass through a **Security Filter Chain** before reaching the controller.

Conceptually:

    Client
      |
      v
    HTTP Request
      |
      v
    Spring Security Filter Chain
      |
      +--> Security Filter
      |
      +--> Authentication Filter
      |
      +--> Authorization Filter
      |
      v
    Controller

The exact filters and their order depend on the Spring Security configuration and the authentication mechanisms being used.

## What is the Security Filter Chain?

The **Security Filter Chain** is a sequence of Spring Security filters responsible for processing security-related aspects of an HTTP request.

It can handle:

- Authentication
- Authorization
- Security context management
- CSRF protection
- Session management
- Bearer tokens
- HTTP Basic authentication
- Exception handling
- Security headers

A simplified flow is:

    Request
       |
       v
    Security Filter Chain
       |
       +--> Is the request authenticated?
       |
       +--> Who is the authenticated user?
       |
       +--> What authorities does the user have?
       |
       +--> Is the user authorized?
       |
       v
    Controller

---

# Authentication vs Authorization

Understanding the difference between authentication and authorization is fundamental when studying Spring Security filters.

## Authentication

Authentication answers:

> **Who are you?**

For example, a client might send:

    Authorization: Bearer <JWT>

A Spring Security authentication filter can:

1. Extract the token
2. Validate the token
3. Identify the user
4. Create an `Authentication` object
5. Store the authentication information in the `SecurityContext`

Conceptually:

    Request
       |
       v
    Authentication Filter
       |
       +--> Extract credentials
       |
       +--> Validate credentials
       |
       +--> Identify user
       |
       v
    SecurityContext

## Authorization

Authorization answers:

> **Are you allowed to perform this operation?**

For example:

    @PreAuthorize("hasRole('ADMIN')")

A user can be authenticated but still not have permission to access a particular resource.

The relationship is:

    Authentication
          |
          v
    Who are you?
          |
          v
    Authorization
          |
          v
    What are you allowed to do?

---

# Custom Filters with Spring Security

A common use case is creating a custom authentication filter.

For example, suppose an application receives a JWT:

    Authorization: Bearer abc123

A custom filter could inspect the token.

    @Component
    public class CustomAuthenticationFilter
            extends OncePerRequestFilter {

        @Override
        protected void doFilterInternal(
                HttpServletRequest request,
                HttpServletResponse response,
                FilterChain filterChain)
                throws ServletException, IOException {

            String authorization =
                    request.getHeader("Authorization");

            if (authorization != null &&
                authorization.startsWith("Bearer ")) {

                String token =
                        authorization.substring(7);

                // Validate token
                // Load user
                // Create Authentication
                // Set SecurityContext
            }

            filterChain.doFilter(request, response);
        }
    }

The filter can then be added to the Spring Security filter chain.

For example:

    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        @Bean
        SecurityFilterChain securityFilterChain(
                HttpSecurity http,
                CustomAuthenticationFilter customFilter)
                throws Exception {

            return http
                    .addFilterBefore(
                            customFilter,
                            UsernamePasswordAuthenticationFilter.class
                    )
                    .authorizeHttpRequests(auth -> auth
                            .requestMatchers("/public/**")
                            .permitAll()
                            .anyRequest()
                            .authenticated()
                    )
                    .build();
        }
    }

The important method here is:

    .addFilterBefore(...)

Spring Security also provides methods such as:

    .addFilterAfter(...)

and:

    .addFilterAt(...)

These methods allow you to control where a custom filter is positioned in the Security Filter Chain.

---

# JWT Authentication Flow

A typical Spring Security application using JWT might follow this flow:

    Client
       |
       | Authorization: Bearer <JWT>
       v
    Spring Security Filter Chain
       |
       v
    JWT Authentication Filter
       |
       +--> Extract JWT
       |
       +--> Validate JWT
       |
       +--> Extract user information
       |
       +--> Create Authentication
       |
       +--> Store Authentication
       |    in SecurityContext
       |
       v
    Authorization Filter
       |
       +--> Check permissions
       |
       v
    Controller

The important idea is that the JWT filter does not normally call the controller directly.

Instead, it authenticates the request and then allows the request to continue through the security chain.

---

# 401 vs 403

Filters are commonly involved when dealing with HTTP security errors.

## 401 Unauthorized

A `401 Unauthorized` response generally means that authentication is missing or invalid.

Example:

    Request
       |
       v
    Authentication Filter
       |
       X
    401 Unauthorized

In simple terms:

    401 = "I don't know who you are."

## 403 Forbidden

A `403 Forbidden` response generally means that the user is authenticated but does not have sufficient permissions.

Example:

    Request
       |
       v
    Authentication
       |
       v
    Authorization
       |
       X
    403 Forbidden

In simple terms:

    403 = "I know who you are, but you are not allowed to do this."

---

# Important Spring Security Filters

Spring Security contains many filters, and the exact chain depends on the configuration.

Some commonly encountered filters include:

## SecurityContextHolderFilter

Responsible for making the security context available during request processing.

The security context contains information about the current authentication.

Conceptually:

    Request
       |
       v
    SecurityContextHolderFilter
       |
       v
    SecurityContext
       |
       v
    Other Security Filters

## BasicAuthenticationFilter

Processes HTTP Basic authentication credentials.

For example:

    Authorization: Basic <credentials>

It can extract the credentials and create an authenticated security context when authentication succeeds.

## BearerTokenAuthenticationFilter

Processes bearer tokens such as JWTs.

For example:

    Authorization: Bearer eyJhbGciOi...

The filter extracts the bearer token and delegates authentication to the appropriate authentication mechanism.

## AuthorizationFilter

Responsible for evaluating whether the authenticated request has permission to access the requested resource.

Conceptually:

    Authenticated User
           |
           v
    AuthorizationFilter
           |
           +---- Allowed ------> Controller
           |
           +---- Not Allowed --> 403

---

# Complete Request Flow

Putting everything together:

    Client
       |
       v
    HTTP Request
       |
       v
    Servlet Container
       |
       v
    Servlet Filters
       |
       v
    Spring Security Filter Chain
       |
       +--> Security Context
       |
       +--> Authentication
       |
       +--> JWT / Basic Authentication
       |
       +--> Authorization
       |
       v
    DispatcherServlet
       |
       v
    Spring MVC Interceptors
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
    Database
       |
       v
    HTTP Response
       |
       v
    Filters
       |
       v
    Client

This illustrates an important distinction:

    Servlet Filter
        |
        | operates at HTTP / Servlet level
        v
    Spring Security Filter
        |
        | handles security concerns
        v
    Spring MVC Interceptor
        |
        | handles MVC/controller concerns
        v
    Controller

---

# Key Takeaways

## 1. What is a Filter?

A Filter intercepts HTTP requests and responses before and/or after the request reaches the controller.

## 2. What is a Filter Chain?

A Filter Chain is an ordered sequence of filters through which a request passes.

The key method is:

    filterChain.doFilter(request, response);

Calling it allows the request to continue.

## 3. How do you create a custom filter?

A common approach in Spring Boot is:

    public class MyFilter extends OncePerRequestFilter {
        ...
    }

## 4. Can a Filter block a request?

Yes.

A filter can return without calling `filterChain.doFilter()`.

For example:

    if (invalidRequest) {
        response.setStatus(401);
        return;
    }

## 5. What is the Spring Security Filter Chain?

It is a specialized chain of filters responsible for security-related processing such as:

- Authentication
- Authorization
- Security context
- JWT/Bearer tokens
- HTTP Basic authentication
- CSRF
- Exception handling

## 6. Authentication vs Authorization

Remember:

    Authentication
        = Who are you?

    Authorization
        = What are you allowed to do?

## 7. 401 vs 403

Remember:

    401
        = Authentication problem

    403
        = Authorization problem

## 8. The Big Picture

The most important concept to remember is:

    HTTP Request
         |
         v
    Filter Chain
         |
         v
    Spring Security Filters
         |
         v
    Authentication
         |
         v
    Authorization
         |
         v
    Spring MVC
         |
         v
    Controller

Understanding Filters and the Filter Chain makes it much easier to understand how **Spring Security, JWT authentication, authentication, authorization, `SecurityContext`, `401`, `403`, and custom security filters** work internally.
