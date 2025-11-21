# Testing Anti-Patterns: Common Mistakes to Avoid

## Testing Implementation Details

**The Anti-Pattern**: Tests verify internal mechanics rather than observable behavior, causing tests to break during safe refactoring.

### Examples of Testing Implementation Details

**Testing Private Methods Directly**
```python
# ❌ BAD: Testing private method
class OrderService:
    def create_order(self, items):
        self._validate_items(items)
        return self._build_order(items)

    def _validate_items(self, items):
        # Internal implementation detail
        pass

    def _build_order(self, items):
        # Internal implementation detail
        pass

def test_validate_items_rejects_empty():
    service = OrderService()
    # Accessing private method
    with pytest.raises(ValidationError):
        service._validate_items([])
```

**Testing Call Sequences**
```python
# ❌ BAD: Verifying internal method calls
def test_order_creation_process():
    mock_validator = Mock()
    mock_builder = Mock()
    service = OrderService(mock_validator, mock_builder)

    service.create_order(items)

    # Testing HOW it works internally
    mock_validator.validate.assert_called_once_with(items)
    mock_builder.build.assert_called_once()
    assert mock_validator.validate.call_count == 1
```

**Testing Internal State**
```python
# ❌ BAD: Accessing private fields
def test_user_activation():
    user = User()

    user.activate()

    assert user._activated_at is not None  # Private field
    assert user._activation_status == "active"  # Private field
```

### How to Fix It

**Test Through Public Interfaces**
```python
# ✅ GOOD: Testing observable behavior
def test_create_order_with_valid_items():
    service = OrderService()
    items = [Item("Widget", 10)]

    order = service.create_order(items)

    assert order.id is not None
    assert order.total == 10
    assert len(order.items) == 1

def test_create_order_with_empty_items_raises_error():
    service = OrderService()

    with pytest.raises(ValidationError):
        service.create_order([])
```

**Test Outcomes, Not Process**
```python
# ✅ GOOD: Verifying results
def test_order_completion():
    order_service = OrderService(email_service)

    completed_order = order_service.complete_order(order_id=123)

    assert completed_order.status == "completed"
    assert completed_order.completed_at is not None
    # Email sending is internal detail - verify user-visible outcome
```

### Why It Matters

- **Refactoring breaks tests**: Changing internal structure breaks tests even though behavior is unchanged
- **False negatives**: Tests fail when nothing is actually broken
- **Maintenance burden**: Every internal change requires test updates
- **Missed real bugs**: Focus on implementation means less focus on behavior
- **Discourages improvement**: Fear of breaking tests prevents refactoring

---

## Over-Mocking and Excessive Test Doubles

**The Anti-Pattern**: Mocking every dependency regardless of whether it's necessary, creating tests that pass while real code fails.

### Examples of Over-Mocking

**Mocking Everything**
```python
# ❌ BAD: Mocking internal collaborators
def test_user_service():
    mock_validator = Mock()
    mock_repository = Mock()
    mock_email_service = Mock()
    mock_logger = Mock()
    mock_cache = Mock()
    mock_event_bus = Mock()

    service = UserService(
        mock_validator,
        mock_repository,
        mock_email_service,
        mock_logger,
        mock_cache,
        mock_event_bus
    )

    # Test provides no confidence real code works
    service.create_user(data)

    # Verifying every interaction
    mock_validator.validate.assert_called()
    mock_repository.save.assert_called()
    # ... and so on
```

**Mocking What You Own**
```python
# ❌ BAD: Mocking internal business logic
def test_order_total_calculation():
    mock_calculator = Mock()
    mock_calculator.calculate.return_value = 100

    order_service = OrderService(mock_calculator)

    total = order_service.get_order_total(order)

    assert total == 100
    # This test verifies nothing - just that mock returns what we told it to
```

### How to Fix It

**Mock Only at System Boundaries**
```python
# ✅ GOOD: Mock external dependencies, use real internal objects
def test_user_service():
    # Mock external system
    mock_email_service = Mock()

    # Real internal collaborators
    validator = UserValidator()
    repository = InMemoryUserRepository()

    service = UserService(validator, repository, mock_email_service)

    user = service.create_user(valid_user_data)

    assert user.id is not None
    assert repository.find(user.id) == user
```

**Use Real Implementations Where Possible**
```python
# ✅ GOOD: Real implementations for testable components
def test_order_total_calculation():
    calculator = OrderCalculator()  # Real calculator
    order = Order(items=[Item("Widget", 10), Item("Gadget", 20)])

    total = calculator.calculate_total(order)

    assert total == 30
```

