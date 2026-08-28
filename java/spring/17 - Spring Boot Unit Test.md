# 17 - Spring Boot Unit Test

## 1. Introduction to Unit Testing

**Unit Testing** is the practice of testing a small, isolated piece of code, usually a single method or class, to verify that it behaves as expected.

In a Java application, the most common unit-testing tools are:

- **JUnit 5** — test framework used to define and execute tests.
- **Mockito** — mocking framework used to create test doubles and isolate the class under test.
- **AssertJ** — optional assertion library that provides a fluent API for assertions.
- **Spring Boot Test** — provides integration-testing capabilities when we need to test Spring components together.

The main idea behind a unit test is:

> Test one unit of behavior in isolation.

For example, suppose we have:

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User createUser(User user) {
        return userRepository.save(user);
    }
}
```

A unit test should test the behavior of `UserService` without requiring:

- A real database
- A running Spring application
- HTTP requests
- External APIs
- Other infrastructure

Instead, we can replace `UserRepository` with a Mockito mock.

---

# 2. Why Unit Tests Are Important

Unit tests provide several important benefits.

## 2.1 Catch Bugs Early

A unit test can detect a regression immediately after a change is introduced.

For example:

```java
@Test
void shouldCalculateTotalPrice() {
    var result = calculator.calculate(100, 10);

    assertEquals(110, result);
}
```

If someone changes the implementation and the result becomes `100`, the test immediately fails.

---

## 2.2 Enable Refactoring

Good unit tests give developers confidence when changing existing code.

Imagine we have a complex service with hundreds of lines of code.

Without tests:

```text
Change code
   ↓
Hope nothing broke
   ↓
Deploy
   ↓
Find out later
```

With tests:

```text
Change code
   ↓
Run tests
   ↓
Tests pass
   ↓
Much higher confidence
```

Unit tests therefore act as a safety net during refactoring.

---

## 2.3 Document Expected Behavior

Tests can also serve as executable documentation.

For example:

```java
@Test
void shouldRejectUserWhenEmailAlreadyExists() {
    // ...
}
```

Even without reading the implementation, we can understand an important business rule:

> A user cannot be created when the email already exists.

This is one of the strongest benefits of well-written tests.

---

## 2.4 Improve Code Design

Writing unit tests often exposes poorly designed code.

For example:

```java
public class UserService {

    public void createUser() {
        // Database
        // HTTP request
        // File system
        // Business logic
        // Email
        // Logging
        // ...
    }
}
```

This class is difficult to test because it has too many responsibilities.

After applying separation of concerns:

```java
UserService
    ↓
UserRepository

UserService
    ↓
EmailService

UserService
    ↓
UserValidator
```

Each dependency can be mocked independently.

Therefore:

> Testability is often a signal of good software design.

---

# 3. Types of Tests

Before focusing on unit tests, it is important to understand where they fit in the testing pyramid.

```text
             /\
            /  \
           / E2E\
          /------\
         /  Integ \
        /----------\
       /   Unit     \
      /--------------\
```

## Unit Tests

Test a small piece of code in isolation.

Example:

```text
UserService
    ↓
Mockito Mock
    ↓
UserRepository
```

Characteristics:

- Very fast
- Isolated
- Usually no Spring context
- No real database
- No network calls

---

## Integration Tests

Test multiple components working together.

For example:

```text
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

Integration tests are slower but verify that components actually integrate correctly.

---

## End-to-End Tests

Test the complete application from the perspective of the user or another system.

Example:

```text
HTTP Request
    ↓
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
    ↓
HTTP Response
```

These tests provide high confidence but are more expensive and slower.

---

# 4. Getting Started with Unit Testing in a Spring Boot Application

Spring Boot applications typically use JUnit 5.

A typical Maven project can include:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

`spring-boot-starter-test` provides several testing libraries, including:

- JUnit 5
- Mockito
- Spring Test
- AssertJ

You normally don't need to add JUnit and Mockito separately when using this starter.

---

# 5. Test Class Structure

