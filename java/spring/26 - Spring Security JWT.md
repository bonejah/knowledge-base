# 26 - Spring Security JWT

# JWT (JSON Web Token)

## What is JWT?

**JWT (JSON Web Token)** is an open standard used to securely transmit information between parties as a JSON object.

In the context of authentication, JWT is commonly used to represent the identity and permissions of a user after they successfully log in.

A JWT is **digitally signed**, allowing the recipient to verify that the token has not been modified.

### JWT Characteristics

- **Compact:** Small enough to be sent in HTTP headers.
- **Self-contained:** Contains claims about the user and the token.
- **Stateless:** The server can validate the token without storing a session.
- **Signed:** Protects the integrity of the token.
- **Not encrypted by default:** The payload can be decoded, so sensitive information should not be stored in it.

> **Important:** A JWT is not the same as encryption. Signing proves that the token was issued by a trusted party and has not been altered.

---

## Why JWT?

JWT is commonly used to implement **stateless authentication** in modern applications.

### Traditional Session-Based Authentication

In session-based authentication:

1. The user logs in.
2. The server creates a session.
3. The server stores the session.
4. The client receives a session ID.
5. The client sends the session ID with subsequent requests.
6. The server looks up the session to identify the user.

### JWT-Based Authentication

With JWT:

1. The user logs in.
2. The server validates the credentials.
3. The server generates a JWT.
4. The client receives the token.
5. The client sends the token with subsequent requests.
6. The server validates the token and identifies the user.

The server does not need to store a session for every authenticated user.

### Advantages of JWT

- **Stateless authentication**
- **Scalability**
- **Suitable for REST APIs**
- **Works well with microservices**
- **Can carry claims such as user ID and roles**
- **Useful for distributed systems**

### Important Considerations

JWT is not automatically better than session-based authentication.

It introduces challenges such as:

- Token expiration
- Token revocation
- Secure storage
- Refresh token management
- Token theft

> **Remember:** JWT is a token format. It is not an authentication protocol by itself.

---

## JWT Token Format

A JWT consists of **three parts**, separated by dots (`.`):

**Header.Payload.Signature**

Example:

    eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJyb2xlIjoiVVNFUiJ9.signature

### 1. Header

The header contains metadata about the token.

Example:

    {
      "alg": "HS256",
      "typ": "JWT"
    }

Common fields:

- **alg:** Algorithm used to sign the token.
- **typ:** Token type.

### 2. Payload

The payload contains **claims**.

Example:

    {
      "sub": "123",
      "name": "John",
      "role": "USER",
      "iat": 1690000000,
      "exp": 1690003600
    }

Common claims:

| Claim | Meaning                      |
| ----- | ---------------------------- |
| `sub` | Subject, usually the user ID |
| `iss` | Issuer                       |
| `aud` | Audience                     |
| `iat` | Issued at                    |
| `exp` | Expiration time              |
| `nbf` | Not valid before             |
| `jti` | Unique token identifier      |

Custom claims can also be included, such as:

- `role`
- `permissions`
- `email`

### 3. Signature

The signature is used to verify the token's integrity.

For an HMAC-based JWT:

    HMACSHA256(
      base64UrlEncode(header) + "." +
      base64UrlEncode(payload),
      secret
    )

The signature allows the server to verify that the token was created by a trusted issuer and has not been modified.

### JWT Structure

    Header.Payload.Signature
       ↓       ↓       ↓
    Metadata  Claims  Integrity

### Important

The header and payload are **Base64URL-encoded**, not encrypted.

Anyone who has the token can decode the payload.

---

## JWT Flow in Authentication

### Authentication Flow

    Client                    Authentication Server
       |                              |
       |  1. Login credentials        |
       |----------------------------->|
       |                              |
       |  2. Validate credentials     |
       |                              |
       |  3. Generate JWT             |
       |                              |
       |  4. Return JWT               |
       |<-----------------------------|
       |                              |
       |  5. Store token              |
       |                              |

### Accessing a Protected Resource

    Client                    Resource Server
       |                              |
       |  1. Request + JWT            |
       |----------------------------->|
       |                              |
       |  2. Validate JWT             |
       |                              |
       |  3. Extract user identity    |
       |                              |
       |  4. Check authorization      |
       |                              |
       |  5. Return protected data    |
       |<-----------------------------|

