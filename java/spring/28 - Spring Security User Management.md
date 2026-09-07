# 28 - Spring Security User Management

## User Management

User Management is the process of creating, retrieving, updating, and deleting users in an application, as well as managing their credentials, roles, and permissions.

In Spring Security, user management is closely related to **Authentication** and **Authorization**.

- **Authentication:** Who is the user?
- **Authorization:** What can the user do?
- **User Management:** How are users created, stored, updated, and managed?

### User Management Responsibilities

A typical user management system includes:

- User registration
- User login
- Password management
- User profile management
- Role assignment
- Permission management
- Account activation and deactivation
- User retrieval
- User update
- User deletion

### User Management vs Authentication

| User Management         | Authentication                 |
| ----------------------- | ------------------------------ |
| Creates users           | Verifies user identity         |
| Stores user information | Checks credentials             |
| Updates passwords       | Validates passwords            |
| Assigns roles           | Establishes authenticated user |
| Disables accounts       | Rejects disabled accounts      |

User Management provides the data that Spring Security uses during authentication.

---

## Implementation using Spring Boot

We will implement a simple User Management API using:

- Spring Boot
- Spring Security
- Spring Data JPA
- H2 Database
- BCrypt Password Encoder
- REST API

### Project Structure

    src/main/java/com/example/usermanagement
    │
    ├── UserManagementApplication.java
    │
    ├── config
    │   ├── SecurityConfig.java
    │   └── PasswordConfig.java
    │
    ├── controller
    │   └── UserController.java
    │
    ├── dto
    │   ├── UserRequest.java
    │   └── UserResponse.java
    │
    ├── entity
    │   └── User.java
    │
    ├── repository
    │   └── UserRepository.java
    │
    └── service
        ├── UserService.java
        └── CustomUserDetailsService.java

---

## 1. Add Dependencies

Add the following dependencies to your `pom.xml`.

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>

---

## 2. Create the User Entity

The `User` entity represents a user stored in the database.

    package com.example.usermanagement.entity;

    import jakarta.persistence.*;

    @Entity
    @Table(name = "users")
    public class User {

        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @Column(nullable = false, unique = true)
        private String username;

        @Column(nullable = false, unique = true)
        private String email;

        @Column(nullable = false)
        private String password;

        private boolean enabled = true;

        private String role;

        public User() {
        }

        public User(String username, String email, String password, String role) {
            this.username = username;
            this.email = email;
            this.password = password;
            this.role = role;
        }

        public Long getId() {
            return id;
        }

        public String getUsername() {
            return username;
        }

        public String getEmail() {
            return email;
        }

        public String getPassword() {
            return password;
        }

        public boolean isEnabled() {
            return enabled;
        }

        public String getRole() {
            return role;
        }

        public void setUsername(String username) {
            this.username = username;
        }

        public void setEmail(String email) {
            this.email = email;
        }

        public void setPassword(String password) {
            this.password = password;
        }

        public void setEnabled(boolean enabled) {
            this.enabled = enabled;
        }

        public void setRole(String role) {
            this.role = role;
        }
    }

### Important

The password must **never be stored as plain text**.

    Incorrect:
    password = "123456"

    Correct:
    password = "$2a$10$..."

The correct approach is to hash the password using `BCryptPasswordEncoder`.

---

## 3. Create the Repository

The repository provides access to the database.

    package com.example.usermanagement.repository;

    import com.example.usermanagement.entity.User;
    import org.springframework.data.jpa.repository.JpaRepository;

    import java.util.Optional;

    public interface UserRepository extends JpaRepository<User, Long> {

        Optional<User> findByUsername(String username);

        Optional<User> findByEmail(String email);

        boolean existsByUsername(String username);

        boolean existsByEmail(String email);
    }

### Useful Repository Methods

    userRepository.findAll();

    userRepository.findById(id);

    userRepository.findByUsername(username);

    userRepository.save(user);

    userRepository.deleteById(id);

---

## 4. Create the DTOs

DTOs are used to control the data exchanged through the API.

