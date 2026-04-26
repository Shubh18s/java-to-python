# Chapter 15: Putting It Together — Building an AI Application

## Overview

This chapter builds a complete AI application using every concept from the book. Not a toy example — a structured, production-grade pattern for building AI applications in Python.

The application: a multi-model conversation manager that handles concurrent API calls, tracks token usage, streams responses, validates inputs, and is extensible via mixins.

Every design decision maps back to a concept from earlier chapters.

---

## The Architecture

```
ConversationManager (Chapter 14 — Mixins)
    ├── LogMixin           — logging
    ├── TokenTrackingMixin — usage monitoring
    └── RetryMixin         — resilience

Message (Chapter 10 — Dataclass)
    — immutable value object

ConversationRequest (Chapter 10 — Dataclass)
    — validated request builder

PromptTemplate (Chapter 13 — Functor)
    — callable, configurable, reusable

stream_response (Chapter 9 — Generator)
    — lazy streaming

track_tokens (Chapter 8 — Context Manager)
    — guaranteed cleanup and summary

async fetch_concurrent (Chapter 11 — asyncio)
    — concurrent API calls

ValidatedField (Chapter 12 — Descriptor)
    — reusable attribute validation

PaymentTier (Chapter 6 — ABC)
    — enforced interface for extensibility
```

---

## The Data Layer — Dataclasses

```python
from dataclasses import dataclass, field
from typing import Optional
from enum import Enum

class Role(str, Enum):
    USER = "user"
    ASSISTANT = "assistant"
    SYSTEM = "system"

@dataclass(frozen=True)
class Message:
    """Immutable message — value object."""
    role: Role
    content: str

    def __post_init__(self):
        if not self.content.strip():
            raise ValueError("Message content cannot be empty")

@dataclass
class TokenUsage:
    input_tokens: int = 0
    output_tokens: int = 0

    @property
    def total(self) -> int:
        return self.input_tokens + self.output_tokens

    def __add__(self, other: "TokenUsage") -> "TokenUsage":
        return TokenUsage(
            self.input_tokens + other.input_tokens,
            self.output_tokens + other.output_tokens
        )

@dataclass
class ConversationResponse:
    content: str
    usage: TokenUsage
    model: str
    stop_reason: Optional[str] = None

@dataclass
class ConversationConfig:
    model: str = "claude-sonnet-4-20250514"
    max_tokens: int = 1000
    temperature: float = 1.0
    system_prompt: Optional[str] = None
    messages: list = field(default_factory=list)

    def add_message(self, role: Role, content: str) -> "ConversationConfig":
        self.messages.append(Message(role=role, content=content))
        return self    # fluent interface
```

---

## The Validation Layer — Descriptors

```python
class PositiveInt:
    """Reusable descriptor — validates positive integer fields."""

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if not isinstance(value, int) or value <= 0:
            raise ValueError(f"{self.name} must be a positive integer, got {value!r}")
        setattr(obj, self.private_name, value)

class BoundedFloat:
    """Reusable descriptor — validates float within bounds."""

    def __init__(self, min_val: float, max_val: float):
        self.min_val = min_val
        self.max_val = max_val

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if not isinstance(value, (int, float)):
            raise TypeError(f"{self.name} must be numeric")
        if not self.min_val <= value <= self.max_val:
            raise ValueError(
                f"{self.name} must be between {self.min_val} and {self.max_val}"
            )
        setattr(obj, self.private_name, float(value))

class ValidatedConfig:
    """Configuration with descriptor-based validation."""
    max_tokens = PositiveInt()
    temperature = BoundedFloat(0.0, 1.0)

    def __init__(self, max_tokens: int = 1000, temperature: float = 1.0):
        self.max_tokens = max_tokens
        self.temperature = temperature
```

---

## The Behaviour Layer — Mixins

