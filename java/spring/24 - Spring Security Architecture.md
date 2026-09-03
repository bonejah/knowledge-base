# 24 - Spring Security Architecture

## What is Spring Security?

**Spring Security** is a framework for securing Java applications, especially applications built with Spring Boot.

It provides mechanisms to protect application resources by controlling:

- **Who can access the application?** → Authentication
- **What can an authenticated user access?** → Authorization
- **How are security rules applied to HTTP requests?** → Security Architecture

### Why do we need Spring Security?

Imagine an application with the following endpoints:

- `GET /products` → Anyone can access.
- `POST /products` → Only authenticated users can access.
- `DELETE /products/1` → Only administrators can access.

Without a security framework, we would need to implement authentication and authorization manually in every endpoint.

Spring Security provides a centralized and configurable way to enforce these rules.

### Example

    @RestController
    @RequestMapping("/products")
    public class ProductController {

        @GetMapping
        public String getProducts() {
            return "Public products";
        }

        @PostMapping
        public String createProduct() {
            return "Product created";
        }

        @DeleteMapping("/{id}")
        public String deleteProduct(@PathVariable Long id) {
            return "Product deleted";
        }
    }

With Spring Security, we can configure:

    GET /products
        → Public

    POST /products
        → Requires authentication

    DELETE /products/{id}
        → Requires ADMIN role

The controller does not need to implement the authentication logic itself.

---

## Authentication

### What is Authentication?

**Authentication is the process of verifying the identity of a user or system.**

In simple terms:

> Authentication answers: **"Who are you?"**

### Real-world example

When you log in to your bank:

1. You enter your username.
2. You enter your password.
3. The bank verifies your credentials.
4. If they are valid, you are authenticated.

The same concept applies to Spring Security.

### Example

    Username: bruno
    Password: 123456
            ↓
    Spring Security verifies credentials
            ↓
    Credentials are valid
            ↓
    User is authenticated

### Authentication flow

    Client
       ↓
    Sends credentials
       ↓
    Spring Security
       ↓
    AuthenticationManager
       ↓
    AuthenticationProvider
       ↓
    UserDetailsService
       ↓
    PasswordEncoder
       ↓
    Credentials verified
       ↓
    Authenticated user

### Important components

#### Authentication

`Authentication` is an interface that represents the authentication request or the authenticated user.

It contains information such as:

- Principal
- Credentials
- Authorities
- Authentication status

Example:

    Authentication authentication;

After successful authentication:

    authentication.isAuthenticated();

Returns:

    true

#### AuthenticationManager

`AuthenticationManager` is the main interface responsible for authenticating a user.

    public interface AuthenticationManager {

        Authentication authenticate(Authentication authentication)
                throws AuthenticationException;
    }

Its responsibility is to delegate the authentication process to an appropriate `AuthenticationProvider`.

#### AuthenticationProvider

`AuthenticationProvider` contains the actual authentication logic for a particular authentication mechanism.

For example:

- Username and password
- LDAP
- OAuth2
- JWT
- Custom authentication

  public interface AuthenticationProvider {

        Authentication authenticate(Authentication authentication)
                throws AuthenticationException;

        boolean supports(Class<?> authentication);

  }

Example:

    AuthenticationManager
            ↓
    AuthenticationProvider
            ↓
    Verify username and password
            ↓
    Return authenticated user

#### UserDetailsService

`UserDetailsService` is responsible for loading user information.

    public interface UserDetailsService {

        UserDetails loadUserByUsername(String username)
                throws UsernameNotFoundException;
    }

Example:

    @Service
    public class CustomUserDetailsService
            implements UserDetailsService {

        @Override
        public UserDetails loadUserByUsername(String username) {

            return User.withUsername(username)
                    .password("{noop}123456")
                    .roles("USER")
                    .build();
        }
    }

> In a real application, passwords should not be stored using `{noop}`. Use a proper `PasswordEncoder`.

#### PasswordEncoder

`PasswordEncoder` is responsible for encoding passwords and verifying passwords against their encoded values.

    public interface PasswordEncoder {

        String encode(CharSequence rawPassword);

        boolean matches(
                CharSequence rawPassword,
                String encodedPassword
        );
    }

Example:

    PasswordEncoder encoder =
            new BCryptPasswordEncoder();

    String encodedPassword =
            encoder.encode("123456");

The password is not stored as plain text.

    Raw password:
    123456

    Encoded password:
    $2a$10$...

### Authentication vs Credentials

These concepts are related but different.

