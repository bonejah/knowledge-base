# 15 - Spring Boot Actuator

## 1. What Is Spring Boot Actuator?

**Spring Boot Actuator** is a Spring Boot module that provides **production-ready features for monitoring and managing an application**.

It exposes information about the running application through endpoints, such as:

- Application health
- Application metrics
- JVM information
- Memory usage
- HTTP request statistics
- Database connectivity
- Active beans
- Environment properties
- Logging configuration
- Application mappings
- Thread information

In simple terms:

> **Spring Boot Actuator lets you look inside a running Spring Boot application without having to build all the monitoring functionality yourself.**

For example, if your application is running on:

```text
http://localhost:8080
```

Actuator can expose:

```text
http://localhost:8080/actuator/health
```

which might return:

```json
{
  "status": "UP"
}
```

This is extremely useful in production environments where systems such as Kubernetes, Prometheus, monitoring platforms, or load balancers need to know whether an application is healthy.

---

# 2. Why Do We Need Actuator?

Imagine that you have a Spring Boot application running in production.

Everything appears normal from the outside, but you need to answer questions such as:

- Is the application running?
- Can it connect to the database?
- How much memory is it using?
- How many HTTP requests is it receiving?
- Which endpoints are being called?
- How many requests are failing?
- Is the JVM running out of memory?
- How many threads are active?
- What is the current application configuration?
- Is a dependency such as MongoDB or PostgreSQL unavailable?
- Is the application ready to receive traffic?

Without Actuator, you would have to implement many of these features yourself.

Actuator provides many of them automatically.

---

# 3. Adding Actuator to a Spring Boot Application

Add the following dependency to your Maven project:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

With Gradle:

```gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

After adding the dependency, Spring Boot automatically configures Actuator.

By default, Actuator endpoints are available under:

```text
/actuator
```

For example:

```text
/actuator/health
/actuator/info
/actuator/metrics
```

---

# 4. Actuator Endpoints

Actuator functionality is exposed through **endpoints**.

An endpoint represents a specific piece of information or management functionality.

Some important endpoints include:

| Endpoint                | Purpose                           |
| ----------------------- | --------------------------------- |
| `/actuator/health`      | Application health                |
| `/actuator/info`        | Application information           |
| `/actuator/metrics`     | Application and JVM metrics       |
| `/actuator/loggers`     | View/change logging configuration |
| `/actuator/beans`       | Information about Spring beans    |
| `/actuator/mappings`    | Application request mappings      |
| `/actuator/env`         | Environment properties            |
| `/actuator/configprops` | Configuration properties          |
| `/actuator/threaddump`  | Thread information                |
| `/actuator/heapdump`    | JVM heap dump                     |
| `/actuator/caches`      | Cache information                 |
| `/actuator/conditions`  | Auto-configuration conditions     |

Not every endpoint should necessarily be exposed publicly.

---

# 5. The Health Endpoint

One of the most important Actuator endpoints is:

```text
/actuator/health
```

Example:

```json
{
  "status": "UP"
}
```

The purpose is to answer:

> "Is this application healthy enough to operate?"

This endpoint is commonly used by:

- Kubernetes
- Load balancers
- Cloud platforms
- Monitoring systems
- Deployment systems

---

# 6. Health Indicators

The health endpoint can include information about different components of the application.

For example:

```text
Application
    |
    +-- Database
    |
    +-- MongoDB
    |
    +-- Redis
    |
    +-- Disk
    |
    +-- External services
```

A response might look like:

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP"
    },
    "diskSpace": {
      "status": "UP"
    }
  }
}
```

Spring Boot can automatically create health indicators for many technologies.

For example:

- JDBC databases
- MongoDB
- Redis
- Cassandra
- Kafka
- Elasticsearch
- Disk space

This means that if your database becomes unavailable, the health status can reflect that.

---

# 7. Health Status

Spring Boot provides several health statuses.

The most common are:

```text
UP
DOWN
UNKNOWN
OUT_OF_SERVICE
```

For example:

```json
{
  "status": "DOWN"
}
```

could indicate that an important dependency is unavailable.

The overall health status is calculated from the health indicators.

For example:

```text
Database: UP
Redis: UP
Disk: UP

        ↓

Application: UP
```