Suppose we have:

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User findById(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

A unit test can look like:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

}
```

Let's understand these annotations.

---

# 6. JUnit 5

JUnit 5 is the testing framework.

A basic test looks like:

```java
@Test
void shouldCalculateTotal() {
    int result = 10 + 20;

    assertEquals(30, result);
}
```

The `@Test` annotation tells JUnit:

> This method is a test case.

---

## 6.1 Assertions

Assertions verify the expected result.

### assertEquals

```java
assertEquals(30, result);
```

Checks that two values are equal.

---

### assertNotNull

```java
assertNotNull(user);
```

Checks that an object is not `null`.

---

### assertNull

```java
assertNull(user);
```

Checks that an object is `null`.

---

### assertTrue

```java
assertTrue(user.isActive());
```

Checks that an expression is true.

---

### assertFalse

```java
assertFalse(user.isDeleted());
```

Checks that an expression is false.

---

### assertThrows

Very useful when testing exceptions:

```java
assertThrows(
    UserNotFoundException.class,
    () -> userService.findById(10L)
);
```

This verifies that the method throws the expected exception.

---

# 7. Mockito

Mockito allows us to create mock objects.

Suppose:

```java
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

The `UserService` depends on `UserRepository`.

We don't want to use the real repository during a unit test.

Instead:

```java
@Mock
private UserRepository userRepository;
```

Mockito creates a fake implementation of the repository.

---

# 8. @Mock

`@Mock` creates a Mockito mock.

```java
@Mock
private UserRepository userRepository;
```

The mock doesn't execute the real repository implementation.

For example:

```java
userRepository.findById(1L);
```

will return `null` by default unless we configure it.

We can define the behavior:

```java
when(userRepository.findById(1L))
        .thenReturn(Optional.of(user));
```

Now Mockito knows:

> When `findById(1L)` is called, return this user.

---

# 9. @InjectMocks

`@InjectMocks` creates the class under test and injects the mocks into it.

```java
@Mock
private UserRepository userRepository;

@InjectMocks
private UserService userService;
```

Conceptually:

```text
Mockito

UserRepository Mock
       │
       ▼
UserService
```

This allows us to test `UserService` without creating a real `UserRepository`.

---

# 10. @ExtendWith(MockitoExtension.class)

JUnit needs to know that Mockito should initialize the annotations.

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
}
```

This enables:

```java
@Mock
@InjectMocks
@Spy
@Captor
```

for the test class.

---

# 11. Writing a Complete Unit Test

Let's create a complete example.

Production code:

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User findById(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

Test:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldReturnUserWhenUserExists() {

        User user = new User(1L, "John");

        when(userRepository.findById(1L))
                .thenReturn(Optional.of(user));

        User result = userService.findById(1L);

        assertNotNull(result);
        assertEquals("John", result.getName());
    }
}
```

The flow is:

```text
Arrange
   ↓
Act
   ↓
Assert
```

This is commonly called the **AAA pattern**.

---

# 12. Arrange, Act, Assert

## Arrange

Prepare the test scenario.

```java
User user = new User(1L, "John");

when(userRepository.findById(1L))
        .thenReturn(Optional.of(user));
```

---

## Act

Execute the method being tested.

```java
User result = userService.findById(1L);
```

---

## Assert

Verify the result.

```java
assertNotNull(result);
assertEquals("John", result.getName());
```

A good unit test usually makes these three phases easy to identify.

---

# 13. Testing Exceptions

Suppose the service contains:

```java
public User findById(Long id) {

    return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
}
```

We can test the exception:

```java
@Test
void shouldThrowExceptionWhenUserDoesNotExist() {

    when(userRepository.findById(1L))
            .thenReturn(Optional.empty());

    assertThrows(
        UserNotFoundException.class,
        () -> userService.findById(1L)
    );
}
```

We can also capture the exception:

```java
@Test
void shouldThrowExceptionWithCorrectMessage() {

    when(userRepository.findById(1L))
            .thenReturn(Optional.empty());

    UserNotFoundException exception = assertThrows(
        UserNotFoundException.class,
        () -> userService.findById(1L)
    );

    assertEquals(
        "User 1 was not found",
        exception.getMessage()
    );
}
```

---

# 14. Mockito `when()` and `thenReturn()`

Mockito allows us to define behavior:

```java
when(userRepository.findById(1L))
        .thenReturn(Optional.of(user));
```

This means:

```text
When findById(1L) is called
        ↓
Return Optional.of(user)
```

Another example:

```java
when(userRepository.existsByEmail("john@email.com"))
        .thenReturn(true);
```

---

# 15. Testing Save Operations

Suppose we have:

```java
public User createUser(User user) {

    if (userRepository.existsByEmail(user.getEmail())) {
        throw new EmailAlreadyExistsException();
    }

    return userRepository.save(user);
}
```

