---
rfc: 005
title: "Functional Testing Framework for DocumentDB"
status: Draft
owner: "@nitinahuja89"
issue: "https://github.com/documentdb/documentdb/issues/367"
---

# RFC-005: Functional Testing Framework for DocumentDB

## Problem

DocumentDB currently lacks a systematic functional testing framework that can validate correctness and measure compatibility with MongoDB APIs and behaviors.

**Who is impacted:**
- Contributors who cannot easily validate that their changes don't break existing functionality
- Users who may encounter regressions due to insufficient testing coverage
- Product teams who lack visibility into MongoDB compatibility metrics

**Current consequences:**
- **Limited Confidence**: Developers cannot easily validate that their changes don't break existing functionality
- **Compatibility Gaps**: No systematic way to measure and track compatibility with MongoDB
- **Manual Testing Burden**: Contributors rely on manual testing, slowing development velocity
- **Regression Risk**: Lack of automated testing increases the risk of introducing regressions
- **Feature Validation**: There is no testing framework to allow contributors to write new test cases to validate their features and comnpatibility with MongoDB behavior

**Current workarounds:**
- Manual testing by contributors before submitting PRs
- Ad-hoc compatibility testing
- Reliance on unit tests that don't cover end-to-end scenarios
- Post-deployment discovery of functional or compatibility issues

**Success criteria:**
- Users are able to run all functional tests against locally hosted or remotely hosted DocumentDB with a single command and no setup involved
- Contributors can simply add new test files and it would get picked up by the test runner and include in the functional test suite
- Contributors/ users should get a list of all the test failures together (the test execution should not abort on a single failed test)
- Contributors get easy to understand messages/logs for each failing test that points to the exact assertion that has failed

**Non-goals:**
- Performance/load/stress testing
- Unit testing improvements
- Security testing
- Migration testing from other databases

---

## Approach

The proposed solution is a comprehensive functional testing framework that combines two complementary approaches:

1. **Custom Functional Test Suite**: A purpose-built test suite specifically designed for DocumentDB to validate functional correctness and compatibility with MongoDB.
2. **MongoDB Service Tests Integration**: Leveraging existing MongoDB service tests without modification to measure compatibility.

**Why this approach is preferable:**

- **Comprehensive Coverage**: Custom tests ensure DocumentDB-specific scenarios are covered, while MongoDB service tests provide broad compatibility validation
- **Dual Validation**: Running the same tests against both DocumentDB and MongoDB provides direct compatibility measurement
- **Scalability**: The purpose-built framework will be designed for easy test authoring and parallel execution

**Key benefits:**
- Automated compatibility measurement and reporting
- Easy test authoring for contributors using familiar Python/PyMongo
- Integration with existing development workflows (local, CI/CD)
- Clear failure reporting and debugging capabilities
- Systematic test organization using multi-dimensional tagging

**Key tradeoffs:**
- Initial development investment for the testing infrastructure vs long-term development velocity gains
- Test execution time vs comprehensive coverage

**Alignment with existing architecture:**
- Integrates with current CI/CD infrastructure (GitHub Actions)
- Follows Docker-based deployment patterns
- Complements existing unit testing framework

---

## Detailed Design

### Functional Components

The functional testing framework consists of several key components that work together to provide comprehensive testing capabilities.

**1. Test Runner:**
- **Purpose**: Orchestrates overall test execution strategy and coordinates the testing workflow
- **Responsibilities**:
  - Parses test configuration parameters
  - Identifies the right set of tests to execute based on test suite, tags, and filters
  - Schedules tests based on dependencies and parallelism settings
  - Coordinates with Test Executor for test execution
  - Manages overall test execution workflow

**2. Test Executor:**
- **Purpose**: Executes tests using the appropriate test framework and manages test lifecycle
- **Responsibilities**:
  - Executes individual tests from the specified test suite
  - Manages test lifecycle (setup, run test, collect result, cleanup)
  - Handles test isolation (namespace isolation, data seeding)
  - Runs tests in parallel internally using threads or processes based on parallelism configuration
  - Collects and returns test results to Test Runner

**Why Test Executor is Separate:**

The Test Executor is architecturally separated from the Test Runner to enable future extensibility:

1. **Distributed Test Execution**: Test Executor could be deployed across multiple machines for large-scale parallel testing, while Test Runner remains centralized for coordination.

2. **Multiple Execution Engines**: Different Test Executor implementations can support different test frameworks:
   - PyMongo-based executor (initial implementation)
   - JavaScript/Node.js executor (for MongoDB driver tests)
   - Other language-specific executors as needed