But:

```text
Database: DOWN
Redis: UP
Disk: UP

        ↓

Application: DOWN
```

The exact behavior depends on the configured health groups and indicators.

---

# 8. Liveness and Readiness

In modern cloud-native applications, especially Kubernetes environments, two concepts are particularly important:

```text
Liveness
Readiness
```

They answer different questions.

### Liveness

Liveness asks:

> "Is the application still alive?"

If the answer is no, the platform may restart the application.

For example:

```text
Application process is completely broken
        ↓
Liveness = DOWN
        ↓
Kubernetes restarts the container
```

### Readiness

Readiness asks:

> "Is the application ready to receive traffic?"

An application can be alive but not ready.

For example:

```text
Application process = running
Database connection = unavailable

        ↓

Liveness = UP
Readiness = DOWN
```

In this situation, Kubernetes should generally stop sending traffic to the application without necessarily restarting it.

---

# 9. Enabling Liveness and Readiness

Spring Boot can expose:

```text
/actuator/health/liveness
```

and:

```text
/actuator/health/readiness
```

Configuration:

```properties
management.endpoint.health.probes.enabled=true
```

Example:

```text
GET /actuator/health/liveness
```

Response:

```json
{
  "status": "UP"
}
```

And:

```text
GET /actuator/health/readiness
```

Response:

```json
{
  "status": "UP"
}
```

This is particularly useful when deploying Spring Boot applications to Kubernetes.

---

# 10. Metrics Endpoint

Another important endpoint is:

```text
/actuator/metrics
```

It provides access to application metrics.

For example:

```text
/actuator/metrics
```

can return a list of available metrics.

You can then query an individual metric:

```text
/actuator/metrics/jvm.memory.used
```

Example:

```json
{
  "name": "jvm.memory.used",
  "measurements": [
    {
      "statistic": "VALUE",
      "value": 123456789
    }
  ]
}
```

Metrics can provide information about:

- JVM memory
- Garbage collection
- CPU
- Threads
- HTTP requests
- Database connection pools
- Application-specific metrics
- Cache usage

---

# 11. Micrometer

Spring Boot Actuator uses **Micrometer** as its metrics abstraction.

Micrometer provides a common API for collecting metrics.

Think of it as:

```text
Spring Boot
     |
     v
  Actuator
     |
     v
 Micrometer
     |
     +---- Prometheus
     +---- Datadog
     +---- New Relic
     +---- CloudWatch
     +---- Other monitoring systems
```

This allows your application to collect metrics without being tightly coupled to one monitoring platform.

---

# 12. Prometheus Integration

A very common combination is:

```text
Spring Boot
    ↓
Actuator
    ↓
Micrometer
    ↓
Prometheus
    ↓
Grafana
```

Add:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Then expose the Prometheus endpoint:

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```

Prometheus can scrape:

```text
/actuator/prometheus
```

The architecture becomes:

```text
                    Spring Boot
                        |
                     Actuator
                        |
                    Micrometer
                        |
                  /actuator/prometheus
                        |
                    Prometheus
                        |
                     Grafana
```

---

# 13. Exposing Actuator Endpoints

By default, you should **not expose every endpoint**.

You can configure which endpoints are available over HTTP.

For example:

```properties
management.endpoints.web.exposure.include=health,info,metrics
```

Now the application exposes:

```text
/actuator/health
/actuator/info
/actuator/metrics
```

Instead of exposing everything.

You can expose all endpoints with:

```properties
management.endpoints.web.exposure.include=*
```

However:

> **Exposing everything publicly is generally a bad production practice.**

Some endpoints expose sensitive information.

---

# 14. Sensitive Actuator Endpoints

Some endpoints can reveal information that should not be publicly accessible.

For example:

```text
/actuator/env
/actuator/configprops
/actuator/beans
/actuator/mappings
/actuator/heapdump
```

These can potentially reveal:

- Environment variables
- Configuration
- Internal application structure
- Bean information
- Request mappings
- JVM information
- Sensitive configuration details

Therefore, Actuator should be treated as a **management interface**, not as a normal public API.

---

# 15. Securing Actuator

If Spring Security is being used, Actuator endpoints should normally be protected.

For example, you might allow:

```text
/actuator/health
```

without authentication:

```text
GET /actuator/health
```

while requiring authentication for:

```text
/actuator/env
/actuator/beans
/actuator/loggers
```

A simplified Spring Security configuration could look like:

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health").permitAll()
            .requestMatchers("/actuator/**").hasRole("ACTUATOR")
            .anyRequest().authenticated()
        );

    return http.build();
}
```