### When to Mock

**DO mock**:
- External HTTP APIs
- Databases (sometimes - consider in-memory alternatives)
- File systems
- System time/clocks
- Random number generators
- Third-party services

**DON'T mock**:
- Internal business logic
- Value objects and data structures
- Utilities and helpers you own
- Simple collaborators

### Why It Matters

- **False confidence**: Tests pass but real integrations fail
- **Implementation coupling**: Tests break when changing internal code structure
- **Maintenance nightmare**: Every refactoring breaks mocks
- **Missing integration bugs**: Real interactions between components never tested
- **Complex test setup**: More time setting up mocks than writing actual test

---

## Brittle Tests That Block Refactoring

**The Anti-Pattern**: Tests so tightly coupled to current implementation that refactoring becomes impossible without rewriting tests.

### Examples of Brittle Tests

**UI Tests Coupled to Selectors**
```python
# ❌ BAD: Coupled to specific HTML structure
def test_login():
    browser.find_element_by_xpath(
        '/html/body/div[1]/div[2]/form/div[1]/input'
    ).send_keys('user@example.com')

    browser.find_element_by_xpath(
        '/html/body/div[1]/div[2]/form/div[2]/input'
    ).send_keys('password')

    browser.find_element_by_xpath(
        '/html/body/div[1]/div[2]/form/button[1]'
    ).click()
```

**Tests Dependent on Execution Order**
```python
# ❌ BAD: Tests must run in specific order
def test_1_create_user():
    global user_id
    user_id = create_user("test@example.com")

def test_2_activate_user():
    activate_user(user_id)  # Depends on test_1

def test_3_delete_user():
    delete_user(user_id)  # Depends on test_2
```

**Tests Coupled to Data Format**
```python
# ❌ BAD: Expects exact JSON structure
def test_api_response():
    response = api.get_user(123)

    assert response == {
        'id': 123,
        'name': 'Test User',
        'email': 'test@example.com',
        'created_at': '2024-01-01T00:00:00Z',
        'updated_at': '2024-01-01T00:00:00Z'
    }
    # Breaks if any field added or order changes
```

### How to Fix It

**Use Stable Selectors**
```python
# ✅ GOOD: Use semantic selectors
def test_login():
    browser.find_element_by_label_text('Email').send_keys('user@example.com')
    browser.find_element_by_label_text('Password').send_keys('password')
    browser.find_element_by_role('button', name='Login').click()

# Or use data-testid attributes
def test_login_with_test_ids():
    browser.find_element_by_test_id('email-input').send_keys('user@example.com')
    browser.find_element_by_test_id('password-input').send_keys('password')
    browser.find_element_by_test_id('login-button').click()
```

**Make Tests Independent**
```python
# ✅ GOOD: Each test is self-contained
def test_create_user():
    user_id = create_user("test@example.com")
    assert user_id is not None

def test_activate_user():
    user_id = create_user("test2@example.com")
    result = activate_user(user_id)
    assert result.success

def test_delete_user():
    user_id = create_user("test3@example.com")
    delete_user(user_id)
    assert find_user(user_id) is None
```

**Test Required Fields, Ignore Optional Ones**
```python
# ✅ GOOD: Test essential fields only
def test_api_response():
    response = api.get_user(123)

    assert response['id'] == 123
    assert response['email'] == 'test@example.com'
    assert 'created_at' in response
    # Don't care about other fields or order
```

### Why It Matters

- **Prevents refactoring**: Fear of breaking tests stops code improvement
- **High maintenance cost**: Every small change requires test updates
- **False failures**: Tests fail when nothing is actually broken
- **Slows development**: More time fixing tests than improving code
- **Tests become liability**: Team considers tests obstacle rather than asset

---

## The Inverted Test Pyramid (Ice Cream Cone)

**The Anti-Pattern**: More E2E and integration tests than unit tests, leading to slow, flaky test suites.

### What It Looks Like

```
        /\
       /  \      ← Large E2E test suite (slow, flaky)
      /    \
     /------\    ← Moderate integration tests
    /        \
   /   unit   \  ← Small unit test suite
  /____________\
```

**Symptoms**:
- Test suite takes 30+ minutes to run
- Frequent flaky test failures
- Developers skip running tests locally
- CI pipeline is bottleneck
- Hard to diagnose failures

