# ERP AI Agents Integration

This module provides AI agent capabilities to all ERP services through guarded APIs.

## Integration Points

- `ERP-Platform`: plan/entitlement checks before agent invocation.
- `ERP-Directory`: tenant-aware identity and permission propagation.
- `ERP-Productivity`: copilot in docs/sheets/slides.
- `ERP-Meet`: meeting transcript summarization and action extraction.
- `ERP-Drive`: document classification and policy checks.

## AIDD Enforcement

- Every autonomous action must emit confidence and rationale fields.
- High-risk actions require explicit human approval.
- All decisions are audited with immutable event IDs.