### UserRequest

    package com.example.usermanagement.dto;

    public record UserRequest(
            String username,
            String email,
            String password,
            String role
    ) {
    }

### UserResponse

    package com.example.usermanagement.dto;

    public record UserResponse(
            Long id,
            String username,
            String email,
            String role,
            boolean enabled
    ) {
    }

### Why use DTOs?

Without DTOs, it is easy to accidentally expose sensitive information such as the password.

    Entity → Contains password
    DTO    → Does not expose password

---

## 5. Create the User Service

The service contains the business logic for user management.

    package com.example.usermanagement.service;

    import com.example.usermanagement.dto.UserRequest;
    import com.example.usermanagement.dto.UserResponse;
    import com.example.usermanagement.entity.User;
    import com.example.usermanagement.repository.UserRepository;
    import org.springframework.security.crypto.password.PasswordEncoder;
    import org.springframework.stereotype.Service;

    import java.util.List;

    @Service
    public class UserService {

        private final UserRepository userRepository;
        private final PasswordEncoder passwordEncoder;

        public UserService(
                UserRepository userRepository,
                PasswordEncoder passwordEncoder
        ) {
            this.userRepository = userRepository;
            this.passwordEncoder = passwordEncoder;
        }

        public UserResponse createUser(UserRequest request) {

            if (userRepository.existsByUsername(request.username())) {
                throw new RuntimeException("Username already exists");
            }

            if (userRepository.existsByEmail(request.email())) {
                throw new RuntimeException("Email already exists");
            }

            User user = new User();

            user.setUsername(request.username());
            user.setEmail(request.email());

            // Never store plain-text passwords
            user.setPassword(
                    passwordEncoder.encode(request.password())
            );

            user.setRole(
                    request.role() != null ? request.role() : "USER"
            );

            user.setEnabled(true);

            User savedUser = userRepository.save(user);

            return toResponse(savedUser);
        }

        public List<UserResponse> findAllUsers() {

            return userRepository.findAll()
                    .stream()
                    .map(this::toResponse)
                    .toList();
        }

        public UserResponse findById(Long id) {

            User user = userRepository.findById(id)
                    .orElseThrow(() ->
                            new RuntimeException("User not found")
                    );

            return toResponse(user);
        }

        public UserResponse updateUser(Long id, UserRequest request) {

            User user = userRepository.findById(id)
                    .orElseThrow(() ->
                            new RuntimeException("User not found")
                    );

            user.setEmail(request.email());

            if (request.password() != null &&
                    !request.password().isBlank()) {

                user.setPassword(
                        passwordEncoder.encode(request.password())
                );
            }

            User updatedUser = userRepository.save(user);

            return toResponse(updatedUser);
        }

        public void deleteUser(Long id) {

            if (!userRepository.existsById(id)) {
                throw new RuntimeException("User not found");
            }

            userRepository.deleteById(id);
        }

        public void enableUser(Long id) {

            User user = userRepository.findById(id)
                    .orElseThrow(() ->
                            new RuntimeException("User not found")
                    );

            user.setEnabled(true);

            userRepository.save(user);
        }

        public void disableUser(Long id) {

            User user = userRepository.findById(id)
                    .orElseThrow(() ->
                            new RuntimeException("User not found")
                    );

            user.setEnabled(false);

            userRepository.save(user);
        }

        private UserResponse toResponse(User user) {

            return new UserResponse(
                    user.getId(),
                    user.getUsername(),
                    user.getEmail(),
                    user.getRole(),
                    user.isEnabled()
            );
        }
    }

### What happens during registration?

    Client
       │
       ▼
    POST /users
       │
       ▼
    UserController
       │
       ▼
    UserService
       │
       ├── Validate username
       ├── Validate email
       ├── Encode password
       ├── Assign role
       └── Save user
       │
       ▼
    Database

---

## 6. Create the Password Encoder