| Concept            | Meaning                                    |
| ------------------ | ------------------------------------------ |
| Credentials        | Information used to prove identity         |
| Authentication     | The process of verifying identity          |
| Authenticated user | The identity after successful verification |

Example:

    Username + Password
            ↓
    Credentials
            ↓
    Authentication process
            ↓
    Authenticated user

---

## Authorization

### What is Authorization?

**Authorization is the process of determining what an authenticated user is allowed to access.**

In simple terms:

> Authorization answers: **"What are you allowed to do?"**

### Real-world example

Imagine a company system:

    User: Bruno
    Role: USER

Bruno can:

- View products.
- Create orders.

But Bruno cannot:

- Delete products.
- Manage users.

An administrator can perform those operations.

### Authentication vs Authorization

This is one of the most important concepts in Spring Security.

| Authentication               | Authorization                |
| ---------------------------- | ---------------------------- |
| Verifies identity            | Verifies permissions         |
| "Who are you?"               | "What can you do?"           |
| Happens before authorization | Happens after authentication |
| Example: Login               | Example: Access `/admin`     |
| Uses credentials             | Uses roles/authorities       |

### Example

    User enters username and password
            ↓
    Authentication
            ↓
    User is authenticated
            ↓
    User requests /admin
            ↓
    Authorization
            ↓
    Does the user have ADMIN authority?
            ↓
    Yes → Allow
    No  → Deny

### Roles and Authorities

Spring Security uses **authorities** to represent permissions.

A role is a special type of authority.

Example:

    ROLE_USER
    ROLE_ADMIN
    ROLE_MANAGER

Authorities can represent more specific permissions:

    READ_PRODUCTS
    CREATE_PRODUCTS
    DELETE_PRODUCTS

### Role example

    .authorizeHttpRequests(auth -> auth
            .requestMatchers("/admin/**")
            .hasRole("ADMIN")
    )

This means:

    User must have ROLE_ADMIN

### Authority example

    .authorizeHttpRequests(auth -> auth
            .requestMatchers("/products")
            .hasAuthority("READ_PRODUCTS")
    )

This means:

    User must have READ_PRODUCTS authority

### Important difference

    .hasRole("ADMIN")

Internally checks:

    ROLE_ADMIN

While:

    .hasAuthority("ROLE_ADMIN")

Checks the exact authority:

    ROLE_ADMIN

