# Spring Bean Scope

Spring Bean Scope defines the lifecycle and the visibility of a Bean inside the Spring IoC Container.
The scopes determined how many instances will be created and when they will be reused.

## Scopes

- singleton: One shared instance for the entire Spring IoC Container (default)
- prototype: A new instance is created every time the Bean is required
- request: One instance per HTTP request (web application only)
- session: One instance per HTTP session (web application only)
- application: One instance per ServletContext session (web application only)
- websocket: One instance per WebSocket session.

### Singleton Scope

The singleton scope creates only one instance of a Bean for the entire Spring IoC Container.

```
@Service
public class EmailService {

}

this is equivalent to:

@Service
@Scope("singleton)
public class EmailService {

}
```

```
# another example, sharing the same instance and counter variable

@Service
public class CounterService {
  private int counter = 0;

  public int increment() {
    return ++counter;
  }
}

@RestController
public class AController{
  @Autowired
  private CounterService counterService;
}

@RestController
public class BController{
  @Autowired
  private CounterService counterService;
}

# both controllers receive the same instance, so the counter variable is shared.

```

#### Characteristics

- Default Spring scope
- Only one instance per IoC Container
- Created when the Application Context starts (by default)
- Shared accross the entire application
- Thread safety is the developer's responsibility

### Prototype Scope

The prototype scope creates a new instance every time it is requested from the Spring Container.

```
@Component
@Scope("prototype")
public class ReportGenerator {

}
```

#### Characteristics

- New instance every request
- Not shared
- Spring manages only the Bean creation
- Bean destruction is the application's responsibility

Unlike singleton beans, Spring does not invoke destruction callbacks (such as @PreDestroy) for prototype beans. The developer is responsible for destroying prototype beans when they hold resources that need explicit cleanup, because Spring does not invoke destruction callbacks for prototype-scoped beans.

### Request Scope

Available only in web applications, creates one Bean per HTTP request.

```
@Component
@RequestScope
public class UserContext {

}
```

### Session Scope

Available only in web applications, creates one Bean per HTTP session.

```
@Component
@SessionScope
public class UserContext {

}
```

### Application Scope

Available only in web applications, creates one Bean per ServletContext.

```
@Component
@ApplicationScope
public class UserContext {

}
```

This scope is similar to singleton, but its lifecycle is tied to the ServletContext rather than directly to the Spring IoC Container

### WebSocket Scope

Creates one Bean for each WebSocket session. Each connected client receives its own Bean instance.

```
@Component
@Scope("websocket")
public class UserContext {

}
```

## Choosing the Right Scope

- Use singleton when:
  1. The Bean is stateless.
  2. The Bean provides shared services (e.g., @Service, @Repository, @Controller).
  3. You want to minimize object creation and improve performance.

- Use prototype when:
  1. The Bean holds state that should not be shared.
  2. Each consumer needs its own independent instance.
  3. Good to use when needs to handle, file processors, import/export jobs, report generators, temporary data holders, parsing contexts

- Use request, session, or websocket only in web applications when the Bean's lifecycle should follow the corresponding HTTP or WebSocket context.