The exact security configuration depends on the application's authentication architecture.

---

# 16. Using a Separate Management Port

Another useful practice is running Actuator on a different port.

For example:

```properties
server.port=8080
management.server.port=8081
```

Now:

```text
Application
http://localhost:8080

Actuator
http://localhost:8081/actuator
```

This can help isolate management traffic from normal application traffic.

For example:

```text
Internet
   |
   v
Port 8080
   |
   +---- Public API


Internal Network
   |
   v
Port 8081
   |
   +---- Actuator
```

This can be particularly useful in production environments.

However, a separate port does **not** automatically make Actuator secure. Network controls and authentication should still be considered.

---

# 17. The Info Endpoint

The endpoint:

```text
/actuator/info
```

can expose custom application information.

For example:

```properties
info.app.name=User Service
info.app.version=1.5.0
info.app.description=User management service
```

The endpoint could return:

```json
{
  "app": {
    "name": "User Service",
    "version": "1.5.0",
    "description": "User management service"
  }
}
```

This can be useful for identifying which version of an application is running.

---

# 18. Build Information

You can also expose build information.

For example:

```properties
info.app.name=${project.name}
info.app.version=${project.version}
```

Or configure Maven's build information generation.

Then Actuator can expose information such as:

```text
Application version
Build timestamp
Artifact
Group
```

This is particularly useful when several instances of an application are deployed.

For example:

```text
Instance 1 → version 1.5.0
Instance 2 → version 1.5.0
Instance 3 → version 1.4.9
```

This can immediately reveal that an old version is still running.

---

# 19. Application Metrics

Actuator automatically provides many useful metrics.

For example:

```text
jvm.memory.used
jvm.memory.max
jvm.threads.live
jvm.gc.pause
system.cpu.usage
process.cpu.usage
```

HTTP metrics can include:

```text
http.server.requests
```

You can inspect:

```text
/actuator/metrics/http.server.requests
```

This can help answer questions such as:

- How many requests are arriving?
- How long are requests taking?
- Which HTTP status codes are being returned?
- Which endpoints are receiving traffic?

---

# 20. Custom Metrics

You can create your own application metrics.

For example, suppose you want to count how many users register.

Using Micrometer:

```java
@Service
public class UserService {

    private final Counter userRegistrationCounter;

    public UserService(MeterRegistry meterRegistry) {
        this.userRegistrationCounter =
                Counter.builder("users.registered")
                        .description("Number of registered users")
                        .register(meterRegistry);
    }

    public void registerUser() {

        // Register user

        userRegistrationCounter.increment();
    }
}
```

Now you can expose:

```text
/actuator/metrics/users.registered
```

This allows monitoring systems to observe business-level metrics.

---

# 21. Gauges

A **Counter** is useful when a value only increases.

For example:

```text
users.registered
orders.created
payments.processed
```

A **Gauge** is useful when a value can increase and decrease.

For example:

```text
active.users
queue.size
connections.active
```

Example:

```java
Gauge.builder(
        "users.active",
        userService,
        UserService::getActiveUsers
    )
    .register(meterRegistry);
```

---

# 22. Timers

Timers measure how long operations take.

For example:

```java
Timer timer = Timer
        .builder("payment.processing.time")
        .description("Payment processing duration")
        .register(meterRegistry);

timer.record(() -> {
    processPayment();
});
```

You can then monitor:

```text
Average duration
Maximum duration
Number of executions
Total time
```

This is much more useful than simply knowing whether an operation succeeded.

---

# 23. Application Logging

Actuator can also expose logging information.

Endpoint:

```text
/actuator/loggers
```

It can show the current logging configuration.

For example:

```text
ROOT = INFO
com.example.user = DEBUG
com.example.payment = WARN
```

In some configurations, logging levels can also be changed dynamically.

For example:

```text
com.example.payment
        |
        v
     DEBUG
```

This can be useful when investigating production problems.

However, changing logging levels dynamically should be controlled and audited.

---

# 24. Thread Dump

The endpoint:

```text
/actuator/threaddump
```

can provide information about JVM threads.

This can help investigate:

- Deadlocks
- Blocked threads
- Thread pool problems
- High concurrency
- Threads waiting on locks

For example:

```text
Thread-1 → RUNNABLE
Thread-2 → WAITING
Thread-3 → BLOCKED
Thread-4 → TIMED_WAITING
```

This is especially useful when diagnosing JVM performance problems.

---

# 25. Heap Dump

The endpoint:

```text
/actuator/heapdump
```

can generate a JVM heap dump.

Heap dumps are useful for investigating:

- Memory leaks
- Excessive object creation
- OutOfMemoryError
- Unexpected memory consumption

However, heap dumps can be:

- Very large
- Expensive to generate
- Sensitive
- Potentially dangerous to expose

Therefore, this endpoint should be heavily protected.

---

# 26. Beans Endpoint

The endpoint:

```text
/actuator/beans
```

provides information about Spring beans.

For example:

```text
UserController
UserService
UserRepository
DataSource
TransactionManager
```

This can help diagnose problems involving Spring's application context.

For example, if you expect:

```text
UserService
```

to exist but it does not appear, you can investigate configuration or component scanning.

---

# 27. Mappings Endpoint

The endpoint:

```text
/actuator/mappings
```

shows the application's request mappings.

For example:

```text
GET    /users
POST   /users
GET    /users/{id}
DELETE /users/{id}
```

This can be useful when debugging routing problems.

Instead of wondering:

> "Why isn't my controller being called?"

you can inspect the registered mappings.

---

# 28. Environment Endpoint

The endpoint:

```text
/actuator/env
```

shows environment and configuration information.

For example:

```text
spring.datasource.url
server.port
spring.profiles.active
```

This endpoint should be treated as sensitive.

Even when Spring masks certain values, exposing configuration information publicly can reveal useful information to an attacker.

Therefore:

```text
/actuator/env
```

should generally be protected.

---

# 29. Actuator and Kubernetes

Actuator works particularly well with Kubernetes.

A typical setup might look like:

```text
Kubernetes
     |
     +---- Liveness Probe
     |         |
     |         v
     |   /actuator/health/liveness
     |
     +---- Readiness Probe
               |
               v
        /actuator/health/readiness
```

Example Kubernetes configuration:

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

This allows Kubernetes to make intelligent decisions about the application.

---

# 30. Actuator and Prometheus/Grafana

A very common production architecture is:

```text
                  Spring Boot
                      |
                   Actuator
                      |
                  Micrometer
                      |
                 Prometheus
                      |
                   Grafana
```

Actuator provides the metrics.

Micrometer provides the metrics abstraction and integration.

Prometheus stores and queries the metrics.

Grafana visualizes them.

For example, Grafana could display:

```text
HTTP Requests:       15,432
Error Rate:             1.2%
Average Latency:       85 ms
CPU Usage:              62%
Memory Usage:          1.8 GB
Active Threads:         43
```

This is much more powerful than manually checking:

```text
/actuator/health
```

because it provides historical information and dashboards.

---

# 31. Actuator Is Not a Monitoring System

An important distinction is:

> **Actuator is not the same thing as a monitoring platform.**

Actuator exposes information.

A monitoring system collects, stores, analyzes, and visualizes that information.

For example:

```text
Actuator
   ↓
Exposes metrics

Prometheus
   ↓
Collects and stores metrics

Grafana
   ↓
Visualizes metrics
```

Think of Actuator as a **window into the application**.

Prometheus and Grafana are systems that can use that information to build monitoring infrastructure.

---

# 32. Actuator vs Logging

Actuator and logging solve different problems.

### Logging

Answers:

> "What happened?"

Example:

```text
2026-08-18 20:10:32 ERROR Payment failed
```

### Actuator

Answers:

> "What is the current state of the application?"

Example:

```text
Application: UP
Memory: 65%
Threads: 42
HTTP errors: 1.2%
Database: UP
```

They complement each other.

A production system normally needs both.

---

