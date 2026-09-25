# 33 - Spring Dependency Injection

## 1. Understanding the Core Problem

Dependency Injection (DI) is one of the fundamental concepts of the Spring Framework.

The basic idea is simple:

> An object should not be responsible for creating the dependencies it needs. Instead, those dependencies should be provided by another component, typically the Spring IoC Container.

### The Problem

Consider a Spring application with a Singleton bean that needs to use a Prototype bean.

Example scenario:

- `OrderService` is a Singleton
- `OrderProcessor` is a Prototype
- Every operation needs a new `OrderProcessor`

If we simply inject the Prototype bean into the Singleton:

    @Service
    public class OrderService {

        private final OrderProcessor processor;

        public OrderService(OrderProcessor processor) {
            this.processor = processor;
        }
    }

The problem is that Spring resolves the dependency when the Singleton is created.

The Singleton receives one instance of `OrderProcessor` and keeps using that same instance.

Therefore:

    Singleton
        |
        +----> Prototype instance #1

Even though `OrderProcessor` is declared as Prototype, the Singleton does not automatically receive a new instance every time it needs one.

### Why Does This Happen?

Spring bean scopes define how the container manages instances.

Common scopes include:

- Singleton
- Prototype
- Request
- Session
- Application
- WebSocket

A Singleton normally lives for the entire application lifecycle.

A Prototype bean, on the other hand, is created whenever Spring is asked for a new instance.

Therefore, injecting a Prototype directly into a Singleton creates a lifecycle mismatch.

    Singleton
        |
        +----> Prototype

The Singleton is created once, so its injected dependency is also resolved once.

### The Core Question

How can a Singleton obtain a new dependency instance whenever it actually needs one?

Spring provides several solutions.

The most common approaches are:

1. `ObjectProvider`
2. Scoped Proxy
3. `@Lookup` Method Injection

---

# 2. Solution 1 - ObjectProvider

`ObjectProvider` is one of the cleanest ways to request a dependency from the Spring container when it is needed.

Instead of injecting the actual Prototype object, we inject an `ObjectProvider`.

Example:

    @Service
    public class OrderService {

        private final ObjectProvider<OrderProcessor> processorProvider;

        public OrderService(ObjectProvider<OrderProcessor> processorProvider) {
            this.processorProvider = processorProvider;
        }

        public void processOrder() {

            OrderProcessor processor =
                    processorProvider.getObject();

            processor.process();
        }
    }

The Singleton receives the provider:

    OrderService
         |
         +----> ObjectProvider
                    |
                    +----> OrderProcessor

Every time `getObject()` is called, Spring can provide a new Prototype instance.

For example:

    processorProvider.getObject()
            |
            +----> OrderProcessor #1

    processorProvider.getObject()
            |
            +----> OrderProcessor #2

    processorProvider.getObject()
            |
            +----> OrderProcessor #3

### Prototype Bean

The dependency can be declared as:

    @Component
    @Scope("prototype")
    public class OrderProcessor {

        public void process() {
            // processing logic
        }
    }

Or using:

    @Prototype
    public class OrderProcessor {
    }

Conceptually:

    @Service
    OrderService
        |
        +---- ObjectProvider<OrderProcessor>
                     |
                     +---- getObject()
                              |
                              +---- new Prototype instance

### ObjectProvider Methods

Some useful methods include:

    getObject()

Obtains an object from the container.

    getIfAvailable()

Returns the object if available, otherwise returns `null`.

    getIfUnique()

Returns the object if there is exactly one matching candidate.

This makes `ObjectProvider` useful when dependency creation needs to be deferred.

### Advantages

- Simple
- Explicit
- Easy to understand
- Supports lazy retrieval
- Works well with Prototype beans
- Provides additional lookup methods
- Does not require proxying

### Disadvantages

The main disadvantage is that the application code becomes aware of Spring's dependency lookup mechanism.

For example:

    processorProvider.getObject();

The service is now directly interacting with a Spring-specific abstraction.

---

# 3. Solution 2 - Scoped Proxy

Another solution is to use a Scoped Proxy.

A scoped proxy allows Spring to inject a proxy instead of the actual Prototype object.

