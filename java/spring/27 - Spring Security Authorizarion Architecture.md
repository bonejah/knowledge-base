# 27 - Spring Security Authorizarion Architecture

## What is Authorization?

Authorization is the process of determining **what an authenticated user is allowed to do**.

While authentication answers **"Who are you?"**, authorization answers **"What are you allowed to access?"**

For example, a user may be authenticated but still not have permission to delete a user or access an administrative endpoint.

### Authentication vs Authorization

| Authentication                      | Authorization                             |
| ----------------------------------- | ----------------------------------------- |
| Verifies the user's identity        | Verifies what the user can do             |
| Answers: Who are you?               | Answers: What are you allowed to do?      |
| Uses credentials, JWT, OAuth2, etc. | Uses roles, permissions, and access rules |
| Happens before authorization        | Happens after authentication              |

### Example

A user logs in successfully:

    Username: john
    Password: ********

The authentication process confirms that John is a valid user.

The authorization process then checks:

    Can John access /admin/users?
    Can John delete a user?
    Can John view reports?

If John does not have the required permission, access is denied.

---

## Roles & Permissions

### What is a Role?

A **role** is a collection of permissions that represents a user's responsibilities or level of access.

Examples:

    ROLE_USER
    ROLE_MANAGER
    ROLE_ADMIN

A role usually represents **who the user is in the application**.

### What is a Permission?

A **permission** is a specific action that a user is allowed to perform.

Examples:

    USER_READ
    USER_CREATE
    USER_UPDATE
    USER_DELETE

A permission represents **what the user can do**.

### Roles vs Permissions

| Role                                | Permission                   |
| ----------------------------------- | ---------------------------- |
| Represents a group of access rights | Represents a specific action |
| Higher-level abstraction            | More granular                |
| Example: ROLE_ADMIN                 | Example: USER_DELETE         |
| Can contain multiple permissions    | Can be assigned to roles     |

### Example: Role-Based Access Control

    ROLE_USER
        ├── USER_READ
        └── PROFILE_UPDATE

    ROLE_MANAGER
        ├── USER_READ
        ├── USER_CREATE
        └── USER_UPDATE

    ROLE_ADMIN
        ├── USER_READ
        ├── USER_CREATE
        ├── USER_UPDATE
        └── USER_DELETE

A user can have one or more roles.

    John → ROLE_USER

    Maria → ROLE_MANAGER

    Admin → ROLE_ADMIN

### Why Use Permissions?

Roles are useful for simple applications, but permissions provide more flexibility.

For example:

    ROLE_MANAGER → USER_READ, USER_UPDATE

    ROLE_SUPPORT → USER_READ

Both users may access the same endpoint, but only the manager can update users.

### Important distinction

**Role = a group of permissions.**

**Permission = a specific action.**

In Spring Security, roles are commonly represented with the `ROLE_` prefix.

    ROLE_ADMIN

A permission does not necessarily need the `ROLE_` prefix.

    USER_READ

---

## The Authorization Architecture

Spring Security authorization is responsible for deciding whether a request should be allowed to access a protected resource.

The main components are:

    Client
       ↓
    Authentication
       ↓
    SecurityContext
       ↓
    Authorization
       ↓
    Protected Resource

### Main Components

#### 1. SecurityFilterChain

The `SecurityFilterChain` defines the security rules for incoming HTTP requests.

It determines:

- Which endpoints are public.
- Which endpoints require authentication.
- Which endpoints require specific roles or authorities.
- How unauthorized requests are handled.

#### 2. SecurityContext

The `SecurityContext` stores information about the currently authenticated user.

It contains an `Authentication` object.

    SecurityContext
        └── Authentication
                ├── Principal
                ├── Authorities
                └── Authenticated

#### 3. Authentication

The `Authentication` object represents the authenticated user.

It contains:

    Principal → The user
    Authorities → Roles and permissions
    Authenticated → Whether authentication succeeded

Example:

    Authentication
        ├── Principal: john
        ├── Authorities:
        │     ├── ROLE_USER
        │     └── USER_READ
        └── Authenticated: true