```python
import time
import logging

class LogMixin:
    """Adds structured logging to any class."""

    def _log(self, level: str, message: str) -> None:
        logger = logging.getLogger(self.__class__.__name__)
        getattr(logger, level)(message)

    def log_info(self, message: str) -> None:
        self._log("info", message)

    def log_error(self, message: str) -> None:
        self._log("error", message)

class TokenTrackingMixin:
    """Tracks cumulative token usage."""

    def __init__(self):
        super().__init__()
        self._usage = TokenUsage()

    def _track(self, usage: TokenUsage) -> None:
        self._usage = self._usage + usage

    @property
    def usage(self) -> TokenUsage:
        return self._usage

    def reset_usage(self) -> None:
        self._usage = TokenUsage()

class RetryMixin:
    """Adds configurable retry logic."""
    _max_retries: int = 3
    _retry_delay: float = 1.0

    def _with_retry(self, func, *args, **kwargs):
        last_error = None
        for attempt in range(self._max_retries):
            try:
                return func(*args, **kwargs)
            except Exception as e:
                last_error = e
                if attempt < self._max_retries - 1:
                    wait = self._retry_delay * (2 ** attempt)    # exponential backoff
                    time.sleep(wait)
        raise last_error
```

---

## The Abstraction Layer — Abstract Base Class

```python
from abc import ABC, abstractmethod

class BaseConversationClient(ABC):
    """
    Abstract interface for conversation clients.
    Any implementation must provide these methods.
    """

    @abstractmethod
    def send(self, config: ConversationConfig) -> ConversationResponse:
        """Send a conversation and return the response."""
        pass

    @abstractmethod
    async def send_async(self, config: ConversationConfig) -> ConversationResponse:
        """Async version — send a conversation and return the response."""
        pass

    @abstractmethod
    def stream(self, config: ConversationConfig):
        """Stream a conversation response as a generator."""
        pass
```

Any new client — Anthropic, OpenAI, a mock for testing — must implement all three. Can't instantiate `BaseConversationClient` directly.

---

## The Context Manager — Token Tracking Session

```python
from contextlib import contextmanager

@contextmanager
def conversation_session(client, label: str = "Session"):
    """
    Context manager that tracks token usage across multiple calls
    and reports a summary on exit.
    """
    client.reset_usage()
    start_time = time.time()

    try:
        yield client
    finally:
        elapsed = time.time() - start_time
        usage = client.usage
        print(f"\n{'─' * 40}")
        print(f"{label} Summary")
        print(f"{'─' * 40}")
        print(f"Duration:       {elapsed:.2f}s")
        print(f"Input tokens:   {usage.input_tokens:,}")
        print(f"Output tokens:  {usage.output_tokens:,}")
        print(f"Total tokens:   {usage.total:,}")
        print(f"{'─' * 40}")
```

---

## The Generator — Streaming Responses

```python
def stream_conversation(client, config: ConversationConfig):
    """
    Generator that yields text chunks as they stream from the API.
    Automatically tracks token usage.
    """
    with client.anthropic_client.messages.stream(
        model=config.model,
        max_tokens=config.max_tokens,
        system=config.system_prompt or "",
        messages=[
            {"role": msg.role.value, "content": msg.content}
            for msg in config.messages
        ]
    ) as stream:
        for chunk in stream.text_stream:
            yield chunk

        # track final usage after stream completes
        final_message = stream.get_final_message()
        client._track(TokenUsage(
            input_tokens=final_message.usage.input_tokens,
            output_tokens=final_message.usage.output_tokens
        ))
```

---

## The Functor — Reusable Prompt Templates

```python
class PromptTemplate:
    """
    Callable object that formats and sends structured prompts.
    Carries configuration and tracks its own usage.
    """

    def __init__(
        self,
        template: str,
        system_prompt: Optional[str] = None,
        max_tokens: int = 1000
    ):
        self.template = template
        self.system_prompt = system_prompt
        self.max_tokens = max_tokens
        self._call_count = 0

    def __call__(self, client, **kwargs) -> str:
        self._call_count += 1
        prompt = self.template.format(**kwargs)
        config = ConversationConfig(
            max_tokens=self.max_tokens,
            system_prompt=self.system_prompt
        ).add_message(Role.USER, prompt)
        response = client.send(config)
        return response.content

    def __repr__(self) -> str:
        return (
            f"PromptTemplate("
            f"calls={self._call_count}, "
            f"max_tokens={self.max_tokens})"
        )
```

