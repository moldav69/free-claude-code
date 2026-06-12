# MiniMax Provider Integration — Design Spec

**Data**: 2026-06-12
**Autore**: Architettura sistema
**Stato**: Approved — pronto per implementazione
**Riferimento**: https://platform.minimax.io/docs/token-plan/intro

---

## 1. Obiettivo

Integrare MiniMax come nuovo provider LLM nel proxy free-claude-code, sfruttando l'endpoint Anthropic Messages compatibile esposto da MiniMax all'indirizzo `https://api.minimax.io/anthropic/v1/messages`.

Il provider deve supportare:
- Streaming SSE delle risposte
- Tool use Anthropic-compatibile
- Extended thinking (nativo su M3, sempre-on su M2.x)
- Prompt caching via `cache_control: {"type": "ephemeral"}`
- Discovery automatico dei modelli disponibili via endpoint `/v1/models`
- Proxy configurabile
- Rate limiting integrato

---

## 2. Architettura

### 2.1 Trasporto

MiniMax espone un endpoint Anthropic Messages nativo. Il provider eredita da `AnthropicMessagesTransport` (`providers/anthropic_messages.py`), che fornisce:
- Gestione streaming SSE
- Retry trasparenti con backoff
- Recovery mid-stream
- Tool repair per errori streaming
- Rate limiting via `GlobalRateLimiter`
- Gestione errori HTTP (401, 429, 529)

### 2.2 Pattern di riferimento

Il provider segue il pattern di **Kimi (Moonshot)**, che è il provider Anthropic-compat più simile:
- Stessa classe base (`AnthropicMessagesTransport`)
- Stesso meccanismo di autenticazione (`Authorization: Bearer`)
- Stessa tecnica per il model list (endpoint OpenAI-compat separato)
- Stessa struttura file (`__init__.py`, `client.py`, `request.py`)

### 2.3 Modelli target

| Modello ID | Context | Thinking | Multimodale | Note |
|------------|---------|----------|-------------|------|
| `MiniMax-M3` | 1M token | Controllabile | Sì (immagini, video) | Modello principale |
| `MiniMax-M2.7` | 204k | Sempre on | No | — |
| `MiniMax-M2.7-highspeed` | 204k | Sempre on | No | Latenza ridotta |
| `MiniMax-M2.5` | 204k | Sempre on | No | — |
| `MiniMax-M2.5-highspeed` | 204k | Sempre on | No | — |
| `MiniMax-M2.1` | 204k | Sempre on | No | — |
| `MiniMax-M2.1-highspeed` | 204k | Sempre on | No | — |
| `MiniMax-M2` | 200k | Sempre on | No | Legacy |

---

## 3. Specifica Implementazione

### 3.1 File da creare

#### `providers/minimax/__init__.py`
```python
"""MiniMax provider exports."""

from providers.defaults import MINIMAX_DEFAULT_BASE

from .client import MiniMaxProvider

__all__ = [
    "MINIMAX_DEFAULT_BASE",
    "MiniMaxProvider",
]
```

#### `providers/minimax/client.py`

Classe `MiniMaxProvider` che estende `AnthropicMessagesTransport`:

```python
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
        """Models are listed from the OpenAI-compat root, not /anthropic/v1."""
        return await self._client.get(
            _MINIMAX_OPENAI_MODELS_URL,
            headers=self._model_list_headers(),
        )

    def _model_list_headers(self) -> dict[str, str]:
        return {"Authorization": f"Bearer {self._api_key}"}
```

#### `providers/minimax/request.py`

Request body builder per MiniMax:

```python
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
    """Build JSON for MiniMax Anthropic-compat POST …/messages."""
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

### 3.2 File da modificare

#### `config/provider_catalog.py`

**Aggiungere** la costante `MINIMAX_DEFAULT_BASE` dopo `ZAI_DEFAULT_BASE`:
```python
# MiniMax Anthropic-compatible Messages API (POST …/anthropic/v1/messages).
MINIMAX_DEFAULT_BASE = "https://api.minimax.io/anthropic/v1"
```

**Aggiungere** il `ProviderDescriptor` al dict `PROVIDER_CATALOG` dopo la voce `"zai"`:
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

#### `providers/defaults.py`

**Aggiungere** l'import e re-export di `MINIMAX_DEFAULT_BASE`:
```python
from config.provider_catalog import (
    # ... imports esistenti ...
    MINIMAX_DEFAULT_BASE,
    # ...
)

