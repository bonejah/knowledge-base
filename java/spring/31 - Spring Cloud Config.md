# 31 - Spring Cloud Config

## 1. Introduction

In a real-world Spring Boot application, configuration is usually not limited to a single `application.properties` or `application.yml` file.

As the number of microservices grows, configuration becomes difficult to manage:

- Multiple microservices
- Multiple environments
- Different database configurations
- Different API URLs
- Different credentials
- Feature flags
- Environment-specific properties
- Configuration changes without redeploying applications

Spring Cloud Config provides a centralized configuration management solution for distributed systems.

The main idea is:

> Store configuration in one centralized location and allow multiple microservices to retrieve their configuration from a Config Server.

---

# 2. Real-World Project Problems

Imagine a company has the following microservices:

    user-service
    order-service
    payment-service
    notification-service

Each service has its own configuration.

For example:

    user-service
        database.url
        database.username
        jwt.expiration

    order-service
        database.url
        payment-service.url
        order.timeout

    payment-service
        stripe.api.url
        stripe.api.key
        payment.timeout

Now imagine the application is deployed into three environments:

    Development
    QA
    Production

Each environment requires different configuration.

Without centralized configuration, we may end up with:

    user-service
        application-dev.yml
        application-qa.yml
        application-prod.yml

    order-service
        application-dev.yml
        application-qa.yml
        application-prod.yml

    payment-service
        application-dev.yml
        application-qa.yml
        application-prod.yml

This creates several problems.

## Problems

### 1. Configuration duplication

The same configuration may be duplicated across multiple services.

### 2. Configuration inconsistency

One service may use:

    payment-service.url=http://payment-service:8080

while another service accidentally uses:

    payment-service.url=http://payment-service:8081

### 3. Configuration changes require redeployment

Suppose the following property changes:

    order.timeout=30

If the property is packaged inside the application, we may need to:

    Change configuration
          ↓
    Build application
          ↓
    Create Docker image
          ↓
    Deploy application
          ↓
    Restart service

This is inconvenient for configuration-only changes.

### 4. Managing multiple environments

Production, QA and development require different configurations.

Managing all of these independently becomes difficult.

### 5. Configuration management

Configuration should ideally be:

- Centralized
- Version controlled
- Environment aware
- Easily changeable
- Consistent across services

This is where Spring Cloud Config helps.

---

# 3. What is Spring Cloud Config?

Spring Cloud Config provides server-side and client-side support for externalized configuration in distributed systems.

It follows a centralized configuration architecture.

Instead of every microservice managing its configuration independently:

    Microservice
         ↓
    Local configuration

we can have:

    Git Repository
         ↓
    Config Server
         ↓
    Microservices

The Config Server acts as a centralized configuration provider.

---

# 4. Spring Cloud Config Architecture

A typical architecture looks like this:

    ┌─────────────────────┐
    │   Git Repository    │
    │                     │
    │ application.yml     │
    │ user-service.yml    │
    │ order-service.yml   │
    │ payment-service.yml │
    └──────────┬──────────┘
               │
               │ Configuration
               ↓
    ┌─────────────────────┐
    │    Config Server    │
    │   Spring Boot App   │
    └──────────┬──────────┘
               │
        ┌──────┼───────┐
        │      │       │
        ↓      ↓       ↓
    User     Order   Payment
    Service  Service Service

The components are:

### Config Repository

Usually a Git repository containing configuration files.

### Config Server

A Spring Boot application responsible for retrieving configuration from the repository.

### Config Clients

Microservices that request configuration from the Config Server.

---

# 5. Configuration Repository

A common Git repository structure could be:

    config-repo/
    │
    ├── application.yml
    ├── user-service.yml
    ├── order-service.yml
    ├── payment-service.yml
    │
    ├── user-service-dev.yml
    ├── user-service-prod.yml
    │
    ├── order-service-dev.yml
    └── order-service-prod.yml

The naming convention is important.

For example:

    user-service.yml

contains common configuration for `user-service`.

And:

    user-service-prod.yml

contains production-specific configuration.

---

# 6. Configuration File Naming Convention

Spring Cloud Config commonly follows this structure:

    {application}-{profile}.yml

For example:

    user-service-dev.yml

means:

    application = user-service
    profile = dev

Another example:

    order-service-prod.yml

means:

    application = order-service
    profile = prod

There can also be a default configuration:

    application.yml

