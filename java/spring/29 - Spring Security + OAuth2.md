# 29 - Spring Security + OAuth 2

## 1. Introduction

OAuth 2.0 is an authorization framework that allows an application to access protected resources on behalf of a user, without requiring the application to know the user's password.

In a Spring Boot application, OAuth 2.0 can be used to:

- Authenticate users through an external identity provider.
- Allow users to log in with Google, GitHub, or another provider.
- Protect APIs using access tokens.
- Delegate access to resources.
- Implement Single Sign-On (SSO).

The goal is to understand how OAuth 2.0 works and how to implement it using Spring Security.

---

## 2. What is OAuth 2.0?

**OAuth 2.0** is an authorization framework that allows a client application to obtain limited access to protected resources.

Instead of sharing a user's password with every application, the user authorizes an application to access specific resources.

### Example

Imagine an application called **Photo App** that wants to access your Google Photos.

Without OAuth:

    Photo App asks for your Google password
           ↓
    Photo App stores your password
           ↓
    Photo App accesses your photos

This is insecure.

With OAuth 2.0:

    Photo App asks Google for permission
           ↓
    User authenticates with Google
           ↓
    User grants permission
           ↓
    Google issues an access token
           ↓
    Photo App uses the token to access photos

The application never needs to know the user's Google password.

---

## 3. OAuth 2.0 vs Authentication

OAuth 2.0 was originally designed for **authorization**, not authentication.

### Authorization

Answers:

> What is this application allowed to access?

Example:

    Photo App → Can read Google Photos

### Authentication

Answers:

> Who is the user?

Example:

    User → Bruno Lima

### OpenID Connect

**OpenID Connect (OIDC)** is an authentication protocol built on top of OAuth 2.0.

It adds an **ID Token** that contains information about the authenticated user.

Therefore:

    OAuth 2.0 → Authorization
    OpenID Connect → Authentication + Authorization

When you use:

    Login with Google

You are typically using **OpenID Connect**, not OAuth 2.0 alone.

---

## 4. Example of OAuth 2.0

Imagine a user wants to connect a calendar application to their Google account.

### Without OAuth

    Calendar App
         ↓
    "Enter your Google password"
         ↓
    Calendar App accesses your account

### With OAuth 2.0

    Calendar App
         ↓
    Redirect user to Google
         ↓
    User logs in to Google
         ↓
    User authorizes Calendar App
         ↓
    Google redirects back with an authorization code
         ↓
    Calendar App exchanges code for tokens
         ↓
    Calendar App accesses the calendar API

The important idea is:

> The application receives permission to access resources, not the user's password.

---

## 5. Roles in OAuth 2.0 Architecture

OAuth 2.0 defines four main roles.

### 5.1 Resource Owner

The **Resource Owner** is the person or entity that owns the protected resources.

Example:

    User Bruno
         ↓
    Owns Google Photos

The resource owner grants permission to the client.

---

### 5.2 Client

The **Client** is the application requesting access to protected resources.

Examples:

- Web application.
- Mobile application.
- Backend application.
- SPA (Single Page Application).

Example:

    Photo App
         ↓
    Requests access to Google Photos

---

### 5.3 Authorization Server

The **Authorization Server** authenticates the resource owner and issues tokens to the client.

Responsibilities:

- Authenticate the user.
- Ask for consent.
- Validate authorization requests.
- Issue authorization codes.
- Issue access tokens.
- Issue refresh tokens.

Examples:

- Google Identity.
- Okta.
- Microsoft Entra ID.
- Keycloak.
- Auth0.

---

### 5.4 Resource Server

The **Resource Server** hosts the protected resources.

Examples:

- Google Photos API.
- Calendar API.
- Your Spring Boot REST API.

The resource server validates access tokens before allowing access.

---

## 6. OAuth 2.0 Architecture

    +-------------------+
    |   Resource Owner  |
    |       User        |
    +---------+---------+
              |
              | 1. Authorizes
              v
    +---------+---------+
    |      Client       |
    |   Photo App       |
    +---------+---------+
              |
              | 2. Authorization Request
              v
    +---------+---------+
    | Authorization     |
    | Server            |
    | Google / Okta     |
    +---------+---------+
              |
              | 3. Access Token
              v
    +---------+---------+
    | Resource Server   |
    | Google Photos API |
    +-------------------+

---

## 7. OAuth 2.0 Process Flow

