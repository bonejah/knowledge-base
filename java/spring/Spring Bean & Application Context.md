# Spring Beans & ApplicationContext

## Spring Beans

Bean is a Java Object created, configured and managed by Spring IoC (Inversion of Control) Container.

### What turns a Java Object into a Bean?

The object needs to be registered into the Spring Container using annotations like:

- @Controller
- @Service
- @Repository
- @Component
- @Configuration

### Spring Bean x POJO

POJO (Plain Old Java Object):

- it's a simple Java class
- does not extend any specific class
- does not implements any required interface
- does not has any dependecies like, Hibernate or any framework

POJO example:

```
# no dependencies, interfaces and frameworks

public class UserService() {
}

UserService service = new UserService();
```

Bean example:

```
@Service
public class UserService() {
}
```

### How to create a Bean

1. Using stereotype annotations (@Controller, @Service, @Repository, @Component).

```
@Controller
public class UserController {
}
```

2. Using @Bean inside a @Configuration class.

```
@Configuration
public class AppConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper();
    }
}

# Spring will execute the command below and register it as a Bean:

ObjectMapper mapper = new ObjectMapper();
```

```
# Example using a third-party client:

public class StripeClient {
}

# You cannot add: @Component into StripeClient, so:

@Configuration
public class StripeConfig {

    @Bean
    public StripeClient stripeClient() {
        return new StripeClient();
    }
}


```

3. Using XML configuration (legacy approach)
   Example:

```
<beans>
  <bean
      id="paymentService"
      class="com.example.PaymentService"/>
</beans>

# Spring creates and registers it

new PaymentService();

```

## ApplicationContext

It's an implementation of Spring IoC Container. There is an old container "BeanFactory" (more simpler)

## IoC Container X ApplicationContext

IoC Container -> It's a concept (the mechanism that create, configure and managed the objects).

Example:

```
# The Spring finds this class:

@Service
public class PaymentService(){
}

# And create a new instace to store inside the container
PaymentService paymentService = new PaymentService();

```

ApplicationContext -> It's a Spring Interface that implements these concepts with more extra functions, this object represents the container into a Spring application.

Example:

```
# In that way you have all the registered Beans

ApplicationContext context = SpringApplication.run(Application.class, args);

# You can pick one manually

PaymentService paymentService = context.getBean(PaymentService.class);
```

                 Spring Application
                        |
                        |
                        v
              +--------------------+
              | ApplicationContext |
              |                    |
              |  Bean Registry     |
              |                    |
              |  UserService       |
              |  OrderService      |
              |  Repository        |
              +--------------------+
                        |
                        |
                        v
                Dependency Injection

```
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

1. Start JVM
   |
2. Create ApplicationContext
   |
3. Scan classes
   |
4. Find @Component, @Service, @Repository...
   |
5. Create Beans
   |
6. Inject dependencies
   |
7. Start application
