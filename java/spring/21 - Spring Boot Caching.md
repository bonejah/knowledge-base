# 21 - Spring Boot Caching

## What is Cache?

A **cache** is a temporary storage mechanism used to keep frequently accessed data closer to the application so that it can be retrieved faster.

Instead of executing the same expensive operation repeatedly, an application can store the result and reuse it for subsequent requests.

For example, imagine an API that retrieves a product from a database:

```text
Request 1
   ↓
Controller
   ↓
Service
   ↓
Database
   ↓
Product
```

If the same product is requested hundreds of times, querying the database every time is unnecessary.

With caching:

```text
Request 1
   ↓
Controller
   ↓
Service
   ↓
Cache Miss
   ↓
Database
   ↓
Store result in Cache
   ↓
Product


Request 2
   ↓
Controller
   ↓
Service
   ↓
Cache Hit
   ↓
Product
```

The second request does not need to access the database.

### Why Use Cache?

Caching can provide several benefits:

- Reduce database load
- Reduce response time
- Improve application performance
- Reduce network calls
- Reduce calls to external APIs
- Improve scalability
- Handle frequently requested data more efficiently

### Cache Hit vs Cache Miss

A **cache hit** occurs when the requested data is already available in the cache.

```text
Application → Cache → Data Found
                    → Cache Hit
```

A **cache miss** occurs when the data is not available.

```text
Application → Cache → Data Not Found
                    → Cache Miss
                         ↓
                      Database
                         ↓
                    Store in Cache
```

---

# Cache in Spring Boot

Spring Boot provides caching support through **Spring's Cache Abstraction**.

The important idea is that your application does not need to know exactly how the cache is implemented.

For example, your service can simply say:

```java
@Cacheable("products")
public Product findById(Long id) {
    return repository.findById(id)
            .orElseThrow();
}
```

Spring handles the caching behavior.

The actual cache implementation can be:

- In-memory
- Redis
- Hazelcast
- Caffeine
- Other cache providers

This provides an abstraction between the application and the cache technology.

---

## Enabling Caching

Caching must first be enabled in the Spring Boot application.

```java
@SpringBootApplication
@EnableCaching
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

The `@EnableCaching` annotation tells Spring to enable its caching infrastructure.

Without it, annotations such as `@Cacheable`, `@CachePut`, and `@CacheEvict` will not provide the expected caching behavior.

---

# Simple Example

Consider a product service:

```java
@Service
public class ProductService {

    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    @Cacheable("products")
    public Product findById(Long id) {
        System.out.println("Querying database...");

        return repository.findById(id)
                .orElseThrow();
    }
}
```

The first call:

```java
productService.findById(10L);
```

will execute the method.

```text
Cache
  ↓
MISS
  ↓
findById()
  ↓
Database
  ↓
Store result in cache
```

A subsequent call:

```java
productService.findById(10L);
```

will return the cached value.

```text
Cache
  ↓
HIT
  ↓
Return cached Product
```

The database does not need to be queried again.

---

# Types of Caching

There are two major approaches commonly used in Spring Boot applications:

1. In-Memory Caching
2. Distributed Caching

---

# In-Memory Cache

An **in-memory cache** stores data inside the memory of the application instance.

For example:

```text
Spring Boot Application

+-----------------------------+
|                             |
|   Application               |
|                             |
|   +---------------------+   |
|   | In-Memory Cache     |   |
|   |                     |   |
|   | product:10          |   |
|   | product:20          |   |
|   +---------------------+   |
|                             |
+-----------------------------+
```

Common technologies include:

- Caffeine
- Ehcache
- Simple Map-based implementations

## Advantages

- Very fast
- No network communication
- Simple to configure
- Easy to use
- Excellent for frequently accessed local data

## Disadvantages

The biggest problem is that the cache belongs to a specific application instance.

Consider:

```text
              Load Balancer
                   |
          +--------+--------+
          |                 |
          ↓                 ↓
      Instance A        Instance B
          |                 |
      Cache A            Cache B
```

If Instance A stores:

```text
product:10 → Product
```

Instance B does not automatically have that value.

This can create consistency problems.

---

# Distributed Cache

A **distributed cache** is stored outside the application instances and can be shared by multiple instances.

A common example is **Redis**.

```text
                 Load Balancer
                 /            \
                /              \
               ↓                ↓
        Spring Boot A     Spring Boot B
               \                /
                \              /
                 ↓              ↓
                   Redis
                    |
                 Cache