### How It Happens

**Over-Reliance on E2E Tests**
```python
# ❌ BAD: Testing edge cases through UI
def test_registration_empty_email():
    browser.get('/register')
    # ... fill form with empty email ...
    # 10 seconds to test something unit test could verify in milliseconds

def test_registration_invalid_email():
    browser.get('/register')
    # ... fill form with invalid email ...
    # Another 10 seconds

def test_registration_duplicate_email():
    browser.get('/register')
    # ... fill form with existing email ...
    # Another 10 seconds

# 30 seconds to test what should be 3 fast unit tests
```

**Missing Unit Tests**
```python
# ❌ BAD: No unit tests for business logic
# All testing happens through API or UI
# Business logic never tested in isolation
```

### How to Fix It

**Push Tests Down the Pyramid**
```python
# ✅ GOOD: Unit test edge cases
def test_email_validation_empty():
    with pytest.raises(ValidationError):
        validate_email("")

def test_email_validation_invalid_format():
    with pytest.raises(ValidationError):
        validate_email("notanemail")

def test_email_validation_duplicate():
    with pytest.raises(ValidationError):
        validate_email_uniqueness("existing@example.com")

# Each test: milliseconds

# ✅ GOOD: One E2E test for happy path
def test_successful_registration():
    browser.get('/register')
    fill_registration_form(valid_data)
    submit()
    assert_redirected_to_dashboard()

# One E2E test: 10 seconds
```

**Proper Test Distribution**
```
        /\
       /E2E\     ← Few critical path tests (5-10%)
      /____\
     /      \
    /  Int   \   ← Component boundary tests (15-25%)
   /  Tests   \
  /____________\
 /              \
/   Unit Tests   \ ← Most tests here (70-80%)
/__________________\
```

### Why It Matters

- **Slow feedback**: 30-minute test suites kill productivity
- **Flaky tests**: E2E tests are inherently less stable
- **Unclear failures**: Hard to pinpoint bugs in complex E2E tests
- **Resource intensive**: E2E tests require more infrastructure
- **Maintenance burden**: UI changes break many E2E tests

---

## Assertion Roulette and Unclear Failures

**The Anti-Pattern**: Tests with multiple unrelated assertions that make failures ambiguous.

### Examples

**Too Many Assertions**
```python
# ❌ BAD: Which assertion failed?
def test_user_operations():
    user = create_user("test@example.com")
    assert user.id is not None
    assert user.email == "test@example.com"
    assert user.is_active
    user.update(name="Test User")
    assert user.name == "Test User"
    user.deactivate()
    assert not user.is_active
    user.delete()
    assert find_user(user.id) is None
    # If test fails on line 7, was it deactivate() or the assertion that broke?
```

**No Assertion Messages**
```python
# ❌ BAD: Failure gives no context
def test_discount_calculation():
    result = calculate_discount(customer, order)
    assert result == 10
    # Failure: AssertionError: assert 0 == 10
    # (Why did we expect 10? What were the inputs?)
```

### How to Fix It

**One Logical Assertion Per Test**
```python
# ✅ GOOD: Focused tests
def test_user_creation_assigns_id():
    user = create_user("test@example.com")
    assert user.id is not None

def test_user_creation_sets_email():
    user = create_user("test@example.com")
    assert user.email == "test@example.com"

def test_new_user_is_active_by_default():
    user = create_user("test@example.com")
    assert user.is_active
```

**Add Descriptive Assertion Messages**
```python
# ✅ GOOD: Clear failure messages
def test_vip_discount_calculation():
    customer = Customer(type="VIP")
    order = Order(total=100)

    discount = calculate_discount(customer, order)

    assert discount == 10, \
        f"VIP customer with ${order.total} order should get $10 discount, got ${discount}"
```

### Why It Matters

- **Unclear failures**: Can't tell what broke without debugging
- **Wasted time**: Must debug to understand test failure
- **Cascading failures**: One bug causes multiple assertion failures
- **Hard to maintain**: Complex tests are hard to understand and update

---

## Global State and Hidden Dependencies

**The Anti-Pattern**: Tests that depend on global state, singletons, or hidden dependencies, creating order dependencies and mysterious failures.

### Examples

**Global State**
```python
# ❌ BAD: Tests share global state
current_user = None

def test_login():
    global current_user
    current_user = login("user@example.com", "password")
    assert current_user is not None

def test_access_dashboard():
    # Depends on global current_user from previous test
    response = get_dashboard(current_user)
    assert response.status == 200
```