Test:

```java
@Test
void shouldCreateUser() {

    User user = new User(null, "John");

    when(userRepository.existsByEmail("john@email.com"))
            .thenReturn(false);

    when(userRepository.save(user))
            .thenReturn(new User(1L, "John"));

    User result = userService.createUser(user);

    assertNotNull(result);
    assertEquals(1L, result.getId());
}
```

---

# 16. Mockito `verify()`

Sometimes we don't only care about the return value.

We also want to verify that a dependency was called correctly.

Example:

```java
verify(userRepository).save(user);
```

This verifies that:

```text
userRepository.save(user)
```

was actually executed.

---

## 16.1 Verify Number of Calls

```java
verify(userRepository, times(1))
        .save(user);
```

We can also verify that something was never called:

```java
verify(userRepository, never())
        .save(user);
```

For example:

```java
@Test
void shouldNotSaveUserWhenEmailAlreadyExists() {

    User user = new User(null, "John");

    when(userRepository.existsByEmail(user.getEmail()))
            .thenReturn(true);

    assertThrows(
        EmailAlreadyExistsException.class,
        () -> userService.createUser(user)
    );

    verify(userRepository, never())
            .save(user);
}
```

This test verifies an important business rule:

```text
Email exists
     ↓
Exception
     ↓
User is NOT saved
```

---

# 17. Mockito Argument Matchers

Mockito provides argument matchers.

For example:

```java
when(userRepository.findById(anyLong()))
        .thenReturn(Optional.of(user));
```

Common matchers include:

```java
any()
anyString()
anyLong()
anyInt()
eq()
isNull()
isNotNull()
```

Example:

```java
when(userRepository.findByEmail(anyString()))
        .thenReturn(Optional.of(user));
```

---

# 18. `eq()` Matcher

Suppose we need to match a specific value:

```java
when(userRepository.findByEmail(
        eq("john@email.com")
)).thenReturn(Optional.of(user));
```

Be careful when mixing matchers and raw values.

Incorrect:

```java
when(repository.find("John", anyLong()));
```

Correct:

```java
when(repository.find(eq("John"), anyLong()));
```

When using matchers, use matchers for all arguments.

---

# 19. Testing Void Methods

Suppose:

```java
public void deleteUser(Long id) {
    userRepository.deleteById(id);
}
```

We can test it with:

```java
@Test
void shouldDeleteUser() {

    userService.deleteUser(1L);

    verify(userRepository)
            .deleteById(1L);
}
```

Since `deleteById()` returns nothing, there is no result to assert.

Instead, we verify the interaction.

---

# 20. Mockito `doThrow()`

For void methods, we can configure exceptions using `doThrow()`.

```java
doThrow(new DatabaseException())
        .when(userRepository)
        .deleteById(1L);
```

Then:

```java
assertThrows(
    DatabaseException.class,
    () -> userService.deleteUser(1L)
);
```

---

# 21. Testing Controllers

Unit testing a controller is different from testing the complete HTTP stack.

Example controller:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

A pure unit test can simply instantiate the controller and mock the service.

```java
@ExtendWith(MockitoExtension.class)
class UserControllerTest {

    @Mock
    private UserService userService;

    @InjectMocks
    private UserController userController;

    @Test
    void shouldReturnUser() {

        User user = new User(1L, "John");

        when(userService.findById(1L))
                .thenReturn(user);

        User result = userController.getUser(1L);

        assertEquals("John", result.getName());

        verify(userService)
                .findById(1L);
    }
}
```

This is a **unit test** because Spring MVC is not running.

---

# 22. Unit Test vs `@SpringBootTest`

One of the most important distinctions in Spring Boot testing is:

```java
@SpringBootTest
```

versus:

```java
@ExtendWith(MockitoExtension.class)
```

