# Testing Layers: Detailed Guidance

## Unit Tests: The Foundation

Unit tests verify individual units of code in isolation—typically a single function, method, or class. They represent the foundation of most testing strategies because they are fast, focused, and provide precise failure diagnostics.

### Characteristics

- **Speed**: Milliseconds per test
- **Isolation**: No external dependencies (databases, file systems, networks)
- **Focus**: Single unit of functionality
- **Precision**: Failures point directly to the problematic code
- **Coverage**: Can exhaustively test edge cases and error conditions

### When to Use Unit Tests

Unit tests excel in these scenarios:

**Business Logic and Algorithms**
- Order calculations, discount logic, pricing rules
- Validation logic and data transformation
- Complex conditional logic with multiple branches
- Mathematical calculations and algorithms
- Pure functions with deterministic outputs

**Examples**:
- Testing that a discount calculation applies correct percentages based on order total
- Verifying that password validation rejects weak passwords
- Confirming that a sorting algorithm handles edge cases (empty arrays, single elements)
- Checking that currency conversion produces accurate results

**Edge Cases and Boundary Conditions**
- Null or undefined inputs
- Empty collections (arrays, strings, objects)
- Boundary values (0, -1, maximum integers)
- Invalid inputs and error conditions
- Concurrent access scenarios

**Examples**:
- Testing that a function handles null inputs gracefully
- Verifying behavior with empty strings or arrays
- Confirming off-by-one errors don't exist at boundaries
- Ensuring proper error messages for invalid inputs

**Data Transformations and Parsing**
- JSON/XML parsing and serialization
- Data mapping between models
- String manipulation and formatting
- Data validation and sanitization

### When NOT to Use Unit Tests

Unit tests become problematic or low-value in these scenarios:

**Database Operations**
Testing database queries, ORM behavior, or data access layers with mocked repositories provides limited value. The real risk lies in SQL correctness, schema compatibility, and query performance—aspects that mocks cannot verify. Use integration tests with real databases instead.

**Framework-Specific Behavior**
Testing that a web framework correctly handles routing, middleware, request parsing, or dependency injection through pure unit tests creates more problems than solutions. These behaviors depend on framework implementation details and are better verified through integration tests.

**UI Rendering and Interaction**
Testing that a React component correctly calls a callback function provides less value than testing that it renders the right UI for given props. UI testing belongs at the integration or E2E level where actual rendering can be verified.

**Cross-Service Communication**
Testing API clients, message queue handlers, or service-to-service communication with mocks cannot verify actual network behavior, serialization, error handling, or compatibility. Integration or contract tests are more appropriate.

**Simple Pass-Through Code**
Trivial getters, setters, constructors, or delegation code that simply forwards calls without logic don't benefit from unit tests. Testing these creates maintenance burden without meaningful quality improvement.

### Unit Test Design Patterns

**Arrange-Act-Assert (AAA) Structure**
```
# Arrange
user = User(email="test@example.com", age=25)
validator = UserValidator()

# Act
result = validator.validate(user)

# Assert
assert result.is_valid == True
assert result.errors == []
```

**Parameterized Tests for Multiple Cases**
```
@pytest.mark.parametrize("age,expected", [
    (-1, False),  # Below minimum
    (0, False),   # Boundary
    (17, False),  # Just below valid
    (18, True),   # Minimum valid
    (65, True),   # Typical valid
    (120, True),  # Maximum valid
    (121, False), # Above maximum
])
def test_age_validation(age, expected):
    assert validate_age(age) == expected
```

**Test Doubles for Dependencies**
```
# Stub: Provides canned responses
class StubUserRepository:
    def find_by_id(self, user_id):
        return User(id=user_id, name="Test User")

# Mock: Verifies interactions
mock_email_service = Mock()
user_service.register(user)
mock_email_service.send_welcome_email.assert_called_once_with(user.email)
```

### Recommended Coverage Targets

**High Coverage (90-100%)**
- Business logic and domain models
- Algorithmic code and calculations
- Validation and transformation logic
- Complex conditional logic

**Moderate Coverage (70-90%)**
- Utility functions and helpers
- Data access objects (excluding actual DB calls)
- API response mappers
- Configuration and initialization code

**Low Coverage (<70%) Acceptable**
- Simple property accessors
- Framework-generated code
- Third-party library wrappers
- Code better tested at integration level

---

## Integration Tests: Verifying Component Interactions

Integration tests verify how multiple components work together, focusing on boundaries and interactions between modules, services, or layers. They occupy the middle ground between isolated unit tests and full-system E2E tests.

### Characteristics

