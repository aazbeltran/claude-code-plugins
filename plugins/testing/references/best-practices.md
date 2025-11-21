# Test Design Best Practices

## Building Stable, Deterministic Tests

The foundation of a reliable test suite is deterministic tests that produce consistent results. Non-deterministic or "flaky" tests erode trust and waste time debugging phantom failures.

### Eliminating Timing Dependencies

**The Problem**: Fixed sleep statements create unreliable tests that either waste time or fail intermittently.

```python
# ❌ BAD: Fixed sleep
def test_async_operation():
    start_async_task()
    time.sleep(5)  # Hope 5 seconds is enough
    assert task_completed()

# ✅ GOOD: Conditional wait
def test_async_operation():
    start_async_task()
    wait_until(lambda: task_completed(), timeout=10, interval=0.1)
    assert task_completed()
```

**Strategies**:
- Use explicit waits with conditions, not arbitrary sleeps
- Implement timeout-based polling for async operations
- Use testing framework wait utilities (e.g., Selenium WebDriverWait)
- Mock time-dependent code to make tests deterministic
- Use fast fake timers in tests instead of actual delays

**Example: Polling with Timeout**
```python
def wait_until(condition, timeout=10, interval=0.1):
    """Wait until condition is true or timeout expires."""
    end_time = time.time() + timeout
    while time.time() < end_time:
        if condition():
            return True
        time.sleep(interval)
    raise TimeoutError(f"Condition not met within {timeout} seconds")

def test_message_processing():
    queue.publish(message)

    wait_until(lambda: queue.is_empty(), timeout=5)

    assert message_handler.processed_count == 1
```

### Avoiding Shared State

**The Problem**: Tests that share state create order dependencies and cascading failures.

```python
# ❌ BAD: Shared state
shared_user = None

def test_create_user():
    global shared_user
    shared_user = User.create(email="test@example.com")
    assert shared_user.id is not None

def test_user_can_login():
    # Depends on previous test
    result = login(shared_user.email, "password")
    assert result.success

# ✅ GOOD: Independent tests
@pytest.fixture
def user():
    """Each test gets its own user."""
    user = User.create(email=f"test-{uuid4()}@example.com")
    yield user
    user.delete()

def test_create_user(user):
    assert user.id is not None

def test_user_can_login(user):
    result = login(user.email, "password")
    assert result.success
```

**Strategies**:
- Use test fixtures that create fresh instances
- Employ database transactions that rollback after each test
- Use in-memory databases that reset between tests
- Avoid global variables and singletons in tests
- Make each test completely self-contained

**Database Isolation Pattern**
```python
@pytest.fixture
def db_transaction():
    """Each test runs in a transaction that rolls back."""
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)

    yield session

    session.close()
    transaction.rollback()
    connection.close()

def test_user_creation(db_transaction):
    user = User(email="test@example.com")
    db_transaction.add(user)
    db_transaction.commit()

    assert db_transaction.query(User).count() == 1
    # Transaction rolls back after test - no cleanup needed
```

### Handling External Dependencies

**The Problem**: Tests depending on external systems (APIs, databases, file systems) become non-deterministic and slow.

**Strategies**:
- **Mock external HTTP APIs**: Use libraries like `responses`, `httpretty`, or `requests-mock`
- **Use in-memory databases**: SQLite in-memory for SQL databases
- **Fake file systems**: Use `pyfakefs` or similar for file operations
- **Container-based dependencies**: Use Testcontainers for isolated, real dependencies
- **Test doubles**: Stubs, mocks, and fakes for external services

**Mocking HTTP Calls**
```python
import responses

@responses.activate
def test_fetch_user_data():
    responses.add(
        responses.GET,
        'https://api.example.com/users/123',
        json={'id': 123, 'name': 'Test User'},
        status=200
    )

    user_data = api_client.get_user(123)

    assert user_data['name'] == 'Test User'
```

### Controlling Randomness and Time

