# MiniMax Provider Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate MiniMax as a new Anthropic Messages transport provider with full support for streaming, tool use, thinking, and model discovery.

**Architecture:** MiniMax exposes an Anthropic-compatible Messages endpoint at `api.minimax.io/anthropic/v1`. The provider extends `AnthropicMessagesTransport` following the Kimi (Moonshot) pattern: Bearer token auth, `anthropic-version` header, and OpenAI-compat model list endpoint.

**Tech Stack:** Python 3.14, httpx, Pydantic, pytest, uv

---

## Task 1: Create Provider Module Structure

**Files:**
- Create: `providers/minimax/__init__.py`

- [ ] **Step 1: Create provider package**

```python
# providers/minimax/__init__.py
"""MiniMax provider exports."""

from __future__ import annotations

__all__: list[str] = []
```

- [ ] **Step 2: Verify package created**

Run: `ls providers/minimax/`
Expected: `__init__.py`

- [ ] **Step 3: Commit skeleton**

```bash
git add providers/minimax/__init__.py
git commit -m "feat(minimax): add provider package skeleton"
```

---

## Task 2: Add MINIMAX_DEFAULT_BASE to Provider Catalog

**Files:**
- Modify: `config/provider_catalog.py:34` (after `ZAI_DEFAULT_BASE`)

- [ ] **Step 1: Add MINIMAX_DEFAULT_BASE constant**

Open `config/provider_catalog.py` and add after line 34 (`ZAI_DEFAULT_BASE`):

```python
# MiniMax Anthropic-compatible Messages API (POST …/anthropic/v1/messages).
MINIMAX_DEFAULT_BASE = "https://api.minimax.io/anthropic/v1"
```

- [ ] **Step 2: Verify constant defined**

Run: `uv run python -c "from config.provider_catalog import MINIMAX_DEFAULT_BASE; print(MINIMAX_DEFAULT_BASE)"`
Expected: `https://api.minimax.io/anthropic/v1`

- [ ] **Step 3: Commit constant**

```bash
git add config/provider_catalog.py
git commit -m "feat(catalog): add MINIMAX_DEFAULT_BASE constant"
```

---

## Task 3: Export MINIMAX_DEFAULT_BASE from defaults module

**Files:**
- Modify: `providers/defaults.py:4-21` (imports)
- Modify: `providers/defaults.py:23-41` (__all__)

- [ ] **Step 1: Add MINIMAX_DEFAULT_BASE to imports**

Open `providers/defaults.py` and add `MINIMAX_DEFAULT_BASE` to the import list after `MISTRAL_DEFAULT_BASE`:

```python
from config.provider_catalog import (
    CEREBRAS_DEFAULT_BASE,
    CODESTRAL_DEFAULT_BASE,
    DEEPSEEK_ANTHROPIC_DEFAULT_BASE,
    DEEPSEEK_DEFAULT_BASE,
    GEMINI_DEFAULT_BASE,
    GROQ_DEFAULT_BASE,
    KIMI_DEFAULT_BASE,
    LLAMACPP_DEFAULT_BASE,
    LMSTUDIO_DEFAULT_BASE,
    MINIMAX_DEFAULT_BASE,  # <-- ADD THIS LINE
    MISTRAL_DEFAULT_BASE,
    NVIDIA_NIM_DEFAULT_BASE,
    OLLAMA_DEFAULT_BASE,
    OPENCODE_DEFAULT_BASE,
    OPENCODE_GO_DEFAULT_BASE,
    OPENROUTER_DEFAULT_BASE,
    WAFER_DEFAULT_BASE,
    ZAI_DEFAULT_BASE,
)
```

- [ ] **Step 2: Add MINIMAX_DEFAULT_BASE to __all__**

Add `"MINIMAX_DEFAULT_BASE"` to the `__all__` tuple after `"MISTRAL_DEFAULT_BASE"`:

```python
__all__ = (
    "CEREBRAS_DEFAULT_BASE",
    "CODESTRAL_DEFAULT_BASE",
    "DEEPSEEK_ANTHROPIC_DEFAULT_BASE",
    "DEEPSEEK_DEFAULT_BASE",
    "GEMINI_DEFAULT_BASE",
    "GROQ_DEFAULT_BASE",
    "KIMI_DEFAULT_BASE",
    "LLAMACPP_DEFAULT_BASE",
    "LMSTUDIO_DEFAULT_BASE",
    "MINIMAX_DEFAULT_BASE",  # <-- ADD THIS LINE
    "MISTRAL_DEFAULT_BASE",
    "NVIDIA_NIM_DEFAULT_BASE",
    "OLLAMA_DEFAULT_BASE",
    "OPENCODE_DEFAULT_BASE",
    "OPENCODE_GO_DEFAULT_BASE",
    "OPENROUTER_DEFAULT_BASE",
    "WAFER_DEFAULT_BASE",
    "ZAI_DEFAULT_BASE",
)
```