#### 4. GrantedAuthority

A `GrantedAuthority` represents an authority granted to the user.

Examples:

    ROLE_ADMIN
    ROLE_USER
    USER_READ
    USER_DELETE

Spring Security uses authorities to make authorization decisions.

#### 5. AuthorizationFilter

The `AuthorizationFilter` evaluates the authorization rules configured in the `SecurityFilterChain`.

For example:

    /admin/** → ROLE_ADMIN

If the authenticated user does not have the required authority, access is denied.

#### 6. AuthorizationManager

The `AuthorizationManager` is responsible for making authorization decisions.

It evaluates whether the current `Authentication` has access to a protected resource.

Conceptually:

    AuthorizationManager
        ├── Authentication
        ├── Request
        └── Authorization Decision

The decision is:

    GRANTED
    or
    DENIED

---

## Simple Authorization Flow

Consider the following endpoint:

    GET /admin/users

Only users with `ROLE_ADMIN` should be allowed to access it.

### Flow

    Client
       ↓
    GET /admin/users
       ↓
    SecurityFilterChain
       ↓
    Authentication
       ↓
    SecurityContext
       ↓
    AuthorizationFilter
       ↓
    Check ROLE_ADMIN
       ↓
    ┌─────────────────────┐
    │ Has ROLE_ADMIN?     │
    └─────────────────────┘
       ↓              ↓
      YES             NO
       ↓              ↓
    Allow           Deny
       ↓              ↓
    Controller     403 Forbidden

### Example

    User: John
    Authorities:
        ROLE_USER
        USER_READ

    Request:
        GET /admin/users

    Required:
        ROLE_ADMIN

    Result:
        Access Denied

### Another Example

    User: Admin
    Authorities:
        ROLE_ADMIN
        USER_READ
        USER_DELETE

    Request:
        GET /admin/users

    Required:
        ROLE_ADMIN

    Result:
        Access Granted

---

## Request Authorization vs Method Authorization

Spring Security supports authorization at different levels.

### Request Authorization

Request authorization protects endpoints based on URL patterns.

Example:

    /admin/** → ROLE_ADMIN

This is configured in the `SecurityFilterChain`.

### Method Authorization

Method authorization protects individual methods.

Example:

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) {
        ...
    }

This allows authorization rules to be placed directly on service or controller methods.

### Comparison

| Request Authorization             | Method Authorization          |
| --------------------------------- | ----------------------------- |
| Protects URL patterns             | Protects individual methods   |
| Configured in SecurityFilterChain | Configured with annotations   |
| Example: `/admin/**`              | Example: `@PreAuthorize(...)` |
| Good for endpoint-level rules     | Good for business-level rules |

---

## Implementation Using Spring Boot

We will build a simple application with:

- Public endpoint.
- User endpoint.
- Admin endpoint.
- Role-based authorization.
- Permission-based authorization.
- Method-level authorization.

### Project Structure

    src/main/java/com/example/security
    │
    ├── SecurityApplication.java
    ├── config
    │   └── SecurityConfig.java
    ├── controller
    │   └── UserController.java
    └── service
        └── UserService.java

### 1. Create the Spring Boot Project

Add the following dependencies:

    Spring Web
    Spring Security

For Maven:

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

### 2. Create the Security Configuration

The `SecurityFilterChain` defines which endpoints are protected.

    @Configuration
    @EnableMethodSecurity
    public class SecurityConfig {

        @Bean
        public SecurityFilterChain securityFilterChain(
                HttpSecurity http) throws Exception {

            http
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/public/**").permitAll()
                    .requestMatchers("/user/**").hasRole("USER")
                    .requestMatchers("/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
                )
                .httpBasic(Customizer.withDefaults());

            return http.build();
        }
    }

### Explanation

    /public/** → Anyone can access

    /user/** → Requires ROLE_USER

    /admin/** → Requires ROLE_ADMIN

    Any other endpoint → Requires authentication

### Important

When using:

    hasRole("ADMIN")

Spring Security checks for:

    ROLE_ADMIN

The `ROLE_` prefix is added automatically.

---

## 3. Create a Controller

    @RestController
    public class UserController {

        @GetMapping("/public/hello")
        public String publicEndpoint() {
            return "Public endpoint";
        }

        @GetMapping("/user/profile")
        public String userEndpoint() {
            return "User endpoint";
        }

        @GetMapping("/admin/users")
        public String adminEndpoint() {
            return "Admin endpoint";
        }
    }

### Endpoint Access Rules

| Endpoint        | Access     |
| --------------- | ---------- |
| `/public/hello` | Public     |
| `/user/profile` | ROLE_USER  |
| `/admin/users`  | ROLE_ADMIN |

---

## 4. Create Users with In-Memory Authentication

For demonstration purposes, we can configure users in memory.

    @Bean
    public UserDetailsService userDetailsService(
            PasswordEncoder passwordEncoder) {

        UserDetails user = User.builder()
                .username("john")
                .password(passwordEncoder.encode("password"))
                .roles("USER")
                .build();

        UserDetails admin = User.builder()
                .username("admin")
                .password(passwordEncoder.encode("password"))
                .roles("ADMIN")
                .build();

        return new InMemoryUserDetailsManager(user, admin);
    }

### Password Encoder

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

### Complete Configuration

    @Configuration
    @EnableMethodSecurity
    public class SecurityConfig {

        @Bean
        public SecurityFilterChain securityFilterChain(
                HttpSecurity http) throws Exception {

            http
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/public/**").permitAll()
                    .requestMatchers("/user/**").hasRole("USER")
                    .requestMatchers("/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
                )
                .httpBasic(Customizer.withDefaults());

            return http.build();
        }

        @Bean
        public UserDetailsService userDetailsService(
                PasswordEncoder passwordEncoder) {

            UserDetails user = User.builder()
                    .username("john")
                    .password(passwordEncoder.encode("password"))
                    .roles("USER")
                    .build();

            UserDetails admin = User.builder()
                    .username("admin")
                    .password(passwordEncoder.encode("password"))
                    .roles("ADMIN")
                    .build();

            return new InMemoryUserDetailsManager(user, admin);
        }

        @Bean
        public PasswordEncoder passwordEncoder() {
            return new BCryptPasswordEncoder();
        }
    }

---

## 5. Test the Authorization

### Public Endpoint

    GET /public/hello

    Result:
    200 OK

No authentication is required.

### User Endpoint

    GET /user/profile

    Username: john
    Password: password

    Result:
    200 OK

John has:

    ROLE_USER

### Admin Endpoint

    GET /admin/users

    Username: john
    Password: password

    Result:
    403 Forbidden

John is authenticated, but he does not have:

    ROLE_ADMIN

### Admin Access

    GET /admin/users

    Username: admin
    Password: password

    Result:
    200 OK

The admin user has:

    ROLE_ADMIN

---

## 6. Authorization Using Permissions

Instead of checking only roles, we can check specific permissions.

Example:

    USER_READ
    USER_CREATE
    USER_UPDATE
    USER_DELETE

### Configure Authorities

    UserDetails user = User.builder()
            .username("john")
            .password(passwordEncoder.encode("password"))
            .authorities("USER_READ")
            .build();

    UserDetails admin = User.builder()
            .username("admin")
            .password(passwordEncoder.encode("password"))
            .authorities(
                "USER_READ",
                "USER_CREATE",
                "USER_UPDATE",
                "USER_DELETE"
            )
            .build();

### Protect an Endpoint

    @GetMapping("/users")
    @PreAuthorize("hasAuthority('USER_READ')")
    public String getUsers() {
        return "Users list";
    }

### Another Example

    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasAuthority('USER_DELETE')")
    public String deleteUser(@PathVariable Long id) {
        return "User deleted";
    }

### Role vs Authority

    hasRole("ADMIN")

Checks:

    ROLE_ADMIN

    hasAuthority("USER_DELETE")

Checks:

    USER_DELETE

---

## 7. Method-Level Authorization

Method authorization allows us to protect methods using annotations.

### Enable Method Security

    @Configuration
    @EnableMethodSecurity
    public class SecurityConfig {
        ...
    }

### Example

    @Service
    public class UserService {

        @PreAuthorize("hasRole('ADMIN')")
        public void deleteUser(Long id) {
            // Delete user
        }
    }

Only users with `ROLE_ADMIN` can execute the method.

### Example with Permission

    @PreAuthorize("hasAuthority('USER_DELETE')")
    public void deleteUser(Long id) {
        // Delete user
    }

### Why Use Method Authorization?

It is useful when authorization depends on business logic.

For example:

    A manager can update users.

    But only users from their own department.

This type of rule is often better implemented at the service layer.

---

## 8. @PreAuthorize

`@PreAuthorize` checks authorization **before the method is executed**.

If the authorization expression evaluates to `false`, the method is not executed.

### Basic Example

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) {
        // Delete user
    }

### Flow

    Request
       ↓
    Authentication
       ↓
    @PreAuthorize
       ↓
    Check ROLE_ADMIN
       ↓
    ┌─────────────────────┐
    │ Has ROLE_ADMIN?     │
    └─────────────────────┘
       ↓              ↓
      YES             NO
       ↓              ↓
    Execute         Deny
    method
       ↓
    Return result

### Example with Role

    @PreAuthorize("hasRole('ADMIN')")
    @DeleteMapping("/users/{id}")
    public String deleteUser(@PathVariable Long id) {
        return "User deleted";
    }

Only users with:

    ROLE_ADMIN

Can execute the method.

### Example with Permission

    @PreAuthorize("hasAuthority('USER_DELETE')")
    public void deleteUser(Long id) {
        // Delete user
    }

This checks for the specific permission:

    USER_DELETE

### Example with Multiple Roles

    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    public String getReports() {
        return "Reports";
    }

The method is allowed if the user has either:

    ROLE_ADMIN

    or

    ROLE_MANAGER

### Example with Multiple Permissions

    @PreAuthorize("hasAuthority('USER_READ') or hasAuthority('USER_UPDATE')")
    public String getUser() {
        return "User";
    }

The user needs at least one of the permissions.

### Example with AND

    @PreAuthorize("hasRole('MANAGER') and hasAuthority('USER_UPDATE')")
    public void updateUser(Long id) {
        // Update user
    }

The user must have:

    ROLE_MANAGER

    AND

    USER_UPDATE

### Example with Method Parameters

`@PreAuthorize` can also use method parameters.

    @PreAuthorize("#id == authentication.principal.id")
    public User getUser(Long id) {
        return userRepository.findById(id);
    }

This means:

    The requested user ID must match
    the authenticated user's ID.

### Example with Authentication

    @PreAuthorize("authentication.name == 'admin'")
    public String adminOperation() {
        return "Admin operation";
    }

This allows only the user named:

    admin

### Example with a Custom Permission Bean

    @PreAuthorize("@permissionService.canEditUser(authentication, #id)")
    public void updateUser(Long id) {
        // Update user
    }

The authorization logic is delegated to a custom service.

Example:

    @Service
    public class PermissionService {

        public boolean canEditUser(
                Authentication authentication,
                Long userId) {

            return authentication.getName().equals("admin");
        }
    }

### Important

`@PreAuthorize` is evaluated **before** the method executes.

    @PreAuthorize
        ↓
    Method execution
        ↓
    Return result

---

## 9. @PostAuthorize

`@PostAuthorize` checks authorization **after the method has executed**.

This is useful when the authorization decision depends on the **returned object**.

### Basic Example

    @PostAuthorize("returnObject.owner == authentication.name")
    public User getUser(Long id) {
        return userRepository.findById(id);
    }

The method executes first.

After the result is returned, Spring Security checks whether the authenticated user is the owner.

### Flow

    Request
       ↓
    Authentication
       ↓
    Execute method
       ↓
    Return object
       ↓
    @PostAuthorize
       ↓
    Check returned object
       ↓
    ┌─────────────────────┐
    │ Is user the owner?  │
    └─────────────────────┘
       ↓              ↓
      YES             NO
       ↓              ↓
    Return          Deny
    object

### Example

    @PostAuthorize("returnObject.username == authentication.name")
    public User getUser(Long id) {
        return userRepository.findById(id);
    }

If the authenticated user is:

    john

And the returned object is:

    User
        username: john

Access is granted.

But if the returned object is:

    User
        username: maria

Access is denied.

### Important

`@PostAuthorize` checks the **returned object**, not the method parameters.

The expression can access the returned value through:

    returnObject

### Example with a DTO

    @PostAuthorize("returnObject.ownerUsername == authentication.name")
    public Document getDocument(Long id) {
        return documentRepository.findById(id);
    }

This is useful when the returned object contains ownership information.

### Example with a Service

    @Service
    public class DocumentService {

        @PostAuthorize("returnObject.ownerUsername == authentication.name")
        public Document getDocument(Long id) {
            return documentRepository.findById(id);
        }
    }

The method can retrieve the document, but the result is only returned if the user is authorized.

### Important

`@PostAuthorize` is evaluated **after** the method executes.

    Method execution
        ↓
    Return object
        ↓
    @PostAuthorize
        ↓
    Return or deny

---

## 10. @PreAuthorize vs @PostAuthorize

| @PreAuthorize                       | @PostAuthorize                                       |
| ----------------------------------- | ---------------------------------------------------- |
| Runs before the method              | Runs after the method                                |
| Checks before execution             | Checks after execution                               |
| Can prevent the method from running | Method already executed                              |
| Useful for roles and permissions    | Useful for returned-object ownership                 |
| Example: `hasRole('ADMIN')`         | Example: `returnObject.owner == authentication.name` |

### Example Comparison

#### @PreAuthorize

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) {
        // Delete user
    }

The user must be an admin **before** the method runs.

#### @PostAuthorize

    @PostAuthorize("returnObject.owner == authentication.name")
    public Document getDocument(Long id) {
        return documentRepository.findById(id);
    }

The method runs first, and the returned document is checked afterward.

---

## 11. Practical Example: User Service

### User Entity

    public class User {

        private Long id;
        private String username;
        private String role;

        // Getters and setters
    }

### User Service

    @Service
    public class UserService {

        @PreAuthorize("hasRole('ADMIN')")
        public void deleteUser(Long id) {
            // Delete user
        }

        @PreAuthorize("hasAuthority('USER_READ')")
        public User getUser(Long id) {
            return userRepository.findById(id);
        }

        @PostAuthorize("returnObject.username == authentication.name")
        public User getMyUser(Long id) {
            return userRepository.findById(id);
        }
    }

### Explanation

    deleteUser()
        → Requires ROLE_ADMIN

    getUser()
        → Requires USER_READ

    getMyUser()
        → Returns the user only if
          the username matches the
          authenticated user

---

## 12. Practical Example: Document Ownership

Imagine an application where users can access only their own documents.

### Document

    public class Document {

        private Long id;
        private String title;
        private String ownerUsername;

        // Getters and setters
    }

### Service

    @Service
    public class DocumentService {

        @PostAuthorize(
            "returnObject.ownerUsername == authentication.name"
        )
        public Document getDocument(Long id) {
            return documentRepository.findById(id);
        }
    }

### Example

    Authenticated user:
        john

    Requested document:
        Document ID: 10
        Owner: john

    Result:
        Access Granted

### Another Example

    Authenticated user:
        john

    Requested document:
        Document ID: 20
        Owner: maria

    Result:
        Access Denied

### Why Use @PostAuthorize?

Because the authorization decision depends on the returned document.

The application must first retrieve the document to know who owns it.

---

## 13. Important Considerations

### @PreAuthorize

Use `@PreAuthorize` when you can determine authorization **before** executing the method.

Examples:

    hasRole('ADMIN')

    hasAuthority('USER_DELETE')

    #id == authentication.principal.id

### @PostAuthorize

Use `@PostAuthorize` when authorization depends on the **returned object**.

Examples:

    returnObject.ownerUsername == authentication.name

    returnObject.username == authentication.name

### Performance

`@PostAuthorize` executes the method before checking authorization.

This means the method may perform database queries or other operations before access is denied.

For expensive operations, prefer `@PreAuthorize` when possible.

### Security

Do not use `@PostAuthorize` as a replacement for proper data filtering.

For example, if a user should only access their own documents, it may be more efficient to query only documents belonging to that user.

    findByIdAndOwnerUsername(id, authentication.name)

This avoids retrieving an unauthorized object in the first place.

---

## 14. Authorization Architecture with JWT

In a JWT-based application, the authorization process is similar.

The main difference is that the user's authorities are extracted from the JWT.

### JWT Authorization Flow

    Client
       ↓
    Request with JWT
       ↓
    JWT Authentication Filter
       ↓
    Validate JWT
       ↓
    Extract Authorities
       ↓
    SecurityContext
       ↓
    AuthorizationFilter
       ↓
    Check Roles / Permissions
       ↓
    Allow or Deny

### Example JWT Payload

    {
        "sub": "john",
        "roles": ["USER"],
        "permissions": ["USER_READ"]
    }

The JWT contains information about the authenticated user.

The application uses that information to create an `Authentication` object.

### Important

The JWT itself does not automatically authorize requests.

Spring Security must:

1. Validate the token.
2. Extract the authorities.
3. Store them in the `SecurityContext`.
4. Evaluate the authorization rules.

---

## 15. Authorization vs Authentication in JWT

### Authentication

    Is this JWT valid?

### Authorization

    Does this JWT contain the required authority?

### Example

    JWT:
        User: john
        Role: USER

    Request:
        DELETE /admin/users/10

    Required:
        ROLE_ADMIN

    Result:
        403 Forbidden

The token is valid, but the user does not have permission.

---

## 16. Common Authorization Mistakes

### Mistake 1: Confusing Authentication with Authorization

A valid login does not mean the user can access everything.

### Mistake 2: Using Roles Without Understanding Authorities

    hasRole("ADMIN")

Checks:

    ROLE_ADMIN

Not:

    ADMIN

### Mistake 3: Forgetting the ROLE\_ Prefix

If you manually create authorities:

    .authorities("ADMIN")

Then:

    hasRole("ADMIN")

Will not match unless the authority is:

    ROLE_ADMIN

### Mistake 4: Using Only URL Security

Some business rules cannot be expressed only with URL patterns.

For complex rules, use method-level authorization.

### Mistake 5: Trusting User Input for Permissions

Never allow the client to decide which permissions it has.

Permissions must come from a trusted authentication source.

---

## 17. Summary

### Authentication

**Who are you?**

### Authorization

**What are you allowed to do?**

### Roles

Groups of permissions.

### Permissions

Specific actions.

### SecurityFilterChain

Defines endpoint access rules.

### SecurityContext

Stores the authenticated user.

### GrantedAuthority

Represents roles and permissions.

### AuthorizationManager

Makes authorization decisions.

### Method Security

Protects individual methods.

### @PreAuthorize

Checks authorization **before method execution**.

    @PreAuthorize("hasRole('ADMIN')")

### @PostAuthorize

Checks authorization **after method execution**.

    @PostAuthorize("returnObject.ownerUsername == authentication.name")

### JWT

Carries authentication information that can be used to establish authorities.

---

## Final Architecture

    ┌──────────────────────┐
    │       Client         │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │   SecurityFilterChain│
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │    Authentication    │
    │  Basic Auth / JWT    │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │    SecurityContext   │
    │      Principal       │
    │     Authorities      │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │   Authorization      │
    │  Roles / Permissions │
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │   Protected Resource │
    │  Controller / Service│
    └──────────────────────┘

**Authentication establishes identity. Authorization establishes access.**
