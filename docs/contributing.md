# Contributing

## Prerequisites

- Python 3.13+
- Node.js 18+ and npm
- Docker and Docker Compose
- [Ollama](https://ollama.com) installed, with `llama3.2:3b` and `nomic-embed-text` pulled

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