**Random Data**: Use fixed seeds for reproducible randomness
```python
# ❌ BAD: Non-deterministic
def test_shuffle():
    items = [1, 2, 3, 4, 5]
    random.shuffle(items)  # Different result each run
    assert items[0] < items[-1]

# ✅ GOOD: Deterministic
def test_shuffle():
    random.seed(42)
    items = [1, 2, 3, 4, 5]
    random.shuffle(items)
    assert items == [2, 4, 5, 3, 1]  # Always same order
```

**Time-Dependent Code**: Mock current time
```python
from datetime import datetime
from unittest.mock import patch

# ❌ BAD: Depends on actual current time
def test_is_expired():
    expiry = datetime(2024, 1, 1)
    assert is_expired(expiry)  # Fails after 2024-01-01

# ✅ GOOD: Freezes time
@patch('mymodule.datetime')
def test_is_expired(mock_datetime):
    mock_datetime.now.return_value = datetime(2024, 6, 1)

    expiry = datetime(2024, 1, 1)

    assert is_expired(expiry)
```

---

## Boundary Analysis and Equivalence Partitioning

Systematic approaches to test input selection ensure comprehensive coverage without exhaustive testing.

### Boundary Value Analysis

**Principle**: Bugs often occur at the edges of input ranges. Test boundaries explicitly.

**For a function accepting ages 0-120:**
- Test: -1 (below minimum)
- Test: 0 (minimum boundary)
- Test: 1 (just above minimum)
- Test: 119 (just below maximum)
- Test: 120 (maximum boundary)
- Test: 121 (above maximum)

**Example Implementation**
```python
def validate_age(age):
    if age < 0 or age > 120:
        raise ValueError("Age must be between 0 and 120")
    return True

@pytest.mark.parametrize("age,should_pass", [
    (-1, False),   # Below minimum
    (0, True),     # Minimum boundary
    (1, True),     # Just above minimum
    (60, True),    # Typical value
    (119, True),   # Just below maximum
    (120, True),   # Maximum boundary
    (121, False),  # Above maximum
])
def test_age_validation(age, should_pass):
    if should_pass:
        assert validate_age(age) == True
    else:
        with pytest.raises(ValueError):
            validate_age(age)
```

### Equivalence Partitioning

**Principle**: Divide input space into classes that should behave identically. Test one representative from each class.

**For a discount function based on purchase amount:**
- Partition 1: $0-100 (no discount)
- Partition 2: $101-500 (10% discount)
- Partition 3: $501+ (20% discount)

Test representatives: $50, $300, $600

**Example Implementation**
```python
def calculate_discount(amount):
    if amount <= 100:
        return 0
    elif amount <= 500:
        return amount * 0.10
    else:
        return amount * 0.20

@pytest.mark.parametrize("amount,expected_discount", [
    (50, 0),         # Partition 1: 0-100
    (100, 0),        # Boundary
    (101, 10.10),    # Just into partition 2
    (300, 30),       # Partition 2: 101-500
    (500, 50),       # Boundary
    (501, 100.20),   # Just into partition 3
    (1000, 200),     # Partition 3: 501+
])
def test_discount_calculation(amount, expected_discount):
    assert calculate_discount(amount) == pytest.approx(expected_discount)
```

### Property-Based Testing

**Principle**: Define invariants that should always hold, then generate hundreds of test cases automatically.

**Example: Reversing a List**
```python
from hypothesis import given
from hypothesis.strategies import lists, integers

@given(lists(integers()))
def test_reverse_twice_returns_original(items):
    """Reversing a list twice should return the original."""
    reversed_once = list(reversed(items))
    reversed_twice = list(reversed(reversed_once))
    assert reversed_twice == items

@given(lists(integers()))
def test_sorted_list_has_same_elements(items):
    """Sorted list should have same elements as original."""
    sorted_items = sorted(items)
    assert sorted(sorted_items) == sorted(items)
    assert len(sorted_items) == len(items)
```

