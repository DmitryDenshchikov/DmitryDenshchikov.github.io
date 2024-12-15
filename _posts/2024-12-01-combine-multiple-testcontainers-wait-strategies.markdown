---
layout: post
title: "Combine multiple Testcontainers wait strategies"
date: 2024-12-01 12:00:00 +0100
categories: jekyll update
---

# Introduction
At some point, when I was using `PostgreSQLContainer` for integration tests I faced the next exception
during a database migrations invocation:

```
Caused by: org.postgresql.util.PSQLException: Connection to localhost:<port> refused. 
Check that the hostname and port are correct and that the postmaster is accepting TCP/IP connections.
```

The error was intermittent, but eventually, it was discovered, that despite the fact that a database
in the container is ready to accept connections, **the container’s exposed port is not**. It has been caused
by the container environment that was used. The issue is quite common and takes a considerable amount 
of time to be identified and resolved.

I had to use `HostPortWaitStrategy` as a wait strategy for the container. But I also wanted to be 100% sure,
that the database is ready to accept connections. So, the default `PostgreSQLContainer` wait strategy
should have been applied as well. That is where a Testcontainers wait strategies combination helped me.

Below, you can find an example of `PostgreSQLContainer` configuration with multiple wait strategies.

# Configuration class
In the configuration I use the `WITH_MAXIMUM_OUTER_TIMEOUT` mode as an example.
Using this mode, the inner strategies wait with their own timeouts, but the `WaitAllStrategy` throws an exception, 
if its limit is reached (which is specified in `withStartupTimeout`). The other two modes are described comprehensively 
in Testcontainers Javadocs, so you can choose the one that suits you best.

```java
package denshchikov.dmitry.taskcreator.config;

import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.containers.wait.strategy.WaitAllStrategy;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.time.Duration;

import static org.testcontainers.containers.wait.strategy.WaitAllStrategy.Mode.WITH_MAXIMUM_OUTER_TIMEOUT;

@Testcontainers
public interface IntegrationTest {

  @Container
  PostgreSQLContainer<?> postgresContainer =
      new PostgreSQLContainer<>("postgres:latest")
          .waitingFor(
              new WaitAllStrategy(WITH_MAXIMUM_OUTER_TIMEOUT)
                  .withStartupTimeout(Duration.ofSeconds(90))
                  .withStrategy(
                      Wait.forLogMessage(
                          ".*database system is ready to accept connections.*\\s", 2))
                  .withStrategy(Wait.forListeningPort()));

  @DynamicPropertySource
  static void datasourceProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.username", postgresContainer::getUsername);
    registry.add("spring.datasource.password", postgresContainer::getPassword);
    registry.add("spring.datasource.url", postgresContainer::getJdbcUrl);
  }
}
```

# Usage Example
Then you can just implement the interface in your test classes. Here is an example with a Spring Boot test.

```java
import denshchikov.dmitry.taskcreator.config.IntegrationTest;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public class TaskServiceTest implements IntegrationTest {
    // ...
}
```
You can also find the full project in my repository. 
It contains a service class that should be tested, Flyway DB migrations, the configuration and the tests.
- Java version: https://github.com/DmitryDenshchikov/task-creator/tree/main/app-java
- Kotlin version: https://github.com/DmitryDenshchikov/task-creator/tree/main/app-kotlin