The most common OAuth 2.0 flow is the **Authorization Code Flow**.

### Step 1 — Client Requests Authorization

The client redirects the user to the authorization server.

Example:

    https://authorization-server.com/authorize
        ?client_id=photo-app
        &response_type=code
        &redirect_uri=https://photo-app.com/login/oauth2/code/provider
        &scope=photos.read

The request tells the authorization server:

- Which application is requesting access.
- Which response type is expected.
- Where to redirect the user.
- Which permissions are requested.

---

### Step 2 — User Authenticates

The user logs in to the authorization server.

Example:

    User
      ↓
    Google Login Page
      ↓
    Username + Password
      ↓
    Authentication Successful

The client application does not receive the user's password.

---

### Step 3 — User Grants Consent

The authorization server asks the user whether the client can access the requested resources.

Example:

    Photo App wants to:
    - View your photos

    [Allow] [Deny]

If the user approves, the authorization server continues.

---

### Step 4 — Authorization Server Returns an Authorization Code

The authorization server redirects the user back to the client.

Example:

    https://photo-app.com/login/oauth2/code/provider
        ?code=abc123
        &state=xyz789

The **authorization code** is temporary and is not the access token.

---

### Step 5 — Client Exchanges the Code for Tokens

The client sends the authorization code to the token endpoint.

Example:

    POST /oauth2/token

    grant_type=authorization_code
    code=abc123
    redirect_uri=https://photo-app.com/login/oauth2/code/provider
    client_id=photo-app
    client_secret=client-secret

The authorization server validates the request.

---

### Step 6 — Authorization Server Returns Tokens

Example response:

    {
      "access_token": "eyJhbGciOiJSUzI1NiIs...",
      "token_type": "Bearer",
      "expires_in": 3600,
      "refresh_token": "def456",
      "scope": "photos.read"
    }

The client can now use the access token to access protected resources.

---

### Step 7 — Client Accesses the Resource Server

The client sends the access token in the HTTP Authorization header.

    GET /photos

    Authorization: Bearer eyJhbGciOiJSUzI1NiIs...

The resource server validates the token.

---

### Step 8 — Resource Server Returns the Resource

If the token is valid and contains the required permissions:

    HTTP/1.1 200 OK

    [
      {
        "id": 1,
        "name": "Vacation.jpg"
      }
    ]

---

## 8. Complete OAuth 2.0 Flow

    +-------------------+
    |       User        |
    +---------+---------+
              |
              | 1. Login / Authorization Request
              v
    +---------+---------+
    |      Client       |
    |   Photo App       |
    +---------+---------+
              |
              | 2. Redirect to Authorization Server
              v
    +---------+---------+
    | Authorization     |
    | Server            |
    +---------+---------+
              |
              | 3. User Authenticates
              |
              | 4. User Grants Consent
              |
              | 5. Authorization Code
              v
    +---------+---------+
    |      Client       |
    +---------+---------+
              |
              | 6. Exchange Code for Tokens
              v
    +---------+---------+
    | Authorization     |
    | Server            |
    +---------+---------+
              |
              | 7. Access Token
              v
    +---------+---------+
    |      Client       |
    +---------+---------+
              |
              | 8. Bearer Token
              v
    +---------+---------+
    | Resource Server   |
    +---------+---------+
              |
              | 9. Protected Resource
              v
    +-------------------+
    |       User        |
    +-------------------+

---

## 9. OAuth 2.0 Tokens

OAuth 2.0 commonly uses two types of tokens.

### 9.1 Access Token

The **Access Token** is used to access protected resources.

Example:

    GET /api/profile

    Authorization: Bearer eyJhbGciOiJSUzI1NiIs...

The access token usually has a limited lifetime.

Example:

    expires_in: 3600

This means the token expires after 3600 seconds.

---

### 9.2 Refresh Token

The **Refresh Token** is used to obtain a new access token without requiring the user to authenticate again.

Example:

    POST /oauth2/token

    grant_type=refresh_token
    refresh_token=def456

The authorization server may return a new access token.

    {
      "access_token": "new-access-token",
      "token_type": "Bearer",
      "expires_in": 3600
    }

### Access Token vs Refresh Token

    Access Token
        ↓
    Access protected resources
        ↓
    Short lifetime

    Refresh Token
        ↓
    Obtain new access token
        ↓
    Longer lifetime

---

## 10. OAuth 2.0 Scopes

A **Scope** defines the permissions requested by the client.