3. **Pluggable Execution Strategies**: Test Executor can be swapped or extended without modifying Test Runner:
   - Local execution (initial implementation)
   - Remote execution (cloud-based test execution)
   - Containerized execution (each test in isolated container)
   - Custom execution strategies for specific test types

This separation follows the **Strategy Pattern**, where Test Runner defines the orchestration logic, and Test Executor provides the test execution implementation.

**3. Result Analyzer:**
- **Purpose**: Analyzes and compares test results
- **Responsibilities**:
  - Compares results between DocumentDB and MongoDB
  - Generates compatibility metrics and statistics
  - Identifies patterns in test failures

**4. Test Suites:**
- **Custom Functional Test Suite**: 
  - DocumentDB-specific functional tests using Python/PyMongo
  - Multi-dimensional tagging system for test organization
  - Designed for easy test authoring by contributors
- **MongoDB Service Tests**: 
  - Existing MongoDB compatibility tests used without modification
  - Broad MongoDB API coverage for compatibility validation
  - Provides baseline for measuring DocumentDB compatibility

**5. Target Systems:**
- **DocumentDB Instance**: The system under test (local or remote deployment)
- **MongoDB Instance**: Reference system for compatibility validation (local or remote deployment)

### Functional Test Tagging System

The following two-dimensional tagging strategy is used for organizing and filtering tests in the **documentdb-functional-test** suite:

**Horizontal Tags (API Operations):**
- `find`, `insert`, `update`, `delete`, `aggregate`, `index`, `admin`, `collection-mgmt`

**Vertical Tags (Cross-cutting Features):**
- `rbac`, `decimal128`, `collation`, `transactions`, `geospatial`, `text-search`, `validation`, `ttl`

**Example Test Tags:**
- Find operation using decimal128 with collation: `[find, decimal128, collation]`
- Find with RBAC: `[find, rbac]`
- TTL Index creation: `[index, ttl]`

### Deployment Architecture

The functional testing framework is deployed as a single Docker container that packages all components together.

*Architecture:*
- Docker Image: `documentdb/functional-testing-suite`
- Contains: Test Runner + Test Executor + Result Analyzer + Functional Test Suite

*How it works:*
- User runs the container with configuration parameters (e.g., which tests to run, tags to filter, parallelism level)
- Test Runner parses configuration and identifies tests to execute
- Test Executor runs tests in parallel internally (using threads/processes based on parallelism configuration)
- Each test runs against both DocumentDB and MongoDB instances for compatibility validation
- Result Analyzer processes results, categorizes any failures, compares outcomes, and generates compatibility reports
- All components share the same container environment for efficient communication

*Architecture Benefits:*
- **Simplicity**: Single container deployment with no inter-container communication or orchestration
- **Resource Efficiency**: Lower resource overhead, no container orchestration needed
- **True Single Command**: Achieves the success criteria of execution through a single command with no setup
- **Simplified Debugging**: All logs and processes in one container for easier troubleshooting

### Test Execution Flow

```
Test Runner:
  → Parse configuration (test suite, tags, parallelism)
  → Discover and filter tests based on configuration
  → Schedule tests based on dependencies
  → Send scheduled tests to Test Executor
                    ↓
Test Executor:
  → Receive scheduled tests
  → Create thread/process pool (size=N)
  → Execute tests in parallel (N concurrent threads):
      For each test:
        For each target database (MongoDB, DocumentDB):
          → Generate unique namespace
          → Setup (data seeding if needed)
          → Run test against target database
          → Cleanup
          → Collect results
  → Wait for all threads to complete
  → Return aggregated results
                    ↓
Test Runner:
  → Receive all test results
  → Pass results to Result Analyzer
                    ↓
Result Analyzer:
  → Receive test results
  → Compare DocumentDB vs MongoDB results for each test
  → Calculate compatibility metrics and statistics
  → Identify patterns in test failures
  → Generate reports and dashboards
  → Return reports
                    ↓
Test Runner:
  → Receive final reports
  → Return reports to user
```

### Implementation Details

#### Test Runner Implementation