- **Speed**: Seconds per test
- **Real Dependencies**: Uses actual implementations of some components
- **Boundary Focus**: Tests interfaces between modules/services
- **Moderate Scope**: Multiple components but not entire system
- **Balance**: Realistic behavior with manageable complexity

### When to Use Integration Tests

**Database Integration**
- Testing repository methods with real database (or in-memory equivalent)
- Verifying ORM mappings and query correctness
- Confirming transaction handling and rollback behavior
- Testing database constraints and cascading operations
- Validating query performance with realistic data volumes

**Examples**:
- Verify that saving a user with a duplicate email triggers a constraint violation
- Confirm that deleting a parent record cascades to child records
- Test that a complex JOIN query returns expected results
- Validate that pagination works correctly with large datasets

**API Endpoint Testing**
- Testing HTTP endpoints with the full framework
- Verifying request parsing and validation
- Confirming response serialization and status codes
- Testing middleware behavior (auth, logging, rate limiting)
- Validating error handling and exception mapping

**Examples**:
- POST to `/api/users` with valid data returns 201 with created user
- POST to `/api/users` with invalid email returns 400 with error details
- GET `/api/users/:id` for non-existent user returns 404
- Requests without authentication token return 401

**Message Queue and Event-Driven Integration**
- Testing message producers and consumers
- Verifying message serialization and deserialization
- Confirming event handler registration and invocation
- Testing retry logic and dead letter handling
- Validating message ordering and idempotency

**Examples**:
- Publishing an "OrderCreated" event triggers inventory reservation
- Failed message processing moves message to dead letter queue
- Duplicate messages are processed idempotently
- Message batching works correctly under load

**Cross-Cutting Concerns**
- Authentication and authorization flow
- Caching behavior and invalidation
- Transaction management across layers
- Logging and observability instrumentation
- Rate limiting and throttling

**Examples**:
- Verify that cached data is returned on subsequent requests
- Confirm that unauthorized users cannot access protected resources
- Test that failed transactions roll back completely
- Validate that rate limits block excessive requests

### When NOT to Use Integration Tests

**Exhaustive Edge Case Testing**
Integration tests are slower than unit tests and shouldn't be used to test every edge case, boundary condition, or error scenario. Test algorithmic correctness and edge cases at the unit level; use integration tests to verify components integrate properly.

**Complete User Workflows**
Testing entire user workflows through multiple API calls, UI interactions, and system states belongs at the E2E level. Integration tests should focus on specific component interactions, not complete business processes.

**Implementation Detail Verification**
Integration tests should verify observable behavior at component boundaries, not internal implementation details. If an integration test breaks when refactoring internal code without changing external behavior, it's testing the wrong thing.

**Simple Pass-Through Logic**
Code that simply forwards calls between components without transformation or logic doesn't need integration testing. If a service method just delegates to a repository, testing it adds little value.

### Integration Test Design Patterns

**Test Database Management**
```
# Use transactions that rollback
@pytest.fixture
def db_session():
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)

    yield session

    session.close()
    transaction.rollback()
    connection.close()

def test_create_user(db_session):
    user = User(email="test@example.com")
    db_session.add(user)
    db_session.commit()

    found = db_session.query(User).filter_by(email="test@example.com").first()
    assert found is not None
```

**API Testing with Test Client**
```
def test_create_user_api(test_client):
    response = test_client.post('/api/users', json={
        'email': 'test@example.com',
        'name': 'Test User'
    })

    assert response.status_code == 201
    assert response.json['email'] == 'test@example.com'
    assert 'id' in response.json
```

**Container-Based Dependencies**
```
@pytest.fixture(scope="session")
def postgres_container():
    # Start PostgreSQL container
    container = PostgresContainer("postgres:15")
    container.start()

    yield container

    container.stop()

def test_with_real_postgres(postgres_container):
    connection_string = postgres_container.get_connection_url()
    # Test with real PostgreSQL instance
```

### Recommended Coverage Targets

**High Coverage (70-90%)**
- Repository and data access layers
- API controllers and route handlers
- Service layer with framework integration
- Message queue handlers
- Cross-cutting concern implementations

**Selective Coverage**
- Third-party API integrations (sample scenarios)
- File system operations (key workflows)
- External service communication (happy path + critical errors)

---

## End-to-End Tests: Critical Path Validation

End-to-end (E2E) tests verify the entire system from a user's perspective, simulating real user interactions through the UI or API. They provide the highest confidence that the system works as a complete whole but come with significant costs.

### Characteristics

- **Speed**: Seconds to minutes per test
- **Full Stack**: Tests entire system including UI, backend, database
- **User Perspective**: Simulates real user interactions
- **Production-Like**: Runs in environment resembling production
- **Fragility**: More prone to flakiness due to complexity

