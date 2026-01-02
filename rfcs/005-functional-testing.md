---
rfc: 005
title: "Functional Testing Framework for DocumentDB"
status: Draft
owner: "@nitinahuja89"
issue: "https://github.com/documentdb/documentdb/issues/367"
---

# RFC-005: Functional Testing Framework for DocumentDB

## Problem

DocumentDB currently lacks a systematic end-to-end functional testing framework that can validate correctness of its functionality.

**Who is impacted:**
- Contributors who cannot easily validate that their changes don't break existing functionality
- Users who may encounter regressions due to insufficient testing coverage
- Product teams who lack visibility into functional correctness metrics

**Current consequences:**
- **Limited Confidence**: Developers cannot easily validate that their changes don't break existing functionality
- **Functional Gaps**: No systematic way to measure and track end-to-end functional correctness
- **Manual Testing Burden**: Contributors rely on manual testing, slowing development velocity
- **Regression Risk**: Lack of automated end-to-end testing increases the risk of introducing regressions
- **Feature Validation**: There is no testing framework to allow contributors to write end-to-end new test cases to validate their features

**Current workarounds:**
- Manual testing by contributors before submitting PRs
- Ad-hoc end-to-end functional testing
- Reliance on unit tests that don't cover end-to-end scenarios
- Post-deployment discovery of functional issues

**Success criteria:**
- Users are able to run all functional tests against locally hosted or remotely hosted DocumentDB with a single command and no setup involved
- Contributors can simply add new test files and it would automatically get picked up to be run as part of the functional test suite
- Contributors/ users should get a list of all the test failures together (the test execution should not abort on a single failed test)
- Contributors get easy to understand messages/logs for each failing test that points to the exact cause for the failure
- Contributors should get failed tests categorized by feature tags to make it easy to understand which features have issues

**Non-goals:**
- Performance/load/stress testing
- Unit testing improvements
- Security testing
- Migration testing from other databases

---

## Approach

The proposed solution is a end-to-end functional testing framework that uses **specification-based testing** to validate DocumentDB functionality.

**Self-contained Test Suite**: Tests with explicit specifications that define expected behavior for DocumentDB features. Tests can be executed against any engine implementing the MongoDB wire protocol.

**Why this approach is preferable:**

- **Specification-based**: Tests define explicit expectations for DocumentDB behavior
- **Self-contained**: Each test includes all necessary setup and assertions
- **Future-proof**: Has the ability to support DocumentDB-unique features and functionality
- **Leverages pytest**: Uses proven testing infrastructure

**Key benefits:**
- Automated functional correctness validation using pytest
- Easy test authoring for contributors using familiar pytest
- Integration with existing development workflows (local, CI/CD)
- Clear failure reporting and debugging capabilities
- Systematic test organization using pytest markers

**Key tradeoffs:**
- Initial development investment for the testing infrastructure vs long-term development velocity gains
- Test execution time vs comprehensive coverage

**Alignment with existing architecture:**
- Integrates with current CI/CD infrastructure (GitHub Actions)
- Supports both local execution and execution in remote environments (via a Docker image)
- Complements existing unit testing framework

---

## Detailed Design

### Functional Components

The end-to-end functional testing framework leverages pytest as the core testing infrastructure and adds a separate component to process the test results.

**1. pytest Framework:**
- **Purpose**: Handles test discovery, execution, parallelization, and reporting
- **Responsibilities**:
  - Test discovery and filtering using pytest markers
  - Parallel test execution using pytest-xdist (multiprocessing)
  - Multi-engine test execution via parametrization
  - Fixture-based setup and cleanup
  - Multiple output formats via plugins

**2. Result Analyzer:**
- **Purpose**: Post-processes pytest results to generate metrics
- **Responsibilities**:
  - Analyzes pass/fail results for DocumentDB functionality
  - Generates metrics by feature tags
  - Categorizes failure types (UNSUPPORTED, SPEC_FAILURE, UNEXPECTED_ERROR)
  - Creates dashboards and reports

**3. Test Suites:**
- **DocumentDB Functional Test Suite**: 
  - Self-contained tests with explicit specifications for DocumentDB functionality
  - Multi-dimensional tagging system using pytest markers
  - Designed for easy test authoring by contributors
  - Supports DocumentDB unique features and capabilities

### Functional Test Tagging System

The following two-dimensional tagging strategy is used for organizing and filtering tests using pytest markers:

**Horizontal Tags (API Operations):**
- `find`, `insert`, `update`, `delete`, `aggregate`, `index`, `admin`, `collection_mgmt`

**Vertical Tags (Cross-cutting Features):**
- `rbac`, `decimal128`, `collation`, `transactions`, `geospatial`, `text_search`, `validation`, `ttl`