Example:

    @Component
    @Scope(
        value = "prototype",
        proxyMode = ScopedProxyMode.TARGET_CLASS
    )
    public class OrderProcessor {

        public void process() {
            // processing logic
        }
    }

Now the Singleton can inject `OrderProcessor` directly:

    @Service
    public class OrderService {

        private final OrderProcessor processor;

        public OrderService(OrderProcessor processor) {
            this.processor = processor;
        }

        public void processOrder() {
            processor.process();
        }
    }

At first this may look like a normal dependency injection.

However, the injected object is actually a proxy.

Conceptually:

    OrderService
         |
         +----> OrderProcessor Proxy
                         |
                         +----> Real OrderProcessor #1
                         |
                         +----> Real OrderProcessor #2
                         |
                         +----> Real OrderProcessor #3

The proxy delegates the call to the appropriate object according to the bean scope.

### TARGET_CLASS

`ScopedProxyMode.TARGET_CLASS` creates a class-based proxy.

This usually uses CGLIB-based proxying.

Conceptually:

    OrderProcessor
          ^
          |
    Spring Proxy
          |
          +---- real OrderProcessor

### INTERFACES

If the dependency is based on an interface, another option is:

    @Scope(
        value = "prototype",
        proxyMode = ScopedProxyMode.INTERFACES
    )

This creates a proxy based on interfaces.

### Important Concept

The Singleton does not actually receive the Prototype instance directly.

It receives something like:

    Singleton
       |
       +----> Proxy
                 |
                 +----> Prototype instance

The proxy hides the lifecycle management from the business code.

### Advantages

- Transparent to business code
- The consumer can inject the dependency normally
- Spring manages the lifecycle
- No explicit `getObject()` calls
- Useful for request/session scoped dependencies

### Disadvantages

- Adds proxying
- Can make debugging more complicated
- Proxy behavior can sometimes be confusing
- There can be limitations when dealing with final classes or methods
- Adds some Spring-specific configuration

---

# 4. Solution 3 - @Lookup Method Injection

Spring also provides `@Lookup` for method injection.

The idea is to define a method that Spring overrides internally and uses to obtain the dependency from the container.

Example:

    @Service
    public abstract class OrderService {

        public void processOrder() {

            OrderProcessor processor = getOrderProcessor();

            processor.process();
        }

        @Lookup
        protected abstract OrderProcessor getOrderProcessor();
    }

The Prototype dependency:

    @Component
    @Scope("prototype")
    public class OrderProcessor {

        public void process() {
            // processing logic
        }
    }

When `getOrderProcessor()` is called, Spring retrieves the appropriate bean from the container.

Conceptually:

    OrderService
         |
         +---- getOrderProcessor()
                    |
                    +---- Spring Container
                              |
                              +---- OrderProcessor #1

Another call:

    getOrderProcessor()
         |
         +---- Spring Container
                    |
                    +---- OrderProcessor #2

### Why Is the Method Abstract?

Spring creates a subclass dynamically and overrides the lookup method.

Conceptually:

    Your OrderService
           |
           v
    Spring-generated subclass
           |
           +---- overrides getOrderProcessor()
                        |
                        +---- retrieves bean

This is why `@Lookup` is considered a form of method injection.

### Alternative Form

The method can also return the bean type:

    @Lookup
    protected OrderProcessor getOrderProcessor() {
        return null;
    }

Spring replaces the implementation when creating the bean.

### Advantages

- Very clean business code
- The consumer does not explicitly call `ObjectProvider`
- Spring performs the dependency lookup
- Useful when a Singleton needs Prototype instances

### Disadvantages

- More magic
- Can be harder to understand for developers unfamiliar with Spring
- Requires Spring to create a subclass
- Can have limitations with final classes and methods
- Less commonly used than `ObjectProvider`

---

# 5. Comparing the Three Solutions

| Solution       | How It Works                           | Business Code   | Proxy           | Spring Coupling |
| -------------- | -------------------------------------- | --------------- | --------------- | --------------- |
| ObjectProvider | Explicitly asks Spring for an instance | Explicit lookup | No              | Higher          |
| Scoped Proxy   | Injects a proxy                        | Transparent     | Yes             | Medium          |
| @Lookup        | Spring overrides a method              | Transparent     | No direct proxy | Medium/High     |