**Singleton Dependencies**
```python
# ❌ BAD: Singleton holds state between tests
class Database:
    _instance = None
    _data = {}

    @classmethod
    def instance(cls):
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance

def test_save_user():
    db = Database.instance()
    db.save(user)
    # Pollutes singleton state

def test_user_count():
    db = Database.instance()
    # Still has data from previous test
    assert db.count() == 1  # Breaks if other tests ran first
```

### How to Fix It

**Explicit Dependencies**
```python
# ✅ GOOD: Explicit fixtures
@pytest.fixture
def authenticated_user():
    user = User(email="test@example.com")
    session = login(user)
    return session

def test_access_dashboard(authenticated_user):
    response = get_dashboard(authenticated_user)
    assert response.status == 200
```

**Dependency Injection**
```python
# ✅ GOOD: Inject dependencies
@pytest.fixture
def database():
    db = Database()
    yield db
    db.clear()

def test_save_user(database):
    user = User(email="test@example.com")
    database.save(user)
    assert database.count() == 1

def test_find_user(database):
    user = User(email="test@example.com")
    database.save(user)
    found = database.find_by_email("test@example.com")
    assert found == user
```

### Why It Matters

- **Order dependencies**: Tests must run in specific order
- **Flaky failures**: Tests pass individually but fail in suite
- **Parallel execution impossible**: Can't run tests concurrently
- **Hard to debug**: Failures depend on what ran before
- **Misleading results**: Test results inconsistent between runs

---

## High Coverage, Low Value Tests

**The Anti-Pattern**: Achieving high code coverage with tests that don't catch bugs, providing false sense of security.

### Examples

**Tests That Don't Assert**
```python
# ❌ BAD: Executes code but verifies nothing
def test_process_order():
    service = OrderService()
    service.process(order)
    # No assertions - just executes code for coverage
```

**Testing Trivial Code**
```python
# ❌ BAD: Testing getters and setters
def test_user_get_email():
    user = User(email="test@example.com")
    assert user.get_email() == "test@example.com"

def test_user_set_email():
    user = User()
    user.set_email("test@example.com")
    assert user.email == "test@example.com"
```

**Testing Framework Behavior**
```python
# ❌ BAD: Testing that framework works
def test_django_model_save():
    user = User(email="test@example.com")
    user.save()
    # Just testing Django's save() works - not your code
```

### How to Fix It

**Test Behavior, Not Code Execution**
```python
# ✅ GOOD: Verifies actual behavior
def test_order_processing_updates_inventory():
    inventory = Inventory(product_id=1, quantity=10)
    order = Order(product_id=1, quantity=3)

    process_order(order, inventory)

    assert inventory.quantity == 7

def test_order_processing_sends_confirmation():
    mock_email = Mock()
    order = Order(customer_email="test@example.com")

    process_order(order, email_service=mock_email)

    mock_email.send_confirmation.assert_called_with("test@example.com")
```

**Focus on Complex Logic**
```python
# ✅ GOOD: Test non-trivial behavior
def test_discount_calculation_for_bulk_orders():
    items = [Item("Widget", 10, quantity=100)]

    discount = calculate_bulk_discount(items)

    assert discount == 500  # 50% off for orders > 50 units
```

### Why It Matters

- **False confidence**: High coverage numbers hide lack of actual testing
- **Bugs slip through**: Tests don't catch real problems
- **Wasted effort**: Time spent on tests that provide no value
- **Misleading metrics**: Coverage metric becomes meaningless

---

## Summary: Avoiding Anti-Patterns

| Anti-Pattern | Impact | Solution |
|-------------|---------|----------|
| Testing implementation details | Brittle tests, blocks refactoring | Test through public interfaces, verify behavior |
| Over-mocking | False confidence, missed integration bugs | Mock only external boundaries |
| Brittle tests | High maintenance, prevents improvement | Use stable selectors, test essentials |
| Inverted pyramid | Slow, flaky suite | Push tests down: more unit, fewer E2E |
| Assertion roulette | Unclear failures | One logical assertion per test |
| Global state | Order dependencies, flaky tests | Use fixtures, dependency injection |
| Coverage without value | False security | Test behavior, not code execution |

**Core Principle**: Good tests enable confident refactoring and catch real bugs. If tests block improvement or fail without catching bugs, they've become an anti-pattern.
