# Contributing

## Prerequisites

- Python 3.13+
- Node.js 18+ and npm
- Docker
- An AWS account with the AWS CLI configured, and an [OpenRouter](https://openrouter.ai) API key

## Setup

See the [Deployment Guide](deployment.md) for full development environment setup.

## Project Structure

```
subtext-ai/
├── frontend/       # React 19 + TypeScript + Tailwind CSS
├── backend/        # FastAPI + Python 3.13
├── docker/         # Dockerfiles and docker-compose
├── tests/          # Pytest test suite
├── docs/           # Documentation
└── .github/        # GitHub Actions CI/CD
```

## Code Quality

```bash
# Lint and format (backend)
ruff check .
ruff format .

# Run tests
pytest

# Pre-commit hooks
pre-commit install
pre-commit run --all-files
```

## License

Distributed under the MIT License.