Example:

    scope=photos.read photos.write

This means the client is requesting permission to:

- Read photos.
- Write photos.

### Example Scopes

    profile
    email
    openid
    photos.read
    photos.write
    calendar.read

Scopes are used to limit what the client can access.

Example:

    Client has photos.read
         ↓
    Can read photos
         ↓
    Cannot delete photos

---

## 11. OAuth 2.0 Grant Types

A **Grant Type** defines how the client obtains an access token.

### Authorization Code

Used when a user authorizes an application.

Example:

    Web Application
         ↓
    User Login
         ↓
    Authorization Code
         ↓
    Access Token

This is the most common flow for user authentication.

---

### Client Credentials

Used for machine-to-machine communication.

Example:

    Service A
         ↓
    Requests token
         ↓
    Authorization Server
         ↓
    Access Token
         ↓
    Service B

There is no user involved.

Example:

    Order Service → Inventory Service

---

### Refresh Token

Used to obtain a new access token.

Example:

    Refresh Token
         ↓
    Authorization Server
         ↓
    New Access Token

---

### Resource Owner Password Credentials

This flow allowed clients to collect the user's username and password directly.

It is **deprecated** and should not be used for new applications.

---

## 12. Authorization Code Flow vs Client Credentials

### Authorization Code

Used when a user is involved.

    User → Client → Authorization Server → Resource Server

Example:

    User logs in to a web application.

### Client Credentials

Used when one service calls another service.

    Service A → Authorization Server → Service B

Example:

    Payment Service → Order Service

---

## 13. OAuth 2.0 and OpenID Connect

OAuth 2.0 alone does not define a standard way to identify the user.

OpenID Connect adds:

- **ID Token**.
- **UserInfo Endpoint**.
- Standard identity claims.
- Authentication flow.

### Example

    User logs in with Google
           ↓
    Google authenticates the user
           ↓
    Google returns ID Token
           ↓
    Application identifies the user

Example ID Token claims:

    {
      "iss": "https://accounts.google.com",
      "sub": "123456789",
      "aud": "client-id",
      "email": "user@example.com",
      "name": "John"
    }

The `sub` claim is the unique identifier of the user at the identity provider.

---

## 14. Spring Security + OAuth 2.0

Spring Security supports OAuth 2.0 in different roles.

### OAuth2 Client

Used when your application wants to access another service using OAuth 2.0.

Example:

    Spring Boot Application
           ↓
    Google OAuth 2.0
           ↓
    Google API

### OAuth2 Resource Server

Used when your application exposes protected APIs and validates access tokens.

Example:

    Client
       ↓
    Bearer Token
       ↓
    Spring Boot API
       ↓
    Validate Token
       ↓
    Access Granted

### OAuth2 Authorization Server

Used when your application acts as an authorization server.

Example:

    Client Application
           ↓
    Your Authorization Server
           ↓
    Access Token
           ↓
    Protected API

---

## 15. Example — Login with Google

Imagine a Spring Boot application that allows users to log in with Google.

### Architecture

    +-------------------+
    |       User        |
    +---------+---------+
              |
              | 1. Click "Login with Google"
              v
    +---------+---------+
    |  Spring Boot App  |
    |   OAuth2 Client   |
    +---------+---------+
              |
              | 2. Redirect
              v
    +---------+---------+
    |  Google OAuth 2.0 |
    | Authorization     |
    | Server            |
    +---------+---------+
              |
              | 3. Login + Consent
              |
              | 4. Authorization Code
              v
    +---------+---------+
    |  Spring Boot App  |
    +---------+---------+
              |
              | 5. Exchange Code
              v
    +---------+---------+
    |  Google OAuth 2.0 |
    +---------+---------+
              |
              | 6. ID Token + Access Token
              v
    +---------+---------+
    |  Spring Boot App  |
    +-------------------+

---

## 16. Implementing OAuth2 Login in Spring Boot

The following example uses Spring Boot and Spring Security to implement OAuth2 Login with Google.

### 16.1 Create the Project

Create a Spring Boot project with:

- Spring Web.
- Spring Security.
- OAuth2 Client.

### Maven Dependencies

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
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>

---

## 17. Configure Google OAuth2 Client

First, create OAuth 2.0 credentials in Google Cloud Console.

You will need:

- Client ID.
- Client Secret.
- Redirect URI.

Example redirect URI:

    http://localhost:8080/login/oauth2/code/google

