# 35 - Lombok vs Records

## 1. What is Lombok?

**Project Lombok** is a Java library that reduces boilerplate code by generating common methods and constructors automatically during compilation.

Without Lombok, a simple Java class may require:

- Getters and setters
- Constructors
- `equals()`
- `hashCode()`
- `toString()`
- Builder implementations
- Logging fields

With Lombok, annotations can generate much of this code automatically.

For example, instead of manually writing getters, setters, constructor, `equals()`, `hashCode()` and `toString()`, you can annotate a class with:

`@Data`

The important point is:

> Lombok does not change Java itself. It generates code that the Java compiler can use.

---

# 2. Lombok in a Spring Boot Project

Lombok is very common in Spring Boot applications because Spring applications frequently contain classes with repetitive boilerplate.

A typical Maven dependency is:

`org.projectlombok:lombok`

Usually Lombok is configured with compile-time scope because it is primarily needed during compilation.

Example:

`@Getter`
`@Setter`
`@NoArgsConstructor`
`@AllArgsConstructor`

`public class User {`
`    private Long id;`
`    private String name;`
`}`

Conceptually, Lombok causes the compiler to see something similar to:

`public class User {`
`    private Long id;`
`    private String name;`

`    public User() {}`
`` `    public User(Long id, String name) {`
`        this.id = id;`
`        this.name = name;`
`    }` ``
`    public Long getId() {`
`        return id;`
`    }`
`` `    public void setId(Long id) {`
`        this.id = id;`
`    }` ``
`    // ...`
`}`

The developer writes less code, but the resulting compiled application still contains normal Java methods.

---

# 3. Major Lombok Annotations

## @Getter

Generates getter methods.

`@Getter`
`public class User {`
`    private String name;`
`}`

Conceptually generates:

`public String getName() {`
`    return name;`
`}`

---

## @Setter

Generates setter methods.

`@Setter`
`public class User {`
`    private String name;`
`}`

Conceptually:

`public void setName(String name) {`
`    this.name = name;`
`}`

---

## @Data

One of the most commonly used Lombok annotations.

`@Data`

Effectively combines several features:

- `@Getter`
- `@Setter`
- `@RequiredArgsConstructor`
- `@ToString`
- `@EqualsAndHashCode`

Example:

`@Data`
`public class User {`
`    private Long id;`
`    private String name;`
`}`

### Important consideration

`@Data` is convenient, but it can generate more behavior than you actually want.

For example, automatically generated `equals()` and `hashCode()` can be problematic for certain JPA entities.

Therefore, many teams prefer explicit annotations:

`@Getter`
`@Setter`

instead of blindly using:

`@Data`

---

# 4. @Value

`@Value` is designed for immutable objects.

It is conceptually similar to:

- `final` fields
- Getters
- Required constructor
- `equals()`
- `hashCode()`
- `toString()`

Example:

`@Value`
`public class Address {`
`    String city;`
`    String country;`
`}`

The fields are effectively final and there are no setters.

This makes `@Value` useful for immutable value objects.

---

# 5. Constructors

## @NoArgsConstructor

Generates a no-argument constructor.

`@NoArgsConstructor`
`public class User {`
`    private String name;`
`}`

---

## @AllArgsConstructor

Generates a constructor containing all fields.

`@AllArgsConstructor`
`public class User {`
`    private Long id;`
`    private String name;`
`}`

Conceptually:

`new User(10L, "Bruno");`

---

## @RequiredArgsConstructor

Generates a constructor for required fields, generally final fields and fields marked with `@NonNull`.

Example:

`@RequiredArgsConstructor`
`public class UserService {`
`    private final UserRepository repository;`
`}`

This is particularly common in Spring dependency injection.

Instead of:

`private final UserRepository repository;`

`public UserService(UserRepository repository) {`
`    this.repository = repository;`
`}`

you can simply use:

`@RequiredArgsConstructor`

---

# 6. @Builder

Generates the Builder pattern.

Example:

`@Builder`
`public class User {`
`    private Long id;`
`    private String name;`
`    private String email;`
`}`

Usage:

