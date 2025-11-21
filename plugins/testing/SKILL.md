---
name: testing
description: Expert guidance on software testing strategies, methodologies, and best practices. Provides language-agnostic advice for designing test strategies, choosing appropriate test levels (unit, integration, E2E), applying methodologies (TDD, BDD), optimizing CI/CD pipelines, and testing legacy code. Use when designing test strategies, deciding between testing approaches, refactoring tests, or optimizing test suites.
---

# Testing Strategy Expert

## Overview

Transform Claude into an expert testing consultant providing comprehensive, language-agnostic guidance on software testing strategies, methodologies, and best practices. This skill enables informed decisions about test design, coverage strategy, CI/CD integration, and legacy code testing.

**Note on Examples**: This skill uses Python syntax in code examples for clarity and consistency. However, all principles, strategies, and methodologies apply universally across programming languages including JavaScript, PHP, Java, C#, Ruby, Go, Rust, and others. The testing concepts transcend specific languages and frameworks—adapt the patterns to your language's idioms and testing tools.

## Core Testing Principles

Apply these foundational principles across all testing decisions:

### 1. Test Behavior, Not Implementation
Verify observable outcomes and public contracts rather than internal implementation details. Tests coupled to implementation break during safe refactoring. Focus on what the system does from a user's or caller's perspective.

**Example**: Test that a login function succeeds with valid credentials and fails appropriately with invalid ones, not that it calls specific internal methods in a particular order.

### 2. Optimize for Signal Quality Over Coverage
A test suite with 80% coverage that catches real bugs beats one with 100% coverage that never fails meaningfully. Focus on tests that provide clear, actionable feedback when they break, not coverage percentages.

### 3. Design for Fast Feedback Loops
Speed of feedback directly impacts developer productivity. Structure test suites for rapid feedback: fast unit tests run pre-commit, comprehensive suites run post-merge, exhaustive tests run nightly.

### 4. Ensure Test Independence and Determinism
Every test should be independent, produce consistent results, and avoid shared state. Independent tests can run in parallel, fail clearly, and be debugged easily without cascading failures.

### 5. Apply Tests at the Appropriate Level
Use unit tests for logic and edge cases, integration tests for component boundaries, E2E tests for critical paths. Testing everything through the UI wastes time; testing everything with mocks provides false confidence.

## Quick Decision Framework

Use this framework to rapidly determine the appropriate testing approach:

### When to Use Unit Tests
- Business logic, algorithms, calculations
- Data transformations and pure functions
- Edge cases and boundary conditions
- Validation rules and complex conditionals
- Any scenario expressible as "given input X, expect output Y"

**Target**: 70-80% of test suite, focusing on business logic coverage

### When to Use Integration Tests
- Database queries and ORM behavior
- API endpoints with framework integration
- Service-to-service communication
- Cross-cutting concerns (auth, caching, transactions)
- Framework-specific behavior validation

**Target**: 15-25% of test suite, focusing on component boundaries

### When to Use E2E Tests
- Critical user journeys only (login, checkout, payment)
- Complete user workflows through the entire system
- Cross-system integration validation
- Production-like environment verification

**Target**: 5-10% of test suite, focusing on critical paths only

### When to Use Contract Tests
- Microservices architectures with independent service deployment
- Third-party API integrations
- Multi-team service development
- Preventing breaking changes at service boundaries

## Testing Methodology Selection

### Apply Test-Driven Development (TDD) When:
- Building complex business logic with well-understood requirements
- Exploring API design and interface contracts
- Implementing algorithms or data structures
- Building pure functions with clear input/output specifications

**Skip TDD for**: UI implementation, proof-of-concept work, framework exploration, simple glue code

### Apply Behavior-Driven Development (BDD) When:
- Building user-facing features requiring stakeholder collaboration
- Complex workflows where examples clarify requirements better than abstract descriptions
- Needing living documentation that business users can read
- Aligning product, engineering, and QA teams

**Skip BDD for**: Technical components, utility functions, projects without business stakeholder involvement

### Test-Later Approach When:
- Prototyping or spike solutions
- Working with legacy code (characterization tests first)
- Simple scripts or one-off tools
- Time-boxed experiments

## Architecture-Specific Strategies

### Monolithic Applications
Use the traditional testing pyramid: 70-80% unit tests, 15-20% integration tests, 5-10% E2E tests. Optimize for fast unit test execution while ensuring integration tests verify database and framework integration.