---

## The Async Layer — Concurrent API Calls

```python
import asyncio

async def fetch_concurrent(
    async_client,
    configs: list[ConversationConfig]
) -> list[ConversationResponse]:
    """
    Run multiple conversation requests concurrently.
    All requests run simultaneously — not sequentially.
    """
    async def fetch_one(config: ConversationConfig) -> ConversationResponse:
        return await async_client.send_async(config)

    return await asyncio.gather(*[fetch_one(c) for c in configs])

async def process_batch(prompts: list[str], async_client) -> list[str]:
    """Process a batch of prompts concurrently with rate limiting."""
    semaphore = asyncio.Semaphore(5)    # max 5 concurrent requests

    async def limited_fetch(prompt: str) -> str:
        async with semaphore:
            config = ConversationConfig().add_message(Role.USER, prompt)
            response = await async_client.send_async(config)
            return response.content

    return await asyncio.gather(*[limited_fetch(p) for p in prompts])
```

---

## The Full Client — Everything Combined

```python
import anthropic

class ClaudeConversationClient(
    LogMixin,
    TokenTrackingMixin,
    RetryMixin,
    BaseConversationClient
):
    """
    Production conversation client combining:
    - LogMixin: structured logging
    - TokenTrackingMixin: usage monitoring
    - RetryMixin: resilience
    - BaseConversationClient: enforced interface
    """

    def __init__(self, model: str = "claude-sonnet-4-20250514"):
        super().__init__()
        self.model = model
        self.anthropic_client = anthropic.Anthropic()
        self.async_anthropic_client = anthropic.AsyncAnthropic()

    def send(self, config: ConversationConfig) -> ConversationResponse:
        self.log_info(f"Sending request to {self.model}")

        def make_request():
            return self.anthropic_client.messages.create(
                model=config.model or self.model,
                max_tokens=config.max_tokens,
                system=config.system_prompt or "",
                messages=[
                    {"role": msg.role.value, "content": msg.content}
                    for msg in config.messages
                ]
            )

        response = self._with_retry(make_request)
        usage = TokenUsage(
            input_tokens=response.usage.input_tokens,
            output_tokens=response.usage.output_tokens
        )
        self._track(usage)
        self.log_info(f"Response received. Session total: {self.usage.total} tokens")

        return ConversationResponse(
            content=response.content[0].text,
            usage=usage,
            model=response.model,
            stop_reason=response.stop_reason
        )

    async def send_async(self, config: ConversationConfig) -> ConversationResponse:
        response = await self.async_anthropic_client.messages.create(
            model=config.model or self.model,
            max_tokens=config.max_tokens,
            system=config.system_prompt or "",
            messages=[
                {"role": msg.role.value, "content": msg.content}
                for msg in config.messages
            ]
        )
        usage = TokenUsage(
            input_tokens=response.usage.input_tokens,
            output_tokens=response.usage.output_tokens
        )
        self._track(usage)
        return ConversationResponse(
            content=response.content[0].text,
            usage=usage,
            model=response.model,
            stop_reason=response.stop_reason
        )

    def stream(self, config: ConversationConfig):
        return stream_conversation(self, config)
```

---

## Putting It All Together — Usage

