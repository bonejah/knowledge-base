# 22 - Spring Boot Scheduling

## 1. What is Scheduling? Why?

Scheduling is the process of automatically executing a task at a specific time or at regular intervals.

In a Spring Boot application, scheduling is commonly used for tasks that need to run automatically without requiring a user request.

### Common Use Cases

- Running background jobs periodically
- Cleaning old records from a database
- Sending emails or notifications
- Generating reports
- Synchronizing data with external systems
- Refreshing caches
- Processing pending records
- Running maintenance tasks
- Calling external APIs periodically

### Example

Imagine an application that needs to remove expired sessions every hour.

Without scheduling:

    Someone needs to manually trigger the operation.

With scheduling:

    Every hour
        ↓
    Spring executes the scheduled method
        ↓
    Application finds expired sessions
        ↓
    Expired sessions are removed

The main idea is:

> Scheduling allows an application to execute tasks automatically based on a defined schedule.

---

## 2. Enabling Scheduling in Spring Boot

Spring Boot provides scheduling support through the `@Scheduled` annotation.

To enable scheduling, add `@EnableScheduling` to a configuration class, usually the main application class.

### Example

    @SpringBootApplication
    @EnableScheduling
    public class Application {

        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }

`@EnableScheduling` tells Spring to detect methods annotated with `@Scheduled` and execute them according to their configuration.

Without `@EnableScheduling`, the scheduled methods will not be automatically executed.

---

## 3. Basic Scheduling

The simplest scheduled task uses `@Scheduled`.

### Example

    @Component
    public class NotificationScheduler {

        @Scheduled(fixedRate = 5000)
        public void sendNotifications() {
            System.out.println("Checking notifications...");
        }
    }

The method will execute every 5 seconds.

### Scheduled Method Requirements

A scheduled method should generally:

- Be managed by Spring
- Have no parameters
- Usually return `void`
- Be annotated with `@Scheduled`

For example:

    @Component
    public class MyScheduler {

        @Scheduled(fixedRate = 5000)
        public void executeTask() {
            // Task logic
        }
    }

---

# 4. fixedRate

`fixedRate` executes a method at a fixed interval measured from the **start time of the previous execution**.

### Example

    @Scheduled(fixedRate = 5000)
    public void executeTask() {
        System.out.println("Task executed");
    }

The scheduling concept is:

    Execution starts
          ↓
       5 seconds
          ↓
    Next execution starts
          ↓
       5 seconds
          ↓
    Next execution starts

For example:

    @Scheduled(fixedRate = 10000)
    public void processOrders() {
        System.out.println("Processing orders...");
    }

The task is scheduled every 10 seconds.

### When to Use fixedRate

Use `fixedRate` when you want a task to be triggered at regular intervals.

Typical examples:

- Polling an API
- Checking system status
- Refreshing information
- Periodically collecting metrics

---

# 5. fixedDelay

`fixedDelay` executes the method after a fixed amount of time has passed **since the previous execution finished**.

### Example

    @Scheduled(fixedDelay = 5000)
    public void executeTask() {
        System.out.println("Task executed");
    }

The execution flow is:

    Task starts
        ↓
    Task executes
        ↓
    Task finishes
        ↓
    Wait 5 seconds
        ↓
    Task starts again

For example, if the task takes 3 seconds:

    Task starts
        ↓
    3 seconds processing
        ↓
    Task finishes
        ↓
    5 seconds delay
        ↓
    Next execution

Therefore:

    Total interval = execution time + fixedDelay

### When to Use fixedDelay

Use `fixedDelay` when you want to ensure that a specific amount of time passes after a task finishes before it runs again.

This can be useful for:

- Sequential processing
- Polling systems where you want to wait after processing
- Tasks that should not immediately execute again after completion

---

# 6. fixedRate vs fixedDelay

| Feature             | fixedRate                   | fixedDelay                       |
| ------------------- | --------------------------- | -------------------------------- |
| Reference point     | Start of previous execution | End of previous execution        |
| Delay measured from | Execution start             | Execution completion             |
| Typical use         | Regular periodic execution  | Wait between executions          |
| Example             | Poll every 10 seconds       | Wait 10 seconds after processing |

### Simple Rule

Think about it this way:

    fixedRate
        ↓
    "Run every X seconds."

    fixedDelay
        ↓
    "Wait X seconds after finishing, then run again."

---

# 7. initialDelay