### Step-by-Step

**1. User logs in**

The client sends credentials to the authentication server.

**2. Credentials are validated**

The server verifies the username and password.

**3. JWT is generated**

The server creates a signed JWT containing claims about the user.

**4. JWT is returned**

The client receives the token.

**5. Client sends the JWT**

The client includes the token in the `Authorization` header.

    Authorization: Bearer <JWT>

**6. Server validates the JWT**

The server checks:

- Signature
- Expiration
- Issuer
- Audience
- Other required claims

**7. Server authenticates the user**

If the token is valid, the server identifies the user.

**8. Server authorizes the request**

The server checks whether the user has permission to access the requested resource.

**9. Protected resource is returned**

If authentication and authorization succeed, the server returns the requested data.

### Authentication vs Authorization

**Authentication:** Who are you?

**Authorization:** What are you allowed to do?

Example:

    JWT → User ID: 123
           Role: USER

    Authentication → User 123 is authenticated.

    Authorization → User 123 can access their profile.

---

# JWT Authentication in Spring Boot

A common implementation using Spring Security follows this architecture:

    POST /authenticate
           ↓
    Username + Password
           ↓
    AuthenticationManager
           ↓
    Validate credentials
           ↓
    Generate JWT
           ↓
    Return JWT
           ↓
    Client sends JWT
           ↓
    Authorization: Bearer <JWT>
           ↓
    JWT Authentication Filter
           ↓
    Validate JWT
           ↓
    Set SecurityContext
           ↓
    Protected Endpoint

---

## Project Structure

A simple implementation can be organized as:

    src/main/java/com/example/security

    ├── config
    │   └── SecurityConfig.java
    │
    ├── controller
    │   ├── AuthController.java
    │   └── UserController.java
    │
    ├── filter
    │   └── JwtAuthenticationFilter.java
    │
    └── service
        └── JwtService.java

---

## 1. Maven Dependencies

Add Spring Security and JWT dependencies to `pom.xml`.

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.6</version>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>

---

## 2. JWT Service

The `JwtService` is responsible for creating and validating JWT tokens.

    @Service
    public class JwtService {

        private final String secretKey =
                "my-super-secret-key-my-super-secret-key";

        public String generateToken(UserDetails userDetails) {

            return Jwts.builder()
                    .subject(userDetails.getUsername())
                    .issuedAt(new Date())
                    .expiration(
                        new Date(System.currentTimeMillis() + 1000 * 60 * 60)
                    )
                    .signWith(getSigningKey())
                    .compact();
        }

        private SecretKey getSigningKey() {

            return Keys.hmacShaKeyFor(
                    secretKey.getBytes(StandardCharsets.UTF_8)
            );
        }

        public String extractUsername(String token) {

            return Jwts.parser()
                    .verifyWith(getSigningKey())
                    .build()
                    .parseSignedClaims(token)
                    .getPayload()
                    .getSubject();
        }

        public boolean isTokenValid(
                String token,
                UserDetails userDetails) {

            String username = extractUsername(token);

            return username.equals(userDetails.getUsername())
                    && !isTokenExpired(token);
        }

        private boolean isTokenExpired(String token) {

            Date expiration = Jwts.parser()
                    .verifyWith(getSigningKey())
                    .build()
                    .parseSignedClaims(token)
                    .getPayload()
                    .getExpiration();

            return expiration.before(new Date());
        }
    }

### What Does This Service Do?

`generateToken()`:

- Creates the JWT.
- Adds the username as the subject.
- Adds the issue date.
- Adds an expiration date.
- Signs the token.
- Returns the compact JWT string.

`extractUsername()`:

- Validates the JWT signature.
- Reads the `sub` claim.
- Returns the username.

`isTokenValid()`:

- Extracts the username.
- Checks whether it belongs to the expected user.
- Checks whether the token has expired.

---

# 3. Authentication Request

Create a DTO to receive the username and password.

    public record AuthenticationRequest(
            String username,
            String password
    ) {
    }

---

# 4. Authentication Controller

