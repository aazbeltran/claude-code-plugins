# Testing Legacy Code: Strategies for the Untestable

Michael Feathers defines legacy code as "code without tests." Working with such code requires different strategies than greenfield development because you cannot safely change untested code, yet you cannot write tests without changing the code—a classic catch-22.

## The Legacy Code Dilemma

**The Problem**:
1. Want to refactor code to improve it
2. Can't refactor safely without tests
3. Can't write tests because code isn't testable
4. Need to change code to make it testable (back to step 1)

**The Solution**: Break the cycle using characterization tests, seam identification, and careful dependency breaking.

---

## Characterization Testing

Characterization tests document what code actually does right now, not what it should do. They serve as a safety net during refactoring.

### The Characterization Testing Process

**Step 1: Identify the Code to Characterize**
Start with the area you need to change, not the entire system.

```python
# Legacy code you need to modify
def process_order(order_data, customer_id):
    # 200 lines of complex, untested logic
    # involving database calls, external APIs,
    # and business rules you don't understand
    pass
```

**Step 2: Write a Test with Unknown Expected Output**
```python
def test_process_order_behavior():
    order_data = {
        'items': [{'id': 1, 'quantity': 2}],
        'total': 100
    }
    customer_id = 123

    # Don't know what this returns yet
    result = process_order(order_data, customer_id)

    # Write assertion with placeholder
    assert result == ???
```

**Step 3: Run Test and Observe Actual Output**
```
AssertionError: assert {'status': 'processed', 'order_id': 456, 'timestamp': '2024-01-15'} == ???
```

**Step 4: Update Test to Expect Actual Output**
```python
def test_process_order_behavior():
    order_data = {
        'items': [{'id': 1, 'quantity': 2}],
        'total': 100
    }
    customer_id = 123

    result = process_order(order_data, customer_id)

    # Now documents actual behavior
    assert result['status'] == 'processed'
    assert result['order_id'] == 456
    assert 'timestamp' in result
```

**Step 5: Add More Test Cases**
Characterize different input scenarios:

```python
def test_process_order_with_discount():
    order_data = {'items': [{'id': 1, 'quantity': 2}], 'total': 100, 'discount': 10}
    result = process_order(order_data, 123)
    # Document this path

def test_process_order_with_invalid_customer():
    order_data = {'items': [{'id': 1, 'quantity': 2}], 'total': 100}
    result = process_order(order_data, -1)
    # Document error behavior

def test_process_order_with_empty_items():
    order_data = {'items': [], 'total': 0}
    result = process_order(order_data, 123)
    # Document edge case
```

### Value of Characterization Tests

**What They Do**:
- Document current behavior as executable specification
- Catch regressions during refactoring
- Build confidence before making changes
- Reveal unexpected behaviors and edge cases