The redirect URI must match the one configured in Google.

---

## 18. Configure application.properties

    spring.security.oauth2.client.registration.google.client-id=YOUR_CLIENT_ID

    spring.security.oauth2.client.registration.google.client-secret=YOUR_CLIENT_SECRET

    spring.security.oauth2.client.registration.google.scope=openid,profile,email

### Explanation

    client-id
        ↓
    Identifies your application

    client-secret
        ↓
    Authenticates your application

    scope
        ↓
    Defines the requested permissions

### Important

Do not commit the client secret to a public repository.

For production, use environment variables or a secrets manager.

---

## 19. Create the Security Configuration

Create a class called `SecurityConfig`.

    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        @Bean
        SecurityFilterChain securityFilterChain(
                HttpSecurity http) throws Exception {

            http
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/", "/error").permitAll()
                    .anyRequest().authenticated()
                )
                .oauth2Login(Customizer.withDefaults());

            return http.build();
        }
    }

### Explanation

    @Configuration
        ↓
    Defines a Spring configuration class

    @EnableWebSecurity
        ↓
    Enables Spring Security

    authorizeHttpRequests()
        ↓
    Defines authorization rules

    permitAll()
        ↓
    Allows public access

    anyRequest().authenticated()
        ↓
    Requires authentication

    oauth2Login()
        ↓
    Enables OAuth2 Login

---

## 20. Create a Controller

Create a controller with a protected endpoint.

    @RestController
    public class UserController {

        @GetMapping("/")
        public String home() {
            return "Welcome to the application";
        }

        @GetMapping("/profile")
        public String profile(Authentication authentication) {
            return "Hello, " + authentication.getName();
        }
    }

### Explanation

The `/profile` endpoint requires authentication.

If the user is not authenticated:

    GET /profile
         ↓
    Spring Security
         ↓
    Redirect to Google Login

If the user is authenticated:

    GET /profile
         ↓
    Spring Security
         ↓
    Controller
         ↓
    Hello, user

---

## 21. What Happens When the User Accesses /profile?

### Step 1

The user accesses:

    http://localhost:8080/profile

### Step 2

Spring Security checks whether the user is authenticated.

    Is user authenticated?
          ↓
         No

### Step 3

Spring Security redirects the user to Google.

    /oauth2/authorization/google

### Step 4

Google authenticates the user.

### Step 5

Google redirects the user back to:

    /login/oauth2/code/google

### Step 6

Spring Security exchanges the authorization code for tokens.

### Step 7

Spring Security creates an authenticated session.

### Step 8

The user is redirected to `/profile`.

### Step 9

The controller returns:

    Hello, user

---

## 22. OAuth2 Login Flow in Spring Security

    User
      |
      | GET /profile
      v
    Spring Security
      |
      | User not authenticated
      v
    /oauth2/authorization/google
      |
      | Redirect
      v
    Google
      |
      | Login + Consent
      v
    Google
      |
      | Authorization Code
      v
    /login/oauth2/code/google
      |
      | Exchange Code
      v
    Google Token Endpoint
      |
      | ID Token + Access Token
      v
    Spring Security
      |
      | Create Authentication
      v
    SecurityContext
      |
      | Authenticated Request
      v
    Controller
      |
      | Response
      v
    User

---

## 23. Accessing User Information

After authentication, Spring Security provides the authenticated user.

Example:

    @GetMapping("/user")
    public Map<String, Object> user(
            @AuthenticationPrincipal OAuth2User oauth2User) {

        return oauth2User.getAttributes();
    }

The `OAuth2User` contains attributes returned by the provider.

Example:

    {
      "sub": "123456789",
      "name": "John Doe",
      "email": "john@example.com",
      "picture": "https://example.com/photo.jpg"
    }

### Important

The available attributes depend on the identity provider.

For example:

- Google provides `email`, `name`, and `picture`.
- GitHub may provide `login`, `name`, and `avatar_url`.

---

## 24. Example — Display the Authenticated User

    @RestController
    public class UserController {

        @GetMapping("/user")
        public Map<String, Object> user(
                @AuthenticationPrincipal OAuth2User oauth2User) {

            return Map.of(
                "name", oauth2User.getAttribute("name"),
                "email", oauth2User.getAttribute("email")
            );
        }
    }

### Example Response

    {
      "name": "John Doe",
      "email": "john@example.com"
    }

---

## 25. OAuth2 Client vs Resource Server

