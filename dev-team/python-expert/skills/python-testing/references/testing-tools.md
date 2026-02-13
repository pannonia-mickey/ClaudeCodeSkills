# Testing Tools

Reference covering pytest plugins, property-based testing with Hypothesis, time freezing, HTTP mocking, test runners, and mutation testing.

---

## pytest Plugins

### pytest-xdist (Parallel Execution)

Run tests in parallel across multiple CPUs:

```bash
# Auto-detect CPU count
pytest -n auto

# Specific worker count
pytest -n 4

# Load balancing modes
pytest -n 4 --dist=loadscope   # Group by module/class
pytest -n 4 --dist=loadfile    # Group by file
pytest -n 4 --dist=loadgroup   # Custom grouping via markers
```

Configuration:

```toml
[tool.pytest.ini_options]
addopts = "-n auto --dist=loadscope"
```

Caveats: ensure tests are truly independent. Shared mutable state (databases, files, module-level globals) causes flaky failures under parallel execution.

### pytest-randomly

Randomize test order to discover hidden dependencies:

```bash
# Install and it activates automatically
pip install pytest-randomly

# Reproduce a specific order using the seed
pytest -p randomly --randomly-seed=12345

# Disable for debugging
pytest -p no:randomly
```

### pytest-timeout

Prevent tests from hanging:

```toml
[tool.pytest.ini_options]
timeout = 30  # Default 30-second timeout for all tests
```

```python
@pytest.mark.timeout(5)
def test_fast_operation() -> None:
    result = quick_calculation()
    assert result is not None

@pytest.mark.timeout(120)
def test_slow_integration() -> None:
    result = full_pipeline()
    assert result.success
```

### pytest-sugar

Enhanced output formatting with progress bars and instant failure display. Install and it activates automatically:

```bash
pip install pytest-sugar
```

### pytest-clarity / pytest-icdiff

Improved diff output for assertion failures:

```bash
pip install pytest-clarity
# or
pip install pytest-icdiff
```

Shows colored, side-by-side diffs for complex data structure comparisons.

### pytest-benchmark

Performance regression testing:

```python
def test_sorting_performance(benchmark) -> None:
    data = list(range(10000, 0, -1))
    result = benchmark(sorted, data)
    assert result == list(range(1, 10001))

def test_serialization_performance(benchmark) -> None:
    obj = create_large_object()
    benchmark.pedantic(serialize, args=(obj,), rounds=100, warmup_rounds=5)
```

```bash
# Compare against saved baseline
pytest --benchmark-autosave
pytest --benchmark-compare=0001
```

### pytest-cov

Coverage integration:

```bash
pytest --cov=src --cov-report=term-missing --cov-report=html --cov-branch
```

```toml
[tool.pytest.ini_options]
addopts = "--cov=src --cov-report=term-missing --cov-branch"
```

---

## Hypothesis (Property-Based Testing)

### Core Strategies

```python
from hypothesis import strategies as st

# Primitives
st.integers(min_value=0, max_value=100)
st.floats(min_value=0.0, max_value=1.0, allow_nan=False)
st.text(min_size=1, max_size=50)
st.binary(min_size=1, max_size=1024)
st.booleans()
st.none()

# Collections
st.lists(st.integers(), min_size=1, max_size=100)
st.sets(st.text(min_size=1), min_size=1)
st.dictionaries(st.text(min_size=1), st.integers())
st.tuples(st.integers(), st.text())
st.frozensets(st.integers())

# Special
st.emails()
st.uuids()
st.datetimes()
st.dates()
st.timedeltas()
st.ip_addresses()

# Combinators
st.one_of(st.integers(), st.text())
st.sampled_from(["red", "green", "blue"])
st.just("constant_value")
```

### Composite Strategies for Domain Models

```python
from hypothesis import strategies as st
from dataclasses import dataclass

@dataclass
class Address:
    street: str
    city: str
    zip_code: str
    country: str

@dataclass
class Customer:
    name: str
    email: str
    address: Address

@st.composite
def addresses(draw: st.DrawFn) -> Address:
    return Address(
        street=draw(st.text(min_size=5, max_size=100)),
        city=draw(st.text(min_size=2, max_size=50, alphabet=st.characters(whitelist_categories=("L", "Zs")))),
        zip_code=draw(st.from_regex(r"[0-9]{5}(-[0-9]{4})?", fullmatch=True)),
        country=draw(st.sampled_from(["US", "CA", "GB", "DE", "FR"])),
    )

@st.composite
def customers(draw: st.DrawFn) -> Customer:
    return Customer(
        name=draw(st.text(min_size=1, max_size=50)),
        email=draw(st.emails()),
        address=draw(addresses()),
    )
```

