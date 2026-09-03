# 25 - Spring Security Basic Authentication

## 1. Understand Basic Authentication

### What is Basic Authentication?

**Basic Authentication** is an HTTP authentication mechanism where the client sends a username and password with every request to a protected resource.

The credentials are sent using the `Authorization` HTTP header:

    Authorization: Basic <Base64(username:password)>

For example, if the username is `admin` and the password is `password123`, the client creates the string:

    admin:password123

Then encodes it using Base64:

    YWRtaW46cGFzc3dvcmQxMjM=

The request becomes:

    GET /api/users HTTP/1.1
    Host: example.com
    Authorization: Basic YWRtaW46cGFzc3dvcmQxMjM=

### How Basic Authentication works

    Client
       |
       | 1. Request protected resource
       v
    Spring Security
       |
       | 2. Authentication required
       v
    Client
       |
       | 3. Sends username + password
       |    in Authorization header
       v
    Spring Security
       |
       | 4. Decodes credentials
       | 5. Validates username and password
       v
    Authentication successful?
       |
       +---- No ----> 401 Unauthorized
       |
       +---- Yes ---> Request continues
                         |
                         v
                    Controller
                         |
                         v
                    HTTP Response

### Important characteristics

- **Stateless:** The server does not need to maintain a session for each authenticated request.
- **Credentials on every request:** The client sends the `Authorization` header each time.
- **Base64 is not encryption:** Anyone who intercepts the header can decode the credentials.
- **HTTPS is essential:** TLS protects the credentials while they are transmitted.
- **Simple to implement:** It is useful for APIs, internal services, and testing.

> **Important:** Basic Authentication should always be used over HTTPS in production.

### Basic Authentication vs Form Login

| Basic Authentication                                       | Form Login                                                |
| ---------------------------------------------------------- | --------------------------------------------------------- |
| Credentials are sent in an HTTP header                     | Credentials are submitted through a login form            |
| Commonly used for APIs                                     | Commonly used for web applications                        |
| Usually stateless                                          | Often uses a session                                      |
| Client must send credentials with each request             | Browser maintains authentication through a session cookie |
| Returns `401 Unauthorized` when authentication is required | Redirects to a login page                                 |

### Authentication vs Authorization

**Authentication** answers:

> Who are you?

Example:

    Username: admin
    Password: password123

**Authorization** answers:

> What are you allowed to do?

Example:

    admin -> Can access /admin
    user  -> Cannot access /admin

Basic Authentication is responsible for **authentication**. Spring Security can also perform **authorization** after the user has been authenticated.

---

## 2. Implement Basic Authentication

### Step 1 - Create a Spring Boot project

Create a Spring Boot project with the following dependencies:

- Spring Web
- Spring Security

For Maven, add the Spring Security starter:

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

### Step 2 - Create a protected endpoint

Create a controller with an endpoint that requires authentication.

    @RestController
    @RequestMapping("/api")
    public class HelloController {

        @GetMapping("/hello")
        public String hello() {
            return "Hello, authenticated user!";
        }
    }

By default, Spring Security protects application endpoints when the security starter is added.

### Step 3 - Configure a user

For a simple example, configure an in-memory user.

    @Configuration
    public class UserConfig {

        @Bean
        public UserDetailsService userDetailsService(
                PasswordEncoder passwordEncoder) {

            UserDetails user = User
                    .withUsername("admin")
                    .password(passwordEncoder.encode("password123"))
                    .roles("USER")
                    .build();

            return new InMemoryUserDetailsManager(user);
        }

        @Bean
        public PasswordEncoder passwordEncoder() {
            return new BCryptPasswordEncoder();
        }
    }

### Explanation

**UserDetailsService**

Responsible for loading user information, such as:

- Username
- Password
- Roles
- Authorities

**PasswordEncoder**

Responsible for encoding and verifying passwords.

In this example, `BCryptPasswordEncoder` is used to store the password securely.

**InMemoryUserDetailsManager**

Stores user information in memory.

This is useful for:

- Learning
- Testing
- Small demonstrations

For production applications, users are usually stored in a database.

### Step 4 - Configure HTTP Basic Authentication

Create a security configuration.

    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        @Bean
        public SecurityFilterChain securityFilterChain(
                HttpSecurity http) throws Exception {

            http
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/hello").authenticated()
                    .anyRequest().permitAll()
                )
                .httpBasic(Customizer.withDefaults());

            return http.build();
        }
    }

### What does this configuration do?

    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/hello").authenticated()
        .anyRequest().permitAll()
    )

Means:

- `/api/hello` requires authentication.
- Other endpoints are allowed without authentication.

  .httpBasic(Customizer.withDefaults())

Enables HTTP Basic Authentication.

When an unauthenticated client tries to access `/api/hello`, Spring Security returns a `401 Unauthorized` response.

### Step 5 - Test the endpoint

#### Request without credentials

    GET /api/hello HTTP/1.1