**Additional Tags:**
- `smoke`: Quick feature detection tests to determine if functionality is implemented

**Example Test Tags:**
- Find operation using decimal128 with collation: `@pytest.mark.find @pytest.mark.decimal128 @pytest.mark.collation`
- Find with RBAC: `@pytest.mark.find @pytest.mark.rbac`
- TTL Index creation: `@pytest.mark.index @pytest.mark.ttl`
- Smoke test for aggregation: `@pytest.mark.aggregate @pytest.mark.smoke`

The tags are used for grouping, organizing and filtering tests. They allow users to run tests for specific features and enable grouping when reporting test results.

### Running the Tests

The functional testing framework is available in two formats.

**1. Source Code (for contributors):**
- Raw source code with pytest tests and plugins
- Requires local Python environment setup

```bash
git clone https://github.com/documentdb/functional-tests
cd functional-tests
pip install -r requirements.txt
pytest --engine documentdb=mongodb://localhost:27017 --engine mongodb=mongodb://mongo:27017
```

**2. Docker Image (for cluster testing):**
- Pre-built image with all dependencies included
- Published as `documentdb/functional-tests`

```bash
# AWS DocumentDB
docker run documentdb/functional-tests \
  --engine documentdb=mongodb://cluster.docdb.amazonaws.com:27017 \
  --engine mongodb=mongodb://mongo.example.com:27017

# Azure Cosmos DB (MongoDB API)
docker run documentdb/functional-tests \
  --engine cosmosdb=mongodb://myaccount.mongo.cosmos.azure.com:27017 \
  --engine mongodb=mongodb://mongo.example.com:27017
```

*Benefits:*
- **Flexibility**: Contributors can run and debug tests locally
- **Portability**: Docker image provides consistent environment for testing in remote environments
- **CI/CD Integration**: Works with existing GitHub Actions workflows

### Test Execution Flow

```
pytest:
  → Parse configuration (engines, tags, parallelism)
  → Discover and filter tests based on markers
  → Execute tests in parallel using pytest-xdist (multiprocessing)
      For each test:
        For each target engine (parametrized):
          → Generate unique namespace using test name
          → Setup using pytest fixtures
          → Run test against target engine
          → Cleanup via fixture teardown
          → Collect results (pass/fail)
  → Generate output
                    ↓
Result Analyzer:
  → Parse pytest output
  → Categorize results by failure type:
    - UNSUPPORTED (smoke test failed)
    - SPEC_FAILURE (assertion failed)
    - UNEXPECTED_ERROR (infrastructure issue)
  → Calculate metrics by tags
  → Generate weighted overall compatibility scores
  → Create reports and dashboards
                    ↓
Output:
  → Tag-level metrics
  → Overall weighted compatibility scores
  → Detailed failure categorization
  → Multiple report formats (JSON, HTML, JUnit XML)
```

### Implementation Details

#### pytest Test Framework

**Configuration Management:**
- Configuration provided via command line arguments and optional YAML files
- Configuration parameters:
  - `engine`: Engine configurations to test against (e.g., `--engine documentdb=mongodb://localhost:27017 --engine mongodb-7.0=mongodb://mongo:27017`)
  - `tags`: pytest marker filtering (e.g., `-m "find and rbac"`)
  - `parallelism`: Number of concurrent processes via pytest-xdist (e.g., `-n 8`)
  - `output_format`: Report format (JSON via `--json-report`, JUnit XML via `--junitxml`)
  - `fail_fast`: Stop on first failure (`-x`)

**Test Discovery:**
- Uses pytest's built-in discovery for files matching `test_*.py`
- Leverages pytest markers for tagging and filtering
- No custom test registry needed - pytest handles metadata

**Multi-Engine Execution:**
- Engines to run against specfied via configuration
- Uses pytest parametrization to run same test against multiple engines
- Each test automatically runs against all specified engines

**Parallel Execution:**
- Uses pytest-xdist for multiprocessing-based parallelism (avoids Python GIL limitations)
- Automatic load balancing across worker processes
- No custom thread pool management needed

#### Test Organization

**Directory Structure:**
For large features like `find`, tests are split by sub-functionality: basic queries, query operators, logical operators, projections, sorting, cursors, etc. Each file contains focused test cases for that specific aspect.

