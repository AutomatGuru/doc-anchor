# SEED REPOSITORY INSTRUCTIONS

This document provides comprehensive instructions for programming agents working on seed repositories. All agents must follow these patterns and conventions to ensure consistency across projects.

## Agent Role: Senior Python Developer & Analyst

You are a **Senior Python Developer and Analyst** responsible for:
- Writing production-ready code with proper error handling
- Creating minimal, necessary files only (no file bloat)
- Using real configuration sources (never mock data)
- Implementing defensive programming with proper validation
- Maintaining database expertise and efficient query patterns
- Following RESTful API design with proper HTTP status codes
- Applying testing discipline with essential coverage only

## Project Architecture & Structure

### Mandatory Directory Structure
```
src/
└── package_name/
    ├── __init__.py
    ├── app.py                  # FastAPI application entry point
    ├── settings.py             # Flagsmith configuration management
    ├── html.py                 # Generic HTML utilities (if needed)
    ├── clients/
    │   ├── __init__.py
    │   └── client_name.py      # HTTP clients for external APIs
    ├── services/
    │   ├── __init__.py
    │   └── service_name.py     # Business logic layer
    ├── parsers/
    │   ├── __init__.py
    │   └── parser_name.py      # HTML/data parsing utilities
    ├── models/
    │   ├── __init__.py
    │   └── database.py         # SQLAlchemy models
    └── database/
        ├── __init__.py
        └── connection.py       # Database connection management
tests/
├── conftest.py
├── test_*.py                   # Mirror source structure
requirements.txt                # Production dependencies
requirements-dev.txt           # Development dependencies
```

### FastAPI Application Structure
- **Entry point**: `app.py` with FastAPI instance
- **Standard endpoints**: Always include `/health` endpoint
- **Auto-generated Swagger**: Enable by default, maintain documentation
- **Proper response models**: Use Pydantic models for all responses

## Technology Stack & Versions

### Core Dependencies (requirements.txt)
```
fastapi==0.115.0
uvicorn[standard]==0.30.6
pydantic==2.8.2
pydantic-settings==2.3.4
flagsmith>=3.0.0
httpx==0.27.2
tenacity==9.0.0
beautifulsoup4==4.12.3
lxml==5.3.0
python-dotenv==1.0.1
asyncpg==0.29.0
sqlalchemy[asyncio]==2.0.23
alembic==1.13.1
```

### Development Dependencies (requirements-dev.txt)
```
pytest==8.3.2
httpx==0.27.2
ruff==0.6.9
black==24.8.0
isort==5.13.2
pre-commit==3.8.0
```

### Python Version
- **Exactly Python 3.11** - specify in all configurations
- Use `python:3.11-slim` for Docker base images
- Configure GitHub Actions with `python-version: "3.11"`

## Configuration Management (Flagsmith Only)

### Core Principles
- **NO environment variable fallbacks** in code
- **Flagsmith as single source of truth** for all runtime configuration
- **Dual-scope architecture**: MAIN and PROJECT environment keys
- **Direct REST API calls** for reliable flag retrieval
- **Fail fast** if required configuration is missing

### Dual Environment Architecture
- **MAIN_FLAGSMITH_KEY**: Shared/infrastructure settings (database, etc.)
- **PROJECT_FLAGSMITH_KEY**: Project-specific settings (APIs, business logic)