## Unit Test

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
}
```

Characteristics:

- Does not start Spring
- Very fast
- Uses Mockito
- Dependencies are mocked
- Tests one class in isolation

---

## Spring Boot Integration Test

```java
@SpringBootTest
class UserServiceIntegrationTest {
}
```

This starts the Spring application context.

Spring creates and wires the beans.

This is useful when we want to test the integration between Spring components.

However, it is generally unnecessary for a simple unit test.

---

# 23. When Should You Use `@SpringBootTest`?

Don't automatically use:

```java
@SpringBootTest
```

for every test.

Use it when you actually need the Spring context.

For example:

```text
Testing Spring configuration
Testing bean wiring
Testing multiple layers together
Testing integration with infrastructure
```

If you only want to test:

```java
UserService
```

a Mockito-based unit test is usually better.

---

# 24. Testing Repository Behavior

Repositories often don't need extensive unit tests.

For example:

```java
userRepository.findByEmail(...)
```

If you're using Spring Data JPA, much of the repository implementation is provided by Spring.

Instead, integration tests can be more valuable when you need to verify:

- Custom queries
- JPQL
- Native queries
- Database mappings
- Relationships
- Constraints

For example:

```java
@DataJpaTest
class UserRepositoryTest {
}
```

`@DataJpaTest` is designed for JPA-related tests and is generally more appropriate than `@SpringBootTest` for this purpose.

---

# 25. Test Naming

Good test names explain the expected behavior.

Avoid:

```java
@Test
void testUser() {
}
```

Prefer:

```java
@Test
void shouldReturnUserWhenUserExists() {
}
```

Or:

```java
@Test
void shouldThrowExceptionWhenUserDoesNotExist() {
}
```

Another common style is:

```java
@Test
void findById_whenUserDoesNotExist_shouldThrowException() {
}
```

The important thing is consistency.

---

# 26. Test One Behavior

A test should ideally verify one behavior.

Avoid:

```java
@Test
void testEverything() {
    // create user
    // update user
    // delete user
    // search user
    // validate email
}
```

Prefer:

```java
@Test
void shouldCreateUser() {
}

@Test
void shouldUpdateUser() {
}

@Test
void shouldDeleteUser() {
}

@Test
void shouldFindUserById() {
}
```

This makes failures easier to understand.

---

# 27. Don't Test Implementation Details

A common mistake is testing _how_ the code works instead of _what_ it does.

Suppose:

```java
public User findById(Long id) {
    return repository.findById(id)
        .orElseThrow(...);
}
```

The important behavior is:

```text
User exists → return user

User does not exist → throw exception
```

We should not create tests that are tightly coupled to internal implementation details that may change during refactoring.

---

# 28. Avoid Excessive Mocking

Mockito is useful, but don't mock everything.

Bad example:

```text
Service
 ↓
Mock A
 ↓
Mock B
 ↓
Mock C
 ↓
Mock D
```

At some point, the test can become disconnected from real application behavior.

Mock dependencies that are external to the unit being tested.

For example:

```text
UserService
   ↓
Mock UserRepository
```

makes sense.

But if you're testing a simple domain object:

```java
User user = new User();
```

there is usually no reason to mock the `User`.

---

# 29. Test Behavior, Not Coverage

Code coverage is useful, but 100% coverage does not automatically mean 100% quality.

For example:

```java
if (user != null) {
    return user;
}
```

A test that simply executes this line may increase coverage.

But a good test verifies meaningful behavior.

The goal should be:

> High confidence, not a high percentage.

---

# 30. Test Edge Cases

Good unit tests should consider more than the happy path.

For example, instead of only testing:

```text
User exists
```

also consider:

```text
User does not exist
Email already exists
Invalid input
Null values
Empty collections
Boundary values
Unexpected dependency failures
```

For example:

```java
@Test
void shouldThrowExceptionWhenUserDoesNotExist() {
}
```

and:

```java
@Test
void shouldNotCreateUserWhenEmailAlreadyExists() {
}
```

---

# 31. Parameterized Tests

JUnit 5 supports parameterized tests.

Instead of writing:

```java
@Test
void shouldRejectInvalidEmail1() {
}

@Test
void shouldRejectInvalidEmail2() {
}

@Test
void shouldRejectInvalidEmail3() {
}
```

we can use:

```java
@ParameterizedTest
@ValueSource(strings = {
    "invalid",
    "test",
    "hello"
})
void shouldRejectInvalidEmail(String email) {

    assertFalse(
        validator.isValid(email)
    );
}
```

This allows the same test logic to run with multiple inputs.

---

# 32. Testing with `@BeforeEach`

JUnit allows common setup before every test.

```java
@BeforeEach
void setUp() {
    user = new User(1L, "John");
}
```

Example:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    private User user;

    @BeforeEach
    void setUp() {
        user = new User(1L, "John");
    }
}
```

Use `@BeforeEach` when the setup is genuinely shared by multiple tests.

Don't move everything into `@BeforeEach` just because you can.

---

# 33. `@BeforeAll` and `@AfterAll`