# 33. Actuator vs Distributed Tracing

Actuator is also different from distributed tracing.

For example:

```text
Actuator
    ↓
Application health and metrics
```

while:

```text
OpenTelemetry
    ↓
Distributed traces
    ↓
Service A → Service B → Database
```

Actuator might tell you:

```text
HTTP latency = 800 ms
```

Tracing can help you understand:

```text
HTTP request
     ↓
User Service = 50 ms
     ↓
Payment Service = 600 ms
     ↓
Database = 150 ms
```

In modern distributed systems, metrics, logs, and traces are often used together.

---

# 34. Common Production Architecture

A mature Spring Boot production environment might look like:

```text
                     Users
                       |
                    Load Balancer
                       |
                Spring Boot App
                       |
          +------------+------------+
          |            |            |
       Logging      Actuator     OpenTelemetry
          |            |            |
          v            v            v
     Log Platform  Prometheus     Tracing
                       |
                       v
                    Grafana
```

Each component has a different responsibility.

---

# 35. Configuration Example

A reasonable starting configuration might be:

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus

management.endpoint.health.show-details=when_authorized

management.info.env.enabled=true

management.server.port=8081
```

The exact configuration should depend on your infrastructure and security model.

For a simple local development environment, you might expose more endpoints.

For production, expose only what is required.

---

# 36. Health Details

You can configure how much information the health endpoint reveals.

For example:

```properties
management.endpoint.health.show-details=never
```

Possible behavior includes:

```text
never
when-authorized
always
```

For production, exposing detailed health information publicly is generally unnecessary.

For example, instead of publicly returning:

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP"
    }
  }
}
```

you might expose only:

```json
{
  "status": "UP"
}
```

while allowing authorized administrators to see the details.

---

# 37. Custom Health Indicator

You can create your own health indicator.

For example, suppose your application depends on an external payment service.

```java
@Component
public class PaymentServiceHealthIndicator
        implements HealthIndicator {

    @Override
    public Health health() {

        boolean available = checkPaymentService();

        if (available) {
            return Health.up()
                    .withDetail("service", "payment")
                    .build();
        }

        return Health.down()
                .withDetail("service", "payment")
                .build();
    }

    private boolean checkPaymentService() {
        // Check external service
        return true;
    }
}
```

Actuator can then include:

```json
{
  "status": "UP",
  "components": {
    "paymentService": {
      "status": "UP"
    }
  }
}
```

This allows Actuator to represent application-specific dependencies.

---

# 38. A Common Mistake with Health Checks

Be careful when creating custom health checks.

Imagine your application depends on:

```text
Payment Service
Email Service
Analytics Service
```

If you make every dependency part of the application's readiness check, then:

```text
Analytics Service DOWN
        ↓
Application readiness DOWN
        ↓
Traffic removed from application
```

even though the application could continue processing users normally.

Therefore, you should distinguish between:

### Critical dependencies

If unavailable, the application cannot perform its primary function.

Example:

```text
Primary Database
```

### Non-critical dependencies

The application can continue operating with degraded functionality.

Example:

```text
Analytics Service
```

This distinction is important when designing health checks.

---

# 39. Best Practices

## 39.1 Do Not Expose Everything

Avoid:

```properties
management.endpoints.web.exposure.include=*
```

in production unless you have a very specific reason.

Prefer:

```properties
management.endpoints.web.exposure.include=health,info,prometheus
```

---

## 39.2 Protect Sensitive Endpoints

Endpoints such as:

```text
/env
/beans
/configprops
/mappings
/loggers
/heapdump
/threaddump
```

should generally require authentication and authorization.

---

## 39.3 Keep Health Checks Fast

A health check should normally be fast.

Avoid a health endpoint that performs:

```text
10 external HTTP requests
+
5 database queries
+
3 expensive calculations
```

every time Kubernetes checks it.

Health checks should answer:

> "Can this instance safely perform its intended role?"

They should not become a performance problem themselves.

---

## 39.4 Separate Liveness and Readiness

Do not treat these concepts as interchangeable.

```text
Liveness
    =
"Should this process be restarted?"

Readiness
    =
"Should this instance receive traffic?"
```

This distinction is especially important in Kubernetes.

---