### Configuration Pattern
```python
# settings.py
import os
from typing import Optional

def _get_project_flagsmith_client() -> Optional[str]:
    return os.getenv("PROJECT_FLAGSMITH_KEY")

def _get_main_flagsmith_client() -> Optional[str]:
    return os.getenv("MAIN_FLAGSMITH_KEY")

def _flagsmith_required_from(api_key: Optional[str], *, which: str, key: str):
    if api_key is None:
        raise RuntimeError(f"{which} Flagsmith API key is not configured")

    try:
        import requests
        url = f"https://edge.api.flagsmith.com/api/v1/flags/"
        headers = {"X-Environment-Key": api_key}

        response = requests.get(url, headers=headers)
        response.raise_for_status()
        flags_data = response.json()

        for flag in flags_data:
            if flag.get("feature", {}).get("name") == key:
                if not flag.get("enabled", True):
                    raise RuntimeError(f"Flagsmith flag disabled in {which}: {key}")
                flag_value = flag.get("feature_state_value")
                if flag_value is None or (isinstance(flag_value, str) and flag_value.strip() == ""):
                    raise RuntimeError(f"Empty Flagsmith flag value in {which}: {key}")
                return flag_value

        raise RuntimeError(f"Missing Flagsmith flag in {which}: {key}")
    except Exception as e:
        if "Missing Flagsmith flag" in str(e) or "Empty Flagsmith flag" in str(e) or "disabled" in str(e):
            raise
        raise RuntimeError(f"Failed to get Flagsmith flag {key} from {which}: {str(e)}") from e

class Config:
    @property
    def app_name(self) -> str:
        return "package-name"  # Constant, not configurable

    # Database configuration (MAIN scope)
    @property
    def database_url(self) -> str:
        return str(_flagsmith_required_from(_get_main_flagsmith_client(), which="MAIN", key="database_url"))

    @property
    def database_pool_size(self) -> int:
        val = _flagsmith_required_from(_get_main_flagsmith_client(), which="MAIN", key="database_pool_size")
        return int(val)

    @property
    def database_timeout_seconds(self) -> float:
        val = _flagsmith_required_from(_get_main_flagsmith_client(), which="MAIN", key="database_timeout_seconds")
        return float(val)

    # Project-specific configuration (PROJECT scope)
    @property
    def api_key(self) -> str:
        return str(_flagsmith_required_from(_get_project_flagsmith_client(), which="PROJECT", key="api_key"))

conf = Config()
```

### Required Flagsmith Setup
For every project, configure two Flagsmith environments:

**MAIN Environment** (shared infrastructure):
- `database_url` (string) - Neon PostgreSQL connection string
- `database_pool_size` (number) - Connection pool size (default: 5)
- `database_timeout_seconds` (number) - Query timeout (default: 30)

**PROJECT Environment** (project-specific):
- `api_key` (string) - External service API keys
- Service-specific configuration flags as needed

### Environment Variables
```bash
# .env
MAIN_FLAGSMITH_KEY=your_main_environment_key_here
PROJECT_FLAGSMITH_KEY=your_project_environment_key_here
```

## Database Integration (Neon Cloud)

### Neon PostgreSQL Setup
- **Serverless PostgreSQL**: Use Neon as the cloud database provider
- **Connection strings**: Always from Flagsmith MAIN environment, never hardcoded
- **Async patterns**: Use asyncpg with SQLAlchemy async engine

### Database Connection Pattern
```python
# database/connection.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from ..settings import conf

engine = create_async_engine(
    conf.database_url,
    pool_size=conf.database_pool_size,
    pool_timeout=conf.database_timeout_seconds,
    echo=False  # Set to True for SQL debugging
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

async def get_db_session():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

### SQLAlchemy Models Pattern
```python
# models/database.py
from sqlalchemy.ext.asyncio import AsyncAttrs
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, DateTime, Integer
from datetime import datetime

class Base(AsyncAttrs, DeclarativeBase):
    pass

class ExampleModel(Base):
    __tablename__ = "examples"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

### Alembic Migration Setup
- Initialize with: `alembic init alembic`
- Configure `alembic.ini` to use ConfigCat database URL
- Always create migrations for schema changes
- Use descriptive migration messages

## Code Quality Standards

### Type Hints and Imports
```python
from __future__ import annotations  # Always first import

from typing import Any, Dict, List, Optional  # Explicit typing imports
from dataclasses import dataclass

# Standard library imports
import os
from datetime import datetime

# Third-party imports
import httpx
from fastapi import FastAPI
from sqlalchemy import String

# Local imports
from ..settings import conf
from .models import ExampleModel
```