JUnit also provides:

```java
@BeforeAll
```

and:

```java
@AfterAll
```

`@BeforeAll` runs once before all tests.

`@AfterAll` runs once after all tests.

Example:

```java
@BeforeAll
static void setup() {
    // runs once
}
```

These should be used sparingly because shared state can make tests harder to reason about.

---

# 34. Testing with AssertJ

Spring Boot Test also includes AssertJ.

Instead of:

```java
assertEquals("John", result.getName());
```

we can write:

```java
assertThat(result.getName())
        .isEqualTo("John");
```

AssertJ becomes particularly useful for complex assertions.

For example:

```java
assertThat(users)
        .hasSize(2)
        .extracting(User::getName)
        .containsExactly("John", "Mary");
```

This often makes tests easier to read.

---

# 35. Unit Testing a Service with Multiple Dependencies

Suppose:

```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;

    public UserService(
            UserRepository userRepository,
            EmailService emailService) {

        this.userRepository = userRepository;
        this.emailService = emailService;
    }

    public User createUser(User user) {

        User savedUser = userRepository.save(user);

        emailService.sendWelcomeEmail(savedUser);

        return savedUser;
    }
}
```

The unit test can mock both dependencies:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService userService;
}
```

Then:

```java
@Test
void shouldCreateUserAndSendWelcomeEmail() {

    User user = new User(null, "John");
    User savedUser = new User(1L, "John");

    when(userRepository.save(user))
            .thenReturn(savedUser);

    User result = userService.createUser(user);

    assertEquals(1L, result.getId());

    verify(userRepository)
            .save(user);

    verify(emailService)
            .sendWelcomeEmail(savedUser);
}
```

The real email service is never called.

---

# 36. Testing Interaction Order

Sometimes the order of operations matters.

Mockito provides `InOrder`:

```java
InOrder inOrder = inOrder(
    userRepository,
    emailService
);

inOrder.verify(userRepository)
        .save(user);

inOrder.verify(emailService)
        .sendWelcomeEmail(savedUser);
```

This verifies:

```text
1. Save user
2. Send email
```

Use this only when the order is actually part of the behavior.

Don't verify ordering unnecessarily.

---

# 37. Best Practices

## 37.1 Keep Unit Tests Fast

A unit test should generally execute very quickly.

Avoid:

```text
Real database
Real HTTP calls
Real message broker
Real external APIs
Full Spring context
```

when writing a true unit test.

---

## 37.2 Keep Tests Independent

One test should not depend on another test.

Bad:

```text
testCreateUser()
      ↓
testUpdateUser()
      ↓
testDeleteUser()
```

Tests should be independently executable.

---

## 37.3 Don't Use Shared Mutable State

Avoid having tests modify common objects that other tests depend on.

Prefer creating fresh test data for each test.

---

## 37.4 Make Tests Deterministic

A test should produce the same result every time.

Avoid depending on:

```text
Current time
Random numbers
External APIs
Network
Database state
Machine configuration
```

unless these are explicitly controlled.

For example, instead of:

```java
LocalDateTime.now()
```

consider injecting a `Clock`:

```java
Clock clock;
```

Then the test can control the current time.

---

# 38. Follow the AAA Pattern

A clean test usually follows:

```text
Arrange
Act
Assert
```

Example:

```java
@Test
void shouldReturnUser() {

    // Arrange
    User user = new User(1L, "John");

    when(userRepository.findById(1L))
            .thenReturn(Optional.of(user));

    // Act
    User result = userService.findById(1L);

    // Assert
    assertEquals("John", result.getName());
}
```

This makes tests easy to read.

---

# 39. Use Meaningful Test Data

Avoid meaningless values:

```java
User user = new User(1L, "abc");
```

Prefer:

```java
User user = new User(1L, "John Smith");
```

Meaningful data makes the intent of the test clearer.

For larger applications, test-data builders or factories can help:

```java
User user = UserTestDataFactory.validUser();
```

---

# 40. Don't Overuse `verify()`

This:

```java
verify(repository).save(user);
```

is useful when the interaction itself is important.

But avoid verifying every internal call:

```java
verify(repository).findById(id);
verify(repository).save(user);
verify(repository).existsByEmail(email);
verify(validator).validate(user);
verify(mapper).map(user);
```

This can make tests fragile.

Focus primarily on observable behavior.

---

# 41. Don't Test Framework Behavior

You generally don't need to test whether Spring Data itself can execute:

```java
repository.findById()
```

or whether JUnit can execute:

```java
@Test
```

Test your application logic.

For example:

```text
Good:
"UserService throws UserNotFoundException when user doesn't exist."

