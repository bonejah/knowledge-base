# 18 - Spring Batch

## 1. What is Spring Batch?

**Spring Batch** is a framework from the Spring ecosystem designed to process **large volumes of data** in a reliable, structured, and efficient way.

A **batch job** is typically a process that runs without direct user interaction and performs a large amount of work, such as:

- Importing thousands or millions of records.
- Exporting data to files.
- Processing CSV, XML, or JSON files.
- Migrating data between databases.
- Generating reports.
- Updating large numbers of database records.
- Processing financial transactions.
- Sending scheduled notifications.
- Data transformation and ETL processes.

For example, imagine that we receive a CSV file containing **10 million customers**:

```text
customers.csv
     |
     v
Read customer
     |
     v
Validate / Transform
     |
     v
Insert into Database
```

Doing this with a simple loop could consume a significant amount of memory and make error recovery difficult.

Spring Batch provides abstractions for handling these scenarios in a controlled way.

---

## 2. Why Use Spring Batch?

Spring Batch provides features that are especially useful when processing large amounts of data.

### Main benefits

- Chunk-based processing
- Transaction management
- Job execution tracking
- Restartability
- Retry
- Skip
- Parallel processing
- Error handling
- Job parameters
- Execution metadata
- Integration with databases and files

Instead of implementing all these mechanisms manually, Spring Batch provides them as framework features.

---

## 3. Spring Batch vs a Normal Spring Service

Consider a normal Spring service:

```java
@Service
public class CustomerService {

    public void processCustomers(List<Customer> customers) {

        for (Customer customer : customers) {
            customerRepository.save(customer);
        }
    }
}
```

This can work for a small number of records.

However, imagine:

```text
10 records      -> Fine
1,000 records   -> Probably fine
100,000 records -> Potential problems
10,000,000      -> Very problematic
```

A large list can consume a significant amount of memory.

Spring Batch allows us to process data incrementally:

```text
10,000,000 records

     |
     v
Read 100
     |
     v
Process 100
     |
     v
Write 100
     |
     v
Commit Transaction
     |
     v
Read next 100
     |
     v
...
```

This is called **chunk-oriented processing**.

---

# 4. Spring Batch Architecture

A simplified Spring Batch architecture looks like this:

```text
                 +----------------+
                 |   Job Launcher |
                 +-------+--------+
                         |
                         v
                 +----------------+
                 |      Job       |
                 +-------+--------+
                         |
                +--------+--------+
                |                 |
                v                 v
           +---------+       +---------+
           | Step 1  |       | Step 2  |
           +----+----+       +---------+
                |
                v
        +---------------+
        | Reader        |
        +-------+-------+
                |
                v
        +---------------+
        | Processor     |
        +-------+-------+
                |
                v
        +---------------+
        | Writer        |
        +-------+-------+
                |
                v
        +---------------+
        | Database      |
        +---------------+
```

The most important concepts are:

- `Job`
- `Step`
- `JobLauncher`
- `ItemReader`
- `ItemProcessor`
- `ItemWriter`
- `JobRepository`
- `JobExecution`
- `StepExecution`
- `JobParameters`

---

# 5. Job

A **Job** represents an entire batch process.

For example:

```text
Import Customers Job
```

The job could contain multiple steps:

```text
Customer Import Job

Step 1 -> Read customer file
Step 2 -> Validate customers
Step 3 -> Insert customers
Step 4 -> Generate report
```

A job can therefore be viewed as a workflow.

Example:

```java
@Bean
public Job customerJob(JobRepository jobRepository,
                        Step customerStep) {

    return new JobBuilder("customerJob", jobRepository)
            .start(customerStep)
            .build();
}
```

---

# 6. Step

A **Step** represents an independent phase of a batch job.

For example:

```text
Job
 |
 +-- Step 1: Read CSV
 |
 +-- Step 2: Validate records
 |
 +-- Step 3: Insert records
```

A job can contain one or many steps.

```java
@Bean
public Job customerJob(JobRepository jobRepository,
                        Step step1,
                        Step step2) {

    return new JobBuilder("customerJob", jobRepository)
            .start(step1)
            .next(step2)
            .build();
}
```

