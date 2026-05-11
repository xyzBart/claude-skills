---
name: java
description: Java coding conventions and best practices: language features, testing patterns, and standard library preferences. Apply when writing or reviewing Java code.
version: 1.0.0
allowed-tools: Read, Edit, Write, Bash
---

# Java Coding Conventions

## Language features

Prefer Java 21 language features.
Prefer unchecked exceptions over checked, unless there is good reason to use checked exceptions. Wrap IOExceptions in UncheckedIOException in methods and lambdas.
In methods, put unchecked exceptions in method signature.
In lambdas, `Throwing` utility class can be used to wrap checked exceptions.

## Testing

JUnit 5 is used for both unit and integration tests.

Unit tests should not depend on external resources, but can use local file system. For methods which operate on file system, simply create temporary files and directories in OS temp dir, with root name under temp equal to test case name. Delete them after test.

When external resources are needed, use integration tests.

`maven-failsafe-plugin` is properly configured to run integration tests in the verify phase. Integration tests should be named `*IT.java` and placed under the `it` package in the standard `src/test/java` tree. They are standard JUnit tests, but can use `@BeforeAll` and `@AfterAll` static methods to setup and cleanup resources, e.g. start and stop an HTTP server.

- **Unit test** = JUnit test case with no external deps (local file system is fine).
- **Integration test** = JUnit `*IT` test case which may depend on external resources.

## Logging

Prefer slf4j over `System.out.println` (except in `main` methods). In test cases, use info level.

When creating a new class, add a logger instance:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private static final Logger logger = LoggerFactory.getLogger(ClassName.class);
```