### When to Use E2E Tests

**Critical User Journeys**
Focus exclusively on workflows that, if broken, would severely impact users or business:

- User registration and login flow
- Checkout and payment processing
- Core feature workflows (search, book, confirm)
- Account management (password reset, profile update)
- Critical integrations (payment gateways, shipping APIs)

**Examples**:
- Complete e-commerce checkout: browse → add to cart → checkout → payment → confirmation
- User onboarding: sign up → verify email → complete profile → access application
- Booking flow: search → select → customize → pay → receive confirmation
- Content publication: create → review → approve → publish → verify live

**Cross-System Integration Validation**
Verify that deployed systems integrate correctly:

- Multiple microservices working together
- Frontend and backend communication
- Third-party service integration
- Email delivery and notifications
- File uploads and processing

**Examples**:
- Order placement triggers inventory service, payment service, and notification service
- File upload processes correctly through CDN to storage to thumbnail generation
- OAuth integration with external provider completes successfully

**Smoke Tests for Deployment Validation**
Minimal E2E tests that verify deployment succeeded:

- Application starts and responds
- Critical paths are accessible
- Database connections work
- External service connectivity verified

### When NOT to Use E2E Tests

**Edge Cases and Error Handling**
Testing every validation error message, boundary condition, or error scenario through the UI is slow and brittle. Test edge cases at the unit level where they can be verified quickly and precisely.

**Examples of What NOT to E2E Test**:
- ❌ Every possible validation error on a registration form
- ❌ All password strength combinations
- ❌ Boundary conditions on numeric inputs
- ✅ One representative validation failure (tested at E2E)
- ✅ Comprehensive validation (tested at unit/integration level)

**Business Logic Permutations**
Testing all combinations of discount rules, pricing scenarios, or business logic variations through E2E tests creates a massive, slow suite. Test business logic thoroughly at the unit level; verify basic integration works E2E.

**Intermediate System States**
E2E tests should verify complete workflows from user perspective, not intermediate states or backend processes not visible to users. Internal system behavior belongs at unit or integration level.

**Implementation Details**
E2E tests should never verify internal implementation details like specific API calls, database queries, or service interactions. Test observable user-facing behavior only.

### E2E Test Design Patterns

**Page Object Pattern**
```
class LoginPage:
    def __init__(self, browser):
        self.browser = browser

    def navigate(self):
        self.browser.get("https://app.example.com/login")

    def enter_credentials(self, email, password):
        self.browser.find_element_by_id("email").send_keys(email)
        self.browser.find_element_by_id("password").send_keys(password)

    def submit(self):
        self.browser.find_element_by_id("login-button").click()

    def is_error_displayed(self):
        return self.browser.find_element_by_class("error").is_displayed()

def test_login_invalid_credentials():
    login_page = LoginPage(browser)
    login_page.navigate()
    login_page.enter_credentials("invalid@example.com", "wrongpassword")
    login_page.submit()

    assert login_page.is_error_displayed()
```

**Explicit Waits (Not Sleeps)**
```
# ❌ BAD: Fixed sleep
click_button()
time.sleep(3)  # Hope page loads
assert_element_visible()

# ✅ GOOD: Explicit wait for condition
click_button()
wait_until(lambda: element_is_visible("success-message"), timeout=10)
assert_element_visible("success-message")
```

**Test Data Management**
```
@pytest.fixture
def test_user():
    # Create unique test user for this test
    user = create_user(
        email=f"test-{uuid4()}@example.com",
        password="TestPassword123!"
    )

    yield user

    # Cleanup
    delete_user(user.id)
```

### Recommended Coverage Targets

**Must Cover (5-10 E2E tests)**
- User registration and login
- Core value proposition workflow (the main thing users do)
- Payment/checkout flow if applicable
- Critical administrative functions
- Account recovery/password reset

**Should Cover (10-20 E2E tests)**
- Secondary feature workflows
- Important edge cases visible to users
- Cross-browser compatibility for critical paths
- Mobile responsiveness for key workflows

**Don't Cover with E2E**
- Every variation and edge case
- Internal backend processes
- API-only functionality (use integration tests)
- Features with low business impact

---

## Contract Tests: Service Boundary Validation

Contract testing, particularly relevant in microservices architectures, verifies the agreements between service consumers and providers. Rather than testing complete integration, contract tests validate that APIs meet expectations defined in consumer-driven contracts.

### Characteristics