__all__ = (
    # ... exports esistenti ...
    "MINIMAX_DEFAULT_BASE",
    # ...
)
```

#### `config/settings.py`

**Aggiungere** il campo API key dopo `fireworks_api_key`:
```python
# ==================== MiniMax Config ====================
minimax_api_key: str = Field(default="", validation_alias="MINIMAX_API_KEY")
```

**Aggiungere** il campo proxy dopo `fireworks_proxy`:
```python
minimax_proxy: str = Field(default="", validation_alias="MINIMAX_PROXY")
```

#### `providers/registry.py`

**Aggiungere** la factory function dopo `_create_zai`:
```python
def _create_minimax(config: ProviderConfig, _settings: Settings) -> BaseProvider:
    from providers.minimax import MiniMaxProvider

    return MiniMaxProvider(config)
```

**Aggiungere** l'entry al dict `PROVIDER_FACTORIES`:
```python
"minimax": _create_minimax,
```

#### `.env.example`

**Aggiungere** la sezione API key dopo `FIREWORKS_API_KEY`:
```bash
# MiniMax Config (Anthropic-compatible Messages at api.minimax.io/anthropic/v1)
MINIMAX_API_KEY=""
```

**Aggiungere** il proxy dopo `FIREWORKS_PROXY`:
```bash
MINIMAX_PROXY=""
```

**Aggiungere** la voce per smoke model dopo `FCC_SMOKE_MODEL_CEREBRAS`:
```bash
FCC_SMOKE_MODEL_MINIMAX=
```

**Aggiornare** il commento dei valid providers per includere `minimax`:
```bash
# Valid providers: "nvidia_nim" | "open_router" | "gemini" | "deepseek" | "mistral" | "mistral_codestral" | "opencode" | "opencode_go" | "wafer" | "kimi" | "cerebras" | "groq" | "fireworks" | "zai" | "minimax" | "lmstudio" | "llamacpp" | "ollama"
```

### 3.3 Test da creare

#### `tests/providers/test_minimax.py`

Test unitari che replicano il pattern `test_kimi.py`:

| Test | Verifica |
|------|----------|
| `test_init` | API key e base URL configurati correttamente |
| `test_request_headers` | Header `Authorization: Bearer` + `anthropic-version: 2023-06-01` |
| `test_build_request_body_native` | Body con `model`, `stream: True`, `messages` corretti |
| `test_build_request_body_default_max_tokens` | `max_tokens` default = `ANTHROPIC_DEFAULT_MAX_OUTPUT_TOKENS` |
| `test_build_request_body_rejects_extra_body` | `InvalidRequestError` su `extra_body` |
| `test_model_list_uses_minimax_openai_url` | URL model list = `https://api.minimax.io/v1/models` |
| `test_cleanup_aclose` | `client.aclose()` chiamato una volta |

---

## 4. Versioning

Questo cambiamento introduce una **nuova capability** (nuovo provider). Secondo le regole semver del progetto:
- **MINOR bump**: `x.Y+1.0` — backward-compatible feature (nuovo provider)
- Aggiornare `version` in `pyproject.toml`
- Eseguire `uv lock` per allineare il lockfile
- Commit insieme alle modifiche di produzione

---

## 5. Comportamento e Limitazioni Note

### 5.1 Thinking su M2.x
I modelli M2.x non supportano la disattivazione del thinking. Se l'utente invia una richiesta con `thinking: {"type": "disabled"}`, MiniMax lo ignorerà per M2.x. Il proxy inoltra il parametro correttamente; il comportamento è determinato dal modello.

### 5.2 Parametri ignorati da MiniMax
I seguenti parametri Anthropic vengono accettati ma ignorati silenziosamente da MiniMax:
- `top_k`
- `stop_sequences`
- `mcp_servers`
- `context_management`
- `container`