Less useful:
"Spring Data repository can execute findById."
```

---

# 42. Unit Test Checklist

When writing a unit test, ask:

### Isolation

- Am I testing one unit?
- Are external dependencies mocked?

### Behavior

- What behavior am I testing?
- What should happen when it succeeds?
- What should happen when it fails?

### Structure

- Is the test easy to understand?
- Does it follow Arrange → Act → Assert?

### Edge Cases

- What happens with invalid input?
- What happens when data doesn't exist?
- What happens when a dependency fails?

### Maintainability

- Is the test coupled to implementation details?
- Will the test survive a reasonable refactoring?

---

# 43. Example: Complete Service Test

Production code:

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User createUser(User user) {

        if (userRepository.existsByEmail(user.getEmail())) {
            throw new EmailAlreadyExistsException(
                    user.getEmail()
            );
        }

        return userRepository.save(user);
    }

    public User findById(Long id) {

        return userRepository.findById(id)
                .orElseThrow(
                    () -> new UserNotFoundException(id)
                );
    }
}
```

Unit tests:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldCreateUserWhenEmailDoesNotExist() {

        User user = new User(
                null,
                "John",
                "john@email.com"
        );

        User savedUser = new User(
                1L,
                "John",
                "john@email.com"
        );

        when(userRepository.existsByEmail("john@email.com"))
                .thenReturn(false);

        when(userRepository.save(user))
                .thenReturn(savedUser);

        User result = userService.createUser(user);

        assertEquals(1L, result.getId());
        assertEquals("John", result.getName());

        verify(userRepository)
                .save(user);
    }

    @Test
    void shouldThrowExceptionWhenEmailAlreadyExists() {

        User user = new User(
                null,
                "John",
                "john@email.com"
        );

        when(userRepository.existsByEmail("john@email.com"))
                .thenReturn(true);

        assertThrows(
            EmailAlreadyExistsException.class,
            () -> userService.createUser(user)
        );

        verify(userRepository, never())
                .save(user);
    }

    @Test
    void shouldReturnUserWhenUserExists() {

        User user = new User(
                1L,
                "John",
                "john@email.com"
        );

        when(userRepository.findById(1L))
                .thenReturn(Optional.of(user));

        User result = userService.findById(1L);

        assertEquals(1L, result.getId());
        assertEquals("John", result.getName());
    }

    @Test
    void shouldThrowExceptionWhenUserDoesNotExist() {

        when(userRepository.findById(1L))
                .thenReturn(Optional.empty());

        assertThrows(
            UserNotFoundException.class,
            () -> userService.findById(1L)
        );
    }
}
```

This gives us four important scenarios:

```text
Create user
    ↓
Email doesn't exist
    ↓
Save user


Create user
    ↓
Email already exists
    ↓
Throw exception


Find user
    ↓
User exists
    ↓
Return user


Find user
    ↓
User doesn't exist
    ↓
Throw exception
```

---

# 44. Final Mental Model

A useful way to think about unit testing in a Spring Boot application is:

```text
                 UNIT TEST
                     │
                     ▼
              Class Under Test
                     │
            ┌────────┴────────┐
            ▼                 ▼
       Mock Dependency   Mock Dependency
            │                 │
            ▼                 ▼
        Repository         Service/API
```

JUnit 5 provides the testing framework:

```text
JUnit 5
  ├── @Test
  ├── @BeforeEach
  ├── Assertions
  └── Parameterized Tests
```

Mockito provides test doubles:

```text
Mockito
  ├── @Mock
  ├── @InjectMocks
  ├── when()
  ├── thenReturn()
  ├── verify()
  └── Argument Matchers
```

Spring Boot provides integration-testing support:

```text
Spring Boot Test
  ├── @SpringBootTest
  ├── @WebMvcTest
  └── @DataJpaTest
```

The key distinction is:

```text
Unit Test
    ↓
Fast + Isolated + Mock dependencies

Integration Test
    ↓
Multiple components + Real infrastructure

End-to-End Test
    ↓
Complete application flow
```

The ultimate goal of unit testing is not simply to achieve a high code-coverage percentage.

The goal is to create **fast, reliable, maintainable tests that give developers confidence that the application's behavior is correct.**