These are two different use cases.

### OAuth2 Client

Your application logs in users through an external provider.

    User
       ↓
    Google
       ↓
    Spring Boot Application

Example:

    Login with Google

### Resource Server

Your application receives access tokens and protects APIs.

    Client
       ↓
    Access Token
       ↓
    Spring Boot API

Example:

    GET /api/orders
    Authorization: Bearer <token>

---

## 26. Example — Spring Boot as OAuth2 Resource Server

Now imagine a different application.

A frontend application obtains an access token from an authorization server and calls your Spring Boot API.

### Architecture

    +-------------------+
    |   Frontend App    |
    +---------+---------+
              |
              | Authorization: Bearer <token>
              v
    +---------+---------+
    | Spring Boot API   |
    |  Resource Server  |
    +---------+---------+
              |
              | Validate Token
              v
    +---------+---------+
    | Authorization     |
    | Server            |
    +-------------------+

---

## 27. Add Resource Server Dependency

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>

---

## 28. Configure JWT Validation

Example using an authorization server that exposes a JWK Set URI.

    spring.security.oauth2.resourceserver.jwt.jwk-set-uri=https://auth.example.com/oauth2/jwks

The resource server uses the public keys to validate JWT signatures.

### Important

The exact configuration depends on the authorization server.

---

## 29. Configure Security for Resource Server

    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        @Bean
        SecurityFilterChain securityFilterChain(
                HttpSecurity http) throws Exception {

            http
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/public").permitAll()
                    .anyRequest().authenticated()
                )
                .oauth2ResourceServer(oauth2 -> oauth2
                    .jwt(Customizer.withDefaults())
                );

            return http.build();
        }
    }

### Explanation

    oauth2ResourceServer()
        ↓
    Configures the application as a Resource Server

    jwt()
        ↓
    Configures JWT-based authentication

    anyRequest().authenticated()
        ↓
    Requires a valid token

---

## 30. Create a Protected API

    @RestController
    @RequestMapping("/api")
    public class OrderController {

        @GetMapping("/orders")
        public String orders() {
            return "Protected orders";
        }
    }

### Request

    GET /api/orders

    Authorization: Bearer eyJhbGciOiJSUzI1NiIs...

### Response

    Protected orders

If the token is missing or invalid:

    HTTP/1.1 401 Unauthorized

---

## 31. OAuth2 Resource Server Flow

    Client
       |
       | 1. Sends Bearer Token
       v
    Spring Boot Resource Server
       |
       | 2. Extract Token
       v
    JWT Decoder
       |
       | 3. Validate Signature
       |
       | 4. Validate Claims
       v
    SecurityContext
       |
       | 5. Authenticated Request
       v
    Controller
       |
       | 6. Response
       v
    Client

---

## 32. OAuth2 + JWT

OAuth 2.0 defines how tokens are obtained and used.

JWT defines a token format.

Therefore:

    OAuth 2.0
        ↓
    Authorization Framework

    JWT
        ↓
    Token Format

OAuth 2.0 can use different token formats:

- JWT.
- Opaque tokens.

### Example

    OAuth 2.0 Authorization Server
           ↓
    Issues JWT Access Token
           ↓
    Spring Boot Resource Server
           ↓
    Validates JWT

---

## 33. JWT Access Token Example

A JWT contains three parts:

    Header.Payload.Signature

Example:

    eyJhbGciOiJSUzI1NiIs...
    .
    eyJzdWIiOiIxMjM0NTY3ODkiLCJzY29wZSI6InJlYWQifQ
    .
    signature

Example claims:

    {
      "iss": "https://auth.example.com",
      "sub": "123456789",
      "aud": "orders-api",
      "scope": "orders.read",
      "exp": 1735689600
    }

### Important Claims

    iss
        ↓
    Issuer

    sub
        ↓
    Subject

    aud
        ↓
    Audience

    scope
        ↓
    Permissions

    exp
        ↓
    Expiration Time

---

## 34. OAuth2 Scopes and Spring Security Authorities

OAuth2 scopes can be converted into Spring Security authorities.

Example token:

    {
      "scope": "orders.read orders.write"
    }

Spring Security may represent them as:

    SCOPE_orders.read
    SCOPE_orders.write

### Example

    @GetMapping("/orders")
    @PreAuthorize("hasAuthority('SCOPE_orders.read')")
    public String orders() {
        return "Orders";
    }

The user must have the `orders.read` scope.