`User user = User.builder()`
`    .id(10L)`
`    .name("Bruno")`
`    .email("bruno@example.com")`
`    .build();`

Without Lombok, implementing the complete Builder pattern manually requires significantly more code.

---

# 7. @ToString

Generates a `toString()` implementation.

Example:

`@ToString`
`public class User {`
`    private Long id;`
`    private String name;`
`}`

Result conceptually:

`User(id=10, name=Bruno)`

Be careful with sensitive data.

For example, you generally do not want passwords, tokens or secrets appearing in logs.

Lombok provides:

`@ToString.Exclude`

to exclude fields.

---

# 8. @EqualsAndHashCode

Generates:

- `equals()`
- `hashCode()`

Example:

`@EqualsAndHashCode`
`public class User {`
`    private Long id;`
`    private String name;`
`}`

This is useful, but developers need to understand what fields participate in equality.

This becomes particularly important with:

- JPA entities
- Hibernate proxies
- Mutable objects
- Collections such as `HashSet`
- Entity identifiers generated by databases

---

# 9. @Slf4j

Creates a logger automatically.

Instead of manually declaring a logger, you can use:

`@Slf4j`

Then:

`log.info("User created: {}", userId);`

Lombok generates the appropriate logger field.

Other logging annotations include:

- `@Log`
- `@Log4j2`
- `@Slf4j`
- `@XSlf4j`

---

# 10. @SneakyThrows

`@SneakyThrows` allows checked exceptions to be thrown without explicitly declaring them.

Example:

`@SneakyThrows`
`public void process() {`
`    ...`
`}`

This can reduce boilerplate, but it should be used carefully.

It can make exception behavior less obvious to someone reading the method.

---

# 11. @NonNull

`@NonNull` can generate null checks.

Example:

`public User(@NonNull String name) {`
`    this.name = name;`
`}`

Conceptually, Lombok generates a null check.

This is different from Bean Validation annotations such as:

`@NotNull`

because Bean Validation is generally part of runtime/application validation, while Lombok's `@NonNull` is primarily about generated Java code.

---

# 12. How Lombok Works Internally

This is one of the most important concepts for interviews.

Lombok operates during compilation.

The general idea is:

`Java source code`
`       ↓`
`Java compiler`
`       ↓`
`Lombok annotation processing / compiler integration`
`       ↓`
`Generated members`
`       ↓`
`Bytecode`
`       ↓`
`JVM`

For example:

`@Getter`
`class User {`
`    private String name;`
`}`

The developer does not explicitly write:

`getName()`

but Lombok causes the compiled class to contain the corresponding method.

The JVM does not need to understand Lombok.

This is a key distinction:

> Lombok is primarily a compile-time tool, not a runtime framework.

Once the application is compiled, the JVM executes ordinary Java bytecode.

---

# 13. Does Lombok Use Reflection?

Not in the sense that Spring commonly uses reflection at runtime to implement the generated getters or setters.

Lombok generates source-level/compiler-level structures during compilation.

After compilation, the resulting class contains ordinary Java members.

Therefore:

`@Getter`

does not mean:

"At runtime, Lombok dynamically intercepts getName()."

Instead, it means approximately:

"During compilation, generate getName()."

---

# 14. Pros of Lombok

## Less Boilerplate

The biggest advantage.

Instead of writing hundreds of repetitive lines, developers can express intent using annotations.

---

## Better Developer Productivity

Constructors, getters, setters, builders and logging can be generated quickly.

---

## Cleaner Domain Classes

A class can focus more on its actual business behavior.

For example:

`@Builder`
`@Getter`
`public class Order {`
`    ...`
`}`

is much easier to read than a large amount of generated boilerplate.

---

## Builder Pattern Becomes Easy

Implementing a robust Builder manually requires additional code.

Lombok makes it almost trivial.

---

# 15. Cons of Lombok

## Hidden Code

This is probably the biggest criticism.

When you see:

`@Data`

you need to know what Lombok generates.

The actual class behavior is not completely visible from the source.

---

## IDE Dependency

Modern IDEs support Lombok well, but the development experience depends on proper IDE/compiler integration.