### Authorization example

    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        @Bean
        SecurityFilterChain securityFilterChain(
                HttpSecurity http
        ) throws Exception {

            http
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/public/**").permitAll()
                    .requestMatchers("/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
                );

            return http.build();
        }
    }

### What happens here?

    /public/**
        → Anyone can access

    /admin/**
        → Requires ADMIN role

    Any other endpoint
        → Requires authentication

---

## Spring Security Architecture

### Overview

Spring Security is based on a **filter-based security architecture**.

Instead of implementing security logic inside every controller, Spring Security intercepts HTTP requests before they reach the application.

    Client
       ↓
    HTTP Request
       ↓
    Spring Security Filters
       ↓
    Authentication
       ↓
    Authorization
       ↓
    Controller
       ↓
    HTTP Response

### Why use filters?

Filters allow Spring Security to apply security rules consistently across the application.

For example:

    Every request
        ↓
    Check authentication
        ↓
    Check authorization
        ↓
    Allow or reject request

This avoids duplicating security logic in every controller.

---

## Main Components of Spring Security Architecture

### 1. SecurityFilterChain

`SecurityFilterChain` defines the security rules that apply to incoming HTTP requests.

It is one of the most important components in Spring Security.

Example:

    @Bean
    SecurityFilterChain securityFilterChain(
            HttpSecurity http
    ) throws Exception {

        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            );

        return http.build();
    }

### What does it do?

It configures:

- Which requests are public.
- Which requests require authentication.
- Which requests require specific roles.
- How authentication is performed.
- How security exceptions are handled.

### Example

    GET /public/products
            ↓
    SecurityFilterChain
            ↓
    permitAll()
            ↓
    Controller

    GET /admin/users
            ↓
    SecurityFilterChain
            ↓
    Requires ADMIN
            ↓
    Authenticated?
            ↓
    Yes → Authorization check
    No  → 401 Unauthorized

---

### 2. DelegatingFilterProxy

Spring Security integrates with the Servlet container through a filter called `DelegatingFilterProxy`.

Its responsibility is to delegate requests to Spring-managed security components.

    Servlet Container
            ↓
    DelegatingFilterProxy
            ↓
    Spring Security
            ↓
    SecurityFilterChain

### Why is it needed?

The Servlet container manages servlet filters, while Spring manages Spring beans.

`DelegatingFilterProxy` acts as a bridge between them.

    Servlet Filter
          ↓
    DelegatingFilterProxy
          ↓
    Spring Application Context
          ↓
    SecurityFilterChain

---

### 3. FilterChainProxy

`FilterChainProxy` is the main entry point into Spring Security's filter system.

It receives the request and delegates it to the appropriate `SecurityFilterChain`.

    HTTP Request
          ↓
    DelegatingFilterProxy
          ↓
    FilterChainProxy
          ↓
    SecurityFilterChain
          ↓
    Security Filters

### Important distinction

    DelegatingFilterProxy
        → Integrates Servlet container with Spring

    FilterChainProxy
        → Delegates requests to the appropriate security filter chain

---

### 4. Security Filters

Security filters are responsible for processing HTTP requests and applying security logic.

Examples include filters that:

- Authenticate users.
- Process login requests.
- Process JWT tokens.
- Handle security context.
- Apply authorization rules.
- Handle security exceptions.

### Example filter flow

    HTTP Request
          ↓
    SecurityContextHolderFilter
          ↓
    Authentication Filter
          ↓
    Authorization Filter
          ↓
    Controller

> The exact filters and their order depend on the Spring Security configuration and authentication mechanism.

---

### 5. SecurityContextHolder

`SecurityContextHolder` is used to store the security context for the current execution.

The security context contains information about the currently authenticated user.

    SecurityContext context =
            SecurityContextHolder.getContext();

To retrieve the authenticated user:

    Authentication authentication =
            context.getAuthentication();

Example:

    Authentication authentication =
            SecurityContextHolder
                    .getContext()
                    .getAuthentication();

    String username =
            authentication.getName();

### What is stored?

    SecurityContext
            ↓
    Authentication
            ↓
    Principal
            ↓
    Authorities

### Example

    Authenticated user:
    bruno

    Authorities:
    ROLE_USER
    READ_PRODUCTS

### Important concept

The `SecurityContextHolder` provides access to the authentication information during request processing.

---

### 6. SecurityContext

`SecurityContext` is an object that stores the current authentication information.

    public interface SecurityContext {

        Authentication getAuthentication();

        void setAuthentication(Authentication authentication);
    }

Example:

    SecurityContext context =
            SecurityContextHolder.getContext();

    Authentication authentication =
            context.getAuthentication();

---

### 7. Authentication

The `Authentication` object represents the current authentication state.

It contains information such as:

    Principal
    Credentials
    Authorities
    Authenticated status

Example:

    Authentication authentication =
            SecurityContextHolder
                    .getContext()
                    .getAuthentication();

### Example output

    Principal: bruno
    Authorities: ROLE_USER
    Authenticated: true

---

### 8. AuthenticationManager

`AuthenticationManager` is responsible for authenticating the user.

    Authentication Filter
            ↓
    AuthenticationManager
            ↓
    AuthenticationProvider
            ↓
    UserDetailsService
            ↓
    PasswordEncoder

### Example

    Authentication authentication =
            authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(
                            username,
                            password
                    )
            );

### What happens?

    Username + Password
            ↓
    AuthenticationManager
            ↓
    AuthenticationProvider
            ↓
    Load user
            ↓
    Verify password
            ↓
    Return Authentication

---

### 9. AuthenticationProvider

`AuthenticationProvider` performs the actual authentication logic.

For username/password authentication, a common implementation is:

    DaoAuthenticationProvider

### Example flow

    AuthenticationManager
            ↓
    DaoAuthenticationProvider
            ↓
    UserDetailsService
            ↓
    PasswordEncoder
            ↓
    Authenticated user

### Responsibilities

- Load user information.
- Verify credentials.
- Create an authenticated `Authentication` object.
- Return authentication or throw an exception.

---

### 10. UserDetailsService

`UserDetailsService` loads user information from a data source.

The data source could be:

- Database.
- In-memory storage.
- LDAP.
- Custom service.

Example:

    @Service
    public class CustomUserDetailsService
            implements UserDetailsService {

        @Override
        public UserDetails loadUserByUsername(String username) {

            return User.withUsername(username)
                    .password("{noop}123456")
                    .roles("USER")
                    .build();
        }
    }

### Important concept

`UserDetailsService` does **not** authenticate the user by itself.

It loads the user information.

    UserDetailsService
        → Loads user

    AuthenticationProvider
        → Verifies credentials

---

### 11. UserDetails

`UserDetails` represents the user information used by Spring Security.

    public interface UserDetails {

        Collection<? extends GrantedAuthority>
                getAuthorities();

        String getPassword();

        String getUsername();

        boolean isAccountNonExpired();

        boolean isAccountNonLocked();

        boolean isCredentialsNonExpired();

        boolean isEnabled();
    }

### Example

    Username:
    bruno

    Password:
    Encoded password

    Authorities:
    ROLE_USER

---

### 12. GrantedAuthority

`GrantedAuthority` represents an authority granted to an authenticated user.

Example:

    new SimpleGrantedAuthority("ROLE_ADMIN");

Or:

    new SimpleGrantedAuthority("READ_PRODUCTS");

### Example

    User:
    bruno

    Authorities:
    ROLE_USER
    READ_PRODUCTS

---

### 13. PasswordEncoder

`PasswordEncoder` is responsible for password encoding and verification.

Example:

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

### Why use a password encoder?

Passwords should not be stored as plain text.

    Plain password:
    123456

    Encoded password:
    $2a$10$...

During login:

    Raw password
            ↓
    PasswordEncoder.matches()
            ↓
    Compare with stored encoded password
            ↓
    Valid or invalid

---

## Complete Authentication Architecture

### Username and Password Authentication

    ┌──────────────────────┐
    │       Client         │
    │  Username + Password │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │   Security Filters   │
    │ Authentication Filter│
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │ AuthenticationManager│
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │ AuthenticationProvider│
    │ DaoAuthenticationProvider│
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │   UserDetailsService │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │      Database        │
    │  User + Password     │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │   PasswordEncoder    │
    │  Verify Password     │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │ Authenticated User   │
    │ Authentication object│
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │  SecurityContext     │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │    Authorization     │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │     Controller       │
    └──────────────────────┘

---

## Complete Request Flow

### Example: `GET /admin/users`

Imagine a user sends:

    GET /admin/users

### Step 1 — Client sends request

    Client
        ↓
    GET /admin/users

### Step 2 — Request enters Spring Security

    HTTP Request
        ↓
    DelegatingFilterProxy
        ↓
    FilterChainProxy
        ↓
    SecurityFilterChain

### Step 3 — Authentication is checked

Spring Security checks whether the request contains valid authentication information.

For example:

    Session
    JWT
    Basic Authentication
    OAuth2

### Step 4 — Authentication is loaded

If the user is authenticated:

    SecurityContextHolder
            ↓
    Authentication

Example:

    Username:
    bruno

    Authorities:
    ROLE_USER

### Step 5 — Authorization is checked

The security configuration says:

    .requestMatchers("/admin/**")
    .hasRole("ADMIN")

Spring Security checks:

    Does the user have ROLE_ADMIN?

### Step 6 — Access decision

    ROLE_ADMIN exists?
            ↓
    Yes → Allow request
    No  → Deny request

### Step 7 — Controller executes

If authorized:

    @GetMapping("/admin/users")
    public String getUsers() {
        return "Users";
    }

### Step 8 — Response is returned

    Controller
        ↓
    HTTP Response
        ↓
    Client

---

## Authentication and Authorization in One Flow

    ┌──────────────────────┐
    │       Client         │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │   HTTP Request       │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │ Spring Security      │
    │ Security Filters     │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │   Authentication     │
    │   "Who are you?"     │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │ Authenticated User   │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │   Authorization      │
    │ "What can you do?"   │
    └──────────┬───────────┘
               │
               ▼
          ┌─────────┐
          │ Allowed?│
          └────┬────┘
               │
          ┌────┴────┐
          │         │
         Yes        No
          │         │
          ▼         ▼
     Controller   403 Forbidden
          │
          ▼
     HTTP Response

---

## Authentication vs Authorization: Practical Example

Imagine a banking application.

### User logs in

    Username: bruno
    Password: ********

Spring Security verifies the credentials.

    Authentication successful

### User accesses account

    GET /accounts

The user has:

    ROLE_USER

Access is allowed.

### User tries to delete another account

    DELETE /accounts/10

The endpoint requires:

    ROLE_ADMIN

The user only has:

    ROLE_USER

Access is denied.

    403 Forbidden

### Summary

    Authentication:
    "Are you Bruno?"

    Authorization:
    "Are you allowed to delete this account?"

---

## 401 Unauthorized vs 403 Forbidden

These two HTTP status codes are very important.

### 401 Unauthorized

**The request is not authenticated.**

Example:

    User tries to access a protected endpoint
            ↓
    No valid authentication
            ↓
    401 Unauthorized

In simple terms:

> "You need to authenticate first."

### 403 Forbidden

**The user is authenticated but does not have permission.**

Example:

    User is authenticated
            ↓
    User requests admin endpoint
            ↓
    User does not have ADMIN authority
            ↓
    403 Forbidden

In simple terms:

> "We know who you are, but you are not allowed to do this."

### Comparison

| Status | Meaning           | Example                             |
| ------ | ----------------- | ----------------------------------- |
| 401    | Not authenticated | Missing or invalid credentials      |
| 403    | Not authorized    | Authenticated user lacks permission |

---

## Spring Security Architecture with JWT

Spring Security can also authenticate requests using JWT tokens.

### What is JWT?

**JWT (JSON Web Token)** is a token format commonly used for stateless authentication.

Instead of sending username and password on every request, the client sends a token.

### Example

    Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

### JWT authentication flow

    Client
        ↓
    Login
        ↓
    Authentication
        ↓
    JWT Token Generated
        ↓
    Client Stores Token
        ↓
    Client Sends Token
        ↓
    Spring Security JWT Filter
        ↓
    Validate Token
        ↓
    Create Authentication
        ↓
    SecurityContextHolder
        ↓
    Authorization
        ↓
    Controller

### Example request

    GET /products
    Authorization: Bearer <JWT>

### What happens?

    JWT Token
        ↓
    JWT Authentication Filter
        ↓
    Validate token
        ↓
    Extract username and authorities
        ↓
    Create Authentication object
        ↓
    Store in SecurityContext
        ↓
    Authorization
        ↓
    Controller

### Important concept

With JWT authentication, the application usually does not need to store a server-side session for every user.

This is called **stateless authentication**.

---

## Stateless vs Stateful Authentication

### Stateful Authentication

The server stores authentication information in a session.

    Client
        ↓
    Login
        ↓
    Server creates session
        ↓
    Client receives session ID
        ↓
    Client sends session ID
        ↓
    Server loads session
        ↓
    Authenticated user

### Stateless Authentication

The client sends a token with every request.

    Client
        ↓
    Login
        ↓
    Server generates JWT
        ↓
    Client receives JWT
        ↓
    Client sends JWT
        ↓
    Server validates JWT
        ↓
    Authenticated user

### Comparison

| Stateful                           | Stateless                                   |
| ---------------------------------- | ------------------------------------------- |
| Uses server-side session           | Uses token                                  |
| Server stores authentication state | Server does not need to store session state |
| Common with session-based login    | Common with JWT                             |
| Client sends session ID            | Client sends token                          |

---

## SecurityContextHolder and ThreadLocal

By default, Spring Security uses a `ThreadLocal`-based strategy to associate the security context with the current thread.

### What is ThreadLocal?

`ThreadLocal` allows each thread to have its own independent value.

Example:

    Thread 1
        ↓
    SecurityContext → User A

    Thread 2
        ↓
    SecurityContext → User B

### Why is this useful?

When a request is processed, Spring Security can access the authenticated user through:

    SecurityContextHolder.getContext()

### Example

    @GetMapping("/profile")
    public String profile() {

        Authentication authentication =
                SecurityContextHolder
                        .getContext()
                        .getAuthentication();

        return authentication.getName();
    }

If Bruno is authenticated:

    bruno

### Important concept

The security context is associated with the current execution context, not simply with the application as a whole.

---

## SecurityContext Lifecycle

The security context is associated with the current request and execution context.

A simplified flow is:

    Request starts
        ↓
    SecurityContext is loaded
        ↓
    Authentication is available
        ↓
    Controller executes
        ↓
    Request finishes
        ↓
    SecurityContext is handled according to the configured strategy

### Important note

In modern Spring Security, `SecurityContextHolderFilter` is commonly used to load the security context, while persistence behavior depends on the configured security context strategy.

For session-based applications, the context can be associated with the HTTP session.

For stateless applications, the context is typically created for the request and does not need to be persisted as a server-side session.

---

## Request Authorization Architecture

Spring Security can apply authorization rules at different levels.

### URL-based authorization

    http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
    );

### Method-based authorization

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) {
        // Delete user
    }

### Important concept

Authorization can happen:

    HTTP Request Level
            ↓
    Method Level

### Example

    GET /users/1
            ↓
    URL authorization
            ↓
    Controller method
            ↓
    Method authorization
            ↓
    Business logic

---

## Example: Spring Boot Security Configuration

### Dependency

In a Spring Boot application, add:

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

### Security configuration

    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        @Bean
        SecurityFilterChain securityFilterChain(
                HttpSecurity http
        ) throws Exception {

            http
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/public/**").permitAll()
                    .requestMatchers("/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
                )
                .httpBasic(Customizer.withDefaults());

            return http.build();
        }
    }

### What does this configuration do?

    /public/**
        → Public

    /admin/**
        → Requires ADMIN role

    Any other endpoint
        → Requires authentication

    Authentication mechanism
        → HTTP Basic

### Example request

    GET /public/products

Result:

    200 OK

### Example request

    GET /admin/users

Without authentication:

    401 Unauthorized

With authentication but without ADMIN role:

    403 Forbidden

---

## Important Spring Security Interfaces

| Interface                | Responsibility                      |
| ------------------------ | ----------------------------------- |
| `SecurityFilterChain`    | Defines security rules              |
| `AuthenticationManager`  | Authenticates users                 |
| `AuthenticationProvider` | Performs authentication logic       |
| `UserDetailsService`     | Loads user information              |
| `UserDetails`            | Represents user information         |
| `PasswordEncoder`        | Encodes and verifies passwords      |
| `Authentication`         | Represents authentication           |
| `SecurityContext`        | Stores authentication               |
| `SecurityContextHolder`  | Provides access to security context |
| `GrantedAuthority`       | Represents permissions              |

---

## Important Classes

| Class                                 | Responsibility                                |
| ------------------------------------- | --------------------------------------------- |
| `DelegatingFilterProxy`               | Connects Servlet filters to Spring            |
| `FilterChainProxy`                    | Delegates requests to security filter chains  |
| `DaoAuthenticationProvider`           | Authenticates using user details and password |
| `UsernamePasswordAuthenticationToken` | Represents username/password authentication   |
| `SecurityContextHolder`               | Stores and retrieves security context         |
| `BCryptPasswordEncoder`               | Encodes passwords using BCrypt                |

---

## Authentication Flow vs Authorization Flow

### Authentication flow

    Request
        ↓
    Authentication Filter
        ↓
    AuthenticationManager
        ↓
    AuthenticationProvider
        ↓
    UserDetailsService
        ↓
    PasswordEncoder
        ↓
    Authentication
        ↓
    SecurityContextHolder

### Authorization flow

    Request
        ↓
    SecurityContextHolder
        ↓
    Authentication
        ↓
    GrantedAuthority
        ↓
    Authorization rules
        ↓
    Allow or deny

---

## Common Interview Questions

### 1. What is Spring Security?

Spring Security is a framework that provides authentication, authorization, and protection mechanisms for Spring applications.

### 2. What is the difference between authentication and authorization?

Authentication verifies identity.

Authorization verifies permissions.

### 3. What is SecurityFilterChain?

It defines the security rules and filters that apply to HTTP requests.

### 4. What is AuthenticationManager?

It is responsible for authenticating users by delegating to an appropriate `AuthenticationProvider`.

### 5. What is AuthenticationProvider?

It performs the actual authentication logic for a specific authentication mechanism.

### 6. What is UserDetailsService?

It loads user information, usually from a database or another data source.

### 7. What is PasswordEncoder?

It encodes passwords and verifies raw passwords against encoded passwords.

### 8. What is SecurityContextHolder?

It provides access to the security context containing the current authentication.

### 9. What is the difference between 401 and 403?

401 means the request is not authenticated.

403 means the user is authenticated but does not have permission.

### 10. What is the role of GrantedAuthority?

It represents permissions granted to an authenticated user.

---

## Final Summary

Spring Security is built around two fundamental concepts:

    Authentication
        ↓
    Who are you?

    Authorization
        ↓
    What can you do?

The architecture connects these concepts through security filters and authentication components.

    HTTP Request
        ↓
    SecurityFilterChain
        ↓
    Authentication
        ↓
    SecurityContextHolder
        ↓
    Authorization
        ↓
    Controller

### The most important components to remember

    SecurityFilterChain
            ↓
    AuthenticationManager
            ↓
    AuthenticationProvider
            ↓
    UserDetailsService
            ↓
    PasswordEncoder
            ↓
    Authentication
            ↓
    SecurityContextHolder
            ↓
    Authorization

### The most important idea

> **Spring Security intercepts requests, authenticates the user, stores the authentication information, and then checks whether the user is authorized to access the requested resource.**