### Stateful Testing

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, initialize, invariant

class ShoppingCartMachine(RuleBasedStateMachine):
    def __init__(self) -> None:
        super().__init__()
        self.cart = ShoppingCart()
        self.expected_items: dict[str, int] = {}

    @rule(item=st.text(min_size=1), qty=st.integers(min_value=1, max_value=10))
    def add_item(self, item: str, qty: int) -> None:
        self.cart.add(item, qty)
        self.expected_items[item] = self.expected_items.get(item, 0) + qty

    @rule(item=st.text(min_size=1))
    def remove_item(self, item: str) -> None:
        if item in self.expected_items:
            self.cart.remove(item)
            del self.expected_items[item]

    @invariant()
    def items_match(self) -> None:
        for item, qty in self.expected_items.items():
            assert self.cart.quantity(item) == qty

    @invariant()
    def count_matches(self) -> None:
        assert self.cart.unique_items == len(self.expected_items)

TestShoppingCart = ShoppingCartMachine.TestCase
```

### Profiles and Settings

```python
# conftest.py
from hypothesis import settings, Phase, HealthCheck

settings.register_profile(
    "ci",
    max_examples=1000,
    suppress_health_check=[HealthCheck.too_slow],
)
settings.register_profile(
    "dev",
    max_examples=50,
    phases=[Phase.explicit, Phase.generate, Phase.target],
)
settings.register_profile(
    "debug",
    max_examples=10,
    verbosity=hypothesis.Verbosity.verbose,
)
```

```bash
HYPOTHESIS_PROFILE=ci pytest
```

---

## freezegun (Time Freezing)

### Basic Usage

```python
from freezegun import freeze_time

@freeze_time("2024-06-15 12:00:00")
def test_event_scheduling() -> None:
    event = schedule_event(days_ahead=7)
    assert event.date == datetime(2024, 6, 22, 12, 0, 0)

@freeze_time("2024-01-01")
def test_new_year_greeting() -> None:
    assert get_greeting() == "Happy New Year!"
```

### Moving Time

```python
from freezegun import freeze_time

@freeze_time("2024-06-15", tick=True)
def test_with_ticking_clock() -> None:
    start = datetime.now()
    time.sleep(0.1)
    end = datetime.now()
    assert end > start

def test_manual_time_movement() -> None:
    with freeze_time("2024-06-15") as frozen:
        assert datetime.now().day == 15
        frozen.move_to("2024-06-20")
        assert datetime.now().day == 20
```

### Fixture Integration

```python
@pytest.fixture
def at_midnight() -> Generator[None, None, None]:
    with freeze_time("2024-06-15 00:00:00"):
        yield

def test_daily_report(at_midnight: None) -> None:
    report = generate_daily_report()
    assert report.date == date(2024, 6, 15)
```

---

## responses and VCR.py (HTTP Mocking)

### responses Library

```python
import responses

@responses.activate
def test_api_client() -> None:
    responses.add(
        responses.GET,
        "https://api.example.com/users/1",
        json={"id": 1, "name": "Alice"},
        status=200,
    )
    responses.add(
        responses.GET,
        "https://api.example.com/users/999",
        json={"error": "not found"},
        status=404,
    )

    client = APIClient()
    user = client.get_user(1)
    assert user.name == "Alice"

    with pytest.raises(UserNotFoundError):
        client.get_user(999)
```

### Callback-Based Responses

```python
@responses.activate
def test_dynamic_response() -> None:
    def request_callback(request):
        body = json.loads(request.body)
        return (201, {}, json.dumps({"id": "new", **body}))

    responses.add_callback(
        responses.POST,
        "https://api.example.com/users",
        callback=request_callback,
    )

    client = APIClient()
    result = client.create_user(name="Bob")
    assert result["name"] == "Bob"
```

### VCR.py (Record/Replay)

```python
import vcr