**What They Don't Do**:
- Find existing bugs (they codify them)
- Verify correctness (only that behavior hasn't changed)
- Replace proper behavioral tests (those come later)

### Golden Master Testing

For complex outputs (HTML, reports, data transformations), capture entire output as baseline.

```python
def test_generate_invoice_report(golden_master):
    """Characterize complex report generation."""
    order = create_sample_order()

    html_report = generate_invoice_report(order)

    # First run: Captures HTML as baseline
    # Subsequent runs: Compares against baseline
    golden_master.assert_match(html_report, 'invoice_report.html')
```

**When to Use Golden Masters**:
- Complex text/HTML output
- Generated reports or documents
- Data transformations with many fields
- Visual output (images, PDFs)

**Trade-offs**:
- Fast to create comprehensive characterization
- Catches any change to output
- But: Changes to format require updating all golden files
- May hide intentional improvements in approval

---

## Identifying and Exploiting Seams

A **seam** is a place where you can alter behavior without editing code in that place. Seams enable testing by providing points to inject test doubles.

### Types of Seams

**1. Object Seams (Dependency Injection)**
```python
# ❌ BAD: Hard-coded dependency
class OrderService:
    def __init__(self):
        self.db = PostgresDatabase()  # Can't replace for testing

    def create_order(self, data):
        return self.db.save(data)

# ✅ GOOD: Injected dependency (seam)
class OrderService:
    def __init__(self, database):
        self.db = database  # Can inject test double

    def create_order(self, data):
        return self.db.save(data)

# Now testable
def test_order_service():
    fake_db = FakeDatabase()
    service = OrderService(fake_db)

    order = service.create_order(data)

    assert fake_db.saved_items[0] == data
```

**2. Method Seams (Inheritance/Overriding)**
```python
# Legacy code with untestable external call
class PaymentProcessor:
    def process_payment(self, amount):
        result = self.call_payment_gateway(amount)  # External API
        self.send_receipt(result)
        return result

    def call_payment_gateway(self, amount):
        # Actual API call
        return external_api.charge(amount)

# Create testable subclass
class TestablePaymentProcessor(PaymentProcessor):
    def call_payment_gateway(self, amount):
        # Override with fake implementation
        return {'status': 'success', 'transaction_id': '12345'}

def test_payment_processing():
    processor = TestablePaymentProcessor()

    result = processor.process_payment(100)

    assert result['status'] == 'success'
```

**3. Interface Seams (Extract Interface)**
```python
# Legacy code tightly coupled to implementation
class UserService:
    def __init__(self):
        self.repo = UserRepositoryPostgres()  # Concrete class

# Extract interface
class UserRepository(ABC):
    @abstractmethod
    def save(self, user):
        pass

    @abstractmethod
    def find(self, user_id):
        pass

class UserRepositoryPostgres(UserRepository):
    def save(self, user):
        # PostgreSQL implementation
        pass

    def find(self, user_id):
        # PostgreSQL implementation
        pass

class UserRepositoryInMemory(UserRepository):
    def __init__(self):
        self.users = {}

    def save(self, user):
        self.users[user.id] = user

    def find(self, user_id):
        return self.users.get(user_id)

# Now testable
def test_user_service():
    repo = UserRepositoryInMemory()
    service = UserService(repo)

    service.register(user)

    assert repo.find(user.id) == user
```

**4. Parameter Seams (Introduce Parameter)**
```python
# ❌ BAD: Creates dependency internally
def send_notification(user, message):
    email_service = EmailService()  # Hard-coded
    email_service.send(user.email, message)

# ✅ GOOD: Dependency as parameter
def send_notification(user, message, email_service=None):
    if email_service is None:
        email_service = EmailService()  # Default for production

    email_service.send(user.email, message)

# Now testable
def test_send_notification():
    mock_email = Mock()

    send_notification(user, "Test message", mock_email)

    mock_email.send.assert_called_with(user.email, "Test message")
```

---

## Breaking Dependencies Safely

### The Sprout Technique

When adding new functionality to untested code, write new testable code separately, then call it from legacy code.

```python
# Legacy untested code
def process_invoice(invoice):
    # 300 lines of complex logic
    # Need to add discount calculation
    pass

# Sprout: Write new code with tests
def calculate_discount(invoice):
    """New, tested discount logic."""
    if invoice.customer.is_vip:
        return invoice.total * 0.10
    return 0

def test_calculate_discount_for_vip():
    customer = Customer(is_vip=True)
    invoice = Invoice(total=100, customer=customer)

    discount = calculate_discount(invoice)

    assert discount == 10

# Now call from legacy code
def process_invoice(invoice):
    # ... existing logic ...
    discount = calculate_discount(invoice)  # Tested code
    # ... rest of logic ...
```

### The Wrap Technique

When changing behavior, wrap old code in new testable interface.

```python
# Legacy untested code
def legacy_report_generator(data):
    # 500 lines of report generation
    # Need to add caching
    pass

# Wrap with tested code
class CachedReportGenerator:
    def __init__(self, cache):
        self.cache = cache

    def generate(self, data):
        cache_key = self._compute_cache_key(data)

        if cache_key in self.cache:
            return self.cache[cache_key]

        report = legacy_report_generator(data)  # Use legacy code
        self.cache[cache_key] = report
        return report

    def _compute_cache_key(self, data):
        return f"report_{data['id']}"

def test_cached_report_generator():
    cache = {}
    generator = CachedReportGenerator(cache)

    # First call generates report
    report1 = generator.generate({'id': 1, 'data': 'test'})

    # Second call uses cache
    report2 = generator.generate({'id': 1, 'data': 'test'})

    assert report1 == report2
    assert len(cache) == 1
```

### Extract Method Refactoring

Break large untested methods into smaller, testable pieces.

```python
# ❌ BAD: Monolithic untested method
def process_order(order_data):
    # Validate
    if not order_data.get('customer_id'):
        raise ValueError("Missing customer")
    if not order_data.get('items'):
        raise ValueError("No items")

    # Calculate total
    total = 0
    for item in order_data['items']:
        total += item['price'] * item['quantity']

    # Apply discount
    if order_data.get('promo_code') == 'VIP':
        total *= 0.9

    # Save to database
    db.execute("INSERT INTO orders ...", total, ...)

    # Send email
    email.send(order_data['customer_email'], "Order confirmed")

    return {'total': total}

# ✅ GOOD: Extracted testable methods
def validate_order(order_data):
    """Extracted, testable validation."""
    if not order_data.get('customer_id'):
        raise ValueError("Missing customer")
    if not order_data.get('items'):
        raise ValueError("No items")

def calculate_total(items):
    """Extracted, testable calculation."""
    return sum(item['price'] * item['quantity'] for item in items)

def apply_discount(total, promo_code):
    """Extracted, testable discount logic."""
    if promo_code == 'VIP':
        return total * 0.9
    return total

def process_order(order_data):
    validate_order(order_data)
    total = calculate_total(order_data['items'])
    total = apply_discount(total, order_data.get('promo_code'))

    db.execute("INSERT INTO orders ...", total, ...)
    email.send(order_data['customer_email'], "Order confirmed")

    return {'total': total}

# Now individual pieces can be tested
def test_calculate_total():
    items = [{'price': 10, 'quantity': 2}, {'price': 5, 'quantity': 3}]
    assert calculate_total(items) == 35

def test_apply_vip_discount():
    assert apply_discount(100, 'VIP') == 90

def test_apply_no_discount():
    assert apply_discount(100, None) == 100
```

---

## Prioritization Strategies

You cannot test everything in legacy code. Prioritize strategically.

### Test What You Touch

**Principle**: Add tests only for code you're actively changing.

```python
# Working in this area? Add tests
def update_inventory(product_id, quantity):
    # Adding tests here because you're modifying this

# Not working here? Don't add tests yet
def generate_monthly_report():
    # Leave alone for now
```

**Workflow**:
1. Need to modify function X
2. Write characterization tests for X
3. Make changes with test safety net
4. Refactor and add proper behavioral tests
5. Move on (don't test unrelated code)

### Risk-Based Prioritization

**High Priority** (test first):
- Business-critical code (payment, auth, core features)
- Frequently changing code
- Historically bug-prone areas
- Security-sensitive operations

**Medium Priority** (test when touched):
- Important features used regularly
- Moderately complex business logic
- Customer-facing functionality

**Low Priority** (may never need tests):
- Stable code that rarely changes
- Simple pass-through or configuration code
- Deprecated features being phased out
- One-time scripts or utilities

### Change Frequency Analysis

```python
# Analyze git history to find frequently changed files
git log --format=format: --name-only | \
    grep '\.py$' | \
    sort | \
    uniq -c | \
    sort -rn | \
    head -20

# Output:
#   47 src/payment/processor.py     ← High priority: changes often
#   42 src/user/service.py
#   38 src/order/calculator.py
#    3 src/legacy/old_report.py     ← Low priority: stable
#    1 src/config/settings.py
```

### Bug Density Analysis

Track where bugs occur to prioritize testing effort:

```python
# Analyze bug tracker data
SELECT file_path, COUNT(*) as bug_count
FROM bugs
WHERE created_at > DATE_SUB(NOW(), INTERVAL 6 MONTH)
GROUP BY file_path
ORDER BY bug_count DESC
LIMIT 20;

# Focus testing on files with high bug counts
```

---

## Gradual Architectural Improvement

Don't attempt big-bang rewrites. Incrementally improve architecture while maintaining functionality.

### The Strangler Fig Pattern

Gradually replace legacy code by routing new functionality to new implementation while old code continues working.

```python
# Phase 1: Legacy code handles everything
def process_order(order):
    return legacy_order_processor(order)

# Phase 2: Router directs to new or old code
def process_order(order):
    if order.use_new_system:
        return new_order_processor(order)  # New, tested code
    return legacy_order_processor(order)    # Legacy code

# Phase 3: Gradually expand new system coverage
def process_order(order):
    if order.use_new_system or order.created_after('2024-01-01'):
        return new_order_processor(order)
    return legacy_order_processor(order)

# Phase 4: Eventually, all orders use new system
def process_order(order):
    return new_order_processor(order)
    # Legacy code can be removed
```

### Extract Core Domain

Identify and extract business logic from infrastructure concerns.

```python
# ❌ BAD: Business logic mixed with infrastructure
def process_payment(amount, card_number):
    # Business logic
    if amount < 0:
        raise ValueError("Amount must be positive")

    # Infrastructure
    db_connection = psycopg2.connect("postgresql://...")
    cursor = db_connection.cursor()

    # More business logic
    fee = amount * 0.03

    # More infrastructure
    cursor.execute("INSERT INTO payments ...", amount, fee)

    # Business logic
    final_amount = amount + fee

    return final_amount

# ✅ GOOD: Extracted business logic
class PaymentCalculator:
    """Pure business logic - easily testable."""

    FEE_RATE = 0.03

    def validate_amount(self, amount):
        if amount < 0:
            raise ValueError("Amount must be positive")

    def calculate_fee(self, amount):
        return amount * self.FEE_RATE

    def calculate_total(self, amount):
        self.validate_amount(amount)
        fee = self.calculate_fee(amount)
        return amount + fee

# Infrastructure layer
class PaymentService:
    def __init__(self, calculator, repository):
        self.calculator = calculator
        self.repository = repository

    def process_payment(self, amount, card_number):
        total = self.calculator.calculate_total(amount)
        self.repository.save_payment(total, card_number)
        return total

# Now business logic is testable without database
def test_payment_calculation():
    calculator = PaymentCalculator()

    total = calculator.calculate_total(100)

    assert total == 103  # 100 + 3% fee
```

### Dependency Injection at System Boundaries

Introduce seams at architectural boundaries first, then work inward.

```
Old Architecture:
├─ Controllers (hard-coded services)
├─ Services (hard-coded repositories)
└─ Repositories (hard-coded database)

Step 1: Inject at Repository level
├─ Controllers (hard-coded services)
├─ Services (hard-coded repositories)
└─ Repositories (injected database) ← Start here

Step 2: Inject at Service level
├─ Controllers (hard-coded services)
├─ Services (injected repositories)
└─ Repositories (injected database)

Step 3: Inject at Controller level
├─ Controllers (injected services)
├─ Services (injected repositories)
└─ Repositories (injected database) ← Fully testable
```

---

## Documentation and Knowledge Capture

Legacy code often lacks documentation. As you learn, document for the team.

### Architecture Decision Records (ADRs)

Document why the system is structured as it is:

```markdown
# ADR 001: Legacy Order Processing System

## Status
Accepted

## Context
The legacy order processing system uses a monolithic stored procedure
that handles validation, calculation, and persistence in one transaction.

## Decision
We will gradually extract business logic into application layer while
maintaining the stored procedure for backwards compatibility.

## Consequences
- Positive: Can test business logic independently
- Positive: Can gradually migrate to new system
- Negative: Temporary duplication during migration
- Negative: Must maintain both paths during transition
```

### Code Comments for Non-Obvious Behavior

```python
def calculate_discount(order):
    # NOTE: VIP discount logic has historical quirk:
    # Orders placed on weekends get additional 5% even for VIPs
    # due to legacy promotion that can't be removed yet.
    # See ticket #1234 for context.

    base_discount = 0.10 if order.customer.is_vip else 0

    if order.placed_on_weekend():
        # Don't refactor this away - it's required
        base_discount += 0.05

    return order.total * base_discount
```

### Test Documentation

```python
def test_weekend_vip_discount_edge_case():
    """
    Legacy behavior: VIP customers get extra 5% on weekends.

    This is a historical promotion that some customers depend on.
    Do NOT remove without business approval.

    Context: Implemented in 2018 for holiday promotion, never removed.
    See: docs/promotions/weekend-vip-discount.md
    """
    customer = Customer(is_vip=True)
    order = Order(total=100, customer=customer, date=saturday())

    discount = calculate_discount(order)

    assert discount == 15  # 10% VIP + 5% weekend
```

---

## Summary: Legacy Code Testing Strategy

1. **Start with characterization tests** to document current behavior
2. **Identify seams** where you can introduce test doubles
3. **Break dependencies** using safe, mechanical refactorings
4. **Prioritize by risk and touch**: Test critical code and code you're changing
5. **Use sprout and wrap** to add new functionality testably
6. **Extract methods** to create testable units
7. **Apply strangler fig** for gradual architectural improvement
8. **Document discoveries** for the team

**Remember**: You cannot test all legacy code at once. Be strategic, be incremental, and focus on creating safety nets around the areas you're actively working in. Each small improvement makes the next one easier.
