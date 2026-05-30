---
name: dotnet-testing-guide
description: Test writing conventions and preferred frameworks for the FloreriaStore backend.
---

# Testing Guide

All new features should include comprehensive tests.

## Unit Tests
Focus on testing domain logic and application handlers in isolation.
- **Framework:** `xUnit`.
- **Mocking:** `Moq` or `NSubstitute`. Use mocks to fake dependencies like Repositories when testing Application Handlers.
- **Assertions:** `FluentAssertions`.

## Integration Tests
Focus on testing the interaction with the database and external infrastructure.
- **Framework:** Use `Testcontainers` (with the Postgres module). This is mandatory to properly test against a real PostgreSQL instance instead of an in-memory database.
- **Scope:** Test API endpoints end-to-end (from HTTP request to Database and back), or test Repositories querying real dBs via EF Core/Dapper.

## Test Structure
Follow the **Arrange, Act, Assert (AAA)** pattern in all test methods.