`initialDelay` specifies how long Spring should wait before executing the scheduled method for the first time.

### Example

    @Scheduled(
        fixedRate = 10000,
        initialDelay = 5000
    )
    public void executeTask() {
        System.out.println("Task executed");
    }

The execution flow is:

    Application starts
          ↓
       5 seconds
          ↓
    First execution
          ↓
       10 seconds
          ↓
    Second execution
          ↓
       10 seconds
          ↓
    Third execution

### Why Use initialDelay?

`initialDelay` can be useful when:

- The application needs time to initialize
- External services need time to become available
- Database initialization needs to finish
- You do not want a task to execute immediately after startup

---

# 8. Combining fixedRate, fixedDelay and initialDelay

`initialDelay` can be combined with `fixedRate` or `fixedDelay`.

### fixedRate + initialDelay

    @Scheduled(
        fixedRate = 60000,
        initialDelay = 10000
    )
    public void refreshCache() {
        System.out.println("Refreshing cache...");
    }

Execution:

    Application starts
          ↓
       10 seconds
          ↓
    First execution
          ↓
       60 seconds
          ↓
    Second execution
          ↓
       60 seconds
          ↓
    Third execution

### fixedDelay + initialDelay

    @Scheduled(
        fixedDelay = 60000,
        initialDelay = 10000
    )
    public void cleanupDatabase() {
        System.out.println("Cleaning database...");
    }

The application waits 10 seconds before the first execution and then waits 60 seconds after each execution finishes.

---

# 9. Cron Expressions

Cron expressions allow you to define more flexible calendar-based schedules.

Example:

    @Scheduled(cron = "0 0 * * * *")
    public void executeTask() {
        System.out.println("Task executed every hour");
    }

Spring uses a six-field cron expression:

    ┌──────── second (0-59)
    │ ┌────── minute (0-59)
    │ │ ┌──── hour (0-23)
    │ │ │ ┌── day of month (1-31)
    │ │ │ │ ┌ month (1-12 or JAN-DEC)
    │ │ │ │ │ ┌ day of week (0-7 or SUN-SAT)
    │ │ │ │ │ │
    * * * * * *

The structure is:

    second minute hour day-of-month month day-of-week

Cron is useful when the schedule is based on a specific time rather than simply an interval.

---

# 10. Cron Examples

## Every Minute

    @Scheduled(cron = "0 * * * * *")
    public void executeTask() {
        System.out.println("Runs every minute");
    }

## Every Hour

    @Scheduled(cron = "0 0 * * * *")
    public void executeTask() {
        System.out.println("Runs every hour");
    }

## Every Day at Midnight

    @Scheduled(cron = "0 0 0 * * *")
    public void executeTask() {
        System.out.println("Runs every day at midnight");
    }

## Every Day at 8:00 AM

    @Scheduled(cron = "0 0 8 * * *")
    public void executeTask() {
        System.out.println("Runs every day at 8 AM");
    }

## Every Monday at 9:00 AM

    @Scheduled(cron = "0 0 9 * * MON")
    public void executeTask() {
        System.out.println("Runs every Monday at 9 AM");
    }

## Every 15 Minutes

    @Scheduled(cron = "0 */15 * * * *")
    public void executeTask() {
        System.out.println("Runs every 15 minutes");
    }

## Every Weekday at 6 PM

    @Scheduled(cron = "0 0 18 * * MON-FRI")
    public void executeTask() {
        System.out.println("Runs every weekday at 6 PM");
    }

---

# 11. Cron Special Characters

Cron expressions support several special characters.

| Character | Meaning           |
| --------- | ----------------- |
| `*`       | Any value         |
| `,`       | Multiple values   |
| `-`       | Range             |
| `/`       | Increment         |
| `?`       | No specific value |

### `*` - Any Value

    * * * * * *

The `*` means any value for that field.

### `/` - Increment

    */10

Means every 10 units.

Example:

    @Scheduled(cron = "0 */10 * * * *")

Runs every 10 minutes.

### `-` - Range

    MON-FRI

Means Monday through Friday.

Example:

    @Scheduled(cron = "0 0 9 * * MON-FRI")

Runs at 9 AM from Monday to Friday.

### `,` - Multiple Values

    MON,WED,FRI

Means Monday, Wednesday and Friday.

Example:

    @Scheduled(cron = "0 0 9 * * MON,WED,FRI")

Runs at 9 AM on Monday, Wednesday and Friday.

---