This configuration can be shared by multiple services.

---

# 7. Example Configuration Repository

## application.yml

    company:
      name: MyCompany

    logging:
      level:
        root: INFO

## user-service.yml

    server:
      port: 8081

    user:
      max-login-attempts: 5

## user-service-dev.yml

    database:
      url: jdbc:mysql://localhost:3306/userdb

## user-service-prod.yml

    database:
      url: jdbc:mysql://production-db:3306/userdb

The same microservice can receive different configuration depending on the active profile.

---

# 8. Config Server

The Config Server is a Spring Boot application responsible for exposing configuration to clients.

The architecture becomes:

    Git
     ↓
    Config Server
     ↓
    Config Client

The Config Server reads configuration from Git and exposes it through HTTP endpoints.

---

# 9. Creating the Config Server

Create a Spring Boot project.

Typical dependencies:

    Spring Web
    Config Server

The main application class needs the Config Server annotation.

    @SpringBootApplication
    @EnableConfigServer
    public class ConfigServerApplication {

        public static void main(String[] args) {
            SpringApplication.run(
                ConfigServerApplication.class,
                args
            );
        }
    }

The important annotation is:

    @EnableConfigServer

This enables Spring Cloud Config Server functionality.

---

# 10. Config Server Configuration

The Config Server needs to know where the configuration repository is located.

For example:

    server:
      port: 8888

    spring:
      application:
        name: config-server

      cloud:
        config:
          server:
            git:
              uri: https://github.com/company/config-repo

The Config Server runs on:

    http://localhost:8888

---

# 11. Git-Based Configuration

Using Git provides several important advantages.

Configuration becomes:

    Version Controlled
          ↓
    Auditable
          ↓
    Reversible
          ↓
    Centrally Managed

For example:

    Commit 1
    database.url = database-v1

    Commit 2
    database.url = database-v2

If a configuration change causes a problem, Git allows the team to identify or revert the change.

---

# 12. Config Server API

The Config Server exposes configuration through HTTP.

For example:

    GET /user-service/dev

The request means:

    Application = user-service
    Profile = dev

The Config Server retrieves the appropriate configuration and returns it to the client.

Another example:

    GET /order-service/prod

means:

    Application = order-service
    Profile = prod

---

# 13. Config Client

A microservice that consumes configuration from the Config Server is called a Config Client.

For example:

    user-service

can become a Config Client.

The architecture becomes:

    user-service
          |
          | Request configuration
          ↓
    Config Server
          |
          ↓
    Git Repository

---

# 14. Config Client Implementation

The client application needs the appropriate Spring Cloud Config Client dependency.

The service needs to know where the Config Server is located.

A modern Spring Boot / Spring Cloud setup commonly uses:

    spring:
      config:
        import: optional:configserver:http://localhost:8888

The client will contact:

    http://localhost:8888

and retrieve its configuration.

---

# 15. Client Application Name

The Config Server needs to know which configuration belongs to the client.

For example:

    spring:
      application:
        name: user-service

If the active profile is:

    dev

the client will request configuration associated with:

    user-service
    dev

The Config Server can therefore load:

    application.yml
    user-service.yml
    user-service-dev.yml

and combine the applicable properties.

---

# 16. Client Profile

The client can specify an active profile.

    spring:
      application:
        name: user-service

      profiles:
        active: dev

Now the application is effectively requesting:

    user-service + dev

from the Config Server.

---

# 17. Complete Config Client Example

A client application might have:

    spring:
      application:
        name: user-service

      profiles:
        active: dev

      config:
        import: optional:configserver:http://localhost:8888

The Config Server:

    http://localhost:8888

will provide configuration from the Git repository.

---

# 18. Reading Configuration in the Application

Suppose the Git repository contains:

    user:
      max-login-attempts: 5

The Spring Boot application can read the value using `@Value`.

    @Value("${user.max-login-attempts}")
    private int maxLoginAttempts;

The application receives the value from the centralized configuration.

---

# 19. Using @ConfigurationProperties

For larger configurations, `@ConfigurationProperties` is often preferable.

Example:

    @ConfigurationProperties(prefix = "user")
    public class UserProperties {

        private int maxLoginAttempts;

        public int getMaxLoginAttempts() {
            return maxLoginAttempts;
        }

        public void setMaxLoginAttempts(int maxLoginAttempts) {
            this.maxLoginAttempts = maxLoginAttempts;
        }
    }