---

# 7. JobLauncher

The `JobLauncher` is responsible for **starting a Job**.

Conceptually:

```text
Application
     |
     v
JobLauncher
     |
     v
Job
     |
     v
Steps
```

Example:

```java
jobLauncher.run(
    customerJob,
    new JobParameters()
);
```

A job can be started:

- Manually
- Through an API
- Through a scheduler
- Through Spring's scheduling mechanisms
- Through an external scheduler

For example:

```text
Every night at 2 AM

Scheduler
    |
    v
JobLauncher
    |
    v
Customer Import Job
```

---

# 8. JobRepository

The `JobRepository` is responsible for storing metadata about batch executions.

Spring Batch needs to know things such as:

```text
Which job ran?
When did it start?
When did it finish?
Did it succeed?
Which step failed?
How many records were processed?
Can the job be restarted?
```

This information is persisted in database tables.

Typical Spring Batch metadata tables include:

```text
BATCH_JOB_INSTANCE
BATCH_JOB_EXECUTION
BATCH_JOB_EXECUTION_PARAMS

BATCH_STEP_EXECUTION
BATCH_STEP_EXECUTION_CONTEXT
BATCH_JOB_EXECUTION_CONTEXT
```

This metadata is one of the reasons Spring Batch can support **restartability**.

---

# 9. JobExecution

A `JobExecution` represents a specific execution of a Job.

For example:

```text
Job:
customerImport

Execution #1
Status: FAILED

Execution #2
Status: COMPLETED
```

The same logical job can therefore have multiple executions.

---

# 10. JobParameters

`JobParameters` are parameters provided when launching a job.

For example:

```text
customerImport

date = 2026-08-28
file = customers.csv
```

Example:

```java
JobParameters parameters =
        new JobParametersBuilder()
                .addString("file", "customers.csv")
                .addString("date", "2026-08-28")
                .toJobParameters();
```

Parameters can also help identify different executions of the same job.

---

# 11. Chunk-Oriented Processing

One of the most important Spring Batch concepts is **chunk processing**.

Instead of processing everything in one transaction:

```text
10,000 records
       |
       v
ONE HUGE TRANSACTION
```

Spring Batch can process records in chunks:

```text
Chunk size = 100

Read 100
Process 100
Write 100
Commit

Read 100
Process 100
Write 100
Commit

Read 100
Process 100
Write 100
Commit
```

This provides several benefits:

- Lower memory usage
- Smaller transactions
- Better failure recovery
- Better database behavior
- Better scalability

---

# 12. ItemReader

The `ItemReader` is responsible for reading data.

Examples:

```text
Database
CSV
XML
JSON
API
Message Queue
```

Conceptually:

```java
public interface ItemReader<T> {

    T read() throws Exception;
}
```

The reader returns one item at a time.

For example:

```text
Reader
  |
  +--> Customer 1
  +--> Customer 2
  +--> Customer 3
  +--> ...
```

Common implementations include:

```text
JdbcCursorItemReader
JdbcPagingItemReader
FlatFileItemReader
```

---

# 13. ItemProcessor

The `ItemProcessor` is responsible for transforming or validating data.

For example:

```java
@Bean
public ItemProcessor<Customer, Customer> processor() {

    return customer -> {

        customer.setName(
            customer.getName().trim().toUpperCase()
        );

        return customer;
    };
}
```

The flow is:

```text
Input
  |
  v
Customer
  |
  v
Processor
  |
  v
Transformed Customer
```

The processor is also a good place for business validation.

---

# 14. ItemWriter

The `ItemWriter` writes the processed data.

For example:

```text
Reader
   |
   v
Processor
   |
   v
Writer
   |
   v
Database
```

For database operations, Spring Batch provides JDBC-based writers such as:

```text
JdbcBatchItemWriter
```

The writer typically receives a collection of items belonging to the current chunk.

---

# 15. Complete Chunk Processing

The complete flow is:

```text
             CHUNK
               |
               v
       +---------------+
       | ItemReader    |
       +-------+-------+
               |
               v
       +---------------+
       | ItemProcessor |
       +-------+-------+
               |
               v
       +---------------+
       | ItemWriter    |
       +-------+-------+
               |
               v
          Database
               |
               v
            COMMIT
```

For example, with a chunk size of 100:

```text
Read 100
   |
Process 100
   |
Write 100
   |
Commit
```

---

# 16. Implementing Spring Batch in Spring Boot

Let's create a simple application that reads customers and inserts them into a database.

## Project Structure

A possible project structure:

```text
src/main/java
└── com.example.batch
    ├── BatchApplication.java
    ├── config
    │   └── BatchConfig.java
    ├── model
    │   └── Customer.java
    └── repository
        └── CustomerRepository.java
```

---

# 17. Maven Dependencies

For a Spring Boot application:

```xml
<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>

    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>

</dependencies>
```

For production, you would typically replace H2 with a production database such as PostgreSQL or MySQL.

---

# 18. Domain Object

Let's assume our application needs to import customers.

```java
public record Customer(
        Long id,
        String name,
        String email
) {
}
```

---

# 19. Database Table

Our destination table could be:

```sql
CREATE TABLE customer (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255)
);
```

---

# 20. Creating the ItemReader

Suppose our source data is already in another database.

A JDBC paging reader is useful when processing large datasets.

```java
@Bean
public JdbcPagingItemReader<Customer> reader(
        DataSource dataSource) {

    JdbcPagingItemReader<Customer> reader =
            new JdbcPagingItemReader<>();

    reader.setDataSource(dataSource);
    reader.setPageSize(100);

    reader.setRowMapper((rs, rowNum) ->
            new Customer(
                    rs.getLong("id"),
                    rs.getString("name"),
                    rs.getString("email")
            )
    );

    return reader;
}
```

The reader retrieves data in pages instead of loading the entire dataset into memory.

---

# 21. Creating the ItemProcessor

The processor can transform the data.

```java
@Bean
public ItemProcessor<Customer, Customer> processor() {

    return customer -> {

        String normalizedName =
                customer.name().trim();

        return new Customer(
                customer.id(),
                normalizedName,
                customer.email().toLowerCase()
        );
    };
}
```

---

# 22. Creating the ItemWriter

For inserting large amounts of data, use batch database operations rather than executing an individual SQL statement for every record.

```java
@Bean
public JdbcBatchItemWriter<Customer> writer(
        DataSource dataSource) {

    return new JdbcBatchItemWriterBuilder<Customer>()
            .sql("""
                INSERT INTO customer
                    (id, name, email)
                VALUES
                    (:id, :name, :email)
                """)
            .beanMapped()
            .dataSource(dataSource)
            .build();
}
```

---

# 23. Creating the Step

Now we connect the reader, processor, and writer.

```java
@Bean
public Step customerStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager,
        ItemReader<Customer> reader,
        ItemProcessor<Customer, Customer> processor,
        ItemWriter<Customer> writer) {

    return new StepBuilder("customerStep", jobRepository)
            .<Customer, Customer>chunk(100)
            .transactionManager(transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
}
```

The important part is:

```java
.chunk(100)
```

This means:

```text
100 records
    |
    v
Read
    |
    v
Process
    |
    v
Write
    |
    v
COMMIT
```

---

# 24. Creating the Job

Finally, we create the Job.

```java
@Bean
public Job customerJob(
        JobRepository jobRepository,
        Step customerStep) {

    return new JobBuilder("customerJob", jobRepository)
            .start(customerStep)
            .build();
}
```

The complete architecture is now:

```text
                Customer Job
                     |
                     v
               Customer Step
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Reader    Processor    Writer
          |          |          |
          v          v          v
       Source    Transform   Database
```

---

# 25. Complete Example

A simplified configuration could look like this:

```java
@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public Step customerStep(
            JobRepository jobRepository,
            PlatformTransactionManager transactionManager,
            ItemReader<Customer> reader,
            ItemProcessor<Customer, Customer> processor,
            ItemWriter<Customer> writer) {

        return new StepBuilder("customerStep", jobRepository)
                .<Customer, Customer>chunk(100)
                .transactionManager(transactionManager)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }

    @Bean
    public Job customerJob(
            JobRepository jobRepository,
            Step customerStep) {

        return new JobBuilder("customerJob", jobRepository)
                .start(customerStep)
                .build();
    }
}
```

---

# 26. Creating a Job to Insert Large Data

Now let's look at a realistic scenario.

Imagine that we have:

```text
customers.csv
```

containing:

```text
1,John,john@email.com
2,Mary,mary@email.com
3,Robert,robert@email.com
...
10,000,000 records
```

We want to import everything into:

```text
customer
```

The architecture would be:

```text
CSV File
   |
   v
FlatFileItemReader
   |
   v
ItemProcessor
   |
   v
JdbcBatchItemWriter
   |
   v
Database
```

---

# 27. FlatFileItemReader

For CSV files, Spring Batch provides `FlatFileItemReader`.

Example:

```java
@Bean
public FlatFileItemReader<Customer> reader() {

    return new FlatFileItemReaderBuilder<Customer>()
            .name("customerReader")
            .resource(
                new ClassPathResource("customers.csv")
            )
            .delimited()
            .names("id", "name", "email")
            .targetType(Customer.class)
            .build();
}
```

The reader processes the file incrementally.

It does **not** need to load all 10 million records into memory.

---

# 28. Large Insert Flow

Suppose:

```java
.chunk(500)
```

Then Spring Batch approximately works like this:

```text
CSV
 |
 +--> Records 1-500
 |       |
 |       +--> Process
 |       |
 |       +--> INSERT 500
 |       |
 |       +--> COMMIT
 |
 +--> Records 501-1000
 |       |
 |       +--> Process
 |       |
 |       +--> INSERT 500
 |       |
 |       +--> COMMIT
 |
 +--> ...
```

This is much safer than:

```java
List<Customer> customers =
        readEntireFile();

repository.saveAll(customers);
```

for extremely large datasets.

---

# 29. Chunk Size

Chunk size is an important performance consideration.

Example:

```java
.chunk(10)
```

means:

```text
Read 10
Process 10
Write 10
Commit
```

While:

```java
.chunk(1000)
```

means:

```text
Read 1000
Process 1000
Write 1000
Commit
```

There is no universally correct chunk size.

A larger chunk can improve throughput but may:

- Increase memory usage
- Increase transaction duration
- Increase rollback cost
- Put more pressure on the database

A smaller chunk can:

- Reduce memory usage
- Produce shorter transactions
- Improve recovery granularity

But too-small chunks can increase overhead.

Therefore, the appropriate chunk size should be determined through **testing and monitoring**.

---

# 30. Transactions and Chunk Processing

Each chunk normally runs inside a transaction.

For example:

```java
.chunk(100)
```

Conceptually:

```text
BEGIN TRANSACTION

Read 100
Process 100
Write 100

COMMIT
```

If the writer fails:

```text
BEGIN TRANSACTION

Read 100
Process 100
Write

ERROR

ROLLBACK
```

The chunk can therefore provide transactional consistency.

---

# 31. Skip

Sometimes a single invalid record should not cause the entire job to fail.

For example:

```text
Record 1 -> OK
Record 2 -> OK
Record 3 -> INVALID
Record 4 -> OK
```

We may want:

```text
Record 3 -> Skip
```

Spring Batch supports skip policies.

Example:

```java
.faultTolerant()
.skip(ValidationException.class)
.skipLimit(100)
```

This means that up to 100 validation errors can be skipped.

---

# 32. Retry

Transient failures can sometimes be retried.

For example:

```text
Database timeout
Network problem
Temporary external API failure
```

Example:

```java
.faultTolerant()
.retry(TransientDataAccessException.class)
.retryLimit(3)
```

Conceptually:

```text
Write
 |
 +--> Failure
       |
       +--> Retry #1
       |
       +--> Retry #2
       |
       +--> Retry #3
       |
       +--> Success
```