## 39.5 Use Metrics Instead of Polling Health

Health endpoints are excellent for:

```text
UP / DOWN
```

They are not designed to replace metrics.

For example, don't use:

```text
/actuator/health
```

to determine:

```text
Average HTTP latency
CPU utilization
Request rate
Error rate
Memory usage
```

Use metrics for those.

---

## 39.6 Use Business Metrics Carefully

Custom metrics can be extremely valuable.

Examples:

```text
orders.created
payments.failed
users.registered
documents.processed
```

But avoid creating metrics with extremely high-cardinality tags.

Bad:

```text
user.login{userId="123456"}
user.login{userId="123457"}
user.login{userId="123458"}
```

This can create an enormous number of unique time series.

Prefer bounded dimensions such as:

```text
user.login{country="CA"}
user.login{method="password"}
user.login{method="oauth"}
```

---

## 39.7 Avoid Sensitive Information

Never intentionally expose secrets through Actuator.

Be particularly careful with:

```text
Environment variables
Database URLs
Credentials
API keys
Tokens
Internal hostnames
```

Even if Spring masks some sensitive values, Actuator should still be considered sensitive infrastructure.

---

## 39.8 Restrict Network Access

Ideally, management endpoints should only be accessible from trusted systems.

For example:

```text
Internet
   X
   |
   | blocked
   v

Actuator

   ^
   |
Internal monitoring network
   |
Prometheus
```

This provides an additional layer of security.

---

# 40. Development vs Production

A useful approach is to have different Actuator configurations for different environments.

### Development

You might expose:

```text
health
info
metrics
beans
mappings
loggers
env
```

because debugging is easier.

### Production

You might expose only:

```text
health
info
prometheus
```

and protect everything else.

For example:

```text
Development

Actuator
 ├── health
 ├── info
 ├── metrics
 ├── beans
 ├── mappings
 ├── env
 └── loggers


Production

Actuator
 ├── health
 ├── info
 └── prometheus
```

---

# 41. A Practical Example

Imagine a Spring Boot application called:

```text
User Service
```

It has:

```text
REST API
    |
    +--- PostgreSQL
    |
    +--- Redis
    |
    +--- External Identity Service
```

You could configure:

```properties
management.endpoints.web.exposure.include=health,info,prometheus

management.endpoint.health.probes.enabled=true

management.endpoint.health.show-details=when_authorized
```

Then Kubernetes uses:

```text
/actuator/health/liveness
```

for liveness.

And:

```text
/actuator/health/readiness
```

for readiness.

Prometheus uses:

```text
/actuator/prometheus
```

for metrics.

Grafana uses Prometheus to create dashboards.

The result is:

```text
                         Kubernetes
                       /            \
                      /              \
              Liveness              Readiness
                 |                     |
                 v                     v
        /actuator/health/     /actuator/health/
            liveness               readiness


                         Spring Boot
                              |
                           Actuator
                              |
                         Micrometer
                              |
                         Prometheus
                              |
                           Grafana
```

---

# 42. Actuator in a Microservices Architecture

Actuator becomes even more valuable when you have many services.

Imagine:

```text
API Gateway
    |
    +---- User Service
    |
    +---- Order Service
    |
    +---- Payment Service
    |
    +---- Notification Service
```

Every service can expose standardized endpoints:

```text
/user-service/actuator/health
/order-service/actuator/health
/payment-service/actuator/health
/notification-service/actuator/health
```

And metrics:

```text
/user-service/actuator/prometheus
/order-service/actuator/prometheus
/payment-service/actuator/prometheus
/notification-service/actuator/prometheus
```

This standardization is one of Actuator's biggest advantages.

---

# 43. Actuator and Observability

Actuator is an important part of the **observability** ecosystem.

Observability is commonly discussed in terms of:

```text
Logs
Metrics
Traces
```

Often called the **three pillars of observability**.

Actuator primarily contributes to:

```text
Metrics
+
Health
+
Runtime information
```

It can work alongside:

```text
SLF4J / Logback
    → Logs

Micrometer / Actuator
    → Metrics

OpenTelemetry
    → Traces
```

Together they provide a much more complete view of the application.

---

# 44. Advantages

### Easy to configure

Adding the dependency provides many features automatically.

