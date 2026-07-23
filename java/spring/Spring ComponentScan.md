# Spring @ComponentScan Annotation

## What is the @ComponentScan annotation?

It is a Spring annotation responsible for automatically scanning packages and detecting classes managed by Spring (Beans). Such as:

- @RestController
- @Service
- @Repository
- @Component
- @Configuration

In most Spring Boot applications, you do not need to declare @ComponentScan explicitly because @SpringBootApplication already includes it.

@SpringBootApplication is composed of:

- @SpringBootConfiguration
- @EnableAutoConfiguration
- @ComponentScan

## What does it do?

- Scan the configured packages
- Find classes annotated with Spring stereotypes
- Create Bean instances
- Register the Beans in the ApplicationContext
- Resolve dependencies (@Autowired, constructor injection, etc.)

```text
Application starts
   │
   ▼
@ComponentScan
   │
   ▼
Search packages
   │
   ▼
Find annotated classes
   │
   ▼
Create Beans
   │
   ▼
Register in ApplicationContext
```

## Examples

### Basic example

```
@Configuration
@ComponentScan("com.mycompany")
public class AppConfig {

}
```

### Scanning multiple packages

```
@Configuration
@ComponentScan({
  "com.mycompany.api",
  "com.mycompany.secret",
  "com.mycompany.payment"
})
public class AppConfig {

}
```

## Filtering components

It is possible to include or exclude specific classes. This feature is commonly used in advanced configurations or tests.

```
# Exclude
@ComponentScan(
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = Repository.class
    )
)
```

```
# Include
@ComponentScan(
    includeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = Repository.class
    )
)
```

## Common mistakes

### Bean not found - NoSuchBeanDefinitionException

Root cause:

- the class is outside the scanned package.
- missing annotation: @Component, @Service, etc

### Scanning unnecessary packages

This causes Spring to scan a large number of unnecessary classes, increasing the start of the application and causing conflicts. Prefered to be more specific

```
# Wrong
@ComponentScan("com")
```

```
# Right
@ComponentScan("com.mycompany")
```

### Duplicate Beans

Two components implementing the same interface can cause: NoUniqueBeanDefinitionException

```
@Service
class PaypalService implements PaymentService {

}

@Service
class StripeService implements PaymentService{

}

# your application calls
@Autowired
PaymentService paymentService;
```

This error is not caused directly by @ComponentScan. It occurs because component scanning registers both Beans in the ApplicationContext.

## Default Scanning Behavior

By default, @ComponentScan scans the package where the main application class is located and all of its subpackages.

Example:

com.example
├── Application
├── controller
├── service
└── repository

Since `Application` is located in `com.example`, all subpackages are scanned automatically.

If a Bean is located outside this package hierarchy, it will not be detected unless you explicitly configure `@ComponentScan`.