**Configuration Management:**
- Configuration is provided via YAML configuration file
- Configuration parameters:
  - `test_suite`: Which test suite to run (documentdb-functional-test, mongodb-service-tests, or both)
  - `tags`: List of tags to filter tests in documentdb-functional-test suite (e.g., `[find, rbac]`)
  - `parallelism`: Number of concurrent test executors (default: CPU count)
  - `documentdb_uri`: Connection string for DocumentDB instance
  - `mongodb_uri`: Connection string for MongoDB reference instance
  - `output_format`: Report format (json, html, junit)
  - `fail_fast`: Whether to stop on first failure (default: false)

**Test Discovery Mechanism:**
- Scan test directories recursively for Python files matching pattern `test_*.py`
- Parse test files to extract:
  - Test function names (functions starting with `test_`)
  - Test tags (from decorators like `@tags(['find', 'rbac'])`)
  - Test dependencies (from `@depends_on(['test_name'])` decorators) - *Note: Dependency parsing is optional since most tests should not have dependencies*
  - Parallel safety flag (from `@parallel_safe` decorator)
- Build test registry with metadata for each discovered test

**Test Scheduling Algorithm:**
- Build dependency graph from test dependencies
- Perform topological sort to determine execution order
- Group tests into batches based on:
  - Tests with no dependencies can run immediately
  - Tests with dependencies wait for prerequisite tests to complete
  - Parallel-safe tests are grouped for concurrent execution
  - Sequential tests are isolated in separate batches
- Send scheduled test batches to Test Executor for execution (Test Executor will execute tests within each batch in parallel using its internal thread pool based on parallelism configuration)

**State Management:**
- Receive status updates from Test Executor about test execution state:
  - Pending: Tests waiting to be executed
  - Running: Tests currently being executed
  - Completed: Tests that have finished (pass/fail)
  - Skipped: Tests skipped due to failed dependencies
- Use execution state to make scheduling decisions (e.g., when to send next batch based on dependencies)
- Display real-time progress updates to user based on status received from Test Executor
- Handle graceful shutdown on interrupt signals (SIGINT, SIGTERM) and coordinate with Test Executor to stop execution

**Dependency Resolution:**

**Note: Test dependencies should be used sparingly.** In most cases, use `setUp()` methods or fixtures instead to manage test prerequisites. Dependencies introduce tight coupling between tests and can cause:
- Cascading failures (one failed test causes many to be skipped)
- Reduced parallelism (dependent tests must wait)

**When dependencies are appropriate (rare cases):**
- Genuinely sequential tests that cannot be parallelized (e.g., multi-step transaction flows) or Migration/upgrade testing scenarios

**Implementation:**
- Tests can declare dependencies using `@depends_on(['test_name'])` decorator
- Build dependency graph from parsed test metadata
- Validate dependency graph at discovery time to detect cycles
- Use topological sort to determine execution order
- Dependent tests wait for prerequisite tests to complete
- If a prerequisite test fails, dependent tests are marked as skipped

**Preferred alternative:** Use `setUp()` or fixtures to create necessary preconditions for each test independently.

#### Test Executor Implementation

**Parallelism Mechanism:**
- Use Python's `concurrent.futures.ThreadPoolExecutor` for I/O-bound test execution
- Thread pool size determined by parallelism configuration parameter
- Execute all tests within a received batch concurrently using the thread pool (Test Runner has already grouped tests appropriately based on parallel safety)
- Each thread executes tests independently with isolated database namespaces
- Thread-safe result collection using queue or thread-safe data structures

**Namespace Isolation Strategy:**
- Generate unique namespace for each test: `test_{test_name}_{timestamp}_{random_suffix}`
- Create isolated database and collection with the same names in both DocumentDB and MongoDB:
  - Database name: `test_db_{namespace}`
  - Collection name: `test_collection_{namespace}`
- The `DocumentDBTestCase` base class provides pre-scoped PyMongo database and collection objects:
  - `self.test_database`: Database object scoped to the isolated namespace
  - `self.test_collection`: Collection object scoped to the isolated namespace
  - Test authors use these pre-configured objects directly without specifying database/collection names
- For tests requiring multiple collections, test authors can access other collections via `self.test_database['collection_name']` (all within the same isolated database namespace)
- Namespace cleanup will happen after test completion (success or failure)

**Test Lifecycle Management:**
For each test, the following phases are executed (once for each target database):

1. **Setup Phase:**
   - Create isolated namespace
   - Establish connection to the target database
   - Seed test data if specified in test fixture (using `load_fixture()`)
   
2. **Execution Phase:**
   - Execute test function against the target database
   - Capture result (return value, exceptions, execution time)
   