- [ ] **Step 3: Verify export works**

Run: `uv run python -c "from providers.defaults import MINIMAX_DEFAULT_BASE; print(MINIMAX_DEFAULT_BASE)"`
Expected: `https://api.minimax.io/anthropic/v1`

- [ ] **Step 4: Commit export**

```bash
git add providers/defaults.py
git commit -m "feat(defaults): export MINIMAX_DEFAULT_BASE"
```

---

## Task 4: Add minimax_api_key to Settings

**Files:**
- Modify: `config/settings.py:106` (after `fireworks_api_key`)

- [ ] **Step 1: Add minimax_api_key field**

Open `config/settings.py` and add after line 106 (`fireworks_api_key`):

```python
# ==================== MiniMax Config ====================
minimax_api_key: str = Field(default="", validation_alias="MINIMAX_API_KEY")
```

- [ ] **Step 2: Verify field loads from env**

Run: `uv run python -c "from config.settings import Settings; import os; os.environ['MINIMAX_API_KEY']='test123'; s=Settings(); print(s.minimax_api_key)"`
Expected: `test123`

- [ ] **Step 3: Commit settings field**

```bash
git add config/settings.py
git commit -m "feat(settings): add minimax_api_key field"
```

---

## Task 5: Add minimax_proxy to Settings

**Files:**
- Modify: `config/settings.py:173` (after `fireworks_proxy`)

- [ ] **Step 1: Add minimax_proxy field**

Open `config/settings.py` and add after line 173 (`fireworks_proxy`):

```python
minimax_proxy: str = Field(default="", validation_alias="MINIMAX_PROXY")
```

- [ ] **Step 2: Verify field loads from env**

Run: `uv run python -c "from config.settings import Settings; import os; os.environ['MINIMAX_PROXY']='http://proxy:8080'; s=Settings(); print(s.minimax_proxy)"`
Expected: `http://proxy:8080`

- [ ] **Step 3: Commit proxy field**

```bash
git add config/settings.py
git commit -m "feat(settings): add minimax_proxy field"
```

---

## Task 6: Add MiniMax ProviderDescriptor to Catalog

**Files:**
- Modify: `config/provider_catalog.py:199-215` (after `zai` descriptor)

- [ ] **Step 1: Add minimax ProviderDescriptor**

Open `config/provider_catalog.py` and add after the `"zai"` descriptor (around line 215), before `"lmstudio"`:

```python
"minimax": ProviderDescriptor(
    provider_id="minimax",
    transport_type="anthropic_messages",
    credential_env="MINIMAX_API_KEY",
    credential_url="https://platform.minimax.io/user-center/payment/token-plan",
    credential_attr="minimax_api_key",
    default_base_url=MINIMAX_DEFAULT_BASE,
    proxy_attr="minimax_proxy",
    capabilities=(
        "chat",
        "streaming",
        "tools",
        "thinking",
        "native_anthropic",
        "rate_limit",
    ),
),
```

- [ ] **Step 2: Verify descriptor registered**

Run: `uv run python -c "from config.provider_catalog import PROVIDER_CATALOG; print('minimax' in PROVIDER_CATALOG); print(PROVIDER_CATALOG['minimax'].transport_type)"`
Expected: `True` then `anthropic_messages`

- [ ] **Step 3: Commit descriptor**

```bash
git add config/provider_catalog.py
git commit -m "feat(catalog): add minimax ProviderDescriptor"
```

---

## Task 7: Create MiniMax Request Builder

**Files:**
- Create: `providers/minimax/request.py`

- [ ] **Step 1: Create request builder module**