---

## 35. Example — Role-Based Authorization with OAuth2

OAuth2 scopes and application roles are different concepts.

### Scope

Defines what the client is allowed to access.

Example:

    orders.read

### Role

Defines what the user is allowed to do inside the application.

Example:

    ROLE_ADMIN

### Example

    User
       ↓
    ROLE_ADMIN
       ↓
    Can access administrative resources

The authorization server may provide roles or groups as claims, but your application must configure how those claims map to Spring Security authorities.

---

## 36. OAuth2 Login vs Basic Authentication

### Basic Authentication

    Client
       ↓
    Username + Password
       ↓
    Spring Boot

### OAuth2 Login

    Client
       ↓
    Redirect to Identity Provider
       ↓
    User Authenticates
       ↓
    Authorization Code
       ↓
    Tokens
       ↓
    Spring Boot

### Main Difference

Basic Authentication sends credentials directly to the application.

OAuth2 delegates authentication and authorization to an authorization server.

---

## 37. OAuth2 Login vs JWT Authentication

### OAuth2 Login

Used when a user logs in through an external provider.

Example:

    Login with Google

### JWT Authentication

Used when a client sends a JWT to a protected API.

Example:

    Authorization: Bearer <jwt>

### They Can Work Together

    User
       ↓
    Login with Google
       ↓
    Spring Boot OAuth2 Client
       ↓
    Application Session
       ↓
    Protected API

Or:

    User
       ↓
    Login with Identity Provider
       ↓
    Access Token
       ↓
    Spring Boot Resource Server
       ↓
    Protected API

---

## 38. Common OAuth2 Endpoints

### Authorization Endpoint

Used to start the authorization flow.

    /oauth2/authorize

### Token Endpoint

Used to exchange an authorization code for tokens.

    /oauth2/token

### UserInfo Endpoint

Used to retrieve user information in OpenID Connect.

    /userinfo

### JWK Set Endpoint

Used by resource servers to obtain public keys for JWT validation.

    /oauth2/jwks

### Revocation Endpoint

Used to revoke tokens.

    /oauth2/revoke

### Introspection Endpoint

Used to validate opaque tokens.

    /oauth2/introspect

---

## 39. Common OAuth2 Errors

### invalid_client

The client credentials are invalid.

Example:

    Client ID or Client Secret is incorrect.

### invalid_grant

The authorization code is invalid, expired, or already used.

### invalid_scope

The requested scope is invalid or not allowed.

### unauthorized_client

The client is not authorized to use the requested grant type.

### access_denied

The user denied the authorization request.

---

## 40. OAuth2 Security Best Practices

### Use HTTPS

OAuth2 tokens must be transmitted over HTTPS.

### Use Authorization Code Flow

For user-based applications, use Authorization Code Flow with PKCE when appropriate.

### Protect Client Secrets

Never expose client secrets in frontend applications.

### Use Short-Lived Access Tokens

Access tokens should have limited lifetimes.

### Validate Token Claims

Validate:

- Signature.
- Issuer.
- Audience.
- Expiration.
- Not-before time.

### Use Least Privilege

Request only the scopes the application needs.

### Protect Refresh Tokens

Refresh tokens must be stored securely.

### Validate Redirect URIs

Use exact registered redirect URIs.

---

## 41. OAuth2 Authorization Code Flow with PKCE

**PKCE** stands for Proof Key for Code Exchange.

It adds protection against authorization code interception.

### Flow

    Client generates code_verifier
           ↓
    Client generates code_challenge
           ↓
    Client sends code_challenge
           ↓
    Authorization Server
           ↓
    Authorization Code
           ↓
    Client sends code_verifier
           ↓
    Authorization Server validates PKCE
           ↓
    Access Token

PKCE is especially important for public clients such as mobile applications and SPAs.

---

## 42. OAuth2 Example — Mobile Application

Imagine a mobile application that allows users to access their account.

### Flow

    Mobile App
       ↓
    Open System Browser
       ↓
    Identity Provider
       ↓
    User Login
       ↓
    Authorization Code
       ↓
    Mobile App
       ↓
    Access Token
       ↓
    API

The mobile application does not need to collect the user's password.

---

## 43. OAuth2 Example — Machine-to-Machine

Imagine an Order Service calling an Inventory Service.

### Flow

    Order Service
       ↓
    Client Credentials
       ↓
    Authorization Server
       ↓
    Access Token
       ↓
    Inventory Service
       ↓
    Protected Resource