### ObjectProvider

    Singleton
        |
        +---- ObjectProvider
                  |
                  +---- Prototype

The application explicitly asks for an instance.

### Scoped Proxy

    Singleton
        |
        +---- Proxy
                |
                +---- Prototype

The proxy handles the lookup.

### @Lookup

    Singleton
        |
        +---- @Lookup method
                  |
                  +---- Spring Container
                            |
                            +---- Prototype

Spring implements the lookup method.

---

# 6. Use Cases

## Use Case 1 - Prototype Dependency Inside Singleton

A common example is a Singleton service that needs a new processing object for every operation.

Example:

    @Service
    public class PaymentService {

        private final ObjectProvider<PaymentProcessor> provider;

        public PaymentService(
                ObjectProvider<PaymentProcessor> provider) {

            this.provider = provider;
        }

        public void processPayment() {

            PaymentProcessor processor =
                    provider.getObject();

            processor.process();
        }
    }

This is useful when `PaymentProcessor` contains operation-specific state.

---

# 7. Use Case 2 - Request Scoped Bean

Scoped proxies are particularly useful in web applications.

For example:

    @Component
    @RequestScope
    public class RequestContext {

        private String correlationId;

        // getters and setters
    }

A Singleton service can inject it:

    @Service
    public class OrderService {

        private final RequestContext requestContext;

        public OrderService(RequestContext requestContext) {
            this.requestContext = requestContext;
        }
    }

Spring can inject a proxy that resolves the correct `RequestContext` for the current HTTP request.

Conceptually:

    Singleton Service
          |
          +---- RequestContext Proxy
                         |
                         +---- Request #1 Context
                         |
                         +---- Request #2 Context
                         |
                         +---- Request #3 Context

This is one of the most important practical use cases for scoped proxies.

---

# 8. Use Case 3 - Stateful Processing

Suppose a processor maintains state during one operation:

    @Component
    @Scope("prototype")
    public class ReportProcessor {

        private ReportContext context;

        public void process(ReportContext context) {
            this.context = context;

            // processing
        }
    }

If each report requires an independent processor instance, Prototype scope can be appropriate.

A Singleton service can obtain a new processor for each report.

---

# 9. Use Case 4 - Lazy Dependency Creation

`ObjectProvider` can also be useful when a dependency should not be created until it is actually needed.

Example:

    private final ObjectProvider<HeavyService> provider;

    public void execute(boolean required) {

        if (required) {
            HeavyService service =
                    provider.getObject();

            service.execute();
        }
    }

The dependency can be retrieved only when the business operation requires it.

---

# 10. Important Interview Concept

A very common interview question is:

> What happens when a Singleton bean injects a Prototype bean?

The important answer is:

A Prototype dependency injected directly into a Singleton is normally resolved when the Singleton is created.

Therefore, the Singleton does not automatically obtain a new Prototype instance every time a method is called.

Example:

    Singleton
        |
        +---- Prototype #1

Calling the Singleton method multiple times does not automatically produce:

    Prototype #1
    Prototype #2
    Prototype #3

To obtain new instances, mechanisms such as:

- `ObjectProvider`
- Scoped Proxy
- `@Lookup`

can be used.

---

# 11. Another Important Interview Question

> Why can't Spring simply inject a new Prototype every time a method is called?

Because dependency injection happens when the dependency is resolved.

Spring does not automatically analyze every method invocation and recreate injected dependencies.

The Singleton contains a reference to the dependency it received.

Therefore:

    Singleton created
          |
          v
    Prototype created
          |
          v
    Reference stored in Singleton

Later:

    method call
          |
          v
    same reference

If you need dynamic retrieval, you need an appropriate mechanism.

---

# 12. ObjectProvider vs @Lookup

Both can solve the Singleton → Prototype problem, but they approach it differently.

### ObjectProvider

    provider.getObject();

The application explicitly asks Spring for the object.

### @Lookup

    getProcessor();

Spring handles the lookup behind the method.

A simple mental model:

    ObjectProvider
        = "I want to explicitly ask Spring."

    @Lookup
        = "Spring, implement this method and perform the lookup."

---

# 13. ObjectProvider vs Scoped Proxy

### ObjectProvider

The developer controls when the dependency is retrieved.

    processorProvider.getObject();