```python
# providers/minimax/request.py
"""Native Anthropic Messages request builder for MiniMax."""

from __future__ import annotations

from typing import Any

from loguru import logger

from config.constants import ANTHROPIC_DEFAULT_MAX_OUTPUT_TOKENS
from core.anthropic.native_messages_request import (
    build_base_native_anthropic_request_body,
)
from providers.exceptions import InvalidRequestError


def build_request_body(request_data: Any, *, thinking_enabled: bool) -> dict:
    """Build JSON for MiniMax Anthropic-compat ``POST …/messages``."""
    logger.debug(
        "MINIMAX_REQUEST: native build model={} msgs={}",
        getattr(request_data, "model", "?"),
        len(getattr(request_data, "messages", [])),
    )

    body = build_base_native_anthropic_request_body(
        request_data,
        default_max_tokens=ANTHROPIC_DEFAULT_MAX_OUTPUT_TOKENS,
        thinking_enabled=thinking_enabled,
    )
    extra = getattr(request_data, "extra_body", None)
    if extra:
        raise InvalidRequestError(
            "MiniMax native Messages API does not support extra_body on requests."
        )
    body["stream"] = True

    logger.debug(
        "MINIMAX_REQUEST: build done model={} msgs={} tools={}",
        body.get("model"),
        len(body.get("messages", [])),
        len(body.get("tools", [])),
    )
    return body
```

- [ ] **Step 2: Verify module imports**

Run: `uv run python -c "from providers.minimax.request import build_request_body; print('OK')"`
Expected: `OK`

- [ ] **Step 3: Commit request builder**

```bash
git add providers/minimax/request.py
git commit -m "feat(minimax): add request builder module"
```

---

## Task 8: Create MiniMax Client Provider

**Files:**
- Create: `providers/minimax/client.py`

- [ ] **Step 1: Create MiniMaxProvider class**

```python
# providers/minimax/client.py
"""MiniMax provider using native Anthropic-compatible Messages."""

from __future__ import annotations

from typing import Any

import httpx

from providers.anthropic_messages import AnthropicMessagesTransport
from providers.base import ProviderConfig
from providers.defaults import MINIMAX_DEFAULT_BASE

from .request import build_request_body

_MINIMAX_OPENAI_MODELS_URL = "https://api.minimax.io/v1/models"
_ANTHROPIC_VERSION = "2023-06-01"


class MiniMaxProvider(AnthropicMessagesTransport):
    """MiniMax provider using Anthropic-compatible Messages at api.minimax.io/anthropic/v1."""

    def __init__(self, config: ProviderConfig):
        super().__init__(
            config,
            provider_name="MINIMAX",
            default_base_url=MINIMAX_DEFAULT_BASE,
        )

    def _build_request_body(
        self, request: Any, thinking_enabled: bool | None = None
    ) -> dict:
        return build_request_body(
            request,
            thinking_enabled=self._is_thinking_enabled(request, thinking_enabled),
        )

    def _request_headers(self) -> dict[str, str]:
        return {
            "Accept": "text/event-stream",
            "Authorization": f"Bearer {self._api_key}",
            "Content-Type": "application/json",
            "anthropic-version": _ANTHROPIC_VERSION,
        }

    async def _send_model_list_request(self) -> httpx.Response:
        """Models are listed from the OpenAI-compat root, not ``/anthropic/v1``."""
        return await self._client.get(
            _MINIMAX_OPENAI_MODELS_URL,
            headers=self._model_list_headers(),
        )

    def _model_list_headers(self) -> dict[str, str]:
        return {"Authorization": f"Bearer {self._api_key}"}
```

- [ ] **Step 2: Verify class imports**

Run: `uv run python -c "from providers.minimax.client import MiniMaxProvider; print('OK')"`
Expected: `OK`

- [ ] **Step 3: Commit client**

```bash
git add providers/minimax/client.py
git commit -m "feat(minimax): add MiniMaxProvider client class"
```

---

## Task 9: Update Provider Package Exports

**Files:**
- Modify: `providers/minimax/__init__.py`

- [ ] **Step 1: Update __init__.py with exports**

Replace the contents of `providers/minimax/__init__.py` with:

```python
"""MiniMax provider exports."""

from providers.defaults import MINIMAX_DEFAULT_BASE

from .client import MiniMaxProvider

__all__ = [
    "MINIMAX_DEFAULT_BASE",
    "MiniMaxProvider",
]
```

- [ ] **Step 2: Verify exports**

Run: `uv run python -c "from providers.minimax import MiniMaxProvider, MINIMAX_DEFAULT_BASE; print(MINIMAX_DEFAULT_BASE)"`
Expected: `https://api.minimax.io/anthropic/v1`

- [ ] **Step 3: Commit exports**

```bash
git add providers/minimax/__init__.py
git commit -m "feat(minimax): export MiniMaxProvider and MINIMAX_DEFAULT_BASE"
```

---

## Task 10: Add MiniMax Factory to Registry