Questo non causa errori HTTP. Le risposte saranno leggermente diverse da quelle di Anthropic nativo per questi aspetti.

### 5.3 Temperature
MiniMax accetta temperature in [0, 2]. Le temperature Anthropic standard (≤ 1) rientrano nel range e funzionano correttamente.

### 5.4 Model List Discovery
Il model list endpoint (`/v1/models`) è OpenAI-compat. La risposta viene parsata da `extract_openai_model_ids()` che cerca `data[].id` nel JSON. Questo è il pattern già usato da Kimi.

### 5.5 Rate Limits
I rate limits del Token Plan hanno finestre rolling a 5 ore + settimanali. Il `GlobalRateLimiter` del proxy (configurato via `PROVIDER_RATE_LIMIT` / `PROVIDER_RATE_WINDOW`) gestisce il throttling lato client. L'utente deve configurare questi valori in base al proprio piano Token Plan.

### 5.6 Subscription Key vs API Key
MiniMax ha due tipi di credenziali:
- **Subscription Key**: per Token Plan (abbonamento mensile)
- **API Key**: per pay-as-you-go (saldo account)

Il provider accetta entrambe. L'utente configura la credenziale appropriata in `MINIMAX_API_KEY`.

---

## 6. Configurazione Utente

### 6.1 Setup base
```bash
# .env
MINIMAX_API_KEY="your_subscription_or_api_key_here"
MODEL="minimax/MiniMax-M3"
```

### 6.2 Con proxy
```bash
MINIMAX_API_KEY="..."
MINIMAX_PROXY="http://user:pass@host:port"
MODEL="minimax/MiniMax-M3"
```

### 6.3 Modello specifico per tier
```bash
MINIMAX_API_KEY="..."
MODEL="minimax/MiniMax-M3"
MODEL_OPUS="minimax/MiniMax-M3"
MODEL_SONNET="minimax/MiniMax-M3"
MODEL_HAIKU="minimax/MiniMax-M2.7-highspeed"
```

---

## 7. Checklist Implementazione

- [ ] Creare `providers/minimax/__init__.py`
- [ ] Creare `providers/minimax/client.py` con `MiniMaxProvider`
- [ ] Creare `providers/minimax/request.py` con `build_request_body`
- [ ] Aggiungere `MINIMAX_DEFAULT_BASE` in `config/provider_catalog.py`
- [ ] Aggiungere `ProviderDescriptor` per `"minimax"` in `PROVIDER_CATALOG`
- [ ] Aggiungere re-export `MINIMAX_DEFAULT_BASE` in `providers/defaults.py`
- [ ] Aggiungere `minimax_api_key` in `config/settings.py`
- [ ] Aggiungere `minimax_proxy` in `config/settings.py`
- [ ] Aggiungere factory `_create_minimax` in `providers/registry.py`
- [ ] Aggiungere `"minimax"` a `PROVIDER_FACTORIES` in `providers/registry.py`
- [ ] Aggiornare `.env.example` con `MINIMAX_API_KEY`, `MINIMAX_PROXY`, `FCC_SMOKE_MODEL_MINIMAX`
- [ ] Creare `tests/providers/test_minimax.py` con tutti i test
- [ ] Eseguire `uv run ruff format`
- [ ] Eseguire `uv run ruff check`
- [ ] Eseguire `uv run ty check`
- [ ] Eseguire `uv run pytest`
- [ ] Bump version in `pyproject.toml` (MINOR)
- [ ] Eseguire `uv lock`

---

## 8. Rischi Residuali

- **Model list endpoint**: Non verificato live che `/v1/models` restituisca il formato OpenAI-compat con `data[].id`. Il primo test live confermerà. Se MiniMax usa un formato diverso, l'override `_send_model_list_request()` dovrà essere adattato.
- **M2.x thinking non disattivabile**: Accettabile — il proxy inoltra correttamente i thinking blocks. Gli utenti che vogliono controllo sul thinking useranno M3.
- **Nessun altro rischio identificato**: Il pattern Kimi è collaudato e testato. L'endpoint MiniMax è dichiaratamente Anthropic-compatibile.
