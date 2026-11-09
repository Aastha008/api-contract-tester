# API Contract Tester

[![Python Version](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/)
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg)]()
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Code Style: Black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

> Automated runtime OpenAPI specification compliance verification and schema validation for HTTP services.

**API Contract Tester** is an enterprise-grade CLI utility and validation engine designed to verify live HTTP services against their OpenAPI v3.0 and v3.1 specifications. By parsing target schemas, generating dynamic test suites, and executing asynchronous HTTP requests against running endpoints, it automatically flags breaking contract changes, payload structural drift, missing attributes, and status code mismatches before they hit production.

---

## Problem Statement & Target Audience

### The Problem
In modern microservices architectures, web APIs evolve rapidly. Frontends, mobile clients, and downstream microservices rely on documented OpenAPI (Swagger) specifications as a strict contract. However:
* Hand-written or auto-generated documentation frequently drifts from actual deployed behavior during fast-paced development cycles.
* Breaking API changes (e.g., renamed fields, type mutations, missing required attributes, unexpected `500 Internal Server Error` responses) often pass unit tests and escape into staging or production environments.
* Integration failure discovery is delayed until frontend or downstream client integration testing fails late in the release process.

### Target Audience
* **Backend Engineers:** Validate API implementations against technical specifications during pull requests.
* **QA & SDET Engineers:** Automate API contract regression testing within CI/CD pipelines without manually writing assertions for every endpoint.
* **DevOps & Platform Teams:** Implement release gates that prevent non-compliant services from deploying to staging and production clusters.

---

## Key Features

* **OpenAPI Spec Parser (v3.0 & v3.1):** Native support for both JSON and YAML spec formats, including complex `$ref` JSON pointer resolution, parameter schemas, and nested object hierarchies.
* **Automated Test Suite Generation:** Dynamic test case construction backed by Pydantic models for rigid payload structure assertion, enum verification, and data-type validation.
* **Rich Terminal Runner:** Interactive CLI powered by Typer and Rich featuring real-time visual progress bars, status updates, and inline visual schema diffs highlighting payload drift.
* **Comprehensive Export Options:** Export execution runs into structured JSON or standalone HTML report dashboards detailing broken contracts, missing fields, schema violations, and HTTP status code mismatches.

---

## Architecture & System Design

The application uses a decoupled pipeline architecture. The parser reads and normalizes an OpenAPI document, passes structured definitions to the test generator, executes async requests via `httpx`, evaluates responses with Pydantic payload validators, and formats output through Typer/Rich or file exporters.

```
┌───────────────────────────┐
│  OpenAPI Spec File        │
│  (JSON / YAML v3.0, v3.1) │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│     src/parser.py         │  ◄── Resolves $ref JSON Pointers & Normalizes Paths
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│    src/generator.py       │  ◄── Builds Test Matrices & Expected Payloads
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│      src/core.py          │  ◄── Asynchronous HTTP Runner (httpx)
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│    src/validator.py       │  ◄── Structural & Pydantic Schema Validation
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│      src/main.py          │  ◄── Typer CLI, Rich UI & Exporter Engine
└───────────────────────────┘
```

### Module Breakdown

| Module | Purpose |
| :--- | :--- |
| `src/models.py` | Defines Pydantic models for internal spec representation, endpoint configurations, validation rules, and output report models. |
| `src/parser.py` | Parses raw JSON/YAML OpenAPI v3.0/v3.1 files, normalizes routes, and recursively resolves `$ref` JSON pointers. |
| `src/generator.py` | Constructs dynamic operational test suites based on endpoint parameters, request bodies, and expected status codes. |
| `src/core.py` | Handles orchestrations of the overall test execution loop using an asynchronous `httpx.AsyncClient` pipeline. |
| `src/validator.py` | Executes structural schema diffing and type-checking against actual HTTP response payloads and status codes. |
| `src/api.py` | Provides a programmatic Python API wrapper for integrating validation logic directly into custom scripts or ASGI apps. |
| `src/main.py` | Typer CLI entry point responsible for parameter handling, Rich terminal formatting, diff rendering, and reporting. |

---

## Technology Stack

* **Language:** Python 3.10+
* **CLI Framework:** Typer
* **Terminal UI & Diff Formatting:** Rich
* **Data Validation & Schemas:** Pydantic v2
* **HTTP Execution Engine:** httpx (AsyncIO)
* **YAML Parsing:** PyYAML
* **Testing Framework:** pytest

---

## Installation & Virtual Environment Setup

### Prerequisites
* Python 3.10 or higher installed on your system.

### Step-by-Step Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/your-org/api-contract-tester.git
   cd api-contract-tester
   ```

2. **Create and activate a virtual environment:**
   * **Linux/macOS:**
     ```bash
     python3 -m venv venv
     source venv/bin/activate
     ```
   * **Windows (PowerShell):**
     ```powershell
     python -m venv venv
     .\venv\Scripts\Activate.ps1
     ```

3. **Install dependencies:**
   ```bash
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

