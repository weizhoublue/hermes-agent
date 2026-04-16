# Design: delegation.api_key_env Support

**Date:** 2026-04-16  
**Status:** Approved

## Problem

`delegation` in `config.yaml` supports `api_key` (literal key) for custom OpenAI-compatible endpoints (`base_url` path), but does not support `api_key_env` (an env var name). This forces users to embed secrets directly in `config.yaml` rather than referencing them via environment variable names — inconsistent with how `smart_model_routing` and `trajectory_compressor` already handle this pattern.

## Goal

When `delegation.base_url` is configured, resolve the API key using:

```
delegation.api_key  →  os.getenv(delegation.api_key_env)  →  OPENAI_API_KEY  →  error
```

## Out of Scope

- `fallback_model` `api_key_env` gap (not requested)
- Refactoring shared helper (over-engineered for this scope)

## Changes

### `tools/delegate_tool.py` — `_resolve_delegation_credentials()`

Add reading of `api_key_env` from config and insert it into the key resolution chain:

```python
configured_api_key = str(cfg.get("api_key") or "").strip() or None
configured_api_key_env = str(cfg.get("api_key_env") or "").strip() or None

if configured_base_url:
    api_key = (
        configured_api_key
        or (os.getenv(configured_api_key_env) if configured_api_key_env else None)
        or os.getenv("OPENAI_API_KEY", "").strip()
    )
    if not api_key:
        raise ValueError(
            "Delegation base_url is configured but no API key was found. "
            "Set delegation.api_key, delegation.api_key_env, or OPENAI_API_KEY."
        )
```

### `cli-config.yaml.example` — `delegation:` block

Add `api_key_env` documentation alongside `api_key` in the delegation comment block.

### `tests/tools/test_delegate.py` — two new test cases

1. `test_direct_endpoint_uses_api_key_env` — verifies env var is resolved when `api_key` is absent.
2. `test_api_key_takes_priority_over_api_key_env` — verifies `api_key` wins when both are set.

## Pattern Reference

`agent/smart_model_routing.py:142-144` is the canonical existing implementation of this pattern:

```python
api_key_env = str(route.get("api_key_env") or "").strip()
if api_key_env:
    explicit_api_key = os.getenv(api_key_env) or None
```