```python
import asyncio

# initialise client
client = ClaudeConversationClient()

# ── Single conversation with context manager ──────────────────────────────
with conversation_session(client, label="Single Request") as c:
    config = (
        ConversationConfig(system_prompt="You are a helpful assistant.")
        .add_message(Role.USER, "What is a Python decorator?")
    )
    response = c.send(config)
    print(response.content)

# ── Streaming response ────────────────────────────────────────────────────
config = ConversationConfig().add_message(Role.USER, "Explain generators in Python.")

print("\nStreaming response:")
for chunk in client.stream(config):
    print(chunk, end="", flush=True)

# ── Reusable prompt templates ─────────────────────────────────────────────
summarise = PromptTemplate(
    template="Summarise the following in 3 bullet points:\n\n{text}",
    system_prompt="You are a concise summariser.",
    max_tokens=200
)

classify = PromptTemplate(
    template="Classify as positive, negative, or neutral:\n\n{text}",
    max_tokens=50
)

with conversation_session(client, label="Template Calls") as c:
    summary = summarise(c, text="Python is a dynamic language that...")
    sentiment = classify(c, text="I love how Python handles everything as objects!")
    print(summary)
    print(sentiment)
    print(repr(summarise))    # PromptTemplate(calls=1, max_tokens=200)

# ── Concurrent requests ───────────────────────────────────────────────────
async def run_concurrent():
    prompts = [
        "What is a metaclass?",
        "What is a descriptor?",
        "What is a mixin?",
        "What is a generator?",
    ]

    with conversation_session(client, label="Concurrent Requests") as c:
        responses = await process_batch(prompts, c)
        for prompt, response in zip(prompts, responses):
            print(f"\nQ: {prompt}")
            print(f"A: {response[:100]}...")

asyncio.run(run_concurrent())
```

---

## What Each Concept Is Doing

| Concept | Where it appears | What it contributes |
|---|---|---|
| `@dataclass` | `Message`, `ConversationConfig`, `TokenUsage` | Eliminates constructor boilerplate |
| Descriptors | `PositiveInt`, `BoundedFloat` | Reusable attribute validation |
| ABC | `BaseConversationClient` | Enforced interface — all clients must implement send/stream |
| Mixins | `LogMixin`, `TokenTrackingMixin`, `RetryMixin` | Composable behaviour without deep hierarchy |
| Context manager | `conversation_session` | Guaranteed token summary — even if request fails |
| Generator | `stream_conversation` | Lazy streaming — display chunks as they arrive |
| Functor | `PromptTemplate` | Callable with state — configurable, inspectable |
| `asyncio` | `fetch_concurrent`, `process_batch` | Concurrent API calls — 4x faster than sequential |
| `@property` | `TokenUsage.total`, `usage` | Computed attributes — no stored redundancy |
| `@classmethod` | `ConversationConfig.add_message` | Fluent builder pattern |

---

## The Thread Through the Whole Book

From `classmethod` to a production AI client — every concept built on the previous:

```
Everything is an object
    → functions are objects → decorators
    → classes are objects → metaclasses
    → type is the metaclass → ABCs use ABCMeta
        → ABCs enforce interfaces → BaseConversationClient
            → mixins compose behaviour → ClaudeConversationClient
                → descriptors validate attributes → ValidatedConfig
                    → @property is a descriptor → TokenUsage.total
                        → generators pause/resume → stream_conversation
                            → asyncio is built on generators → fetch_concurrent
                                → context managers wrap blocks → conversation_session
                                    → functors are callable objects → PromptTemplate
                                        → dataclasses eliminate boilerplate → Message
```

None of it is magic. All of it is the same object model, applied at different levels.

---

## What to Learn Next

This book covered the concepts a Java developer needs to think in Python. What comes next:

**Testing:**
- `pytest` vs JUnit — simpler, fixture-based
- `unittest.mock` — mocking without frameworks
- Property-based testing with `hypothesis`

**Type system:**
- `mypy` for static analysis
- `Protocol` — structural typing (duck typing with type checking)
- `TypeVar` and generics

**Package management:**
- `pip`, `venv`, `pyproject.toml`
- `poetry` vs `pip-tools`

**Performance:**
- Profiling with `cProfile`
- `numpy` and vectorisation — where Python gets fast
- When to reach for Cython or Rust extensions

**The Python ecosystem for ML/AI:**
- `pydantic` — validation done right (replaces many descriptor use cases)
- `FastAPI` — async web framework (uses everything in this book)
- LangChain, LlamaIndex — AI orchestration frameworks

---

## Final Thought

Java taught you to think in structures. Python rewards that thinking — but hides the structures behind simpler syntax.

Every time Python felt like magic, it was just the object model applied one level deeper than you expected. Now you know where to look.

You're not a Java developer learning Python. You're a developer who understands both — and that's worth more than knowing either one alone.