```

All application instances can access the same cache.

## Advantages

- Shared between application instances
- Better for horizontally scaled applications
- Cache survives individual application instance restarts
- Centralized cache management
- Suitable for microservices environments

## Disadvantages

- Network communication is required
- More infrastructure
- Additional operational complexity
- Cache server can become a dependency

---

# In-Memory vs Distributed Cache

| Feature                         | In-Memory          | Distributed               |
| ------------------------------- | ------------------ | ------------------------- |
| Location                        | Application memory | External cache            |
| Speed                           | Extremely fast     | Very fast                 |
| Shared between instances        | No                 | Yes                       |
| Infrastructure                  | Simple             | Additional infrastructure |
| Scalability                     | Limited            | Better                    |
| Example                         | Caffeine           | Redis                     |
| Network call                    | No                 | Usually yes               |
| Suitable for multiple instances | Limited            | Yes                       |

### When Should You Use In-Memory?

In-memory caching is a good choice when:

- The application has a single instance
- Cached data is small
- Data does not need to be shared
- Extremely low latency is important
- Losing the cache during restart is acceptable

### When Should You Use Distributed Cache?

Distributed caching is generally better when:

- The application has multiple instances
- The application runs in Kubernetes
- Horizontal scaling is required
- Multiple services need to access the same cached data
- Cache consistency across instances matters

---

# Spring Cache Annotations

Spring provides several important annotations for caching:

- `@EnableCaching`
- `@Cacheable`
- `@CachePut`
- `@CacheEvict`
- `@Caching`

Each annotation has a different purpose.

---

# @EnableCaching

`@EnableCaching` enables Spring's annotation-driven caching support.

Example:

```java
@Configuration
@EnableCaching
public class CacheConfig {
}
```

It is also commonly placed on the main application class:

```java
@SpringBootApplication
@EnableCaching
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Think of it as:

```text
@EnableCaching
       ↓
Turn on Spring Cache
       ↓
@Cacheable
@CachePut
@CacheEvict
@Caching
```

---

# @Cacheable

`@Cacheable` is the most commonly used caching annotation.

It tells Spring:

> If the result for this method is already in the cache, return it without executing the method.

Example:

```java
@Cacheable("products")
public Product findById(Long id) {

    System.out.println("Executing database query...");

    return repository.findById(id)
            .orElseThrow();
}
```

The cache name is:

```text
products
```

Spring uses the method arguments to generate a cache key.

For:

```java
findById(10L)
```

the conceptual cache entry is:

```text
products
    |
    +--- 10 → Product
```

---

## @Cacheable Flow

First request:

```text
findById(10)

     ↓

Cache lookup

     ↓

Cache MISS

     ↓

Execute method

     ↓

Database

     ↓

Store result in cache
```

Second request:

```text
findById(10)

     ↓

Cache lookup

     ↓

Cache HIT

     ↓

Return cached result
```

The method itself is not executed on a cache hit.

---

# Custom Cache Key

You can explicitly define the cache key using `key`.

```java
@Cacheable(
    value = "products",
    key = "#id"
)
public Product findById(Long id) {
    return repository.findById(id)
            .orElseThrow();
}
```

Now the cache key is explicitly based on:

```java
#id
```

For:

```java
findById(10L)
```

the key becomes:

```text
10
```

---

# Multiple Parameters

Consider:

```java
@Cacheable(
    value = "products",
    key = "#category + ':' + #id"
)
public Product findProduct(String category, Long id) {
    ...
}
```

For:

```text
category = electronics
id = 10
```

the key becomes:

```text
electronics:10
```

---

# Conditional Caching

You can also decide whether a method result should be cached.

```java
@Cacheable(
    value = "products",
    condition = "#id > 0"
)
public Product findById(Long id) {
    return repository.findById(id)
            .orElseThrow();
}
```

The method will only be cached when:

```text
id > 0
```

---

# unless

`unless` can prevent a result from being cached.

For example:

```java
@Cacheable(
    value = "products",
    unless = "#result == null"
)
public Product findById(Long id) {
    return repository.findById(id)
            .orElse(null);
}
```

If the result is `null`, Spring will not store it in the cache.

---

# @CachePut

`@CachePut` is different from `@Cacheable`.

`@Cacheable` can prevent the method from executing.

`@CachePut` **always executes the method** and updates the cache with the returned result.

Example:

```java
@CachePut(
    value = "products",
    key = "#product.id"
)
public Product update(Product product) {

    return repository.save(product);
}
```

The flow is:

```text
update()
   ↓
Execute method
   ↓
Database
   ↓
Return updated Product
   ↓
Update Cache
```

This is useful when the database and cache need to be synchronized after an update.

---

# @Cacheable vs @CachePut

| Annotation   | Executes Method? | Uses Existing Cache? | Updates Cache? |
| ------------ | ---------------: | -------------------: | -------------: |
| `@Cacheable` | Not on cache hit |                  Yes |            Yes |
| `@CachePut`  |           Always |                   No |            Yes |

A simple way to remember:

```text
@Cacheable
"Use the cache if possible."

@CachePut
"Execute the method and update the cache."
```

---

# @CacheEvict

`@CacheEvict` removes entries from the cache.

This is especially useful when data is deleted or when cached data becomes invalid.

Example:

```java
@CacheEvict(
    value = "products",
    key = "#id"
)
public void delete(Long id) {

    repository.deleteById(id);
}
```

The flow is:

```text
delete()
   ↓
Database
   ↓
Remove cache entry
```

---

# Evict All Entries

You can remove everything from a cache:

```java
@CacheEvict(
    value = "products",
    allEntries = true
)
public void clearCache() {
}
```

This removes all entries from:

```text
products
```

cache.

Conceptually:

```text
products

10 → Product
20 → Product
30 → Product
40 → Product

        ↓
   @CacheEvict
   allEntries=true

        ↓

products

(empty)
```

---

# beforeInvocation

By default, cache eviction happens after the method successfully completes.

You can change this behavior using:

```java
beforeInvocation = true
```

Example:

```java
@CacheEvict(
    value = "products",
    key = "#id",
    beforeInvocation = true
)
public void delete(Long id) {
    repository.deleteById(id);
}
```

The cache entry is removed before the method executes.

This option should be used carefully because the database operation could still fail afterward.

---

# @Caching

`@Caching` allows you to combine multiple caching annotations on the same method.

For example:

```java
@Caching(
    put = {
        @CachePut(
            value = "products",
            key = "#product.id"
        )
    },
    evict = {
        @CacheEvict(
            value = "productList",
            allEntries = true
        )
    }
)
public Product update(Product product) {

    return repository.save(product);
}
```

Here, one operation performs two cache actions:

```text
Update Product
      |
      +---- Update "products" cache
      |
      +---- Clear "productList" cache
```

This is useful when a single database operation affects multiple cached representations of the same data.

---

# Complete Example

Let's build a simple `ProductService`.

```java
@Service
public class ProductService {

    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    @Cacheable(
        value = "products",
        key = "#id"
    )
    public Product findById(Long id) {

        System.out.println("Loading product from database...");

        return repository.findById(id)
                .orElseThrow();
    }

    @CachePut(
        value = "products",
        key = "#product.id"
    )
    public Product update(Product product) {

        return repository.save(product);
    }

    @CacheEvict(
        value = "products",
        key = "#id"
    )
    public void delete(Long id) {

        repository.deleteById(id);
    }
}
```

The behavior becomes:

```text
GET Product
     ↓
@Cacheable
     ↓
Cache Hit?
   /      \
 Yes       No
 |          |
Return     Database
             |
             ↓
         Store Cache
```

Update:

```text
Update Product
      ↓
@CachePut
      ↓
Database
      ↓
Update Cache
```

Delete:

```text
Delete Product
      ↓
Database
      ↓
@CacheEvict
      ↓
Remove Cache Entry
```

---

# Cache Invalidation

One of the most important concepts in caching is **cache invalidation**.

When the underlying data changes, the cached version can become outdated.

For example:

```text
Database

Product 10
Price = $100

        ↓

Cache

Product 10
Price = $100
```

Now the product is updated:

```text
Database

Product 10
Price = $120
```

But if the cache still contains:

```text
Product 10
Price = $100
```

the application can return stale data.

Therefore, caching strategies must consider how and when cached data becomes invalid.

Common strategies include:

- Evict the cache after updates
- Update the cache after updates
- Use TTL (Time To Live)
- Clear caches periodically
- Use cache versioning
- Use event-driven invalidation

---

# TTL - Time To Live

A cache entry can have a limited lifetime.

For example:

```text
Product cached
     ↓
TTL = 10 minutes
     ↓
10 minutes pass
     ↓
Entry expires
     ↓
Next request → Database
```

TTL is particularly useful for data that changes periodically but does not need to be perfectly up to date.

Examples:

- Exchange rates
- Weather information
- Product catalogs
- Configuration data
- External API responses

---

# Cache Stampede

A **cache stampede** can happen when a popular cache entry expires and many requests attempt to rebuild it simultaneously.

Example:

```text
Cache expires
      ↓
100 requests arrive
      ↓
100 cache misses
      ↓
100 database queries
```

This can create a sudden load spike.

Strategies to mitigate cache stampedes include:

- Locking
- Request coalescing
- Refresh-ahead caching
- Proper TTL configuration
- Distributed locking
- Background cache refresh

---

# Caching and Transactions

Caching and database transactions must be designed carefully.

Consider:

```text
Transaction starts
      ↓
Update database
      ↓
Update cache
      ↓
Transaction fails
```

Now the cache may contain data that was never successfully committed to the database.

When using caching together with transactions, consider:

- Transaction boundaries
- Cache update timing
- Rollbacks
- Consistency requirements
- Event-driven cache invalidation

For critical systems, cache consistency should be treated as part of the overall system design.

---

# Common Caching Architecture

A typical Spring Boot application might look like:

```text
                 Client
                   |
                   ↓
              REST API
                   |
                   ↓
               Service
                   |
             +-----+-----+
             |           |
             ↓           ↓
           Cache      Database
             |
             ↓
           Redis
```

For a distributed application:

```text
                  Load Balancer
                  /           \
                 /             \
                ↓               ↓
        Spring Boot A    Spring Boot B
                \               /
                 \             /
                  ↓           ↓
                     Redis
                       |
                       ↓
                   PostgreSQL
```

---

# Best Practices

## 1. Cache Only Appropriate Data

Not every piece of data should be cached.

Good candidates are usually:

- Frequently accessed
- Expensive to calculate
- Expensive to retrieve
- Relatively stable
- Safe to store temporarily

Avoid caching data when it changes constantly and strict real-time consistency is required.

---

## 2. Choose Good Cache Keys

Cache keys should uniquely identify the data.

Good:

```java
key = "#id"
```

or:

```java
key = "#userId + ':' + #productId"
```

Poor cache keys can cause collisions and incorrect data retrieval.

---

## 3. Define an Expiration Strategy

Avoid allowing cache entries to live forever unless there is a very good reason.

Consider:

```text
TTL
+
Explicit eviction
+
Cache refresh
```

depending on the application's requirements.

---

## 4. Avoid Caching Huge Objects

Caching large objects can consume significant memory.

Instead of caching enormous responses, consider:

- Smaller DTOs
- Frequently used fields
- Pagination
- Appropriate TTLs

---

## 5. Monitor the Cache

Caching should be observable.

Useful metrics include:

```text
Cache Hit Rate
Cache Miss Rate
Evictions
Cache Size
Memory Usage
Latency
Errors
```

A cache that is not monitored can become difficult to troubleshoot.

---

## 6. Do Not Assume Cache Is the Source of Truth

In most applications:

```text
Database = Source of Truth
Cache    = Performance Layer
```

The cache should generally be treated as a temporary representation of the underlying data.

---

## 7. Be Careful With Sensitive Data

Avoid caching sensitive information unless there is a clear security design around it.

Consider:

- Encryption
- Access control
- TTL
- Data isolation
- Cache eviction
- Logging

---

# Common Mistakes

### Mistake 1 - Forgetting @EnableCaching

```java
@SpringBootApplication
public class Application {
}
```

If caching is expected but caching is not enabled, the annotations will not work as intended.

Use:

```java
@SpringBootApplication
@EnableCaching
public class Application {
}
```

---

### Mistake 2 - Updating the Database Without Updating or Evicting the Cache

For example:

```java
public Product update(Product product) {
    return repository.save(product);
}
```

If the product already exists in the cache, the cache may now contain stale data.

Consider:

```java
@CachePut(
    value = "products",
    key = "#product.id"
)
public Product update(Product product) {
    return repository.save(product);
}
```

or an appropriate eviction strategy.

---

### Mistake 3 - Caching Everything

Adding `@Cacheable` everywhere is not a caching strategy.

Caching introduces complexity.

Always consider:

```text
Is the operation expensive?
        +
Is the data requested frequently?
        +
Can stale data be tolerated?
        +
Is the cache size acceptable?
```

---

### Mistake 4 - Ignoring Multiple Application Instances

An in-memory cache behaves differently when the application runs as:

```text
1 instance
```

versus:

```text
10 instances
```

With multiple instances:

```text
Instance A → Cache A
Instance B → Cache B
Instance C → Cache C
```

The caches are independent.

A distributed cache may be more appropriate:

```text
Instance A ─┐
Instance B ─┼──→ Redis
Instance C ─┘
```

---

# Summary

Spring Boot caching provides a convenient abstraction for improving application performance by avoiding unnecessary and expensive operations.

The main annotations are:

```text
@EnableCaching
      ↓
Enables caching support

@Cacheable
      ↓
Read from cache if available

@CachePut
      ↓
Execute method and update cache

@CacheEvict
      ↓
Remove cached data

@Caching
      ↓
Combine multiple cache operations
```

The two main caching models are:

```text
In-Memory
   ↓
Fast
   ↓
Local to application instance


Distributed
   ↓
Shared between instances
   ↓
Better for horizontally scaled systems
```

A good caching strategy is not simply about making requests faster. It requires carefully balancing:

```text
Performance
     +
Consistency
     +
Memory
     +
Scalability
     +
Complexity
```

The goal is to use caching where it provides a measurable benefit without introducing unnecessary consistency and operational problems.