# 12. Cron with application.properties

Instead of hardcoding the cron expression directly in Java, it can be stored in application configuration.

### application.properties

    scheduler.cleanup.cron=0 0 2 * * *

Java:

    @Component
    public class CleanupScheduler {

        @Scheduled(cron = "${scheduler.cleanup.cron}")
        public void cleanup() {
            System.out.println("Running cleanup...");
        }
    }

This approach is useful because the schedule can be changed without modifying the Java source code.

---

# 13. Cron with application.yml

The same configuration can be represented using YAML.

### application.yml

    scheduler:
      cleanup:
        cron: "0 0 2 * * *"

Java:

    @Scheduled(cron = "${scheduler.cleanup.cron}")
    public void cleanup() {
        System.out.println("Running cleanup...");
    }

Using configuration makes it easier to have different schedules for different environments.

For example:

    Development:
        every 5 minutes

    Production:
        every day at 2 AM

The Java code does not need to change.

---

# 14. Dynamic Scheduling

Static scheduling is appropriate when the schedule is known in advance.

For example:

    @Scheduled(cron = "0 0 2 * * *")

However, sometimes the schedule needs to be changed dynamically while the application is running.

Examples:

- A user chooses when a report should run
- A schedule is stored in a database
- An administrator changes a schedule through an API
- Different customers have different schedules
- The application calculates the next execution dynamically

In these cases, `@Scheduled` alone may not be sufficient.

---

# 15. TaskScheduler

Spring provides `TaskScheduler` for programmatic scheduling.

A scheduler can be configured as a Spring bean.

Example:

    @Configuration
    @EnableScheduling
    public class SchedulingConfig {

        @Bean
        public TaskScheduler taskScheduler() {
            return new ThreadPoolTaskScheduler();
        }
    }

The scheduler can then be injected into a service.

    @Service
    public class DynamicSchedulerService {

        private final TaskScheduler taskScheduler;

        public DynamicSchedulerService(TaskScheduler taskScheduler) {
            this.taskScheduler = taskScheduler;
        }

        public void scheduleTask() {

            taskScheduler.schedule(
                () -> System.out.println("Dynamic task executed"),
                new CronTrigger("0 */5 * * * *")
            );
        }
    }

The important difference is that the schedule can now be provided programmatically.

---

# 16. Dynamic Cron Expression

Imagine that a cron expression is stored in a database.

For example:

    String cronExpression =
        scheduleRepository.getCronExpression();

The application can create a `CronTrigger` dynamically.

    CronTrigger trigger =
        new CronTrigger(cronExpression);

    taskScheduler.schedule(
        () -> executeTask(),
        trigger
    );

This makes it possible to change the schedule without changing the source code.

---

# 17. Cancelling a Dynamic Task

When using dynamic scheduling, you may need to cancel an existing task.

`TaskScheduler.schedule()` returns a `ScheduledFuture`.

Example:

    ScheduledFuture<?> scheduledTask =
        taskScheduler.schedule(
            () -> executeTask(),
            new CronTrigger("0 */5 * * * *")
        );

The task can be cancelled:

    scheduledTask.cancel(false);

This can be useful when:

- A user disables a schedule
- An administrator changes a schedule
- A scheduled job is deleted
- A customer cancels a scheduled operation

---

# 18. Dynamic Scheduling Example

Imagine an application where an administrator can configure when a report should be generated.

The architecture could look like:

    Admin
      ↓
    Configure Schedule
      ↓
    REST API
      ↓
    Save Schedule
      ↓
    Database
      ↓
    DynamicSchedulerService
      ↓
    TaskScheduler
      ↓
    Generate Report

Example:

    @Service
    public class ReportSchedulerService {

        private final TaskScheduler taskScheduler;

        private ScheduledFuture<?> scheduledTask;

        public ReportSchedulerService(TaskScheduler taskScheduler) {
            this.taskScheduler = taskScheduler;
        }

        public void scheduleReport(String cronExpression) {

            if (scheduledTask != null) {
                scheduledTask.cancel(false);
            }

            scheduledTask = taskScheduler.schedule(
                this::generateReport,
                new CronTrigger(cronExpression)
            );
        }

        private void generateReport() {
            System.out.println("Generating report...");
        }
    }

Now the application can change the schedule without restarting.

---

# 19. Scheduling and Thread Pools

When an application has multiple scheduled tasks, thread-pool configuration becomes important.