- **Speed**: Fast (similar to unit tests)
- **Isolation**: Tests consumer and provider independently
- **Contract Focus**: Verifies API structure and behavior expectations
- **Consumer-Driven**: Contracts defined by consumer needs
- **Independent Deployment**: Enables service deployment without full integration

### When to Use Contract Tests

**Microservices Architectures**
Contract tests excel when services are developed and deployed independently:

- Multiple teams own different services
- Services evolve at different rates
- Coordination overhead for integration testing is high
- Fast feedback about breaking changes is crucial

**Examples**:
- User service publishes contract defining how it provides user data
- Order service consumes this contract to verify it can fetch user information
- When user service changes its API, contract tests immediately fail if breaking
- Teams can verify compatibility without coordinating deployment timing

**Third-Party API Integration**
Contract tests verify your code correctly consumes external APIs:

- Define expectations about external API structure and behavior
- Verify your code handles expected responses
- Detect when external API changes break your integration
- Test against contract without calling actual external service

**Examples**:
- Define contract for payment gateway API (charge endpoint, response format)
- Test that your payment service correctly calls payment gateway
- Verify handling of success, failure, and error responses
- Run tests without calling real payment gateway

**Multi-Team Service Development**
Contract tests enable teams to work independently:

- Provider team defines what they offer
- Consumer team defines what they need
- Contract tests verify compatibility
- Teams deploy independently with confidence

### When NOT to Use Contract Tests

**Monolithic Applications**
Contract tests are unnecessary when all components deploy together. Internal modules within a monolith can be tested with integration tests that verify actual runtime behavior.

**Single Team Codebases**
When one team owns both consumer and provider, traditional integration tests may be simpler and more valuable than contract testing. The coordination benefits of contract testing don't apply.

**Business Logic Verification**
Contract tests validate structure and format, not business logic or data correctness. They ensure the API shape matches expectations but don't verify that calculations are correct or workflows complete properly.

**UI Behavior Testing**
Contract tests focus on API contracts, not user interface behavior. Use integration or E2E tests for UI validation.

### Contract Test Design Patterns

**Consumer-Driven Contract Testing (Pact)**
```
# Consumer defines expectations
def test_get_user_contract(pact):
    expected = {
        'id': pact.like(1),
        'email': pact.like('user@example.com'),
        'name': pact.like('Test User'),
        'created_at': pact.like('2024-01-01T00:00:00Z')
    }

    (pact
        .given('User 123 exists')
        .upon_receiving('A request for user 123')
        .with_request('GET', '/api/users/123')
        .will_respond_with(200, body=expected))

    with pact:
        user_service = UserService('http://localhost')
        user = user_service.get_user(123)

        assert user.email == 'user@example.com'
        assert user.name == 'Test User'

# Provider verifies contract
def test_provider_honors_contract():
    verifier = Verifier(provider='UserService', pact_url='./pacts')
    verifier.verify_pacts()
```

**Schema-Based Contract Testing**
```
# Define API schema
user_schema = {
    "type": "object",
    "properties": {
        "id": {"type": "integer"},
        "email": {"type": "string", "format": "email"},
        "name": {"type": "string"},
        "created_at": {"type": "string", "format": "date-time"}
    },
    "required": ["id", "email", "name"]
}

# Consumer validates response against schema
def test_get_user_response_format():
    response = requests.get('http://localhost/api/users/123')
    validate(response.json(), user_schema)

# Provider validates their implementation produces valid schema
def test_user_endpoint_matches_schema():
    response = client.get('/api/users/123')
    validate(response.json, user_schema)
```

### Recommended Coverage

**Essential Contracts**
- All public APIs consumed by other services
- Critical third-party integrations
- Service boundaries between teams

**Contract Scope**
- Request/response structure and types
- Required vs. optional fields
- Status codes and error responses
- Key business rules at the boundary

**Don't Contract Test**
- Internal implementation details
- Database schemas (use integration tests)
- UI components (use integration/E2E tests)
- Complete end-to-end workflows (use E2E tests)

---

## Summary: Choosing the Right Test Level

Use this decision tree to quickly determine appropriate test level:

1. **Is it business logic, calculations, or algorithms?** → Unit test
2. **Does it involve databases, APIs, or framework integration?** → Integration test
3. **Is it a critical user-facing workflow?** → E2E test (minimal coverage)
4. **Is it a service boundary in microservices architecture?** → Contract test
5. **Are edge cases or error conditions being tested?** → Unit test
6. **Does it require the full system running?** → E2E test (only if critical)

**Remember**: The testing pyramid (70-80% unit, 15-25% integration, 5-10% E2E) guides proportional investment, not rigid rules. Adapt based on your architecture, risks, and context.