### Microservices Architectures
Shift toward the testing honeycomb with greater emphasis on integration and contract tests. In distributed systems, the highest risk lies in service interactions. Prioritize contract testing for service compatibility.

### Legacy Code
Start with characterization tests to document current behavior, identify seams for testing, break dependencies using safe refactorings, and prioritize testing by risk (critical + frequently changed code first).

## Test Design Best Practices

### Structure Tests with Arrange-Act-Assert (AAA)
- **Arrange**: Set up test environment, create objects, prepare test data
- **Act**: Execute the code being tested (single, clear action)
- **Assert**: Verify expected outcome with specific, meaningful assertions

Keep tests focused on one logical assertion per test for clarity.

### Ensure Deterministic Tests
- Eliminate timing dependencies (use conditional waits, not fixed sleeps)
- Avoid shared state between tests
- Mock external dependencies (APIs, databases, file systems)
- Use fixed seeds for random data or freeze time in tests

### Apply Boundary Analysis
Test edge cases systematically: empty inputs, single element, boundary values (0, -1, max), just below/above limits, invalid inputs. This technique finds off-by-one errors and edge case handling issues.

### Write Expressive Tests
- Use descriptive test names: `test_[function]_[scenario]_[expected_result]`
- Include meaningful assertion messages for complex conditions
- Keep tests short and readable
- Make test failures immediately communicate what broke and why

### Mock Strategically
- **Stub** query dependencies that provide data
- **Mock** command dependencies where verifying interaction matters
- **Fake** complex dependencies needing realistic behavior (in-memory databases)
- **Avoid over-mocking** internal collaborators—mock only at system boundaries

## Common Anti-Patterns to Avoid

1. **Testing Implementation Details**: Tests break during refactoring despite unchanged behavior
2. **Over-Mocking**: Mocking every dependency couples tests to implementation
3. **Brittle Tests**: Tests coupled to UI selectors, private methods, internal state
4. **Inverted Pyramid**: More integration/E2E tests than unit tests leads to slow, flaky suites
5. **Coverage Chasing**: Optimizing for coverage percentage over test quality
6. **Flaky Tests**: Non-deterministic tests that erode trust in the test suite
7. **Global State**: Shared mutable state creates order dependencies

## Coverage Strategy

Focus on behavior coverage (scenarios validated) over code coverage (lines executed):

- **Critical paths and business logic**: 90-100% coverage at unit level
- **Integration-heavy code**: 70-90% coverage at integration level
- **Edge cases and error handling**: Comprehensive coverage at unit level
- **Cross-cutting concerns**: Integration-level coverage

Prioritize testing based on risk: business-critical code, frequently changed code, historically bug-prone areas deserve more thorough coverage.

## CI/CD Integration

Structure test execution for optimal feedback:

1. **Pre-commit/Pre-push**: Fast unit tests (<30s) with linting
2. **Pull Request Pipeline**: All unit tests + critical integration tests (10-15 min)
3. **Post-merge**: Full regression suite including slower integration tests (20-45 min)
4. **Nightly/Scheduled**: Exhaustive E2E, performance, security tests (hours)

Parallelize test execution, detect and quarantine flaky tests immediately, and track test health metrics (defect detection efficiency, execution time, flakiness rate).

## Bundled Resources

This skill includes comprehensive reference documentation for deep dives into specific testing topics:

### references/testing-layers.md
Detailed guidance on unit, integration, E2E, and contract testing including specific scenarios, characteristics, and examples for each level.

### references/methodologies.md
Comprehensive coverage of TDD, BDD, test-later approaches, and public behavior testing with workflow examples.

### references/best-practices.md
In-depth test design patterns including deterministic testing, boundary analysis, test data management (fixtures, factories, builders), and mocking strategies.

### references/anti-patterns.md
Common testing mistakes with explanations of why they're problematic and how to fix them.

### references/ci-cd-optimization.md
Strategies for structuring test suites for fast feedback, parallel execution, flaky test management, and test health metrics.

### references/legacy-code.md
Techniques for testing untestable code including characterization testing, identifying seams, breaking dependencies, and prioritization strategies.

### references/coverage-strategy.md
Guidance on code vs behavior coverage, appropriate coverage targets by level, when coverage becomes harmful, and risk-based prioritization.

## Usage

When users ask about testing topics, consult the appropriate reference file for detailed guidance. Apply the core principles and quick decision framework first, then dive into specific methodologies or patterns as needed.

Always provide context-aware recommendations that balance pragmatism with best practices, explaining trade-offs and helping teams make informed decisions rather than prescriptive rules.