There is no user login.

The service authenticates using its own credentials.

---

## 44. OAuth2 Example — Single Sign-On

Imagine a company has multiple applications:

- HR Application.
- Payroll Application.
- Internal Portal.

All applications use the same identity provider.

### Flow

    User
       ↓
    Login to Internal Portal
       ↓
    Identity Provider
       ↓
    Authenticated
       ↓
    Access HR Application
       ↓
    Access Payroll Application

The user does not need to log in separately to every application.

This is the idea of **Single Sign-On (SSO)**.

---

## 45. Important OAuth2 Terminology

| Term                 | Meaning                                               |
| -------------------- | ----------------------------------------------------- |
| Resource Owner       | Entity that owns the protected resources              |
| Client               | Application requesting access                         |
| Authorization Server | Issues tokens                                         |
| Resource Server      | Hosts protected resources                             |
| Access Token         | Token used to access resources                        |
| Refresh Token        | Token used to obtain a new access token               |
| Authorization Code   | Temporary code exchanged for tokens                   |
| Scope                | Permission requested by the client                    |
| Grant Type           | Method used to obtain a token                         |
| Redirect URI         | URL where the authorization server redirects the user |
| ID Token             | Token containing authentication information           |
| PKCE                 | Protection mechanism for authorization code flow      |
| JWT                  | Token format                                          |
| OIDC                 | Authentication protocol built on OAuth 2.0            |

---

## 46. OAuth2 Interview Questions

### What is OAuth 2.0?

OAuth 2.0 is an authorization framework that allows applications to access protected resources without sharing the user's password.

### What is the difference between OAuth2 and OpenID Connect?

OAuth2 is used for authorization. OpenID Connect adds authentication capabilities.

### What is the difference between an access token and a refresh token?

An access token accesses protected resources. A refresh token obtains a new access token.

### What is the difference between an authorization code and an access token?

An authorization code is temporary and exchanged for tokens. An access token is used to access protected resources.

### What is the difference between OAuth2 Client and Resource Server?

An OAuth2 Client obtains tokens to access resources. A Resource Server validates tokens and protects APIs.

### What is the difference between OAuth2 and JWT?

OAuth2 is an authorization framework. JWT is a token format.

### What is PKCE?

PKCE protects the authorization code flow by requiring the client to prove that it initiated the authorization request.

### What is the purpose of scopes?

Scopes define the permissions requested by the client.

### What is the purpose of the redirect URI?

It defines where the authorization server sends the user after authorization.

---

## 47. Summary

OAuth 2.0 is an authorization framework that allows applications to access protected resources without sharing user passwords.

The main roles are:

    Resource Owner
    Client
    Authorization Server
    Resource Server

The most common user-based flow is:

    Authorization Code Flow

The main tokens are:

    Access Token
    Refresh Token

For authentication, OAuth2 is commonly combined with:

    OpenID Connect

In Spring Security, OAuth2 can be used to:

- Implement Login with Google.
- Implement Single Sign-On.
- Access external APIs.
- Protect REST APIs.
- Validate JWT access tokens.
- Implement machine-to-machine authentication.

The most important distinction is:

    OAuth2 Client
        ↓
    Obtains tokens

    OAuth2 Resource Server
        ↓
    Validates tokens

    OAuth2 Authorization Server
        ↓
    Issues tokens

Understanding these three roles is essential for designing secure Spring Boot applications.

---

## 48. Practical Exercise

Implement a Spring Boot application with the following requirements:

### Part 1 — OAuth2 Login

- Configure Google OAuth2.
- Create a protected `/profile` endpoint.
- Display the authenticated user's name and email.
- Allow public access to `/`.

### Part 2 — Resource Server

- Configure JWT validation.
- Create a protected `/api/orders` endpoint.
- Require authentication.
- Return `401 Unauthorized` when the token is missing or invalid.

### Part 3 — Authorization

- Create a scope called `orders.read`.
- Allow only users with `SCOPE_orders.read` to access `/api/orders`.

### Part 4 — Questions

1. What is the difference between OAuth2 and OpenID Connect?
2. What is the difference between an access token and a refresh token?
3. What is the role of the authorization server?
4. What is the role of the resource server?
5. What is the difference between OAuth2 Client and Resource Server?
6. What is PKCE?
7. What is the purpose of scopes?
8. Why should client secrets never be exposed in frontend applications?