Now configuration can be grouped into a dedicated object.

Example configuration:

    user:
      max-login-attempts: 5

This approach becomes easier to maintain when many related properties exist.

---

# 20. Environment-Specific Configuration

One of the biggest benefits of Config Server is environment separation.

For example:

    user-service-dev.yml

    database:
      url: jdbc:mysql://localhost:3306/userdb

And:

    user-service-prod.yml

    database:
      url: jdbc:mysql://prod-db:3306/userdb

The application code remains the same.

Only the configuration changes.

Architecture:

    Same Application
          │
          ├── DEV → dev configuration
          │
          ├── QA  → qa configuration
          │
          └── PROD → prod configuration

---

# 21. Centralized Configuration Flow

The complete startup flow can be represented as:

    User Service starts
            ↓
    Reads application name
            ↓
    Reads active profile
            ↓
    Contacts Config Server
            ↓
    Config Server contacts Git
            ↓
    Git returns configuration
            ↓
    Config Server returns configuration
            ↓
    User Service loads properties
            ↓
    Application starts

This allows the application configuration to be externalized.

---

# 22. Refreshing Configuration at Runtime

One common problem is:

> What happens if configuration changes while the application is running?

For example:

    order.timeout=30

The value changes in Git:

    order.timeout=60

The application may still have the old value in memory.

We want to refresh the configuration without restarting the entire application.

Spring Boot Actuator can help expose a refresh endpoint.

---

# 23. Spring Boot Actuator

Add the Spring Boot Actuator dependency.

The Actuator provides management endpoints such as:

    /actuator/health
    /actuator/info
    /actuator/metrics
    /actuator/refresh

For refresh functionality, the `/actuator/refresh` endpoint is particularly important.

---

# 24. Exposing the Refresh Endpoint

The client application can configure Actuator:

    management:
      endpoints:
        web:
          exposure:
            include: refresh

Now the refresh endpoint can be accessed through:

    POST /actuator/refresh

Important:

The endpoint should normally be protected with authentication and authorization.

Never expose sensitive management endpoints publicly without appropriate security controls.

---

# 25. @RefreshScope

A bean whose configuration should be refreshed dynamically can use:

    @RefreshScope

Example:

    @RefreshScope
    @RestController
    public class ConfigController {

        @Value("${message}")
        private String message;

        @GetMapping("/message")
        public String getMessage() {
            return message;
        }
    }

The `@RefreshScope` annotation allows the bean to be recreated when a refresh occurs.

---

# 26. Runtime Refresh Flow

Suppose Git contains:

    message=Hello

The application starts.

The endpoint returns:

    Hello

Now change Git:

    message=Hello from Config Server

The application is still running.

Execute:

    POST /actuator/refresh

The configuration is refreshed.

The endpoint can now return:

    Hello from Config Server

No complete application restart is required.

---

# 27. Important Refresh Concept

The refresh process can be summarized as:

    Git configuration changes
              ↓
       Config Server
              ↓
    Client requests refresh
              ↓
       /actuator/refresh
              ↓
    Environment updated
              ↓
      @RefreshScope beans
        recreated
              ↓
      New configuration
         becomes active

---

# 28. Configuration Refresh Limitations

Not every configuration change should automatically be considered safe to refresh.

Some properties may require:

- Application restart
- Connection recreation
- Bean recreation
- Infrastructure changes
- Additional application logic

For example:

    database connection configuration

may require more consideration than:

    feature.enabled=true

Therefore, runtime refresh should be designed carefully.

---

# 29. Security Considerations

Configuration can contain sensitive information such as:

    database.password
    api.key
    client.secret
    OAuth credentials

Therefore, storing secrets directly in a public Git repository is dangerous.

In production, sensitive configuration should generally be handled using appropriate secret-management solutions such as:

    HashiCorp Vault
    AWS Secrets Manager
    Azure Key Vault
    Kubernetes Secrets

Spring Cloud Config should not be treated as a replacement for a dedicated secrets-management system.

---

# 30. Spring Cloud Config vs Local Configuration

## Without Config Server

    Service A → application.yml
    Service B → application.yml
    Service C → application.yml

Configuration is distributed across applications.

## With Config Server

    Git Repository
          ↓
    Config Server
          ↓
    ┌─────┼─────┐
    ↓     ↓     ↓
    A     B     C