A scheduler can be configured with multiple threads.

Example:

    @Configuration
    @EnableScheduling
    public class SchedulingConfig {

        @Bean
        public TaskScheduler taskScheduler() {

            ThreadPoolTaskScheduler scheduler =
                new ThreadPoolTaskScheduler();

            scheduler.setPoolSize(5);
            scheduler.setThreadNamePrefix("scheduler-");
            scheduler.initialize();

            return scheduler;
        }
    }

The pool allows multiple scheduled tasks to execute using different scheduler threads.

Without appropriate configuration, multiple tasks may compete for a limited number of scheduler threads.

---

# 20. Long-Running Scheduled Tasks

Be careful with scheduled methods that take a long time to execute.

For example:

    @Scheduled(fixedRate = 5000)
    public void processLargeJob() {

        // Takes 30 minutes
    }

A long-running task can interfere with other scheduled tasks depending on the scheduler configuration.

For heavy workloads, consider:

- Configuring a thread pool
- Using asynchronous execution
- Using Spring Batch
- Using a message queue
- Using a distributed job scheduler
- Using an external scheduling platform

Scheduling is usually responsible for **triggering** work, not necessarily for performing large-scale processing.

---

# 21. Scheduling vs Spring Batch

Spring Scheduling and Spring Batch solve different problems.

### Spring Scheduling

Spring Scheduling focuses on:

> **When should something execute?**

Example:

    Every day at 2 AM
            ↓
    Start cleanup task

### Spring Batch

Spring Batch focuses on:

> **How should a large batch workload be processed?**

Example:

    2 AM
     ↓
    Start Spring Batch Job
     ↓
    Read 10 million records
     ↓
    Process records in chunks
     ↓
    Write results
     ↓
    Finish

They can also be used together.

    Spring Scheduler
          ↓
    Starts
          ↓
    Spring Batch Job
          ↓
    Processes large dataset

A common architecture is:

    @Scheduled
        ↓
    Start Batch Job
        ↓
    Spring Batch
        ↓
    Process large amount of data

---

# 22. Time Zones

Cron schedules can be affected by the application's time zone.

You can explicitly specify a time zone.

Example:

    @Scheduled(
        cron = "0 0 9 * * MON-FRI",
        zone = "America/Vancouver"
    )
    public void executeTask() {
        System.out.println("Runs at 9 AM Vancouver time");
    }

This is especially important for applications deployed across multiple regions.

For example:

    Application Server
          ↓
        UTC

    Business Requirement
          ↓
    Vancouver Time

Explicitly specifying the time zone helps avoid unexpected execution times.

---

# 23. Best Practices

## 23.1 Keep Scheduled Methods Small

Avoid putting large amounts of business logic directly inside the scheduler.

Prefer:

    @Scheduled(cron = "0 0 2 * * *")
    public void execute() {
        reportService.generateReports();
    }

The scheduler should primarily be responsible for triggering the operation.

The business logic should remain inside the appropriate service.

---

## 23.2 Use Configuration for Schedules

Avoid hardcoding schedules when they may change.

Instead of:

    @Scheduled(cron = "0 0 2 * * *")

Prefer:

    @Scheduled(cron = "${scheduler.report.cron}")

This makes configuration easier across environments.

---

## 23.3 Add Logging

Scheduled tasks should provide enough logging to understand when they execute.

Example:

    @Scheduled(cron = "${scheduler.cleanup.cron}")
    public void cleanup() {

        log.info("Starting cleanup task");

        cleanupService.cleanup();

        log.info("Cleanup task completed");
    }

Logging becomes especially important when scheduled tasks run in production without direct user interaction.

---

## 23.4 Handle Exceptions

Scheduled tasks should handle failures appropriately.

Example:

    @Scheduled(fixedRate = 60000)
    public void execute() {

        try {
            service.process();
        } catch (Exception ex) {
            log.error("Error executing scheduled task", ex);
        }
    }

The exact error-handling strategy depends on the application.

For critical jobs, consider:

- Retry mechanisms
- Alerting
- Dead-letter queues
- Job status tracking
- Persistent job execution history

---

## 23.5 Be Careful with Multiple Application Instances

Consider an application deployed with three instances:

    Load Balancer
          |
      +---+---+
      |   |   |
     App App App
      #1  #2  #3

If all three instances contain:

    @Scheduled(cron = "0 0 2 * * *")

the scheduled task may execute on all three instances.

That can cause duplicate processing.