Problems can occur when:

- Lombok version changes
- Java version changes
- IDE plugins are incompatible
- Build configuration is incorrect

---

## Debugging Can Be Less Obvious

A developer may debug a generated method without immediately realizing that the method was generated by Lombok.

---

## Overuse

Using Lombok everywhere can hide important design decisions.

For example:

`@Data`

on a complex JPA entity may generate methods that are not appropriate for the entity's lifecycle or equality semantics.

---

# 16. Why Do People Say Lombok Is "Heavy"?

When developers say Lombok is "heavy", they usually do not mean that Lombok adds a large runtime library that makes the application slow.

The criticism is often about:

- Build/compiler integration
- IDE integration
- Generated code
- Annotation processing
- Compatibility with Java versions
- Compatibility with compiler internals
- Debugging complexity
- Hidden behavior

Lombok historically interacts quite deeply with Java compiler internals.

That is different from a typical annotation processor that simply generates additional source files.

This deeper integration is one reason Lombok can sometimes be more sensitive to changes in:

- `javac`
- Java versions
- IDEs
- compiler tooling

However:

> Lombok is generally not considered a significant runtime performance overhead simply because a project uses Lombok.

The main trade-off is developer/build complexity rather than runtime execution cost.

---

# 17. Java Records: What Are They?

Java Records were introduced as a language feature for modeling simple data-oriented classes.

Example:

`public record User(Long id, String name) {`

This automatically provides important functionality associated with the record components, including:

- Private final fields
- Accessor methods
- Canonical constructor
- `equals()`
- `hashCode()`
- `toString()`

For example:

`User user = new User(10L, "Bruno");`

Access values using:

`user.id()`
`user.name()`

Notice that records use:

`user.name()`

instead of the JavaBean-style:

`user.getName()`

---

# 18. What Does a Record Replace?

Conceptually:

`public record User(Long id, String name) {}`

represents a compact form of a data carrier.

It is roughly equivalent to writing a class with:

`private final Long id;`
`private final String name;`

plus:

- Constructor
- Accessors
- `equals()`
- `hashCode()`
- `toString()`

The compiler generates these members.

The important difference is that Records are a native Java language feature.

Lombok is an external library.

---

# 19. Records Are Not Just "Lombok Without Annotations"

This distinction is important.

Lombok:

`@Getter`
`@Setter`
`@Builder`
`@Data`

is a library that generates code.

Records are part of the Java language itself.

That means the compiler, JVM ecosystem and Java tooling understand Records directly.

You do not need Lombok to create a Record.

---

# 20. Mutability: The Biggest Difference

This is probably the most important conceptual difference.

A traditional Lombok class can be mutable.

Example:

`@Getter`
`@Setter`
`public class User {`
`    private String name;`
`}`

You can do:

`user.setName("Carlos");`

The object changes state.

A Record has final components.

Example:

`public record User(String name) {}`

You cannot do:

`user.setName("Carlos");`

because there is no setter.

Instead, to represent a different value, you generally create another Record instance:

`User updated = new User("Carlos");`

---

# 21. Important: Record Does Not Mean Deep Immutability

This is a common interview trap.

Consider:

`public record User(List<String> roles) {}`

The record component reference is final.

But the List itself may still be mutable.

For example, the following conceptually remains possible:

`user.roles().add("ADMIN");`

if the provided list is mutable.

Therefore:

> Records provide shallow immutability of their components/references, not automatic deep immutability of the entire object graph.

If deep immutability is required, the contained objects must also be immutable or defensively copied.

---

# 22. Record Accessors Are Different

Traditional JavaBean:

`user.getName()`

Record:

`user.name()`

This can matter when integrating with frameworks or libraries that expect JavaBean conventions.

Modern frameworks generally support Records well, but you should always verify framework-specific behavior.

---

# 23. Can Records Have Methods?

Yes.

A Record is not simply a bag of fields.

Example:

`public record User(String name) {`

`    public String displayName() {`
`        return name.toUpperCase();`
`    }`
`}`

Records can contain:

- Methods
- Static fields
- Static methods
- Validation logic
- Compact constructors
- Interfaces

They are restricted from extending arbitrary classes because Records already extend `java.lang.Record`.

---

# 24. Record Validation

Records can validate their constructor arguments.

Example:

`public record User(String name, int age) {`

`    public User {`
`        if (age < 0) {`
`            throw new IllegalArgumentException("Age cannot be negative");`
`        }`
`    }`
`}`

This is called a compact canonical constructor.

It is useful for enforcing invariants at object creation.

---

# 25. Is Record a Replacement for Lombok?

**No.**

Records and Lombok solve overlapping but different problems.

Records are primarily a Java language feature for concise data-oriented types.

Lombok is a general-purpose code-generation library.

A Record does not provide all the capabilities Lombok provides.

For example, Lombok supports features such as:

- `@Builder`
- `@Slf4j`
- `@With`
- `@Delegate`
- `@RequiredArgsConstructor`
- `@Getter`
- `@Setter`
- `@SneakyThrows`

A Record does not automatically replace all of these.

---

# 26. Record vs Lombok: Conceptual Comparison

| Feature                 | Lombok Class            | Java Record                    |
| ----------------------- | ----------------------- | ------------------------------ |
| Native Java feature     | No                      | Yes                            |
| External dependency     | Yes                     | No                             |
| Boilerplate reduction   | Yes                     | Yes                            |
| Mutable objects         | Yes                     | Not directly                   |
| Final components        | Optional                | Yes                            |
| Getters                 | `getName()`             | `name()`                       |
| Setters                 | Supported               | No                             |
| Builder                 | `@Builder`              | Not built-in                   |
| `equals()`              | Generated by annotation | Automatically provided         |
| `hashCode()`            | Generated by annotation | Automatically provided         |
| `toString()`            | Generated by annotation | Automatically provided         |
| Custom methods          | Yes                     | Yes                            |
| Extends arbitrary class | Yes                     | No                             |
| Implements interfaces   | Yes                     | Yes                            |
| Best for                | General classes         | Data-oriented immutable models |

---

# 27. When to Use Lombok?

Lombok can be useful when you need a normal class with features such as:

- Mutable state
- Dependency injection constructors
- Builders
- Getters/setters
- Logging
- More flexible class inheritance
- Framework-specific class structures

For example, a Spring service:

`@Service`
`@RequiredArgsConstructor`
`public class PaymentService {`
`    private final PaymentRepository repository;`
`}`

This is a very reasonable Lombok use case.

---

# 28. When to Use Records?

Records are particularly useful for:

- DTOs
- API request/response models
- Read-only projections
- Value objects
- Configuration-like data structures
- Events
- Messages
- Simple data carriers

Example:

`public record UserResponse(`
`    Long id,`
`    String name,`
`    String email`
`) {}`

This clearly communicates:

> This type primarily represents data, and its state should not be changed after construction.

---

# 29. Records and Spring Boot DTOs

Records are especially attractive for DTOs.

Traditional DTO:

`@Getter`
`@Setter`
`@NoArgsConstructor`
`@AllArgsConstructor`
`public class UserResponse {`
`    private Long id;`
`    private String name;`
`}`

Record:

`public record UserResponse(`
`    Long id,`
`    String name`
`) {}`

The Record communicates the DTO's purpose more directly.

This is one reason Records have become increasingly common in modern Spring Boot applications.

---

# 30. Should You Use Records for JPA Entities?

Usually, Records are not a direct replacement for traditional JPA entity classes.

JPA entities often need:

- Mutable state
- Entity lifecycle management
- Proxies
- A no-argument constructor
- Framework-managed identity
- Specific persistence behavior

Records are designed around immutable data carriers and have language-level restrictions that make them a poor fit for typical JPA entity modeling.

A common architecture is:

`Database`
`   ↓`
`JPA Entity`
`   ↓`
`Mapping`
`   ↓`
`Record DTO`
`   ↓`
`REST API`

For example:

`UserEntity`

can represent persistence state, while:

`UserResponse`

can be a Record representing data exposed by the API.

---