**Benefits**:
- Finds edge cases you didn't think to test
- Tests many more scenarios than manual tests
- Shrinks failing examples to minimal reproducible cases
- Documents properties/invariants of the code

---

## Writing Expressive, Maintainable Tests

### Arrange-Act-Assert (AAA) Pattern

**Structure**: Organize tests into three clear sections for maximum readability.

```python
def test_user_registration():
    # Arrange: Set up test data and dependencies
    email = "newuser@example.com"
    password = "SecurePass123!"
    user_service = UserService(database, email_service)

    # Act: Execute the behavior being tested
    result = user_service.register(email, password)

    # Assert: Verify expected outcomes
    assert result.success == True
    assert result.user.email == email
    assert result.user.is_verified == False
```

**Benefits**:
- Immediately clear what the test does
- Easy to identify what's being tested
- Simple to understand failure location
- Consistent structure across test suite

**Variations**:
- Use blank lines to separate sections
- Use comments to label sections
- BDD Given-When-Then is equivalent structure

### Descriptive Test Names

**Principle**: Test names should describe scenario and expected outcome without reading test code.

**Naming Patterns**:
```python
# Pattern 1: test_[function]_[scenario]_[expected_result]
def test_calculate_discount_for_vip_customer_returns_ten_percent():
    pass

# Pattern 2: test_[scenario]_[expected_result]
def test_invalid_email_raises_validation_error():
    pass

# Pattern 3: Natural language with underscores
def test_user_cannot_login_with_expired_password():
    pass

# Pattern 4: BDD-style
def test_given_vip_customer_when_ordering_then_receives_discount():
    pass
```

**Good Test Names**:
- ✅ `test_empty_cart_checkout_raises_error`
- ✅ `test_duplicate_email_registration_rejected`
- ✅ `test_expired_token_authentication_fails`
- ✅ `test_concurrent_order_updates_handled_correctly`

**Bad Test Names**:
- ❌ `test_1`, `test_2`, `test_3` (meaningless)
- ❌ `test_order` (what about orders?)
- ❌ `test_the_thing_works` (what thing? how?)
- ❌ `test_bug_fix` (what bug?)

### Clear, Actionable Assertions

**Principle**: Assertion failures should immediately communicate what went wrong.

```python
# ❌ BAD: Unclear failure message
def test_discount_calculation():
    assert calculate_discount(100) == 10

# Output: AssertionError: assert 0 == 10
# (Why did we expect 10? What was the input context?)

# ✅ GOOD: Clear failure message
def test_discount_calculation():
    customer = Customer(type="VIP")
    order_total = 100

    discount = calculate_discount(customer, order_total)

    assert discount == 10, \
        f"VIP customer with ${order_total} order should get $10 discount, got ${discount}"

# Output: AssertionError: VIP customer with $100 order should get $10 discount, got $0
```

**Custom Assertion Messages**:
```python
# Complex conditions benefit from explanatory messages
assert user.is_active, \
    f"User {user.email} should be active after verification, but is_active={user.is_active}"

assert len(results) > 0, \
    f"Search for '{search_term}' should return results, got empty list"

assert response.status_code == 200, \
    f"GET /api/users/{user_id} should return 200, got {response.status_code}: {response.text}"
```

### Keep Tests Focused and Small

**Principle**: Each test should verify one behavior or logical assertion.

```python
# ❌ BAD: Testing too much
def test_user_lifecycle():
    # Create user
    user = User.create(email="test@example.com")
    assert user.id is not None

    # Activate user
    user.activate()
    assert user.is_active

    # Update profile
    user.update_profile(name="Test User")
    assert user.name == "Test User"

    # Deactivate user
    user.deactivate()
    assert not user.is_active

    # Delete user
    user.delete()
    assert User.find(user.id) is None

# ✅ GOOD: Focused tests
def test_create_user_assigns_id():
    user = User.create(email="test@example.com")
    assert user.id is not None

def test_activate_user_sets_active_flag():
    user = User.create(email="test@example.com")

    user.activate()

    assert user.is_active == True

def test_update_profile_changes_name():
    user = User.create(email="test@example.com")

    user.update_profile(name="Test User")

    assert user.name == "Test User"
```