```
functional-tests/
├── find/                        # Find operation tests
│   ├── test_basic_queries.py    # Simple find(), findOne()
│   ├── test_query_operators.py  # $eq, $ne, $gt, $lt, $in, etc.
│   ├── test_logical_operators.py # $and, $or, $not, $nor
│   ├── test_projections.py      # Field inclusion/exclusion
│   ├── test_sorting.py          # sort(), compound sorts
│   └── test_cursors.py          # Cursor behavior, iteration
├── aggregate/                   # Aggregation pipeline tests
│   ├── test_match_stage.py
│   ├── test_group_stage.py
│   └── test_pipeline_combinations.py
├── common/                      # Shared utilities
│   ├── conftest.py              # pytest fixtures
│   └── assertions.py            # Custom assertion helpers
└── config/
    └── engines.yaml             # Engine configurations
```

**File Naming Conventions:**
- Test files: `test_<feature>.py` (e.g., `test_basic_queries.py`, `test_rbac.py`)
- Test functions: `test_<scenario>` (e.g., `test_find_with_filter`, `test_rbac_read_permission`)
- Use snake_case for all method names following Python conventions

**Test Example:**

```python
import pytest
from pymongo import MongoClient

@pytest.fixture
def collection(request, database_client):
    """Create isolated collection for test"""
    collection = database_client.test_db[request.node.name]
    yield collection
    collection.drop()

@pytest.mark.documents([{"status": "active"}, {"status": "inactive"}])
@pytest.mark.find
@pytest.mark.rbac
def test_rbac_read_permission(collection):
    """Verify read-only user can query documents"""
    # Test defines explicit specification for DocumentDB behavior
    result = collection.find({"status": "active"})
    result_list = list(result)
    
    # Specification-based assertions - defines expected DocumentDB behavior
    assert len(result_list) == 1, "Expected to find exactly 1 active document"
    assert result_list[0]["status"] == "active"
```

**Tagging Conventions:**
- Use pytest markers: `@pytest.mark.find`, `@pytest.mark.rbac`
- Combine horizontal and vertical tags: `@pytest.mark.find @pytest.mark.rbac @pytest.mark.decimal128`
- Required: At least one horizontal tag (API operation)
- Optional: Vertical tags for cross-cutting features
- Smoke tests: `@pytest.mark.smoke` for quick feature detection


#### Result Analyzer Implementation

**Result Analysis:**
The Result Analyzer is a post-processing script that parses pytest output to generate functionality metrics. Tests use specification-based assertions that define expected DocumentDB behavior.

**Test Outcome Classification:**
For each test, analyze the result.

1. **Passed**
   - Test assertions passed
   - Feature works as specified

2. **Failed**
   - Test assertions failed
   - Feature does not work as specified

**Failure Type Categorization:**
- **UNSUPPORTED**: Smoke test failed, feature not implemented
- **SPEC_FAILURE**: Test assertion failed, feature is either partially implemented or has bugs
- **UNEXPECTED_ERROR**: Infrastructure/connection issues

**Failure Categorization by Tags:**
Group test results by their pytest markers to identify patterns

- **By API Operation Tags**: Which operations have issues?
  - Example: "5 out of 20 `aggregate` tests failed"

- **By Feature Tags**: Which cross-cutting features have issues?
  - Example: "8 out of 12 `decimal128` tests failed"

**Metrics Calculation:**
- **Feature coverage**: Functionality validation across different DocumentDB features
- **Tag-level metrics**: Pass rate for each tag
- **Overall functionality**: Weighted compatibility metrics to prevent high-volume tags from dominating

**Report Generation:**

The framework generates multiple types of outputs for different consumption models

1. **JSON Report** (for programmatic access and automation):
   - Machine-readable format for CI/CD pipelines and automation scripts
   - Structured data with test outcomes and metrics by engine and tag
   - Brief error identification (error type only, not full details)

```json
{
  "summary": {
    "total_tests": 150,
    "passed": 140,
    "failed": 10,
    "pass_rate": 93.3
  },
  "by_tags": {
    "find": {"passed": 45, "failed": 5, "pass_rate": 90.0},
    "aggregate": {"passed": 30, "failed": 3, "pass_rate": 90.9}
  },
  "tests": [
    {
      "name": "test_find_with_filter",
      "status": "passed",
      "duration": 0.52,
      "tags": ["find"]
    },
    {
      "name": "test_aggregate_decimal",
      "status": "failed",
      "duration": 0.41,
      "error_type": "AssertionError",
      "tags": ["aggregate", "decimal128"]
    }
  ]
}
```

2. **JUnit XML** (for GitHub Actions integration):
   - Standard test report format for CI/CD integration
   - Rich UI integration in GitHub Actions for PR reviews
   - Test results appear in "Checks" tab with failure details

3. **Dashboard** (for visual consumption):
   - Summary statistics with charts
   - Test results table with filtering by status/tags
   - Detailed failure information
   - Trend analysis using historical data