Spring Security provides `PasswordEncoder` to securely hash passwords.

    package com.example.usermanagement.config;

    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
    import org.springframework.security.crypto.password.PasswordEncoder;

    @Configuration
    public class PasswordConfig {

        @Bean
        public PasswordEncoder passwordEncoder() {
            return new BCryptPasswordEncoder();
        }
    }

### Encoding a Password

    String encodedPassword =
            passwordEncoder.encode("myPassword");

### Checking a Password

    boolean matches =
            passwordEncoder.matches(
                    "myPassword",
                    encodedPassword
            );

Spring Security uses the encoded password to verify credentials.

---

## 7. Create the User Controller

The controller exposes the User Management API.

    package com.example.usermanagement.controller;

    import com.example.usermanagement.dto.UserRequest;
    import com.example.usermanagement.dto.UserResponse;
    import com.example.usermanagement.service.UserService;
    import org.springframework.http.ResponseEntity;
    import org.springframework.web.bind.annotation.*;

    import java.util.List;

    @RestController
    @RequestMapping("/users")
    public class UserController {

        private final UserService userService;

        public UserController(UserService userService) {
            this.userService = userService;
        }

        @PostMapping
        public ResponseEntity<UserResponse> createUser(
                @RequestBody UserRequest request
        ) {
            return ResponseEntity.ok(
                    userService.createUser(request)
            );
        }

        @GetMapping
        public ResponseEntity<List<UserResponse>> findAllUsers() {

            return ResponseEntity.ok(
                    userService.findAllUsers()
            );
        }

        @GetMapping("/{id}")
        public ResponseEntity<UserResponse> findById(
                @PathVariable Long id
        ) {
            return ResponseEntity.ok(
                    userService.findById(id)
            );
        }

        @PutMapping("/{id}")
        public ResponseEntity<UserResponse> updateUser(
                @PathVariable Long id,
                @RequestBody UserRequest request
        ) {
            return ResponseEntity.ok(
                    userService.updateUser(id, request)
            );
        }

        @DeleteMapping("/{id}")
        public ResponseEntity<Void> deleteUser(
                @PathVariable Long id
        ) {
            userService.deleteUser(id);

            return ResponseEntity.noContent().build();
        }

        @PatchMapping("/{id}/enable")
        public ResponseEntity<Void> enableUser(
                @PathVariable Long id
        ) {
            userService.enableUser(id);

            return ResponseEntity.noContent().build();
        }

        @PatchMapping("/{id}/disable")
        public ResponseEntity<Void> disableUser(
                @PathVariable Long id
        ) {
            userService.disableUser(id);

            return ResponseEntity.noContent().build();
        }
    }

---

## 8. Configure Spring Security

The security configuration defines which endpoints are public and which require authentication.

    package com.example.usermanagement.config;

    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
    import org.springframework.security.config.annotation.web.builders.HttpSecurity;
    import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
    import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
    import org.springframework.security.crypto.password.PasswordEncoder;
    import org.springframework.security.web.SecurityFilterChain;

    @Configuration
    @EnableMethodSecurity
    public class SecurityConfig {

        @Bean
        public SecurityFilterChain securityFilterChain(
                HttpSecurity http
        ) throws Exception {

            http
                .csrf(AbstractHttpConfigurer::disable)

                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/users").permitAll()
                    .anyRequest().authenticated()
                )

                .httpBasic(basic -> {});

            return http.build();
        }

        @Bean
        public PasswordEncoder passwordEncoder() {
            return new BCryptPasswordEncoder();
        }
    }

### What does this configuration do?

    POST /users
        │
        └── Public
            Anyone can register

    GET /users
        │
        └── Requires authentication

    GET /users/{id}
        │
        └── Requires authentication

    PUT /users/{id}
        │
        └── Requires authentication

    DELETE /users/{id}
        │
        └── Requires authentication

> In a production application, user registration, administration, and account management should have more specific authorization rules.

---

## 9. Load Users from the Database

Spring Security needs a way to retrieve users during authentication.