**Files:**
- Modify: `providers/registry.py:109-112` (after `_create_zai`)
- Modify: `providers/registry.py:153` (in PROVIDER_FACTORIES dict)

- [ ] **Step 1: Add factory function**

Open `providers/registry.py` and add after `_create_zai` (around line 112):

```python
def _create_minimax(config: ProviderConfig, _settings: Settings) -> BaseProvider:
    from providers.minimax import MiniMaxProvider

    return MiniMaxProvider(config)
```

- [ ] **Step 2: Add to PROVIDER_FACTORIES dict**

Add `"minimax": _create_minimax,` to the `PROVIDER_FACTORIES` dict after `"zai": _create_zai,` (around line 153):

```python
PROVIDER_FACTORIES: dict[str, ProviderFactory] = {
    "nvidia_nim": _create_nvidia_nim,
    "open_router": _create_open_router,
    "gemini": _create_gemini,
    "deepseek": _create_deepseek,
    "mistral": _create_mistral,
    "mistral_codestral": _create_mistral_codestral,
    "opencode": _create_opencode,
    "opencode_go": _create_opencode_go,
    "wafer": _create_wafer,
    "kimi": _create_kimi,
    "cerebras": _create_cerebras,
    "groq": _create_groq,
    "fireworks": _create_fireworks,
    "zai": _create_zai,
    "minimax": _create_minimax,  # <-- ADD THIS LINE
    "lmstudio": _create_lmstudio,
    "llamacpp": _create_llamacpp,
    "ollama": _create_ollama,
}
```

- [ ] **Step 3: Verify factory registered**

Run: `uv run python -c "from providers.registry import PROVIDER_FACTORIES; print('minimax' in PROVIDER_FACTORIES)"`
Expected: `True`

- [ ] **Step 4: Commit registry**

```bash
git add providers/registry.py
git commit -m "feat(registry): add minimax factory function"
```

---

## Task 11: Update .env.example with MiniMax Configuration

**Files:**
- Modify: `.env.example:38` (after FIREWORKS_API_KEY)
- Modify: `.env.example:121` (after FIREWORKS_PROXY)
- Modify: `.env.example:92` (after FCC_SMOKE_MODEL_CEREBRAS)
- Modify: `.env.example:67` (update valid providers comment)

- [ ] **Step 1: Add MINIMAX_API_KEY section**

Open `.env.example` and add after line 38 (`FIREWORKS_API_KEY=""`):

```bash
# MiniMax Config (Anthropic-compatible Messages at api.minimax.io/anthropic/v1)
MINIMAX_API_KEY=""
```

- [ ] **Step 2: Add MINIMAX_PROXY**

Add after line 121 (`FIREWORKS_PROXY=""`):

```bash
MINIMAX_PROXY=""
```

- [ ] **Step 3: Add FCC_SMOKE_MODEL_MINIMAX**

Add after line 92 (`FCC_SMOKE_MODEL_CEREBRAS=`):

```bash
FCC_SMOKE_MODEL_MINIMAX=
```

- [ ] **Step 4: Update valid providers comment**

Update the comment on line 67 to include `minimax`:

```bash
# Valid providers: "nvidia_nim" | "open_router" | "gemini" | "deepseek" | "mistral" | "mistral_codestral" | "opencode" | "opencode_go" | "wafer" | "kimi" | "cerebras" | "groq" | "fireworks" | "zai" | "minimax" | "lmstudio" | "llamacpp" | "ollama"
```

- [ ] **Step 5: Verify .env.example syntax**

Run: `uv run python -c "import dotenv; dotenv.dotenv_values('.env.example'); print('OK')"`
Expected: `OK`

- [ ] **Step 6: Commit .env.example**

```bash
git add .env.example
git commit -m "docs(env): add MiniMax configuration to .env.example"
```

---

## Task 12: Create Test File for MiniMax Provider

**Files:**
- Create: `tests/providers/test_minimax.py`

- [ ] **Step 1: Create test file with all test cases**