3. **Cleanup Phase:**
   - Drop the test database created during the test (identified by namespace)
   - Drop all users/roles etc created in the admin database (except admin user)
   - Close database connections
   - Log test execution details
   
   **Note:** Cleanup is handled automatically by the base class regardless of test success or failure. Test authors do not need to implement `tearDown()` unless they have additional cleanup requirements beyond what the base class provides.

The test executor runs the complete lifecycle against each target database (DocumentDB/ MongoDB) ensuring isolated and independent execution for each database.

**Error Handling:**
- Catch and log all exceptions during test execution
- Distinguish between:
  - Test failures (assertion errors)
  - Test errors (unexpected exceptions)
  - Infrastructure errors (connection failures, timeouts)
- Continue execution of other tests even if one test fails (unless fail_fast is enabled)
- Provide detailed error messages with stack traces

#### Test Organization

**Directory Structure:**
```
functional-tests/
├── documentdb-functional-test/  # DocumentDB functional tests
│   ├── test_find.py
│   ├── test_insert.py
│   ├── test_aggregate.py
│   └── fixtures/                # Test data fixtures
│       ├── users.json
│       └── products.json
├── common/                      # Shared utilities
│   ├── test_base.py             # Base test class
│   ├── fixtures.py              # Fixture loading utilities
│   └── assertions.py            # Custom assertion helpers
└── config/
    ├── default.yaml             # Default configuration
    └── ci.yaml                  # CI-specific configuration
```

**File Naming Conventions:**
- Test files: `test_<feature>.py` (e.g., `test_find.py`, `test_rbac.py`)
- Test functions: `test_<scenario>` (e.g., `test_find_with_filter`, `test_rbac_read_permission`)
- Fixture files: `<entity>.json` (e.g., `products.json`, `orders.json`)

**Test Example:**

```python
from common.test_base import DocumentDBTestCase
from common.decorators import tags, parallel_safe

@tags(['find', 'rbac'])
@parallel_safe
class TestRBACFind(DocumentDBTestCase):
    
    def setUp(self):
        """Setup RBAC users and test data"""
        # Create a user with read role (base class method handles admin operations)
        self.create_user(
            username='reader_user',
            password='reader_pass',
            roles=[{'role': 'read', 'db': self.test_database.name}]
        )
        
        # Load product fixture data (base class method)
        # Reads products.json and inserts into both DocumentDB and MongoDB
        self.load_fixture('products.json')
    
    def test_rbac_read_permission(self):
        """Verify read-only user can query products"""
        # Authenticate as reader_user using base class method
        self.authenticate('reader_user', 'reader_pass')
        
        # Execute find operation - should succeed with read permission
        result = self.test_collection.find({'category': 'electronics'})
        result_list = list(result)
        
        # Assertions
        assert len(result_list) == 2, "Expected to find exactly 2 electronics products"
    
    # Note: tearDown is handled automatically by the base class
    # The base class will:
    # - Drop the test database created during the test
    # - Drop all users/roles etc created (except admin user)
    # 
    # Custom tearDown is only needed for additional cleanup beyond the base class
```

**Fixture File Format:**

Fixtures contain test data (documents) that get inserted into the test collection.

Example - Product documents for testing (products.json):
```json
[
  {
    "_id": "prod1",
    "name": "Laptop",
    "category": "electronics",
    "price": 999.99,
    "in_stock": true
  },
  {
    "_id": "prod2",
    "name": "Desk Chair",
    "category": "furniture",
    "price": 299.99,
    "in_stock": true
  },
  {
    "_id": "prod3",
    "name": "Monitor",
    "category": "electronics",
    "price": 399.99,
    "in_stock": false
  }
]
```

These documents will be inserted into the test collection by `load_fixture()`, allowing your test to access the data in the collection and verify the feature.

**Tagging Conventions:**
- Use lowercase tags
- Combine horizontal and vertical tags: `@tags(['find', 'rbac', 'decimal128'])`
- Required tags for all tests: At least one horizontal tag (API operation)
- Optional tags: Vertical tags for cross-cutting features


#### Result Analyzer Implementation

**Result Analysis:**
The Result Analyzer examines test execution results from both MongoDB and DocumentDB to identify compatibility issues and patterns. Each test contains assertions that validate correct behavior - the analyzer's role is to surface failures and categorize them.

**Test Outcome Classification:**
For each test, analyze the results from both databases:

1. **Passed on Both (Compatible)**
   - Test assertions passed on both MongoDB and DocumentDB
   - No compatibility issues