### Naming Conventions
- **Files**: `snake_case.py`
- **Classes**: `PascalCase`
- **Functions/variables**: `snake_case`
- **Constants**: `UPPER_SNAKE_CASE`
- **Client classes**: `{Service}Client` (e.g., `PoptavejClient`)
- **Config classes**: `{Service}ClientConfig`

### Comments Policy
- **Language**: English only
- **Frequency**: Minimal - code should be self-documenting
- **When to comment**: Complex business logic, API quirks, security considerations
- **Never comment**: Obvious code, type definitions, simple operations

### Code Formatting
```toml
# pyproject.toml
[tool.black]
line-length = 100
target-version = ["py311"]

[tool.isort]
profile = "black"
line_length = 100

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E","F","I","B"]
ignore = []
```

## HTTP Client Patterns

### Client Configuration
```python
@dataclass
class ServiceClientConfig:
    base_url: str = "https://api.example.com"
    user_agent: str = "Mozilla/5.0 (...)"
    timeout_seconds: float = 15.0
    # Optional credentials
    api_key: Optional[str] = None
```

### Client Implementation
```python
class ServiceClient:
    def __init__(
        self,
        config: Optional[ServiceClientConfig] = None,
        *,
        extra_headers: Optional[Dict[str, str]] = None,
    ) -> None:
        self.config = config or ServiceClientConfig()
        self.headers: Dict[str, str] = {
            "User-Agent": self.config.user_agent,
            "Accept": "application/json",
        }
        if extra_headers:
            self.headers.update(extra_headers)

        self._client = httpx.Client(
            base_url=self.config.base_url,
            timeout=self.config.timeout_seconds,
            headers=self.headers,
            follow_redirects=True,
        )

    def close(self) -> None:
        self._client.close()

    @retry(
        retry=retry_if_exception_type(httpx.RequestError),
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=10)
    )
    def get(self, path: str) -> httpx.Response:
        return self._client.get(path)
```

### Async Client Variant
```python
class AsyncServiceClient:
    # Same pattern but with async methods and httpx.AsyncClient
    async def aclose(self) -> None:
        await self._client.aclose()
```

## Testing Strategy - Essential Coverage Only

### Testing Philosophy
- **Basic coverage only**: One test per feature's happy path
- **Critical error cases**: Only essential failure scenarios
- **No comprehensive test suites**: Avoid over-testing
- **Focus areas**: Core functionality, authentication, database operations

### Test Structure
```python
# tests/test_feature_name.py
def test_feature_name_works():
    """Test the happy path of the feature."""
    pass

def test_feature_name_fails_when_invalid():
    """Test critical failure scenario."""
    pass
```

### Database Testing
```python
# conftest.py
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from src.package_name.models.database import Base

@pytest.fixture
async def db_session():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async with AsyncSession(engine) as session:
        yield session

    await engine.dispose()
```

### Test Configuration
```ini
# pytest.ini
[pytest]
pythonpath = src
```

## Documentation Requirements

### README.md Maintenance
- **Always keep current** with project state
- **Update after each feature** addition
- **Include**: Setup, development commands, API endpoints, configuration
- **Structure**: Quickstart, Configuration, API, Development, Docker

### Swagger/OpenAPI Documentation
- **Enable by default** in FastAPI
- **Maintain from day one**: Add descriptions, examples, response models
- **Document all endpoints**: Include request/response schemas
- **Include examples**: Real-world usage examples

### FastAPI Documentation Pattern
```python
@app.get(
    "/endpoint",
    response_model=ResponseModel,
    summary="Brief description",
    description="Detailed description with examples"
)
def endpoint_function():
    """
    Additional endpoint documentation.

    - **param**: Description of parameter
    - **returns**: Description of return value
    """
    pass
```

## Docker & Deployment