This makes the lifecycle interaction explicit.

### Scoped Proxy

The developer interacts with the dependency normally.

    processor.process();

The proxy handles the lifecycle behavior.

Therefore:

    ObjectProvider
        -> Explicit lookup

    Scoped Proxy
        -> Transparent lookup

---

# 14. Advantages

## Dependency Injection

- Promotes loose coupling
- Improves testability
- Separates object creation from business logic
- Centralizes dependency management
- Supports different bean scopes
- Makes applications easier to configure

## ObjectProvider

- Explicit
- Flexible
- Easy to understand
- Supports lazy retrieval
- Good for optional dependencies

## Scoped Proxy

- Transparent to business code
- Works well with request/session scopes
- Simplifies access to scoped dependencies

## @Lookup

- Clean consumer code
- Useful for dynamic Prototype lookup
- Keeps lookup implementation inside Spring

---

# 15. Disadvantages

## Dependency Injection

Dependency Injection can introduce additional complexity.

For very small applications, extensive dependency configuration can sometimes feel unnecessary.

---

## ObjectProvider

Main disadvantages:

- Introduces Spring-specific API into application code
- Explicit lookup can make dependencies less obvious
- Can encourage service locator-like patterns if overused

Example:

    provider.getObject();

The class is no longer working exclusively with the dependency abstraction.

---

## Scoped Proxy

Main disadvantages:

- Introduces proxy behavior
- Can make debugging harder
- Proxy limitations may apply
- Behavior can be less obvious to developers unfamiliar with Spring

Conceptually:

    service
       |
       +---- proxy
                |
                +---- real object

---

## @Lookup

Main disadvantages:

- Relies on Spring-generated subclasses
- More implicit behavior
- Can be confusing during debugging
- Has limitations with final classes/methods
- Less common in modern Spring applications

---

# 16. Recommended Mental Model

Think about the problem this way:

### Direct Injection

    Singleton
        |
        +---- Prototype instance

One dependency reference is injected.

### ObjectProvider

    Singleton
        |
        +---- Provider
                |
                +---- getObject()
                +---- getObject()
                +---- getObject()

The provider asks Spring for instances.

### Scoped Proxy

    Singleton
        |
        +---- Proxy
                |
                +---- Real object

The proxy hides the scope management.

### @Lookup

    Singleton
        |
        +---- Lookup method
                |
                +---- Spring Container
                        |
                        +---- New object

Spring implements the lookup behavior.

---

# 17. Practical Recommendation

For most modern Spring applications:

### Use ObjectProvider when:

- You explicitly need a new instance
- You want lazy dependency retrieval
- You need optional dependency handling
- You want clear control over when the dependency is requested

### Use Scoped Proxy when:

- You need request/session/application scoped dependencies
- You want transparent access to a scoped bean
- You do not want business code to perform explicit lookups

### Use @Lookup when:

- You specifically need method injection
- You want Spring to perform the lookup transparently
- The use case fits Spring's subclass-based lookup mechanism

---

# 18. Key Takeaways

1. Dependency Injection separates object creation from object usage.

2. A Singleton directly injecting a Prototype does not automatically receive a new Prototype instance on every method call.

3. `ObjectProvider` allows explicit retrieval of a dependency from the Spring container.

4. Scoped Proxy injects a proxy that resolves the appropriate scoped object.

5. `@Lookup` allows Spring to override a method and perform dependency lookup.

6. `ObjectProvider` is explicit.

7. Scoped Proxy is transparent.

8. `@Lookup` is implicit and relies on Spring-generated behavior.

9. Request-scoped dependencies are a common practical use case for Scoped Proxy.

10. Prototype dependencies inside Singleton beans are a classic example of when these mechanisms become necessary.

### Interview Summary

If asked:

> "How can a Singleton use a Prototype bean and get a new instance when needed?"

A strong answer is:

> "A Prototype bean injected directly into a Singleton is resolved when the Singleton is created, so the same instance would normally be reused. If I need a new Prototype instance on demand, I can use `ObjectProvider`, a scoped proxy, or `@Lookup` method injection. `ObjectProvider` gives explicit control over retrieval, scoped proxies provide transparent scope-aware access, and `@Lookup` lets Spring implement a lookup method dynamically."