For example:

    App #1 → Cleanup
    App #2 → Cleanup
    App #3 → Cleanup

The same operation may therefore execute three times.

For distributed environments, consider:

- Distributed locks
- ShedLock
- Quartz
- External schedulers
- Database-based coordination
- Kubernetes CronJobs
- Cloud scheduling services

---

# 24. Common Mistakes

## Forgetting @EnableScheduling

Incorrect:

    @SpringBootApplication
    public class Application {
    }

Correct:

    @SpringBootApplication
    @EnableScheduling
    public class Application {
    }

Without `@EnableScheduling`, scheduled methods will not execute automatically.

---

## Forgetting to Register the Class as a Spring Bean

Incorrect:

    public class MyScheduler {

        @Scheduled(fixedRate = 5000)
        public void execute() {
        }
    }

Correct:

    @Component
    public class MyScheduler {

        @Scheduled(fixedRate = 5000)
        public void execute() {
        }
    }

Spring needs to manage the object for scheduling annotations to be detected.

---

## Confusing fixedRate and fixedDelay

Remember:

    fixedRate
        ↓
    Interval based on execution start

    fixedDelay
        ↓
    Delay after execution finishes

---

## Hardcoding Configuration

Instead of:

    @Scheduled(cron = "0 0 2 * * *")

Consider:

    @Scheduled(cron = "${scheduler.cleanup.cron}")

when the schedule is environment-specific or expected to change.

---

# 25. Complete Example

A simple scheduled service can look like this:

    @Component
    public class CleanupScheduler {

        private static final Logger log =
            LoggerFactory.getLogger(CleanupScheduler.class);

        private final CleanupService cleanupService;

        public CleanupScheduler(CleanupService cleanupService) {
            this.cleanupService = cleanupService;
        }

        @Scheduled(
            cron = "${scheduler.cleanup.cron}",
            zone = "${scheduler.timezone}"
        )
        public void cleanup() {

            log.info("Starting scheduled cleanup");

            try {
                cleanupService.cleanup();

                log.info("Scheduled cleanup completed");

            } catch (Exception ex) {
                log.error("Scheduled cleanup failed", ex);
            }
        }
    }

### application.yml

    scheduler:
      cleanup:
        cron: "0 0 2 * * *"
      timezone: "America/Vancouver"

### Main Application

    @SpringBootApplication
    @EnableScheduling
    public class Application {

        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }

The execution flow is:

    Spring Boot starts
          ↓
    @EnableScheduling
          ↓
    Spring detects @Scheduled
          ↓
    Cron expression is loaded
          ↓
    Every day at 2:00 AM
          ↓
    CleanupScheduler executes
          ↓
    CleanupService performs business operation

---

# 26. Summary

Spring Boot Scheduling provides a simple way to execute tasks automatically.

| Concept                   | Purpose                                               |
| ------------------------- | ----------------------------------------------------- |
| `@EnableScheduling`       | Enables scheduling support                            |
| `@Scheduled`              | Defines a scheduled task                              |
| `fixedRate`               | Executes at a fixed interval based on execution start |
| `fixedDelay`              | Waits a fixed time after execution finishes           |
| `initialDelay`            | Delays the first execution                            |
| `cron`                    | Provides calendar-based scheduling                    |
| `TaskScheduler`           | Provides programmatic/dynamic scheduling              |
| `CronTrigger`             | Defines a dynamic cron-based trigger                  |
| `ScheduledFuture`         | Allows a dynamic task to be cancelled                 |
| `ThreadPoolTaskScheduler` | Configures scheduler threads                          |

### Choosing the Right Approach

    Need a simple periodic task?
            ↓
    @Scheduled + fixedRate/fixedDelay

    Need a calendar-based schedule?
            ↓
    @Scheduled + cron

    Need a configurable schedule?
            ↓
    @Scheduled + application.properties/application.yml

    Need a schedule that changes at runtime?
            ↓
    TaskScheduler

    Need to process millions of records?
            ↓
    Consider Spring Batch

    Need distributed scheduling across multiple instances?
            ↓
    Consider distributed locks or an external scheduler

---

## Key Takeaway

> **Spring Scheduling answers "when should this task run?", while the business service should define "what should the task do?".**

Scheduling is ideal for triggering periodic or time-based operations, while technologies such as Spring Batch, message queues, or distributed job schedulers may be more appropriate when the actual workload is large, long-running, or distributed.