```python
# tests/providers/test_minimax.py
"""Tests for MiniMax native Anthropic Messages provider."""

from contextlib import asynccontextmanager
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from api.models.anthropic import Message, MessagesRequest
from config.constants import ANTHROPIC_DEFAULT_MAX_OUTPUT_TOKENS
from providers.base import ProviderConfig
from providers.defaults import MINIMAX_DEFAULT_BASE
from providers.exceptions import InvalidRequestError
from providers.minimax import MiniMaxProvider


@pytest.fixture
def minimax_config():
    return ProviderConfig(
        api_key="test_minimax_key",
        base_url=MINIMAX_DEFAULT_BASE,
        rate_limit=10,
        rate_window=60,
        enable_thinking=True,
    )


@pytest.fixture(autouse=True)
def mock_rate_limiter():
    @asynccontextmanager
    async def _slot():
        yield

    with patch("providers.anthropic_messages.GlobalRateLimiter") as mock:
        instance = mock.get_scoped_instance.return_value

        async def _passthrough(fn, *args, **kwargs):
            return await fn(*args, **kwargs)

        instance.execute_with_retry = AsyncMock(side_effect=_passthrough)
        instance.concurrency_slot.side_effect = _slot
        yield instance


@pytest.fixture
def minimax_provider(minimax_config):
    return MiniMaxProvider(minimax_config)


def test_init(minimax_config):
    with patch("httpx.AsyncClient") as mock_client:
        provider = MiniMaxProvider(minimax_config)
    assert provider._api_key == "test_minimax_key"
    assert provider._base_url == MINIMAX_DEFAULT_BASE
    assert mock_client.called


def test_request_headers(minimax_provider):
    h = minimax_provider._request_headers()
    assert h["Authorization"] == "Bearer test_minimax_key"
    assert h["anthropic-version"] == "2023-06-01"


def test_build_request_body_native(minimax_provider):
    request = MessagesRequest(
        model="MiniMax-M3",
        max_tokens=50,
        messages=[Message(role="user", content="hi")],
    )
    body = minimax_provider._build_request_body(request)
    assert body["model"] == "MiniMax-M3"
    assert body["stream"] is True
    assert body["messages"][0]["role"] == "user"


def test_build_request_body_default_max_tokens(minimax_provider):
    request = MessagesRequest(
        model="m",
        messages=[Message(role="user", content="x")],
    )
    body = minimax_provider._build_request_body(request)
    assert body["max_tokens"] == ANTHROPIC_DEFAULT_MAX_OUTPUT_TOKENS


def test_build_request_body_rejects_extra_body(minimax_provider):
    request = MessagesRequest.model_validate(
        {
            "model": "m",
            "messages": [{"role": "user", "content": "x"}],
            "extra_body": {"x": 1},
        }
    )
    with pytest.raises(InvalidRequestError, match="does not support extra_body"):
        minimax_provider._build_request_body(request)


@pytest.mark.asyncio
async def test_model_list_uses_minimax_openai_url(minimax_provider):
    called: dict[str, str] = {}

    async def fake_get(url: str, **_k):
        called["url"] = url
        mock_resp = MagicMock()
        mock_resp.raise_for_status = lambda: None
        mock_resp.json = lambda: {"data": [{"id": "MiniMax-M3"}]}
        mock_resp.aclose = AsyncMock()
        return mock_resp

    minimax_provider._client.get = fake_get

    await minimax_provider.list_model_infos()

    assert called["url"] == "https://api.minimax.io/v1/models"


@pytest.mark.asyncio
async def test_cleanup_aclose(minimax_provider):
    minimax_provider._client = AsyncMock()

    await minimax_provider.cleanup()

    minimax_provider._client.aclose.assert_awaited_once()
```

- [ ] **Step 2: Verify test file created**

Run: `ls tests/providers/test_minimax.py`
Expected: File exists

- [ ] **Step 3: Commit test file**

```bash
git add tests/providers/test_minimax.py
git commit -m "test(minimax): add comprehensive test suite"
```

---

## Task 13: Run Tests and Verify

- [ ] **Step 1: Run MiniMax tests**

Run: `uv run pytest tests/providers/test_minimax.py -v`
Expected: All 7 tests PASS

- [ ] **Step 2: Run full test suite**

Run: `uv run pytest`
Expected: All tests PASS (no regressions)

- [ ] **Step 3: Commit test results (if failures fixed)**

If any tests failed and were fixed:
```bash
git add -A
git commit -m "fix(minimax): resolve test failures"
```

---

## Task 14: Run Code Quality Checks

- [ ] **Step 1: Format code**

Run: `uv run ruff format`
Expected: Files formatted (exit code 0)

- [ ] **Step 2: Check formatting**

Run: `uv run ruff format --check`
Expected: All checks PASS (exit code 0)

- [ ] **Step 3: Lint code**

Run: `uv run ruff check`
Expected: No issues (exit code 0)

- [ ] **Step 4: Type check**

Run: `uv run ty check`
Expected: No type errors (exit code 0)

- [ ] **Step 5: Commit any formatting/lint fixes**