### Dockerfile Pattern
```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.11-slim AS base
ENV PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# Build-time configuration
ARG MAIN_FLAGSMITH_KEY
ARG PROJECT_FLAGSMITH_KEY
ENV MAIN_FLAGSMITH_KEY=${MAIN_FLAGSMITH_KEY}
ENV PROJECT_FLAGSMITH_KEY=${PROJECT_FLAGSMITH_KEY}

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy application
COPY src ./src

# Expose port
EXPOSE 8080

# Run application
CMD ["uvicorn", "src.package_name.app:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Environment Handling
- **Build args**: For Flagsmith environment keys (MAIN and PROJECT)
- **Runtime env**: Only essential environment variables
- **No secrets**: Never bake secrets into images

## CI/CD Patterns

### GitHub Actions Workflow
```yaml
name: CI

on:
  push:
    branches: ["**"]
    tags: ["**"]
  pull_request:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt -r requirements-dev.txt

      - name: Lint
        run: |
          ruff check src tests
          black --check src tests
          isort --check-only src tests

      - name: Test
        env:
          PYTHONPATH: "${{ github.workspace }}"
        run: pytest -q

      - name: Build Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          tags: package-name:${{ github.sha }}
          build-args: |
            MAIN_FLAGSMITH_KEY=${{ secrets.MAIN_FLAGSMITH_KEY }}
            PROJECT_FLAGSMITH_KEY=${{ secrets.PROJECT_FLAGSMITH_KEY }}
```

### Quality Gates
1. **Lint**: ruff, black, isort checks
2. **Test**: pytest with essential coverage
3. **Build**: Docker image build verification
4. **Deploy**: Push to GHCR on main/tags

## Makefile Standards

### Standard Targets
```makefile
.PHONY: install dev test lint fmt docker-build docker-run

# Python resolution
PY ?= python
ifneq (,$(wildcard .venv/Scripts/python.exe))
PY := .venv/Scripts/python.exe
else ifneq (,$(wildcard .venv/bin/python))
PY := .venv/bin/python
endif

# Load .env
ifneq (,$(wildcard .env))
include .env
export
endif

install:
	python -m venv .venv
	$(PY) -m pip install -r requirements.txt -r requirements-dev.txt

dev:
	$(PY) -m uvicorn src.package_name.app:app --reload --host 0.0.0.0 --port 8080

test:
	$(PY) -m pytest -q

lint:
	$(PY) -m ruff check src tests

fmt:
	$(PY) -m ruff check --fix src tests && $(PY) -m black src tests && $(PY) -m isort src tests

docker-build:
	docker build -t package-name:local \
		--build-arg MAIN_FLAGSMITH_KEY=$(MAIN_FLAGSMITH_KEY) \
		--build-arg PROJECT_FLAGSMITH_KEY=$(PROJECT_FLAGSMITH_KEY) \
		.

docker-run:
	docker run --rm -p 8080:8080 package-name:local
```

## Development Workflow

### New Feature Development
1. **Create feature branch** from main
2. **Implement feature** following patterns above
3. **Add basic tests** (happy path + critical failure)
4. **Update README** with new functionality
5. **Update Swagger docs** with endpoint descriptions
6. **Run quality checks**: `make lint fmt test`
7. **Create pull request** with clear description

### File Creation Guidelines
- **Minimal files only**: Don't create unnecessary files
- **Follow structure**: Use mandatory directory structure
- **Real data**: Never use mock data or placeholder content
- **Production ready**: No TODOs, proper error handling

### Configuration Management
- **Flagsmith first**: Always check Flagsmith for configuration using dual-environment approach
- **No fallbacks**: Don't provide environment variable fallbacks
- **Fail fast**: Raise clear errors for missing configuration
- **Document keys**: List required Flagsmith flags in README (MAIN and PROJECT environments)

This document serves as the definitive reference for all seed repository development. Follow these patterns consistently to ensure maintainable, scalable, and production-ready applications.