Response:

    HTTP/1.1 401 Unauthorized

#### Request with credentials

    GET /api/hello HTTP/1.1
    Authorization: Basic YWRtaW46cGFzc3dvcmQxMjM=

Response:

    HTTP/1.1 200 OK

    Hello, authenticated user!

### Testing with cURL

    curl -u admin:password123 http://localhost:8080/api/hello

The `-u` option sends the username and password using Basic Authentication.

### Testing with Postman

1. Open Postman.
2. Create a `GET` request.
3. Enter the URL:

   http://localhost:8080/api/hello

4. Open the **Authorization** tab.
5. Select **Basic Auth**.
6. Enter:

   Username: admin
   Password: password123

7. Click **Send**.

The request should return:

    Hello, authenticated user!

---

## 3. Understanding the Spring Security Flow

When a request arrives, Spring Security processes it through a chain of filters.

    HTTP Request
         |
         v
    SecurityFilterChain
         |
         v
    BasicAuthenticationFilter
         |
         v
    Extract Authorization header
         |
         v
    Decode Base64 credentials
         |
         v
    Authenticate user
         |
         v
    AuthenticationManager
         |
         v
    UserDetailsService
         |
         v
    PasswordEncoder
         |
         v
    Authentication successful
         |
         v
    SecurityContext
         |
         v
    Controller

### Important components

**BasicAuthenticationFilter**

Reads the `Authorization: Basic ...` header and attempts to authenticate the user.

**AuthenticationManager**

Coordinates the authentication process.

**UserDetailsService**

Loads the user information.

**PasswordEncoder**

Verifies the submitted password against the stored password.

**SecurityContext**

Stores the authenticated user's information during the request.

**SecurityFilterChain**

Defines which security filters and rules are applied to incoming requests.

---

## 4. Authentication Failure

If the username or password is incorrect, authentication fails.

    Client
       |
       | Invalid credentials
       v
    Spring Security
       |
       v
    Authentication failed
       |
       v
    401 Unauthorized

Example response:

    HTTP/1.1 401 Unauthorized

The client must send valid credentials to access the protected resource.

---

## 5. Basic Authentication with Roles

Basic Authentication can also be combined with authorization rules.

    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        @Bean
        public SecurityFilterChain securityFilterChain(
                HttpSecurity http) throws Exception {

            http
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/admin").hasRole("ADMIN")
                    .requestMatchers("/api/user").hasAnyRole("USER", "ADMIN")
                    .anyRequest().permitAll()
                )
                .httpBasic(Customizer.withDefaults());

            return http.build();
        }
    }

### Example

    User: admin
    Role: ADMIN

    /api/admin -> Allowed
    /api/user  -> Allowed

    User: john
    Role: USER

    /api/admin -> Forbidden
    /api/user  -> Allowed

Authentication verifies the identity.

Authorization verifies whether the authenticated user has permission to access the resource.

---

## 6. Basic Authentication vs JWT

| Basic Authentication                           | JWT                                          |
| ---------------------------------------------- | -------------------------------------------- |
| Sends username and password with every request | Sends a token with every request             |
| Credentials are encoded using Base64           | Token contains encoded claims                |
| Requires HTTPS to protect credentials          | Requires HTTPS to protect tokens             |
| Simple to implement                            | More complex to implement                    |
| Suitable for simple APIs and internal services | Suitable for modern distributed applications |
| Server validates credentials on each request   | Server validates the token on each request   |

> **Note:** Basic Authentication and JWT are both authentication mechanisms, but they solve the problem in different ways.

---

## 7. Important Security Considerations

### Never use plain-text passwords

Do not store passwords directly.

    password("password123")

Instead, use a password encoder.

    password(passwordEncoder.encode("password123"))

### Always use HTTPS

Basic Authentication sends credentials with every request.

Without HTTPS, an attacker could intercept the credentials.

### Avoid exposing credentials in URLs

Do not send credentials like this:

    http://admin:password123@example.com/api/hello

Use the `Authorization` header instead.

### Use strong passwords

Weak passwords can be easily guessed or attacked.

### Consider rate limiting

For public APIs, rate limiting can help reduce brute-force attacks.

### Use Basic Authentication for the right use case

Basic Authentication is useful for:

- Simple REST APIs
- Internal services
- Development and testing
- Service-to-service communication

For more complex applications, other authentication mechanisms may be more appropriate.

---

## 8. Summary

- **Basic Authentication** sends a username and password using the HTTP `Authorization` header.
- Credentials are encoded using **Base64**, not encrypted.
- **HTTPS is required** to protect credentials in transit.
- Spring Security provides **HTTP Basic Authentication** through `httpBasic()`.
- `UserDetailsService` loads user information.
- `PasswordEncoder` verifies passwords securely.
- `BasicAuthenticationFilter` processes Basic Authentication requests.
- Authentication determines **who the user is**.
- Authorization determines **what the user can access**.
- Basic Authentication is simple and useful for APIs, testing, and internal services.