This is done using `UserDetailsService`.

    package com.example.usermanagement.service;

    import com.example.usermanagement.entity.User;
    import com.example.usermanagement.repository.UserRepository;
    import org.springframework.security.core.authority.SimpleGrantedAuthority;
    import org.springframework.security.core.userdetails.*;
    import org.springframework.stereotype.Service;

    @Service
    public class CustomUserDetailsService
            implements UserDetailsService {

        private final UserRepository userRepository;

        public CustomUserDetailsService(
                UserRepository userRepository
        ) {
            this.userRepository = userRepository;
        }

        @Override
        public UserDetails loadUserByUsername(String username)
                throws UsernameNotFoundException {

            User user = userRepository.findByUsername(username)
                    .orElseThrow(() ->
                            new UsernameNotFoundException(
                                    "User not found"
                            )
                    );

            return org.springframework.security.core.userdetails.User
                    .withUsername(user.getUsername())
                    .password(user.getPassword())
                    .authorities(
                            new SimpleGrantedAuthority(
                                    "ROLE_" + user.getRole()
                            )
                    )
                    .disabled(!user.isEnabled())
                    .build();
        }
    }

### Authentication Flow

    Client
       │
       ▼
    Spring Security Filter
       │
       ▼
    UserDetailsService
       │
       ▼
    UserRepository
       │
       ▼
    Database
       │
       ▼
    UserDetails
       │
       ▼
    PasswordEncoder
       │
       ▼
    Authentication Success

---

## 10. Configure Authentication Manager

Spring Security can use the database-backed `UserDetailsService` to authenticate users.

    package com.example.usermanagement.config;

    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.security.authentication.AuthenticationManager;
    import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;

    @Configuration
    public class AuthenticationConfig {

        @Bean
        public AuthenticationManager authenticationManager(
                AuthenticationConfiguration configuration
        ) throws Exception {

            return configuration.getAuthenticationManager();
        }
    }

---

## 11. Configure H2 Database

For this example, we will use H2 as an in-memory database.

    spring.datasource.url=jdbc:h2:mem:usermanagement

    spring.datasource.driver-class-name=org.h2.Driver

    spring.datasource.username=sa

    spring.datasource.password=

    spring.jpa.hibernate.ddl-auto=update

    spring.h2.console.enabled=true

### H2 Console

    http://localhost:8080/h2-console

### JDBC URL

    jdbc:h2:mem:usermanagement

---

## 12. Test User Registration

### Request

    POST http://localhost:8080/users

### Body

    {
      "username": "bruno",
      "email": "bruno@example.com",
      "password": "123456",
      "role": "USER"
    }

### Response

    {
      "id": 1,
      "username": "bruno",
      "email": "bruno@example.com",
      "role": "USER",
      "enabled": true
    }

### Important

The response does not contain the password.

    Password is stored in the database as a BCrypt hash.
    Password is not returned by the API.

---

## 13. Test User Authentication

After creating a user, you can authenticate using Basic Authentication.

### Request

    GET http://localhost:8080/users

### Credentials

    Username: bruno
    Password: 123456

### Example

    curl -u bruno:123456 http://localhost:8080/users

### What happens?

    Client
       │
       ▼
    Authorization: Basic ...
       │
       ▼
    Spring Security
       │
       ▼
    UserDetailsService
       │
       ▼
    Database
       │
       ▼
    PasswordEncoder
       │
       ▼
    Authenticated User
       │
       ▼
    UserController

---

## 14. Test User Update

### Request

    PUT http://localhost:8080/users/1

### Body

    {
      "username": "bruno",
      "email": "newemail@example.com",
      "password": "newPassword123",
      "role": "USER"
    }

### What happens?

    UserService
       │
       ├── Find user
       ├── Update email
       ├── Encode new password
       └── Save changes

---

## 15. Test User Deletion

### Request

    DELETE http://localhost:8080/users/1

### Response

    204 No Content

### What happens?

    Client
       │
       ▼
    DELETE /users/1
       │
       ▼
    UserService
       │
       ▼
    UserRepository
       │
       ▼
    Database
       │
       ▼
    User Deleted