2. **Failed on DocumentDB Only (Compatibility Issue)**
   - Test passed on MongoDB but failed on DocumentDB
   - Indicates a compatibility gap
   - **This is the primary concern** - these tests identify areas where DocumentDB has gaps in functionality

3. **Failed on MongoDB Only (Difference in Behavior)**
   - Test passed on DocumentDB but failed on MongoDB
   - Indicates differing behavior between the databases

4. **Failed on Both (Test or Infrastructure Issue)**
   - Test failed on both MongoDB and DocumentDB
   - Could indicate test logic errors, infrastructure problems, etc
   - Requires investigation to determine root cause

**Failure Categorization by Tags:**
Group failed tests by their tags to identify patterns and problem areas:

- **By API Operation Tags**: Which operations have compatibility issues?
  - Example: "5 out of 20 `aggregate` tests failed on DocumentDB"

- **By Feature Tags**: Which cross-cutting features have compatibility issues?
  - Example: "8 out of 12 `decimal128` tests failed on DocumentDB"

- **Combined Analysis**: Identify specific combinations with issues
  - Example: "`aggregate` + `decimal128` tests are failing"

**Metrics Calculation:**
- **DocumentDB Pass Rate**: `tests_passed_on_documentdb / total_tests * 100`
- **MongoDB Pass Rate**: `tests_passed_on_mongodb / total_tests * 100`

**Report Generation:**

The framework generates multiple types of outputs, each serving different consumption models:

- **JSON Report**: For machine consumption - enables programmatic access, custom analysis tools, monitoring system integration, historical data tracking, and building custom dashboards
- **JUnit XML**: For Github integration - provides rich UI integration in GitHub Actions for PR reviews and quick status checks
- **Dashboard**: For visual consumption - offers rich charts, tables, and trend analysis for deep investigation

1. **JSON Report** (for programmatic access and automation):
   - **Primary purpose**: Machine-readable format for CI/CD pipelines, APIs, and automation scripts
   - **Use cases**: 
     - Custom analysis and reporting tools
     - Integration with monitoring and alerting systems
     - Historical data storage and trend tracking
     - Building custom dashboards and visualizations
     - Feeding data to other automation workflows
   - **Content**: Concise, structured data with high-level test outcomes and metrics
   - Brief error identification (error type only, not full details)

```json
{
  "summary": {
    "total_tests": 150,
    "passed_on_both": 140,
    "failed_on_documentdb_only": 8,
    "failed_on_mongodb_only": 1,
    "failed_on_both": 1,
    "documentdb_pass_rate": 93.3,
    "mongodb_pass_rate": 99.3
  },
  "tests": [
    {
      "name": "test_find_with_filter",
      "mongodb_status": "passed",
      "documentdb_status": "passed",
      "outcome": "compatible",
      "mongodb_duration": 0.45,
      "documentdb_duration": 0.52,
      "tags": ["find"]
    },
    {
      "name": "test_aggregate_decimal",
      "mongodb_status": "passed",
      "documentdb_status": "failed",
      "outcome": "compatibility_issue",
      "mongodb_duration": 0.38,
      "documentdb_duration": 0.41,
      "error_type": "AssertionError",
      "tags": ["aggregate", "decimal128"]
    }
  ]
}
```

Note: The JSON report includes only `error_type` for quick identification. Full error messages, stack traces, and debugging details are available in the test execution logs.

2. **JUnit XML (for GitHub Actions integration):**
   
   JUnit XML is a standard test report format that integrates seamlessly with GitHub Actions to provide rich test result visualization in pull requests.
   
   **How it works with GitHub Actions:**
   - Framework generates JUnit XML file after test execution
   - Various test reporter actions are available in the GitHub Actions Marketplace that can parse JUnit XML and visualize results
   - Test results appear in the "Checks" tab of pull requests
   - Test suites are grouped by tags for organized reporting
   - Includes failure messages and stack traces for quick debugging
   
   **Benefits:**
   - **Build gating**: Automatically mark builds as failed when tests fail
   - **Test tracking**: Identify flaky tests and track pass/fail trends over time
   - **Quick debugging**: View per-test status and failure details directly in PR UI
   - **No log diving**: See test results without reading workflow logs
   
   Note: JUnit XML is an industry-standard format, so the framework could support other CI/CD platforms in the future if needed.

3. **Dashboard:**
   - Summary statistics with charts
   - Test results table with filtering by status/tags
   - Detailed failure information with diffs
   - Trend analysis using historical data