**When Multiple Assertions Are OK**:
- Verifying different properties of the same object
- Checking related side effects of single action
- Asserting preconditions and postconditions

```python
# ✅ GOOD: Multiple related assertions
def test_order_creation():
    items = [Item("Widget", 10), Item("Gadget", 20)]

    order = Order.create(items)

    assert order.id is not None
    assert order.total == 30
    assert len(order.items) == 2
    assert order.status == "pending"
```

---

## Test Data Management

### Fixtures: Providing Test Dependencies

**Fixtures** provide reusable test dependencies and setup/teardown logic.

**Pytest Fixtures**
```python
@pytest.fixture
def database():
    """Provide a test database."""
    db = create_test_database()
    yield db
    db.close()

@pytest.fixture
def user(database):
    """Provide a test user."""
    user = User.create(email="test@example.com")
    yield user
    user.delete()

def test_user_can_place_order(user, database):
    order = Order.create(user=user, items=[Item("Widget", 10)])
    assert order.user_id == user.id
```

**Fixture Scopes**:
- `function`: New instance per test (default)
- `class`: Shared across test class
- `module`: Shared across test file
- `session`: Shared across entire test run

```python
@pytest.fixture(scope="session")
def app():
    """Start application once for entire test session."""
    app = create_app()
    app.start()
    yield app
    app.stop()
```

### Factories: Flexible Test Data Creation

**Factories** create test objects with sensible defaults that can be customized.

```python
# Factory pattern
class UserFactory:
    _counter = 0

    @classmethod
    def create(cls, **kwargs):
        cls._counter += 1
        defaults = {
            'email': f'user{cls._counter}@example.com',
            'name': f'Test User {cls._counter}',
            'role': 'user',
            'is_active': True
        }
        defaults.update(kwargs)
        return User(**defaults)

# Usage
def test_admin_user():
    admin = UserFactory.create(role='admin', name='Admin User')
    assert admin.role == 'admin'

def test_inactive_user():
    inactive = UserFactory.create(is_active=False)
    assert not inactive.is_active
```

**FactoryBoy Library**
```python
import factory

class UserFactory(factory.Factory):
    class Meta:
        model = User

    email = factory.Sequence(lambda n: f'user{n}@example.com')
    name = factory.Faker('name')
    role = 'user'
    is_active = True

# Usage
def test_user_creation():
    user = UserFactory()
    assert user.email.endswith('@example.com')

def test_admin_user():
    admin = UserFactory(role='admin')
    assert admin.role == 'admin'
```

### Builders: Constructing Complex Objects

**Builders** provide fluent APIs for creating complex test objects.

```python
class OrderBuilder:
    def __init__(self):
        self._user = None
        self._items = []
        self._status = 'pending'
        self._shipping_address = None

    def with_user(self, user):
        self._user = user
        return self

    def with_item(self, name, price, quantity=1):
        self._items.append(Item(name, price, quantity))
        return self

    def with_status(self, status):
        self._status = status
        return self

    def with_shipping(self, address):
        self._shipping_address = address
        return self

    def build(self):
        return Order(
            user=self._user,
            items=self._items,
            status=self._status,
            shipping_address=self._shipping_address
        )

# Usage
def test_order_total_calculation():
    order = (OrderBuilder()
        .with_item("Widget", 10, quantity=2)
        .with_item("Gadget", 20)
        .build())

    assert order.total == 40
```

### Snapshot/Golden Master Testing

**Principle**: Capture complex output as baseline, verify future runs match.