The `/authenticate` endpoint is responsible for authenticating the user and generating the JWT.

    @RestController
    @RequestMapping("/auth")
    public class AuthController {

        private final AuthenticationManager authenticationManager;
        private final UserDetailsService userDetailsService;
        private final JwtService jwtService;

        public AuthController(
                AuthenticationManager authenticationManager,
                UserDetailsService userDetailsService,
                JwtService jwtService) {

            this.authenticationManager = authenticationManager;
            this.userDetailsService = userDetailsService;
            this.jwtService = jwtService;
        }

        @PostMapping("/authenticate")
        public ResponseEntity<String> authenticate(
                @RequestBody AuthenticationRequest request) {

            authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(
                            request.username(),
                            request.password()
                    )
            );

            UserDetails user =
                    userDetailsService.loadUserByUsername(
                            request.username()
                    );

            String token = jwtService.generateToken(user);

            return ResponseEntity.ok(token);
        }
    }

### Authentication Flow

When the client calls:

    POST /auth/authenticate

with:

    {
      "username": "john",
      "password": "password"
    }

The following happens:

    /authenticate
         ↓
    AuthenticationManager
         ↓
    Validate username/password
         ↓
    UserDetailsService
         ↓
    Generate JWT
         ↓
    Return JWT

Example response:

    eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIiwiZXhwIjoxNzAwMDAwMDAwfQ.signature

---

# 5. JWT Authentication Filter

The JWT filter intercepts incoming HTTP requests.

Its responsibility is to:

1.  Read the `Authorization` header.
2.  Check for the `Bearer` prefix.
3.  Extract the JWT.
4.  Validate the JWT.
5.  Load the user.
6.  Authenticate the user in Spring Security.

    @Component
    public class JwtAuthenticationFilter
    extends OncePerRequestFilter {

        private final JwtService jwtService;
        private final UserDetailsService userDetailsService;

        public JwtAuthenticationFilter(
                JwtService jwtService,
                UserDetailsService userDetailsService) {

            this.jwtService = jwtService;
            this.userDetailsService = userDetailsService;
        }

        @Override
        protected void doFilterInternal(
                HttpServletRequest request,
                HttpServletResponse response,
                FilterChain filterChain)
                throws ServletException, IOException {

            final String authHeader =
                    request.getHeader("Authorization");

            if (authHeader == null ||
                !authHeader.startsWith("Bearer ")) {

                filterChain.doFilter(request, response);
                return;
            }

            final String jwt =
                    authHeader.substring(7);

            final String username =
                    jwtService.extractUsername(jwt);

            if (username != null &&
                SecurityContextHolder
                    .getContext()
                    .getAuthentication() == null) {

                UserDetails userDetails =
                        userDetailsService
                                .loadUserByUsername(username);

                if (jwtService.isTokenValid(jwt, userDetails)) {

                    UsernamePasswordAuthenticationToken authentication =
                            new UsernamePasswordAuthenticationToken(
                                    userDetails,
                                    null,
                                    userDetails.getAuthorities()
                            );

                    authentication.setDetails(
                            new WebAuthenticationDetailsSource()
                                    .buildDetails(request)
                    );

                    SecurityContextHolder
                            .getContext()
                            .setAuthentication(authentication);
                }
            }

            filterChain.doFilter(request, response);
        }

    }

---

# 6. Security Configuration

The JWT filter needs to be registered in the Spring Security filter chain.

    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        private final JwtAuthenticationFilter jwtAuthenticationFilter;

        public SecurityConfig(
                JwtAuthenticationFilter jwtAuthenticationFilter) {

            this.jwtAuthenticationFilter = jwtAuthenticationFilter;
        }

        @Bean
        public SecurityFilterChain securityFilterChain(
                HttpSecurity http) throws Exception {

            http
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/auth/authenticate")
                    .permitAll()
                    .anyRequest()
                    .authenticated()
                )
                .sessionManagement(session ->
                    session.sessionCreationPolicy(
                        SessionCreationPolicy.STATELESS
                    )
                )
                .addFilterBefore(
                    jwtAuthenticationFilter,
                    UsernamePasswordAuthenticationFilter.class
                );

            return http.build();
        }

        @Bean
        public PasswordEncoder passwordEncoder() {

            return new BCryptPasswordEncoder();
        }
    }

### Important Configuration

The following configuration makes JWT authentication stateless:

    SessionCreationPolicy.STATELESS

