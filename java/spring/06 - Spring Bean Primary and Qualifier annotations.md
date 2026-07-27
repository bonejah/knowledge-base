# Spring Bean @Primary and @Qualifier annotations

Spring inject dependencies by type default.
When there is only one Bean of a given type, Spring
knows exactly which Bean should be injected.

## What is the problem?

```
@Service
public class EmailNotificationService implements NotificationService {

}

@Service
public class UserService {
  private final NotificationService notificationService;

  public UserService(NotificationService notificationService) {
    this.notificationService = notificationService;
  }
}

# Since there is only one implementation of NotificationService, Spring injects automatically.
```

```
# The problem happens when there are multiple Beans of the same type.

@Service
public class EmailNotificationService implements NotificationService {
}

@Service
public class SmsNotificationService implements NotificationService {
}
```

```
# Now Spring doesn't know which implementation to inject.
@Service
public class UserService {

    private final NotificationService notificationService;

    public UserService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

The Application startup fails when an error similar to:

```
No qualifying bean of type 'NotificationService' available:
expected single matching bean but found 2
```

## Solution by using annotations?

Spring provides two annotations to resolve this ambiguity:

- @Primary
- @Qualifier

Both tell Spring which Bean should be injected, but they solve the problem in different ways.

## Use of @Primary and @Qualifier?

### @Primary

marks one Bean as the default choice whenever multiple Beans of the same type exist.

```
@Service
@Primary
public class EmailNotificationService implements NotificationService {
}

@Service
public class SmsNotificationService implements NotificationService {
}
```

Now this injection works, Spring automatically injects EmailNotificationService

```
@Service
public class UserService {

    private final NotificationService notificationService;

    public UserService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

### @Qualifier

Lets you explicitly choose which Bean to inject.

```
@Service("emailService")
public class EmailNotificationService implements NotificationService {
}

@Service("smsService")
public class SmsNotificationService implements NotificationService {
}

# Spring injects SmsNotificationService, even if another Bean
is marked as @Primary

@Service
public class UserService {

    private final NotificationService notificationService;

    public UserService(
        @Qualifier("smsService")
        NotificationService notificationService) {

        this.notificationService = notificationService;
    }
}
```

## @Primary vs @Qualifier?

## When to use @Primary and when to use @Qualifier?

Use @Primary when:

- One implementation is used most of the time
- You want a default Bean for the application
- The majority of injection points should receive the same implementation

```
NotificationService
    ├── EmailNotificationService (@Primary)
    └── SmsNotificationService
```

Use @Qualifier when:

- Different services require different implementations
- You need to explicit control over which Bean is injected
- Multiple implementations are equally important and there is no obvious default

```
@Service
public class MarketingService {

    public MarketingService(
        @Qualifier("emailService")
        NotificationService notificationService) {
    }
}

@Service
public class EmergencyService {

    public EmergencyService(
        @Qualifier("smsService")
        NotificationService notificationService) {
    }
}
```

# Priority Order

When Spring resolves dependencies, it follows this order:

1. @Qualifier (highest priority)
2. @Primary
3. Bean name (in some resolution scenarios)
4. If ambiguity still exists, Springs throws an exception

```
# Although EmailNotificationService is marked with @Primary, Spring injects SmsNotificationService because @Qualifier always takes precedence.

@Service
@Primary
public class EmailNotificationService implements NotificationService {
}

@Service("smsService")
public class SmsNotificationService implements NotificationService {
}

public UserService(
    @Qualifier("smsService")
    NotificationService notificationService) {
}
```