Configuration is centralized.

---

# 31. Benefits

Spring Cloud Config provides:

- Centralized configuration
- Environment-specific configuration
- Git-based version control
- Configuration history
- Easier configuration management
- Externalized configuration
- Runtime refresh support
- Consistent configuration across microservices

---

# 32. Potential Drawbacks

There are also trade-offs.

### Config Server becomes infrastructure

If clients depend on Config Server during startup, the availability of the Config Server becomes important.

### Additional complexity

The architecture now includes:

    Git
    Config Server
    Config Clients

### Security

Configuration endpoints must be protected.

### Secrets

Sensitive values need proper secret-management strategies.

### Compatibility

Spring Boot and Spring Cloud versions must be compatible.

---

# 33. Real-World Architecture

A production architecture could look like:

    ┌─────────────────────────┐
    │       Git Repository    │
    │                         │
    │ application.yml         │
    │ user-service.yml        │
    │ order-service.yml       │
    │ payment-service.yml     │
    └────────────┬────────────┘
                 │
                 ↓
    ┌─────────────────────────┐
    │      Config Server      │
    │                         │
    │ Spring Cloud Config     │
    └────────────┬────────────┘
                 │
        ┌────────┼────────┐
        │        │        │
        ↓        ↓        ↓
    User      Order    Payment
    Service   Service  Service
        │        │        │
        └────────┼────────┘
                 ↓
           Spring Actuator
                 │
                 ↓
          Refresh Configuration

---

# 34. Complete Example

## Config Repository

    application.yml

    company:
      name: MyCompany

    user-service.yml

    user:
      max-login-attempts: 5

    user-service-dev.yml

    message: Hello from DEV

    user-service-prod.yml

    message: Hello from PROD

---

## Config Server

    @SpringBootApplication
    @EnableConfigServer
    public class ConfigServerApplication {

        public static void main(String[] args) {
            SpringApplication.run(
                ConfigServerApplication.class,
                args
            );
        }
    }

Configuration:

    server:
      port: 8888

    spring:
      application:
        name: config-server

      cloud:
        config:
          server:
            git:
              uri: https://github.com/company/config-repo

---

## Config Client

    spring:
      application:
        name: user-service

      profiles:
        active: dev

      config:
        import: optional:configserver:http://localhost:8888

The client retrieves:

    user-service.yml
    user-service-dev.yml
    application.yml

from the Config Server.

---

## Dynamic Configuration

    @RefreshScope
    @RestController
    public class MessageController {

        @Value("${message}")
        private String message;

        @GetMapping("/message")
        public String message() {
            return message;
        }
    }

Actuator configuration:

    management:
      endpoints:
        web:
          exposure:
            include: refresh

Then:

    POST /actuator/refresh

causes refreshable configuration to be reloaded.

---

# 35. Interview Questions

## What is Spring Cloud Config?

Spring Cloud Config provides centralized externalized configuration management for distributed applications and microservices.

## Why use Config Server?

To centralize configuration and avoid duplicating configuration across multiple microservices and environments.

## Where can configuration be stored?

Commonly in a Git repository, but other backends can also be supported.

## What is a Config Client?

A microservice that retrieves its configuration from the Config Server.

## What does @EnableConfigServer do?

It enables the Spring Cloud Config Server functionality in a Spring Boot application.

## What is @RefreshScope?

It allows a bean to be recreated when configuration is refreshed.

## How can configuration be refreshed?

A client can use the Actuator refresh endpoint:

    POST /actuator/refresh

## Does Spring Cloud Config replace a secret manager?

No. Sensitive secrets should generally be managed using a dedicated secret-management solution.

## Why use Git with Config Server?

Git provides version control, history, auditing and the ability to revert configuration changes.

---

# 36. Key Takeaways

Spring Cloud Config solves a common microservices problem:

> How do we centrally manage configuration for many applications and environments?

The architecture is:

    Git Repository
          ↓
    Config Server
          ↓
    Config Clients

The Config Server centralizes configuration.

The Config Client consumes the configuration.

Spring Boot Actuator can expose:

    /actuator/refresh

to trigger configuration refresh.

`@RefreshScope` allows selected beans to use refreshed configuration without requiring a complete application restart.

The overall idea is:

    Centralize
        ↓
    Version Control
        ↓
    Externalize
        ↓
    Consume
        ↓
    Refresh