---

## Environment Variables Configuration

Copy `.env.example` to create your local `.env` configuration file:

```bash
cp .env.example .env
```

### Supported Configuration Keys

| Variable Name | Default Value | Description |
| :--- | :--- | :--- |
| `ACT_TARGET_BASE_URL` | `http://localhost:8000` | Target HTTP service base URL to execute tests against. |
| `ACT_SPEC_PATH` | `./openapi.json` | Path to the default OpenAPI specification JSON or YAML file. |
| `ACT_TIMEOUT_SECONDS` | `10.0` | Timeout threshold per HTTP request in seconds. |
| `ACT_STRICT_MODE` | `true` | If true, extra fields present in API responses not documented in the spec fail validation. |
| `ACT_LOG_LEVEL` | `INFO` | Application logging verbosity (`DEBUG`, `INFO`, `WARNING`, `ERROR`). |

---

## Usage

### 1. Command Line Interface (CLI)

#### Run Validation against a Target Service
Execute contract tests by pointing to a local or remote OpenAPI spec file and your target base service URL:

```bash
python -m src.main test \
  --spec ./openapi.yaml \
  --base-url http://localhost:8000 \
  --strict
```

#### CLI Options & Flags
```bash
python -m src.main test --help
```

* `--spec, -s`: Path or URL to the OpenAPI specification file (JSON or YAML).
* `--base-url, -u`: Base URL of the live running service.
* `--output-json, -j`: Optional output file path for saving JSON test reports.
* `--output-html, -h`: Optional output file path for generating a visual HTML report dashboard.
* `--strict / --no-strict`: Toggle strict schema validation (disallows undocumented payload fields).

#### Exporting Validation Reports
To run the suite and generate both JSON and HTML report artifacts:

```bash
python -m src.main test \
  --spec ./docs/openapi.json \
  --base-url https://api.staging.example.com \
  --output-json ./reports/contract-results.json \
  --output-html ./reports/dashboard.html
```

---

### 2. Programmatic Python API

You can also use API Contract Tester directly inside your test suites or automation scripts:

```python
import asyncio
from src.api import ContractTesterEngine

async def run_contract_check():
    engine = ContractTesterEngine(
        spec_path="./docs/openapi.yaml",
        base_url="http://127.0.0.1:8000",
        strict=True
    )
    
    results = await engine.execute_suite()
    
    if not results.passed:
        print(f"Contract Test Failed! Found {results.error_count} violations.")
        for violation in results.violations:
            print(f"[{violation.endpoint}] {violation.message}")
    else:
        print("API Contract Verification Succeeded cleanly!")

if __name__ == "__main__":
    asyncio.run(run_contract_check())
```

---

## Automated Testing Instructions

The repository includes test suites covering unit and integration scenarios using `pytest`.

### Running Tests

Execute the full test suite:
```bash
pytest
```

Run test suite with verbose output:
```bash
pytest -vv
```

Run specific test files:
```bash
pytest tests/test_core.py
pytest tests/test_api.py
```

Run tests with coverage reporting (if `pytest-cov` is installed):
```bash
pytest --cov=src --cov-report=term-missing
```

---

## Project Structure

```
.
├── .env.example          # Sample environment configuration file
├── .gitignore            # Git exclusion rules
├── requirements.txt      # Project dependencies file
├── src/
│   ├── __init__.py       # Package marker
│   ├── api.py            # High-level Python API wrapper interface
│   ├── core.py           # Execution lifecycle & async httpx request runner
│   ├── generator.py      # Dynamic test case payload generation engine
│   ├── main.py           # Typer CLI application & Rich report formatting
│   ├── models.py         # Pydantic data schemas for OpenAPI & test results
│   ├── parser.py         # OpenAPI spec parser & JSON pointer reference resolver
│   └── validator.py      # Response payload schema validator and structural diff engine
└── tests/
    ├── __init__.py       # Test package marker
    ├── test_api.py       # Unit tests for the programmatic API module
    └── test_core.py      # Core execution loop & HTTP interaction unit tests
```

---

## Roadmap & Future Enhancements

- [ ] **OpenAPI 3.1 Discriminator & AnyOf Support:** Enhanced polymorphic schema support for complex polymorphic payloads.
- [ ] **OAuth2 / Bearer Token Auth Generators:** Built-in capability to automatically obtain and inject authorization headers during test execution.
- [ ] **GitHub Actions Integration:** Pre-packaged CI GitHub Action for zero-setup PR validation gates.
- [ ] **Mock Server Generator:** Spin up a light ASGI mock server directly from the parsed OpenAPI schema for local development.

---

## License

This project is licensed under the **MIT License**. See the [LICENSE](LICENSE) file for details.