If any fixes applied:
```bash
git add -A
git commit -m "style(minimax): apply ruff formatting and lint fixes"
```

---

## Task 15: Bump Version and Update Lockfile

**Files:**
- Modify: `pyproject.toml` (version field)

- [ ] **Step 1: Determine version bump type**

This is a **MINOR** bump (new provider = new capability, backward-compatible).

- [ ] **Step 2: Read current version**

Run: `uv run python -c "import tomllib; print(tomllib.load(open('pyproject.toml', 'rb'))['project']['version'])"`
Expected: Current version (e.g., `1.2.38`)

- [ ] **Step 3: Bump MINOR version**

If current version is `1.2.38`, new version is `1.3.0`.

Edit `pyproject.toml` and update the `version` field:
```toml
[project]
version = "1.3.0"
```

- [ ] **Step 4: Update lockfile**

Run: `uv lock`
Expected: Lockfile updated (exit code 0)

- [ ] **Step 5: Verify version updated**

Run: `uv run python -c "import tomllib; print(tomllib.load(open('pyproject.toml', 'rb'))['project']['version'])"`
Expected: `1.3.0`

- [ ] **Step 6: Commit version bump**

```bash
git add pyproject.toml uv.lock
git commit -m "chore: bump version to 1.3.0 for MiniMax provider integration

Add MiniMax as new Anthropic Messages transport provider with support for
streaming, tool use, extended thinking, and model discovery via OpenAI-compat
endpoint. Includes full test coverage and configuration support.

BREAKING CHANGE: None (backward-compatible feature addition)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 16: Final Verification and Smoke Test

- [ ] **Step 1: Verify all CI checks pass locally**

Run:
```bash
uv run ruff format --check
uv run ruff check
uv run ty check
uv run pytest
```
Expected: All PASS

- [ ] **Step 2: Test MiniMax provider instantiation**

Run:
```bash
uv run python -c "
import os
os.environ['MINIMAX_API_KEY'] = 'test_key'
from config.settings import Settings
from providers.registry import create_provider
settings = Settings()
provider = create_provider('minimax', settings)
print(f'Provider: {type(provider).__name__}')
print(f'Base URL: {provider._base_url}')
print(f'API Key: {provider._api_key[:10]}...')
"
```
Expected: Provider instantiated successfully with correct attributes

- [ ] **Step 3: Verify MiniMax in supported providers list**

Run:
```bash
uv run python -c "
from config.provider_ids import SUPPORTED_PROVIDER_IDS
print('minimax' in SUPPORTED_PROVIDER_IDS)
print(f'Total providers: {len(SUPPORTED_PROVIDER_IDS)}')
"
```
Expected: `True`, total providers = 18 (was 17)

- [ ] **Step 4: Verify model format validation accepts minimax prefix**

Run:
```bash
uv run python -c "
from config.settings import Settings
import os
os.environ['MODEL'] = 'minimax/MiniMax-M3'
settings = Settings()
print(f'Model: {settings.model}')
print(f'Provider type: {settings.provider_type}')
print(f'Model name: {settings.model_name}')
"
```
Expected: Model parsed correctly

- [ ] **Step 5: Document residual risks**

Create a brief summary:

**Residual Risks:**
- Model list endpoint format (`/v1/models`) not verified live — will confirm on first API call
- M2.x thinking always-on behavior is provider-side, not controlled by proxy
- Token Plan rate limits (5h rolling + weekly) require user to configure `PROVIDER_RATE_LIMIT` appropriately

**None of these block merge** — they are operational considerations for users.

---

## Task 17: Create Summary Commit Message

- [ ] **Step 1: Review all commits**

Run: `git log --oneline -20`
Expected: Series of focused commits for MiniMax integration

- [ ] **Step 2: Verify no uncommitted changes**

Run: `git status`
Expected: `nothing to commit, working tree clean`

- [ ] **Step 3: Push to remote (if applicable)**

```bash
git push origin main
```

---

## Summary

**Files Created:** 3
- `providers/minimax/__init__.py`
- `providers/minimax/client.py`
- `providers/minimax/request.py`
- `tests/providers/test_minimax.py`

**Files Modified:** 6
- `config/provider_catalog.py`
- `config/settings.py`
- `providers/defaults.py`
- `providers/registry.py`
- `.env.example`
- `pyproject.toml`

**Total Commits:** 17 (one per task)

**Version Bump:** MINOR (1.x.x → 1.(x+1).0)

**Test Coverage:** 7 unit tests covering init, headers, request building, model list, cleanup