---

## 16. Test Account Activation and Deactivation

### Disable User

    PATCH http://localhost:8080/users/1/disable

### Enable User

    PATCH http://localhost:8080/users/1/enable

### Disabled User

If the user is disabled, authentication will fail.

    User
       │
       ▼
    enabled = false
       │
       ▼
    Spring Security
       │
       ▼
    Authentication Rejected

---

## 17. Add Role-Based Authorization

User Management often includes different roles.

    USER
    ADMIN
    MANAGER

### Example

    USER
    ├── Read own profile
    └── Update own profile

    ADMIN
    ├── Read all users
    ├── Create users
    ├── Update users
    ├── Delete users
    └── Disable users

### Method-Level Authorization

    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping
    public ResponseEntity<List<UserResponse>> findAllUsers() {

        return ResponseEntity.ok(
                userService.findAllUsers()
        );
    }

### Another Example

    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> findById(
            @PathVariable Long id
    ) {
        return ResponseEntity.ok(
                userService.findById(id)
        );
    }

### Authorization Flow

    Request
       │
       ▼
    Authentication
       │
       ▼
    UserDetails
       │
       ▼
    Roles / Authorities
       │
       ▼
    Authorization
       │
       ├── Allowed → Controller
       │
       └── Denied → 403 Forbidden

---

## 18. User Management with JWT

In modern applications, User Management is commonly combined with JWT Authentication.

### Flow

    1. User registers
           │
           ▼
    2. Password is hashed
           │
           ▼
    3. User is saved in database
           │
           ▼
    4. User logs in
           │
           ▼
    5. Credentials are validated
           │
           ▼
    6. JWT is generated
           │
           ▼
    7. Client sends JWT in Authorization header
           │
           ▼
    8. Spring Security validates JWT
           │
           ▼
    9. User is authenticated

### Example Header

    Authorization: Bearer <JWT_TOKEN>

### User Management vs JWT

    User Management
        │
        ├── Creates users
        ├── Stores passwords
        ├── Updates users
        └── Manages roles
        │
        ▼
    JWT Authentication
        │
        ├── Authenticates users
        ├── Generates tokens
        ├── Validates tokens
        └── Provides user identity

---

## 19. Important Security Considerations

### Never Store Plain-Text Passwords

    user.setPassword(
            passwordEncoder.encode(request.password())
    );

### Never Return Passwords

Use DTOs to hide sensitive fields.

### Validate User Input

    Username cannot be empty
    Email must be valid
    Password must meet minimum requirements

### Prevent Duplicate Users

    userRepository.existsByUsername(username);

### Use Unique Constraints

    @Column(nullable = false, unique = true)
    private String username;

### Disable Instead of Delete

For some applications, disabling an account is safer than deleting it.

    enabled = false

### Protect Administrative Endpoints

    @PreAuthorize("hasRole('ADMIN')")

---

## 20. Complete User Management Flow

    ┌──────────────────┐
    │      Client      │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │  UserController  │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │   UserService    │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │  UserRepository  │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │     Database     │
    └──────────────────┘

### Authentication Flow

    ┌──────────────────┐
    │      Client      │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ Spring Security  │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ UserDetailsService│
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │  UserRepository  │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ PasswordEncoder  │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ Authentication   │
    │     Success      │
    └──────────────────┘

---

## Summary

User Management is responsible for managing the lifecycle of users in an application.

Spring Boot provides the tools needed to implement a complete User Management system using:

- `User`
- `UserRepository`
- `UserService`
- `UserController`
- `PasswordEncoder`
- `UserDetailsService`
- `SecurityFilterChain`
- `AuthenticationManager`

The main idea is:

    User Management
        │
        ├── Create user
        ├── Store user
        ├── Update user
        ├── Delete user
        ├── Enable / Disable user
        └── Manage roles
        │
        ▼
    Spring Security
        │
        ├── Authenticate user
        └── Authorize user

**User Management manages the user.**

**Authentication verifies the user.**

**Authorization controls what the user can do.**
