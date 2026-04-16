# delegation.api_key_env Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow `delegation.api_key_env` in `config.yaml` so users can reference an env var name instead of embedding a literal API key when using `delegation.base_url`.

**Architecture:** One-line change in `_resolve_delegation_credentials()` reads `api_key_env` from the delegation config dict and resolves it via `os.getenv()`. Priority chain: `api_key` (literal) → `os.getenv(api_key_env)` → `OPENAI_API_KEY` → error. No new abstractions needed.

**Tech Stack:** Python stdlib (`os.getenv`), existing `tools/delegate_tool.py` patterns.

---

## File Map

| File | Change |
|------|--------|
| `tools/delegate_tool.py` | Modify `_resolve_delegation_credentials()` lines 866–877 |
| `cli-config.yaml.example` | Extend `delegation:` comment block (lines 748–754) |
| `tests/tools/test_delegate.py` | Add 2 test cases to `TestResolveDelegationCredentials` class |

---

### Task 1: Add failing tests for `api_key_env`

**Files:**
- Modify: `tests/tools/test_delegate.py` (after `test_direct_endpoint_does_not_fall_back_to_openrouter_api_key_env`, around line 657)

- [ ] **Step 1: Add two failing test cases**

Open `tests/tools/test_delegate.py`. After the `test_direct_endpoint_does_not_fall_back_to_openrouter_api_key_env` method (ends around line 657), add these two methods inside the same `TestResolveDelegationCredentials` class:

```python
def test_direct_endpoint_uses_api_key_env(self):
    """api_key_env is resolved from the environment when api_key is absent."""
    parent = _make_mock_parent(depth=0)
    cfg = {
        "model": "my-model",
        "base_url": "https://my-endpoint.example.com/v1",
        "api_key_env": "MY_DELEGATION_KEY",
    }
    with patch.dict(os.environ, {"MY_DELEGATION_KEY": "env-secret"}, clear=False):
        creds = _resolve_delegation_credentials(cfg, parent)
    self.assertEqual(creds["api_key"], "env-secret")
    self.assertEqual(creds["provider"], "custom")
    self.assertEqual(creds["base_url"], "https://my-endpoint.example.com/v1")

def test_api_key_takes_priority_over_api_key_env(self):
    """Literal api_key wins when both api_key and api_key_env are set."""
    parent = _make_mock_parent(depth=0)
    cfg = {
        "model": "my-model",
        "base_url": "https://my-endpoint.example.com/v1",
        "api_key": "explicit-key",
        "api_key_env": "MY_DELEGATION_KEY",
    }
    with patch.dict(os.environ, {"MY_DELEGATION_KEY": "env-secret"}, clear=False):
        creds = _resolve_delegation_credentials(cfg, parent)
    self.assertEqual(creds["api_key"], "explicit-key")
```

- [ ] **Step 2: Run the new tests to confirm they fail**

```bash
cd hermes-agent && source venv/bin/activate
python -m pytest tests/tools/test_delegate.py::TestResolveDelegationCredentials::test_direct_endpoint_uses_api_key_env tests/tools/test_delegate.py::TestResolveDelegationCredentials::test_api_key_takes_priority_over_api_key_env -v
```

Expected: both FAIL — `test_direct_endpoint_uses_api_key_env` raises `ValueError` ("no API key was found") because `api_key_env` is ignored. `test_api_key_takes_priority_over_api_key_env` may pass (coincidence — `api_key` already works), that's fine.

---

### Task 2: Implement `api_key_env` in `_resolve_delegation_credentials()`

**Files:**
- Modify: `tools/delegate_tool.py:863–877`

- [ ] **Step 1: Edit `_resolve_delegation_credentials()`**

In `tools/delegate_tool.py`, find `_resolve_delegation_credentials()`. Replace lines 863–877:

**Before:**
```python
    configured_model = str(cfg.get("model") or "").strip() or None
    configured_provider = str(cfg.get("provider") or "").strip() or None
    configured_base_url = str(cfg.get("base_url") or "").strip() or None
    configured_api_key = str(cfg.get("api_key") or "").strip() or None

    if configured_base_url:
        api_key = (
            configured_api_key
            or os.getenv("OPENAI_API_KEY", "").strip()
        )
        if not api_key:
            raise ValueError(
                "Delegation base_url is configured but no API key was found. "
                "Set delegation.api_key or OPENAI_API_KEY."
            )
```

**After:**
```python
    configured_model = str(cfg.get("model") or "").strip() or None
    configured_provider = str(cfg.get("provider") or "").strip() or None
    configured_base_url = str(cfg.get("base_url") or "").strip() or None
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

- [ ] **Step 2: Run the new tests — expect both to pass**

```bash
python -m pytest tests/tools/test_delegate.py::TestResolveDelegationCredentials::test_direct_endpoint_uses_api_key_env tests/tools/test_delegate.py::TestResolveDelegationCredentials::test_api_key_takes_priority_over_api_key_env -v
```

Expected: both PASS.

- [ ] **Step 3: Run the full delegate test class to check for regressions**

```bash
python -m pytest tests/tools/test_delegate.py::TestResolveDelegationCredentials -v
```

Expected: all existing tests still pass.

- [ ] **Step 4: Commit**

```bash
git add tools/delegate_tool.py tests/tools/test_delegate.py
git commit -m "feat: support api_key_env in delegation config

Adds delegation.api_key_env so users can reference an env var name
instead of embedding a literal key when using delegation.base_url.
Priority: api_key > api_key_env > OPENAI_API_KEY."
```

---

### Task 3: Document `api_key_env` in `cli-config.yaml.example`

**Files:**
- Modify: `cli-config.yaml.example:748–754`

- [ ] **Step 1: Update the delegation comment block**

In `cli-config.yaml.example`, replace the `delegation:` block (lines 748–754):

**Before:**
```yaml
delegation:
  max_iterations: 50                          # Max tool-calling turns per child (default: 50)
  default_toolsets: ["terminal", "file", "web"]  # Default toolsets for subagents
  # model: "google/gemini-3-flash-preview"    # Override model for subagents (empty = inherit parent)
  # provider: "openrouter"                    # Override provider for subagents (empty = inherit parent)
  #                                           # Resolves full credentials (base_url, api_key) automatically.
  #                                           # Supported: openrouter, nous, zai, kimi-coding, minimax
```

**After:**
```yaml
delegation:
  max_iterations: 50                          # Max tool-calling turns per child (default: 50)
  default_toolsets: ["terminal", "file", "web"]  # Default toolsets for subagents
  # model: "google/gemini-3-flash-preview"    # Override model for subagents (empty = inherit parent)
  # provider: "openrouter"                    # Override provider for subagents (empty = inherit parent)
  #                                           # Resolves full credentials (base_url, api_key) automatically.
  #                                           # Supported: openrouter, nous, zai, kimi-coding, minimax
  #
  # For custom OpenAI-compatible endpoints:
  # base_url: "https://my-endpoint.example.com/v1"
  # api_key: "sk-..."          # Literal key (highest priority)
  # api_key_env: "MY_API_KEY"  # Env var name (used when api_key is absent)
  #                            # Falls back to OPENAI_API_KEY if neither is set
```

- [ ] **Step 2: Commit**

```bash
git add cli-config.yaml.example
git commit -m "docs: document delegation.api_key_env in cli-config.yaml.example"
```

---

### Task 4: Final verification

- [ ] **Step 1: Run the full delegate test file**

```bash
python -m pytest tests/tools/test_delegate.py -v
```

Expected: all tests pass, no regressions.

- [ ] **Step 2: Run the broader tools test suite**

```bash
python -m pytest tests/tools/ -q
```

Expected: all pass.