**Use Cases**:
- Testing rendered HTML or UI components
- Verifying generated reports or documents
- Validating complex JSON/XML output
- Testing data transformations with many fields

```python
def test_user_profile_rendering(snapshot):
    user = User(name="John Doe", email="john@example.com", age=30)

    html = render_user_profile(user)

    snapshot.assert_match(html, 'user_profile.html')

# First run: Captures output as baseline
# Subsequent runs: Compares against baseline
# To update baseline: pytest --snapshot-update
```

**Trade-offs**:
- **Pro**: Easy to test complex outputs
- **Pro**: Catches unintended changes
- **Con**: Changes to output format require updating all snapshots
- **Con**: Can hide real regressions if snapshots updated without review

---

## Mocking, Stubbing, and Fakes

### Understanding the Distinctions

**Stub**: Provides canned responses to method calls
```python
class StubEmailService:
    def send_email(self, to, subject, body):
        return True  # Always succeeds, doesn't actually send
```

**Mock**: Records interactions and verifies they occurred as expected
```python
from unittest.mock import Mock

email_service = Mock()
user_service.register(user)

email_service.send_welcome_email.assert_called_once_with(user.email)
```

**Fake**: Simplified working implementation
```python
class FakeDatabase:
    def __init__(self):
        self._data = {}

    def save(self, id, value):
        self._data[id] = value

    def find(self, id):
        return self._data.get(id)
```

### When to Use Each Type

**Use Stubs** for query dependencies that provide data
```python
def test_calculate_user_discount():
    stub_repository = StubUserRepository()
    stub_repository.set_return_value(User(type="VIP"))

    discount_service = DiscountService(stub_repository)

    discount = discount_service.calculate_for_user(user_id=123)

    assert discount == 10
```

**Use Mocks** for command dependencies where interaction matters
```python
def test_order_completion_sends_confirmation():
    mock_email_service = Mock()
    order_service = OrderService(mock_email_service)

    order_service.complete_order(order_id=123)

    mock_email_service.send_confirmation.assert_called_once()
```

**Use Fakes** for complex dependencies needing realistic behavior
```python
def test_user_repository_operations():
    fake_db = FakeDatabase()
    repository = UserRepository(fake_db)

    user = User(email="test@example.com")
    repository.save(user)

    found = repository.find_by_email("test@example.com")
    assert found.email == user.email
```

### Avoiding Over-Mocking

**The Problem**: Mocking every dependency couples tests to implementation.

```python
# ❌ BAD: Over-mocking
def test_order_creation():
    mock_validator = Mock()
    mock_repository = Mock()
    mock_calculator = Mock()
    mock_logger = Mock()

    service = OrderService(mock_validator, mock_repository, mock_calculator, mock_logger)

    service.create_order(data)

    mock_validator.validate.assert_called_once()
    mock_calculator.calculate_total.assert_called_once()
    mock_repository.save.assert_called_once()
    mock_logger.log.assert_called()

# ✅ GOOD: Mock only external boundaries
def test_order_creation():
    repository = InMemoryOrderRepository()
    service = OrderService(repository)

    order = service.create_order(data)

    assert order.total == 100
    assert repository.find(order.id) is not None
```

**Guidelines**:
- Mock external systems (databases, APIs, file systems)
- Use real implementations for internal collaborators
- Don't verify every method call
- Focus on observable behavior, not interactions

---

## Summary: Test Design Principles

1. **Deterministic tests**: Eliminate timing issues, shared state, and external dependencies
2. **Boundary testing**: Test edges systematically where bugs hide
3. **Expressive tests**: Clear names, AAA structure, actionable failures
4. **Focused tests**: One behavior per test, small and understandable
5. **Flexible test data**: Use factories and builders for maintainable test data
6. **Strategic mocking**: Mock at system boundaries, use real implementations internally
7. **Fast execution**: Keep unit tests fast for rapid feedback loops

These practices work together to create test suites that catch bugs, enable refactoring, and remain maintainable over time.
