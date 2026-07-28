# 09 - Spring Exception Handlers and Controller Advice

## Traditional Way of Handling Exceptions

In a basic Spring application, exceptions are often handled directly inside controller methods using `try-catch` blocks.

Example:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public User findById(@PathVariable Long id) {

        try {
            return userService.findById(id);
        } catch (UserNotFoundException ex) {
            throw new ResponseStatusException(
                    HttpStatus.NOT_FOUND,
                    ex.getMessage()
            );
        }
    }
}
```

### Problems with this approach

- Repeated code across multiple controllers.
- Difficult to maintain.
- Business logic becomes mixed with error handling.
- Inconsistent error responses.
- Violates the Separation of Concerns principle.

As the application grows, this approach becomes hard to manage.

---

## Exception Handler

Spring provides the `@ExceptionHandler` annotation to centralize exception handling inside a controller.

Example:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public User findById(@PathVariable Long id) {
        return userService.findById(id);
    }

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<String> handleUserNotFound(
            UserNotFoundException ex) {

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(ex.getMessage());
    }
}
```

### How it works

When a `UserNotFoundException` is thrown:

1. Spring looks for an `@ExceptionHandler`.
2. The matching handler method is executed.
3. A custom HTTP response is returned.

### Advantages

- Removes `try-catch` blocks from controller methods.
- Cleaner controller code.
- Centralized exception handling within a controller.
- Easier maintenance.

### Limitation

The exception handler only works for the controller where it is declared.

If you have multiple controllers, you may end up duplicating the same exception handling code.

---

## Need of Controller Advice and Implementation

To avoid duplicating exception handlers across controllers, Spring provides the `@ControllerAdvice` annotation.

`@ControllerAdvice` acts as a global exception handler for all controllers in the application.

### Why use Controller Advice?

Without `@ControllerAdvice`:

```text
UserController
 └── @ExceptionHandler(UserNotFoundException)

ProductController
 └── @ExceptionHandler(UserNotFoundException)

OrderController
 └── @ExceptionHandler(UserNotFoundException)
```

The same code is repeated multiple times.

With `@ControllerAdvice`:

```text
GlobalExceptionHandler
 └── @ExceptionHandler(UserNotFoundException)

UserController
ProductController
OrderController
```

One handler serves the entire application.

---

### Implementation Example

#### Custom Exception

```java
public class UserNotFoundException extends RuntimeException {

    public UserNotFoundException(String message) {
        super(message);
    }
}
```

#### Global Exception Handler

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<String> handleUserNotFound(
            UserNotFoundException ex) {

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(ex.getMessage());
    }
}
```

#### Controller

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public User findById(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

When the exception is thrown:

```java
throw new UserNotFoundException("User not found");
```

Spring automatically routes the exception to the global handler.

Response:

```http
HTTP/1.1 404 Not Found

User not found
```

---

## Returning a Custom Error Response

Instead of returning a simple string, it is common to return a structured JSON response.

### Error DTO

```java
public record ErrorResponse(
        String message,
        int status,
        LocalDateTime timestamp) {
}
```

### Handler

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException ex) {

        ErrorResponse error = new ErrorResponse(
                ex.getMessage(),
                HttpStatus.NOT_FOUND.value(),
                LocalDateTime.now()
        );

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(error);
    }
}
```

Response:

```json
{
  "message": "User not found",
  "status": 404,
  "timestamp": "2026-07-27T20:30:00"
}
```

---

## @ControllerAdvice vs @RestControllerAdvice

### @ControllerAdvice

Used for MVC applications and can return:

- Views
- Model objects
- ResponseEntity

```java
@ControllerAdvice
public class GlobalExceptionHandler {
}
```

### @RestControllerAdvice

Combination of:

```java
@ControllerAdvice
@ResponseBody
```

Recommended for REST APIs because responses are automatically serialized to JSON.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
}
```

---

## Summary

| Approach | Scope | Recommended |
|-----------|---------|------------|
| Try-Catch in Controller | Single method | ❌ No |
| `@ExceptionHandler` | Single controller | ⚠️ Small applications |
| `@ControllerAdvice` | Multiple controllers | ✅ Yes |
| `@RestControllerAdvice` | REST APIs | ✅ Best Practice |

---

## Best Practices

- Create custom exceptions for business scenarios.
- Throw exceptions from the service layer.
- Handle exceptions globally using `@RestControllerAdvice`.
- Return standardized JSON error responses.
- Keep controllers focused on request processing and business flow.
- Avoid duplicating exception handling logic across controllers.
- Include useful information in error responses, such as:
  - Error message
  - HTTP status code
  - Timestamp
  - Request path (optional)
  - Error code (optional)

For modern Spring Boot REST APIs, `@RestControllerAdvice` combined with `@ExceptionHandler` is the recommended approach for centralized and maintainable exception handling.