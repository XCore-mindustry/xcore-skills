---
name: xcore-python-services
description: Standards and testing procedures for Python services (XCore-discord-bot, xcore-metrics-gateway). Use when modifying Python microservices, Redis stream consumers, or Prometheus exporters.
---

# XCore Python Services Guide

## Tooling & Environment
Always use `uv` for dependency management, testing, and linting:
```bash
uv sync
uv run pytest
uv run ruff check .
uv run ruff format .
```

## Redis Streams Patterns
- Use consumer groups (`XREADGROUP`) with proper ACK handling:
  - Read from `xcore:evt:chat:message` or command queues.
  - Process message payload using strict `from_payload()` validation from `xcore_protocol.generated.*`.
  - Acknowledge (`XACK`) only after successful processing.

## Testing Standards
- All tests must use `pytest-asyncio` with `asyncio_mode = "auto"` (configured in `pyproject.toml`).
- Use `fakeredis` for Redis stream unit testing without requiring an external Redis container.
- For MongoDB tests, mock Motor collections or use async fixtures.
