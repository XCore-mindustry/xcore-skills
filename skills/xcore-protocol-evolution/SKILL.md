---
name: xcore-protocol-evolution
description: Workflow for modifying, generating, and verifying cross-service protocol contracts in xcore-protocol, XCore-plugin, and XCore-discord-bot. Use whenever adding or updating events, commands, or telemetry models.
---

# XCore Protocol Evolution Workflow

`xcore-protocol` is the canonical Single Source of Truth for communication between Mindustry servers, Discord bots, and telemetry gateways.

## Order of Operations (MANDATORY)

1. **Modify Schema Specs (`xcore-protocol/spec/`)**:
   - Locate the target family in `xcore-protocol/spec/` (`moderation`, `discord`, `maps`, `chat`, `telemetry`).
   - Use strict `camelCase` for all property names.
   - Maintain backward compatibility where required; prefer additive optional fields.

2. **Update Fixtures (`xcore-protocol/fixtures/`)**:
   - Add/update JSON fixtures in `fixtures/<family>/valid/` and `invalid/`.

3. **Run Code Generation**:
   ```bash
   cd /home/ospx/workspace/xcore/xcore-protocol
   uv run python -m generators generate
   ```

4. **Verify Generation & Compatibility Tests**:
   ```bash
   uv run python -m generators check
   uv run pytest
   ./gradlew test
   ```

5. **Update Consumers**:
   - In `XCore-plugin`: Use generated Java records (`org.xcore.protocol.generated.*`).
   - In `XCore-discord-bot`: Use generated Python dataclasses (`xcore_protocol.generated.*`).
   - In `xcore-metrics-gateway`: Use generated telemetry snapshot decoders.
   - **NEVER** redefine protocol DTOs locally in consumer repositories!
