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
    ├── settings.py             # ConfigCat configuration management
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
configcat-client>=9,<10
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

## Configuration Management (ConfigCat Only)

### Core Principles
- **NO environment variable fallbacks** in code
- **ConfigCat as single source of truth** for all runtime configuration
- **`.env` file contains ONLY**: `CONFIGCAT_SDK_KEY`
- **Fail fast** if required configuration is missing

### Configuration Pattern
```python
# settings.py
import os
from typing import Optional
from configcatclient import ConfigCatClient

_configcat_client: Optional[ConfigCatClient] = None

def _get_configcat_client() -> Optional[ConfigCatClient]:
    global _configcat_client
    if _configcat_client is not None:
        return _configcat_client
    sdk_key = os.getenv("CONFIGCAT_SDK_KEY")
    if not sdk_key:
        return None
    return ConfigCatClient.get(sdk_key)

def _cc_required(key: str):
    client = _get_configcat_client()
    if client is None:
        raise RuntimeError("CONFIGCAT_SDK_KEY is not configured")
    value = client.get_value(key, None)
    if value is None or (isinstance(value, str) and value.strip() == ""):
        raise RuntimeError(f"Missing ConfigCat key: {key}")
    return value

class Config:
    @property
    def app_name(self) -> str:
        return "package-name"  # Constant, not configurable

    # Database configuration
    @property
    def database_url(self) -> str:
        return str(_cc_required("database_url"))

    @property
    def database_pool_size(self) -> int:
        return int(_cc_required("database_pool_size"))

    @property
    def database_timeout_seconds(self) -> float:
        return float(_cc_required("database_timeout_seconds"))

conf = Config()
```

### Required ConfigCat Keys
For every project, configure these keys in ConfigCat:
- `database_url` (string) - Neon PostgreSQL connection string
- `database_pool_size` (number) - Connection pool size (default: 5)
- `database_timeout_seconds` (number) - Query timeout (default: 30)
- Service-specific keys as needed

## Database Integration (Neon Cloud)

### Neon PostgreSQL Setup
- **Serverless PostgreSQL**: Use Neon as the cloud database provider
- **Connection strings**: Always from ConfigCat, never hardcoded
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
ARG CONFIGCAT_SDK_KEY
ENV CONFIGCAT_SDK_KEY=${CONFIGCAT_SDK_KEY}

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
- **Build args**: For ConfigCat SDK key
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
            CONFIGCAT_SDK_KEY=${{ secrets.CONFIGCAT_SDK_KEY }}
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
		--build-arg CONFIGCAT_SDK_KEY=$(CONFIGCAT_SDK_KEY) \
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
- **ConfigCat first**: Always check ConfigCat for configuration
- **No fallbacks**: Don't provide environment variable fallbacks
- **Fail fast**: Raise clear errors for missing configuration
- **Document keys**: List required ConfigCat keys in README

This document serves as the definitive reference for all seed repository development. Follow these patterns consistently to ensure maintainable, scalable, and production-ready applications.