@vcr.use_cassette("tests/cassettes/user_fetch.yaml")
def test_real_api_call() -> None:
    """First run records the HTTP interaction, subsequent runs replay it."""
    client = APIClient(base_url="https://api.example.com")
    user = client.get_user(1)
    assert user.name is not None
```

```yaml
# pytest configuration
# pyproject.toml
[tool.pytest.ini_options]
vcr_cassette_dir = "tests/cassettes"
```

### respx (for httpx/async)

```python
import respx
import httpx

@respx.mock
async def test_async_api_call() -> None:
    respx.get("https://api.example.com/data").mock(
        return_value=httpx.Response(200, json={"result": "ok"})
    )

    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        assert response.json()["result"] == "ok"
```

---

## tox and nox (Test Runners)

### tox Configuration

```ini
# tox.ini
[tox]
envlist = py310, py311, py312, lint, typecheck

[testenv]
deps =
    pytest
    pytest-cov
    hypothesis
commands =
    pytest --cov=src --cov-report=term-missing {posargs}

[testenv:lint]
deps = ruff
commands = ruff check src tests

[testenv:typecheck]
deps = mypy
commands = mypy src
```

### nox Configuration (Python-Based)

```python
# noxfile.py
import nox

@nox.session(python=["3.10", "3.11", "3.12"])
def tests(session: nox.Session) -> None:
    session.install(".", "pytest", "pytest-cov")
    session.run("pytest", "--cov=src", *session.posargs)

@nox.session
def lint(session: nox.Session) -> None:
    session.install("ruff")
    session.run("ruff", "check", "src", "tests")

@nox.session
def typecheck(session: nox.Session) -> None:
    session.install(".", "mypy")
    session.run("mypy", "src")

@nox.session
def safety(session: nox.Session) -> None:
    session.install("pip-audit")
    session.run("pip-audit")
```

Prefer nox over tox for new projects because the Python-based configuration is more flexible and easier to debug.

---

## mutmut (Mutation Testing)

Mutation testing verifies that tests actually catch bugs by introducing small changes (mutations) to source code and checking that tests fail.

### Setup

```bash
pip install mutmut

# Run mutation testing
mutmut run --paths-to-mutate=src/

# View results
mutmut results

# Show a specific mutant
mutmut show 42
```

### Configuration

```toml
# pyproject.toml or setup.cfg
[tool.mutmut]
paths_to_mutate = "src/"
tests_dir = "tests/"
runner = "python -m pytest -x --tb=no -q"
```

### Interpreting Results

| Status    | Meaning                                     |
|-----------|---------------------------------------------|
| Killed    | Test caught the mutation (good)             |
| Survived  | No test caught the mutation (gap in tests)  |
| Timeout   | Mutation caused infinite loop (usually fine) |
| Suspicious| Unusual behavior (investigate)              |

Target a mutation score of 70-80% for well-tested code. Focus on surviving mutants in critical business logic rather than pursuing 100%.

### Common Surviving Mutations

```python
# Boundary mutations: >= changed to >
# Fix: Add boundary tests
def test_discount_at_exact_threshold() -> None:
    assert calculate_discount(quantity=10) == 0.1  # Exactly at boundary

# Operator mutations: + changed to -
# Fix: Add more specific assertions
def test_total_calculation() -> None:
    assert calculate_total(price=10, tax=2) == 12  # Not just > 0

# Return value mutations: True changed to False
# Fix: Assert on the actual return value
def test_is_valid() -> None:
    assert is_valid(good_input) is True  # Not just truthy
    assert is_valid(bad_input) is False   # Not just falsy
```

---

## Plugin Compatibility Matrix

| Plugin           | pytest 7.x | pytest 8.x | Async | Parallel |
|------------------|:----------:|:----------:|:-----:|:--------:|
| pytest-cov       | Yes        | Yes        | Yes   | Yes      |
| pytest-xdist     | Yes        | Yes        | Yes   | N/A      |
| pytest-asyncio   | Yes        | Yes        | N/A   | Yes      |
| pytest-randomly  | Yes        | Yes        | Yes   | Yes      |
| pytest-timeout   | Yes        | Yes        | Yes   | Yes      |
| pytest-benchmark | Yes        | Yes        | No    | No       |
| pytest-sugar     | Yes        | Yes        | Yes   | Partial  |
| freezegun        | Yes        | Yes        | No    | Caution  |
| hypothesis       | Yes        | Yes        | Yes   | Yes      |