Retry should be used carefully.

A retry is appropriate for **transient failures**, not permanent validation errors.

---

# 33. Restartability

One of the biggest advantages of Spring Batch is the ability to restart failed jobs.

Imagine:

```text
10,000,000 records

Processed:
7,500,000

Failure!
```

With a properly configured Spring Batch job, execution metadata can allow the job to restart from the appropriate point instead of necessarily starting from zero.

Conceptually:

```text
Initial execution

1
2
3
...
7,500,000
X FAILURE
```

After fixing the problem:

```text
Restart

7,500,001
7,500,002
...
10,000,000
```

This is especially valuable for long-running jobs.

---

# 34. Job Status

Spring Batch tracks execution status.

Common statuses include:

```text
STARTING
STARTED
COMPLETED
FAILED
STOPPING
STOPPED
ABANDONED
UNKNOWN
```

For example:

```text
customerImport
        |
        v
     STARTED
        |
        v
    COMPLETED
```

or:

```text
customerImport
        |
        v
     STARTED
        |
        v
      FAILED
```

---

# 35. Scheduling Batch Jobs

Batch jobs are often executed on a schedule.

For example:

```text
Every day at 2:00 AM
```

A Spring scheduler could trigger the job.

```java
@Scheduled(cron = "0 0 2 * * *")
public void runJob() throws Exception {

    jobLauncher.run(
            customerJob,
            new JobParametersBuilder()
                    .addLong(
                        "timestamp",
                        System.currentTimeMillis()
                    )
                    .toJobParameters()
    );
}
```

For more complex enterprise environments, external schedulers can also trigger batch jobs.

Examples include:

```text
Kubernetes CronJob
Airflow
Control-M
Jenkins
Cloud schedulers
```

---

# 36. Batch Processing vs Real-Time Processing

Spring Batch is designed for **batch workloads**, not necessarily real-time workloads.

### Batch

```text
Every night

10 million records
        |
        v
Process everything
```

### Real-time

```text
HTTP Request
     |
     v
Process one request
     |
     v
Return response
```

A REST API should generally not be replaced by Spring Batch simply because it needs to process data.

Choose the architecture according to the workload.

---

# 37. Common Use Cases

Spring Batch is commonly used for:

### Data Migration

```text
Database A
    |
    v
Spring Batch
    |
    v
Database B
```

### File Import

```text
CSV
 |
 v
Spring Batch
 |
 v
Database
```

### File Export

```text
Database
 |
 v
Spring Batch
 |
 v
CSV
```

### Data Transformation

```text
Raw Data
   |
   v
Validation
   |
   v
Transformation
   |
   v
Database
```

### Periodic Processing

```text
Every night

Database
   |
   v
Batch Job
   |
   v
Generate Reports
```

---

# 38. Best Practices

## 38.1 Use Chunk Processing

For large datasets, prefer:

```java
.chunk(500)
```

over loading everything into memory.

---

## 38.2 Use Batch Database Operations

Prefer:

```text
JdbcBatchItemWriter
```

or another batching mechanism over:

```java
for (...) {
    repository.save(item);
}
```

for very large workloads.

---

## 38.3 Keep Transactions Reasonably Small

Avoid extremely large transactions such as:

```text
10 million records
       |
       v
ONE TRANSACTION
```

Prefer:

```text
500 records
   |
COMMIT

500 records
   |
COMMIT

500 records
   |
COMMIT
```

---

## 38.4 Make Jobs Restartable

Design jobs so that a failure does not require the entire process to start from scratch.

Think about:

```text
Where did the job fail?

Can it continue?

Is the operation idempotent?

Can already-processed records be safely processed again?
```

---

## 38.5 Make Batch Operations Idempotent When Possible

Suppose a job inserts:

```text
Customer #100
```

If the job is restarted, you don't want:

```text
Customer #100
Customer #100
Customer #100
```

Consider using:

- Unique constraints
- Upsert operations
- Business keys
- Deduplication
- Checkpoints

---

## 38.6 Monitor Batch Jobs

For production systems, monitor:

```text
Job status
Execution duration
Records read
Records processed
Records written
Skip count
Retry count
Failure count
```

This makes operational problems much easier to identify.

---

## 38.7 Avoid Excessive Logging

Avoid logging every record:

```java
log.info("Processing customer {}", customer.id());
```

for millions of records.

This can generate enormous log volumes and slow down the application.

Prefer aggregated information:

```text
Processed 100,000 records
Processed 200,000 records
Processed 300,000 records
```

---

## 38.8 Tune the Chunk Size

Don't assume:

```java
.chunk(100)
```

is always optimal.

Test different values:

```text
50
100
500
1000
5000
```

and measure:

- Throughput
- Memory usage
- Database load
- Transaction duration
- Failure recovery time

---

# 39. Common Mistakes

### Loading Everything Into Memory

Bad:

```java
List<Customer> customers =
        repository.findAll();
```

for millions of records.

Better:

```text
Paging Reader
+
Chunk Processing
```

---

### One Transaction for Everything

Bad:

```text
10 million records
        |
        v
ONE TRANSACTION
```

Better:

```text
Chunk
  |
Commit
  |
Next Chunk
  |
Commit
```

---

### Individual Database Inserts

Bad:

```java
for (Customer customer : customers) {
    repository.save(customer);
}
```

for extremely large datasets.

Better:

```text
JdbcBatchItemWriter
```

or an equivalent batch-oriented writer.

---

### Ignoring Restartability

A batch job can run for hours.

If it fails at 95%:

```text
95% complete
      |
      v
Failure
      |
      v
Restart everything
```

can be extremely expensive.

Design for restartability from the beginning.

---

# 40. Spring Batch Mental Model

A simple way to remember Spring Batch is:

```text
JOB
 |
 +----------------------+
 |                      |
 v                      v
STEP 1                 STEP 2
 |
 v
+--------+
| READER |
+---+----+
    |
    v
+---------+
|PROCESSOR|
+----+----+
     |
     v
+--------+
| WRITER |
+---+----+
    |
    v
 DATABASE
```

And for large data:

```text
             LARGE DATASET
                    |
                    v
             +-------------+
             |    Reader   |
             +------+------+
                    |
                    v
             +-------------+
             |   Processor |
             +------+------+
                    |
                    v
             +-------------+
             |    Writer   |
             +------+------+
                    |
                    v
               DATABASE

        Everything happens in CHUNKS
```

---

# 41. Spring Batch vs Spring @Async

These concepts are different.

### `@Async`

Designed primarily for executing work asynchronously:

```text
Request
  |
  v
@Async
  |
  +----> Background execution
```

### Spring Batch

Designed for structured, large-scale processing:

```text
Job
 |
 +--> Step
       |
       +--> Reader
       +--> Processor
       +--> Writer
       +--> Transaction
       +--> Checkpoint
       +--> Retry
       +--> Skip
       +--> Execution metadata
```

Spring Batch is therefore much more than simply running something in the background.

---

# 42. Key Takeaways

Spring Batch is a framework for **reliable and scalable batch processing**.

The most important concepts are:

```text
Job
Step
JobLauncher
JobRepository
JobExecution
JobParameters
ItemReader
ItemProcessor
ItemWriter
Chunk
Transaction
Skip
Retry
Restartability
```

The most important processing model to remember is:

```text
             JOB
              |
             STEP
              |
      +-------+-------+
      |       |       |
   READER PROCESSOR WRITER
      |       |       |
      +-------+-------+
              |
            CHUNK
              |
          TRANSACTION
              |
            COMMIT
```

For large data insertion, the recommended approach is generally:

```text
Large Dataset
      |
      v
Paging/File Reader
      |
      v
Processor
      |
      v
Batch Writer
      |
      v
Database
```

rather than:

```text
Large Dataset
      |
      v
Load everything into memory
      |
      v
Loop
      |
      v
Individual INSERTs
```

The main goal of Spring Batch is not simply to process data in the background. It is to provide a **structured, transactional, restartable, and scalable way of processing large amounts of data**.