# 31. A Useful Spring Boot Architecture

A practical pattern is:

`Entity`
`    ↓`
`Service`
`    ↓`
`Mapper`
`    ↓`
`Record DTO`
`    ↓`
`Controller`

For example:

`UserEntity`

represents the database model.

`UserResponse`

represents the API response:

`public record UserResponse(`
`    Long id,`
`    String name,`
`    String email`
`) {}`

This separation avoids exposing persistence entities directly through the API.

---

# 32. Lombok + Records Can Coexist

Using Records does not mean Lombok must disappear from the project.

You can use:

`@RequiredArgsConstructor`

for Spring dependency injection classes while using Records for DTOs.

For example:

`@Service`
`@RequiredArgsConstructor`
`public class UserService {`
`    private final UserRepository repository;`
`}`

and:

`public record UserResponse(`
`    Long id,`
`    String name`
`) {}`

This is often a practical combination.

---

# 33. When Should You Prefer a Normal Class?

Use a normal class when the object needs:

- Mutable state
- Setters
- Complex lifecycle
- Inheritance from another class
- Framework requirements
- Complex construction patterns
- Identity that differs from its complete state

Example:

`public class ShoppingCart {`
`    private final List<Item> items = new ArrayList<>();`

`    public void add(Item item) {`
`        items.add(item);`
`    }`
`}`

A shopping cart is not simply a static data carrier. Its state changes through behavior.

A Record would not naturally represent this model.

---

# 34. When Should You Prefer a Record?

Use a Record when the type's main purpose is:

> "These values together represent one piece of data."

Examples:

`public record Coordinates(double latitude, double longitude) {}`

`public record Money(BigDecimal amount, Currency currency) {}`

`public record UserResponse(Long id, String name) {}`

`public record PaymentEvent(Long paymentId, String status) {}`

These types are naturally modeled as immutable values.

---

# 35. Interview Perspective

A strong interview answer is not:

> "Records replace Lombok."

A better answer is:

> "Records and Lombok address different problems. Records are a native Java feature designed primarily for concise data carriers with final components and automatically generated value-based methods. Lombok is a compile-time code-generation library that can reduce boilerplate in general-purpose classes, including mutable classes, builders, constructors, logging and accessors."

Then mention:

- Records are native to Java.
- Lombok is an external dependency.
- Records naturally model immutable data carriers.
- Lombok provides much broader code-generation features.
- Records do not provide setters.
- Records do not provide a built-in Builder.
- Records are usually excellent for DTOs.
- Lombok remains useful for services, entities and other general-purpose classes.

---

# 36. The Big Picture

Think about the two technologies this way:

**Lombok**

"Generate the Java code I don't want to write."

**Record**

"This type is fundamentally a data carrier."

That distinction is more important than memorizing individual annotations.

---

# 37. Decision Guide

| Situation                     | Good Choice                    |
| ----------------------------- | ------------------------------ |
| REST response DTO             | Record                         |
| REST request DTO              | Record                         |
| Immutable value object        | Record                         |
| Event/message object          | Record                         |
| Simple projection             | Record                         |
| Spring Service                | Class + Lombok                 |
| Spring Repository             | Interface                      |
| Mutable domain object         | Class                          |
| JPA Entity                    | Usually Class                  |
| Need setters                  | Class + Lombok                 |
| Need `@Builder`               | Lombok                         |
| Need automatic logger         | Lombok                         |
| Need constructor injection    | Lombok or explicit constructor |
| Need Java-native data carrier | Record                         |

---

# 38. Final Mental Model

The easiest way to remember the difference:

**Lombok = code generation**

**Record = language-level data modeling**

Lombok asks:

> "Which repetitive Java code can the compiler generate for me?"

Record asks:

> "Is this type fundamentally a compact representation of immutable data?"

They can coexist.

A modern Spring Boot application might legitimately use:

`@RequiredArgsConstructor`

for dependency injection,

`@Slf4j`

for logging,

and Records for DTOs and events.

The goal is not to eliminate Lombok or use Records everywhere.

The goal is to choose the construct that best communicates the intended design of each type.