This tells Spring Security that the application should not create or use an HTTP session to maintain authentication state.

Instead:

    Request
       ↓
    JWT
       ↓
    Validate JWT
       ↓
    Authenticate request

---

# 7. Protected Endpoint

Now create an endpoint that requires authentication.

    @RestController
    @RequestMapping("/api/users")
    public class UserController {

        @GetMapping("/me")
        public String getCurrentUser(
                Authentication authentication) {

            return "Hello " + authentication.getName();
        }
    }

Because the security configuration contains:

    .anyRequest().authenticated()

the endpoint requires a valid JWT.

---

# 8. Testing the Authentication

### Step 1 - Authenticate

Send:

    POST /auth/authenticate

Request body:

    {
      "username": "john",
      "password": "password"
    }

Response:

    eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIn0.signature

The client now has the JWT.

---

### Step 2 - Call Protected Endpoint

Send:

    GET /api/users/me

with:

    Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIn0.signature

---

### Step 3 - JWT Filter Processes the Request

The filter reads:

    Authorization: Bearer <JWT>

Then:

    Authorization Header
            ↓
    Extract "Bearer <JWT>"
            ↓
    Remove "Bearer "
            ↓
    Get JWT
            ↓
    Validate Signature
            ↓
    Check Expiration
            ↓
    Extract Username
            ↓
    Load UserDetails
            ↓
    Create Authentication
            ↓
    Set SecurityContext
            ↓
    Continue Request

---

### Step 4 - Access Protected Resource

If the JWT is valid:

    GET /api/users/me

returns:

    Hello john

If the JWT is missing or invalid, the request is not authenticated and Spring Security rejects access to the protected endpoint.

---

# Complete JWT Authentication Flow

The complete flow can be summarized as:

    ┌──────────────────────────┐
    │          Client          │
    └────────────┬─────────────┘
                 │
                 │ POST /auth/authenticate
                 │ username + password
                 ▼
    ┌──────────────────────────┐
    │   AuthenticationManager   │
    └────────────┬─────────────┘
                 │
                 │ Validate credentials
                 ▼
    ┌──────────────────────────┐
    │      UserDetailsService   │
    └────────────┬─────────────┘
                 │
                 │ User authenticated
                 ▼
    ┌──────────────────────────┐
    │        JwtService         │
    │      Generate JWT         │
    └────────────┬─────────────┘
                 │
                 │ JWT
                 ▼
    ┌──────────────────────────┐
    │          Client           │
    └────────────┬─────────────┘
                 │
                 │ GET /api/users/me
                 │ Authorization: Bearer JWT
                 ▼
    ┌──────────────────────────┐
    │ JwtAuthenticationFilter  │
    └────────────┬─────────────┘
                 │
                 │ Validate JWT
                 ▼
    ┌──────────────────────────┐
    │        JwtService         │
    └────────────┬─────────────┘
                 │
                 │ Valid JWT
                 ▼
    ┌──────────────────────────┐
    │    SecurityContext        │
    │       Authentication      │
    └────────────┬─────────────┘
                 │
                 ▼
    ┌──────────────────────────┐
    │    Protected Endpoint     │
    └──────────────────────────┘

---

## Key Takeaways

- `/auth/authenticate` is responsible for **authenticating credentials and generating the JWT**.
- The client sends the JWT in the `Authorization` header.
- The standard format is:

      Authorization: Bearer <JWT>

- `JwtAuthenticationFilter` intercepts incoming requests.
- The filter extracts and validates the JWT.
- If valid, it creates an `Authentication` object.
- The authentication is stored in the `SecurityContext`.
- Spring Security then considers the request authenticated.
- Protected endpoints can access the authenticated user's information through `Authentication` or `SecurityContext`.
- `SessionCreationPolicy.STATELESS` ensures that authentication is based on the JWT instead of an HTTP session.

### The Big Picture

    LOGIN
      ↓
    username + password
      ↓
    AuthenticationManager
      ↓
    JWT generated
      ↓
    Client receives JWT
      ↓
    Authorization: Bearer JWT
      ↓
    JwtAuthenticationFilter
      ↓
    Validate JWT
      ↓
    SecurityContext
      ↓
    Authenticated Request
      ↓
    Protected Resource