### Production-ready

It was designed specifically for operational concerns.

### Standardized

Different Spring Boot services can expose similar endpoints.

### Integrates with monitoring platforms

Especially:

```text
Prometheus
Grafana
Kubernetes
Cloud monitoring platforms
```

### Extensible

You can create:

```text
Custom health indicators
Custom metrics
Custom endpoints
```

### Excellent for microservices

Every service can expose consistent health and metrics endpoints.

---

# 45. Disadvantages and Limitations

### Security risks

Exposing sensitive endpoints can reveal internal information.

### Not a complete monitoring solution

Actuator does not replace:

```text
Prometheus
Grafana
Datadog
New Relic
OpenTelemetry
```

### Possible performance overhead

Some endpoints, especially:

```text
heapdump
threaddump
```

can be expensive.

### Health checks can be badly designed

A poorly designed health check can overload dependencies or cause unnecessary restarts.

### Metrics can become expensive

Poorly designed metric tags can create high cardinality and increase monitoring-system costs.

---

# 46. Common Interview Questions

## What is Spring Boot Actuator?

Spring Boot Actuator is a Spring Boot module that provides production-ready endpoints for monitoring and managing applications.

---

## What is `/actuator/health` used for?

It provides the current health status of the application and its configured health indicators.

---

## What is the difference between liveness and readiness?

**Liveness** determines whether the application should be considered alive.

**Readiness** determines whether the application is ready to receive traffic.

---

## Does Actuator monitor my application by itself?

Not exactly.

Actuator exposes health information and metrics. External systems can collect and visualize that information.

---

## Is Actuator a replacement for Prometheus?

No.

A common architecture is:

```text
Actuator
    ↓
Micrometer
    ↓
Prometheus
    ↓
Grafana
```

---

## Should I expose all Actuator endpoints?

Generally, no.

Expose only what is necessary and protect sensitive endpoints.

---

## Can I create custom health checks?

Yes.

You can implement custom `HealthIndicator` components.

---

## Can I create custom metrics?

Yes.

You can use Micrometer APIs such as:

```text
Counter
Gauge
Timer
DistributionSummary
```

---

# 47. Recommended Mental Model

Think about Spring Boot Actuator like this:

```text
Spring Boot Application
        |
        v
     Actuator
        |
        +-------------------+
        |                   |
        v                   v
      Health              Metrics
        |                   |
        |                Micrometer
        |                   |
        |              Prometheus
        |                   |
        |                Grafana
        |
        +---- Kubernetes
        |
        +---- Load Balancer
        |
        +---- Monitoring
```

Actuator is essentially the **operational interface of your Spring Boot application**.

Your normal REST API answers questions from your users:

```text
GET /users
POST /users
GET /orders
```

Actuator answers questions from your infrastructure and operators:

```text
GET /actuator/health
GET /actuator/metrics
GET /actuator/prometheus
```

That distinction is very important.

---

# 48. Final Summary

Spring Boot Actuator provides a standardized way to expose information about a running Spring Boot application.

The most important concepts are:

```text
Actuator
   |
   +--- Health
   |
   +--- Liveness
   |
   +--- Readiness
   |
   +--- Metrics
   |
   +--- Application information
   |
   +--- Runtime information
   |
   +--- Logging configuration
   |
   +--- Custom health indicators
   |
   +--- Custom metrics
```

For a modern production application, a common architecture is:

```text
                    Spring Boot
                         |
                      Actuator
                         |
              +----------+----------+
              |                     |
            Health                Metrics
              |                     |
        Kubernetes             Micrometer
                                    |
                               Prometheus
                                    |
                                  Grafana
```

The most important best practices are:

1. **Expose only the endpoints you actually need.**
2. **Protect sensitive Actuator endpoints.**
3. **Do not expose management endpoints unnecessarily to the public internet.**
4. **Use liveness and readiness for different purposes.**
5. **Keep health checks fast and meaningful.**
6. **Use metrics for performance and operational monitoring.**
7. **Avoid high-cardinality metric tags.**
8. **Never expose secrets or sensitive configuration.**
9. **Use Actuator together with logs and distributed tracing for full observability.**
10. **Treat Actuator as an operational/management interface, not as part of your public business API.**
