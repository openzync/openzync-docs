LLM Backend Abstraction
=======================

.. note::

   This document covers the LLM abstraction layer in
   ``openzync-core/core/llm.py`` and ``openzync-core/core/llm_backends.py``.
   Code examples assume the package is importable as ``from core import ...``
   from the monolith's Python path.

   The abstraction follows a **strategy / registry** pattern:

   * ``core/llm.py`` defines the abstract interface (:class:`LLMBackend`),
     standardised response types, a registry
     (:class:`LLMBackendRegistry`), and a factory function
     (:func:`resolve_backend`) that wires org-level configuration to the
     correct provider.
   * ``core/llm_backends.py`` contains concrete implementations — one class
     per provider — each registering itself with the registry at module load
     time.

   **Design principle**: every backend is self-contained; there is no shared
   base class beyond :class:`LLMBackend`.  Each provider owns its SDK client,
   retry logic, caching semantics, and error handling.

.. contents:: Sections
   :local:
   :depth: 2
   :class: this-will-duplicate-information-and-it-is-still-useful-here


Abstract Interface — ``core.llm``
----------------------------------

Module: ``core.llm``

Data Types
~~~~~~~~~~

.. class:: LLMProvider(value)

   :metaclass: :class:`Enum`
   :members:

   Supported LLM provider identifiers.  Used throughout the system to
   reference providers by a short string name.

   .. attribute:: OLLAMA

      :value: ``"ollama"``

   .. attribute:: OPENAI

      :value: ``"openai"``

   .. attribute:: AZURE

      :value: ``"azure"``

   .. attribute:: ANTHROPIC

      :value: ``"anthropic"``

   Note that ``"openrouter"`` is **not** in this enum — it is registered
   directly with :class:`LLMBackendRegistry` under the name ``"openrouter"``
   without a corresponding enum member.  This is a
   TODO: needs author clarification — either add the enum member or document
   the intentional divergence.

.. class:: TokenUsage

   :class:`~dataclasses.dataclass`

   Token consumption report for a single LLM call.  Captures both standard
   token counts and provider-specific cache metrics.

   .. attribute:: prompt_tokens

      :type: int
      :value: ``0``

      Number of tokens in the prompt (input).

   .. attribute:: completion_tokens

      :type: int
      :value: ``0``

      Number of tokens in the completion (output).

   .. attribute:: cache_read_input_tokens

      :type: int
      :value: ``0``

      Tokens served from provider cache (cache hit).  Populated by
      OpenAI/Azure (``prompt_tokens_details.cached_tokens``) and Anthropic
      (``cache_read_input_tokens``).  Always 0 for Ollama.

   .. attribute:: cache_creation_input_tokens

      :type: int
      :value: ``0``

      Tokens written to provider cache (cache miss).  Populated by
      OpenAI/Azure (``prompt_tokens_details.cache_write_tokens``) and
      Anthropic (``cache_creation_input_tokens``).  Always 0 for Ollama.

   .. attribute:: total_tokens

      :type: int
      :property:

      Convenience property returning ``prompt_tokens + completion_tokens``.

   .. attribute:: total_cache_tokens

      :type: int
      :property:

      Convenience property returning
      ``cache_read_input_tokens + cache_creation_input_tokens``.

   Usage::

       usage = TokenUsage(
           prompt_tokens=150,
           completion_tokens=42,
           cache_read_input_tokens=128,
           cache_creation_input_tokens=0,
       )
       print(usage.total_tokens)       # → 192
       print(usage.total_cache_tokens)  # → 128

.. class:: PromptCachingConfig

   :class:`~dataclasses.dataclass`

   Per-call configuration for provider-side prompt caching.  Each backend
   interprets the fields relevant to its provider; unsupported fields are
   silently ignored.

   ================================ ========== ==================================
   Provider                         Interprets Notes
   ================================ ========== ==================================
   Anthropic                        ``enabled``, ``anthropic_min_tokens``,
                                    ``anthropic_cache_ttl``
   OpenAI / Azure                   ``enabled`` (automatic prefix caching
                                    needs no additional config)
   OpenRouter                       ``enabled``, ``session_id`` (for
                                    sticky routing to the same upstream)
   Ollama                           *(none)*   Caching is always disabled.
   ================================ ========== ==================================

   .. attribute:: enabled

      :type: bool
      :value: ``True``

      Master switch for this call.  When ``False``, no caching headers or
      markers are emitted for any provider.  When ``True``, caching is
      attempted according to each provider's mechanism.

   .. attribute:: anthropic_min_tokens

      :type: int
      :value: ``1024``

      Minimum estimated token count for a system prompt block before
      ``cache_control`` markers are applied to Anthropic calls.  The
      estimate is calculated as ``len(system_text) // 4``.  Models vary:
      1024 covers Claude Sonnet 4.6, 4096 covers Claude Opus 4.6.

   .. attribute:: anthropic_cache_ttl

      :type: str
      :value: ``"5m"``

      Anthropic cache TTL.  ``"5m"`` (1.25× write cost, default) or
      ``"1h"`` (2× write cost).  Cache reads are always 0.1× regardless
      of write TTL.

   .. attribute:: session_id

      :type: str | None
      :value: ``None``

      Session identifier for OpenRouter sticky routing.  When set, the
      ``session_id`` is included in the request body, encouraging
      OpenRouter to route to the same upstream provider across calls,
      improving cache hit rates.

.. class:: ChatResponse

   :class:`~dataclasses.dataclass`

   Uniform response from any LLM chat backend.  Designed so that callers
   never need provider-specific response parsing.

   When ``response_model`` was passed to :meth:`LLMBackend.chat` and
   validation succeeded, :attr:`validated_data` holds the parsed Pydantic
   model instance — callers can access typed fields directly instead of
   re-parsing ``content``.

   .. attribute:: content

      :type: str

      The generated text content.  When the LLM returns tool calls (OpenAI /
      Azure / OpenRouter), the function arguments are extracted and returned
      here so the caller sees a single string regardless of transport.

   .. attribute:: model

      :type: str

      The model identifier that produced this response
      (e.g. ``"gpt-4o-mini"``).

   .. attribute:: usage

      :type: TokenUsage

      Token consumption report for this call.

   .. attribute:: validated_data

      :type: BaseModel | None
      :value: ``None``

      When a Pydantic ``response_model`` was supplied and validation
      succeeded, this field holds the parsed model instance.  ``None`` when
      no ``response_model`` was requested, or when the response is
      returned before the validation loop completes (the outer caller
      should not rely on this being set on every return path; see
      :meth:`LLMBackend.chat` for details).

.. class:: EmbeddingResponse

   :class:`~dataclasses.dataclass`

   Uniform response from any embedding backend.

   .. attribute:: embeddings

      :type: list[list[float]]

      The embedding vectors.  One vector per input text, in the same order.

   .. attribute:: model

      :type: str

      The model identifier that produced these embeddings.

   .. attribute:: dim

      :type: int

      Dimensionality of each embedding vector (e.g. 768 for
      ``nomic-embed-text``, 1536 for ``text-embedding-3-small``).


Helper Functions
~~~~~~~~~~~~~~~~

.. function:: build_cache_config(org_config=None, session_id=None)

   :param org_config: Optional per-org LLM config dict (from the
       :class:`~core.org_config.OrgConfigBase` JSONB column).  May contain a
       ``"prompt_caching"`` key with ``enabled``, ``anthropic_min_tokens``,
       ``anthropic_cache_ttl``.
   :type org_config: dict | None
   :param session_id: Optional session ID for OpenRouter sticky routing.
   :type session_id: str | None
   :rtype: PromptCachingConfig

   Build a :class:`PromptCachingConfig` from per-org settings, falling back
   to global defaults from the :class:`~core.config.Settings` singleton.

   Resolution order:

   1. ``settings.PROMPT_CACHING_ENABLED`` — hard kill switch.  If
      ``False``, returns ``PromptCachingConfig(enabled=False)`` immediately
      with no further lookup.
   2. ``org_config.get("prompt_caching")`` dict — per-org overrides for
      ``anthropic_min_tokens`` and ``anthropic_cache_ttl``.
   3. ``settings.PROMPT_CACHING_ANTHROPIC_MIN_TOKENS`` and
      ``settings.PROMPT_CACHING_ANTHROPIC_TTL`` — global defaults.

   Usage::

       from core.config import get_settings
       from core.llm import build_cache_config

       # Within a request with per-org config
       org_cfg = await get_org_config(org_id)
       cache_cfg = build_cache_config(
           org_config=org_cfg.dict() if org_cfg else None,
           session_id="my-session-42",
       )
       # cache_cfg.enabled is True unless OZ_PROMPT_CACHING_ENABLED=False


Internal Helpers
~~~~~~~~~~~~~~~~

.. function:: _last_validation_error(content, model)

   :param str content: The raw LLM response content string.
   :param type[BaseModel] model: The Pydantic model to validate against.
   :rtype: str

   Return a short diagnostic message when a structured-output validation
   fails.  First attempts JSON parsing via ``orjson.loads``; if that fails
   the error message indicates invalid JSON.  Otherwise attempts
   ``model.model_validate(parsed)`` and returns the validation error string.
   This is an internal helper used by :meth:`LLMBackend.chat` when
   constructing :class:`LLMStructuredOutputError` detail — callers should
   not need to invoke it directly.


Abstract Backend — :class:`LLMBackend`
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. class:: LLMBackend()

   :metaclass: :class:`abc.ABC`

   Abstract base class for all LLM providers.  Subclasses implement
   :meth:`_chat` and :meth:`embed` using the provider's SDK or HTTP API.
   Every backend reports which model it is using (:attr:`model_name`) and
   the embedding dimensionality (:attr:`embedding_dim`).

   The public :meth:`chat` method adds optional structured-output
   validation on top of the raw provider call.

   .. rubric:: Class Data

   .. attribute:: VALIDATION_RETRIES

      :type: int
      :value: ``2``

      Number of validation retries when a Pydantic ``response_model`` is
      provided but the LLM output fails to parse into it.  Can be
      overridden per-call via the ``validation_retries`` parameter.

   .. rubric:: Public Methods

   .. method:: chat(messages, response_model=None, validation_retries=None, cache_config=None, **kwargs)

      :param list[dict] messages: List of message dicts with ``role`` and
          ``content`` keys, following the OpenAI message format.
      :param response_model: Optional Pydantic model to validate the
          output content against.  When provided, the LLM is instructed to
          emit JSON matching the model's schema via a system instruction
          injection.
      :type response_model: type[BaseModel] | None
      :param validation_retries: Override for the number of validation
          retry attempts.  Defaults to :attr:`VALIDATION_RETRIES`.
      :type validation_retries: int | None
      :param cache_config: Optional prompt caching configuration.  Passed
          through to the backend's :meth:`_chat`.
      :type cache_config: PromptCachingConfig | None
      :param \\**kwargs: Additional provider-specific parameters
          (``temperature``, ``max_tokens``, ``top_p``, etc.).
      :rtype: ChatResponse
      :raises LLMStructuredOutputError: If the output cannot be validated
          against *response_model* after exhausting retries.
      :raises RuntimeError: In the internal validation loop if the
          uncreachable branch is hit (``pragma: no cover`` — should never
          occur in practice).

      Send a chat completion, optionally validating against a Pydantic model.

      **Fast path** (no validation)::

          response = await backend.chat([
              {"role": "user", "content": "Hello!"},
          ])
          print(response.content)

      **Structured output** path::

          from pydantic import BaseModel

          class Quote(BaseModel):
              symbol: str
              price: float
              currency: str = "USD"

          response = await backend.chat(
              messages=[
                  {"role": "user", "content": "What is AAPL at?"},
              ],
              response_model=Quote,
          )
          # response.validated_data is a Quote instance
          print(response.validated_data.symbol)  # → "AAPL"

      When ``response_model`` is provided, the flow is:

      1. A system instruction with the model's JSON schema is injected into
         *messages* so the LLM knows the expected output shape.
      2. The provider is called via :meth:`_chat`.
      3. The response is parsed and validated against *response_model* via
         :meth:`model_validate_json`.
      4. On success the :class:`ChatResponse` is returned with
         :attr:`~ChatResponse.validated_data` set.
      5. On JSON parse failure, a fallback stripping pass is attempted
         via :meth:`_extract_json` to handle markdown fences and thinking
         blocks.
      6. On validation failure the conversation history is amended with
         the bad output and a retry prompt explaining *why* it failed,
         then the provider is called again.
      7. After exhausting ``validation_retries`` attempts a
         :class:`LLMStructuredOutputError` is raised.

   .. rubric:: Abstract Methods

   .. method:: _chat(messages, cache_config=None, **kwargs)

      :param list[dict] messages: List of message dicts following OpenAI
          format.
      :param cache_config: Optional prompt caching configuration.  Each
          backend interprets this according to its provider's caching
          capabilities.
      :type cache_config: PromptCachingConfig | None
      :param \\**kwargs: Provider-specific parameters.
      :rtype: ChatResponse

      Provider-specific chat implementation.  Override this in each
      backend.  The public :meth:`chat` wraps this with validation, retry,
      and structured-output logic.

   .. method:: embed(texts, **kwargs)

      :param list[str] texts: List of input strings to embed.
      :param \\**kwargs: Additional provider-specific parameters
          (e.g. ``model`` to override the embedding model).
      :rtype: EmbeddingResponse
      :raises NotImplementedError: If the provider does not support
          embeddings (Anthropic, OpenRouter).

      Generate embeddings for one or more text strings.  Returns a
      standardised :class:`EmbeddingResponse` containing vectors and
      dimensionality.

   .. rubric:: Abstract Properties

   .. attribute:: model_name

      :type: str
      :property:

      The model identifier currently in use (e.g. ``"gpt-4o"``,
      ``"claude-sonnet-4-20250514"``).

   .. attribute:: embedding_dim

      :type: int
      :property:

      Dimensionality of the embedding vectors produced by this backend.
      Returns ``0`` if the backend does not support embeddings.

   .. rubric:: Static / Internal Helpers

   .. method:: _inject_schema_instr(messages, model)

      :param list[dict] messages: The current message list.
      :param type[BaseModel] model: The Pydantic model to generate a
          schema instruction for.
      :rtype: list[dict]

      Prepend (or append to an existing system message) a schema directive
      telling the LLM to output valid JSON matching the model's schema.
      If the first message already has ``role == "system"``, the schema
      instruction is appended to its content.  Otherwise a new system
      message is prepended.

   .. method:: _build_retry_messages(messages, bad_content, model)

      :param list[dict] messages: The current (good) message history.
      :param str bad_content: The failed LLM output.
      :param type[BaseModel] model: The Pydantic model that was used for
          validation.
      :rtype: list[dict]

      Append assistant error + user retry prompt after a validation
      failure.  Adds the failed output as an ``assistant`` message and
      follows it with a ``user`` message explaining the failure and
      repeating the expected schema.

   .. method:: _extract_json(text)

      :param str text: Raw text that may contain JSON wrapped in markdown
          fences, thinking blocks, or other non-JSON wrappers.
      :rtype: Any | None

      Strip markdown fences (`` ```json `` / `` ``` ``) and other wrappers,
      then attempt to parse JSON via ``orjson.loads``.  Returns the parsed
      Python value on success, or ``None`` if no valid JSON could be
      extracted.  Used by :meth:`chat` as a fallback when
      :meth:`model_validate_json` fails on the raw content.


Backend Registry — :class:`LLMBackendRegistry`
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. class:: LLMBackendRegistry

   Registry of available LLM backend *classes* (not instances).  Backends
   register themselves at import time (see the auto-registration block at
   the bottom of :mod:`core.llm_backends`).  The registry is used by
   :func:`resolve_backend` to look up the correct class by name.

   .. attribute:: _backends

      :type: ClassVar[dict[str, type[LLMBackend]]]
      :value: ``{}``

      Internal mapping of provider name → backend class.

   .. method:: register(name, backend_cls)

      :param str name: Provider name (e.g. ``"openai"``).  Must match
          the provider identifier used by :func:`resolve_backend`.
      :param type[LLMBackend] backend_cls: The class to instantiate when
          this provider is selected.
      :raises ValueError: If a backend with the same name is already
          registered.

      Register a backend class under a provider name.  Typically called at
      module load time::

          LLMBackendRegistry.register("my_provider", MyBackend)

   .. method:: get(name)

      :param str name: Provider name.
      :rtype: type[LLMBackend]
      :raises ValueError: If the provider name is not registered.

      Look up a registered backend class::

          cls = LLMBackendRegistry.get("openai")
          backend = cls(api_key="sk-...", model="gpt-4o")

   .. method:: list_available()

      :rtype: list[str]

      List all registered provider names::

          providers = LLMBackendRegistry.list_available()
          # → ["ollama", "openai", "azure", "anthropic", "openrouter"]


Resolution — :func:`resolve_backend`
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: resolve_backend(provider=None, org_config=None)

   :param provider: Explicit override.  If provided, org config is skipped.
   :type provider: str | None
   :param org_config: Optional dict with per-organisation LLM settings.
       Supported keys include ``llm_backend``, ``ollama_base_url``,
       ``openai_api_key``, ``openai_model``, ``azure_endpoint``,
       ``azure_api_key``, ``azure_deployment``, ``anthropic_api_key``,
       ``anthropic_model``, ``api_key``, ``model``.
   :type org_config: dict | None
   :rtype: LLMBackend
   :raises LLMConfigurationError: If no backend could be resolved.
   :raises ValueError: If the resolved provider name is unknown.

   Resolve the appropriate LLM backend via org config, explicit argument,
   or environment variable.

   **Resolution order**:

   1. **Org-level config** — If ``provider`` is ``None`` and
      ``org_config.get("llm_backend")`` is set, that value is used.
   2. **Explicit argument** — the ``provider`` parameter.
   3. **Error** — raises :class:`LLMConfigurationError`.

   .. code-block:: python

       from core.llm import resolve_backend

       # Via explicit argument
       backend = await resolve_backend(provider="openai")

       # Via per-org config
       org_cfg = {"llm_backend": "anthropic", "anthropic_api_key": "...", "anthropic_model": "claude-sonnet-4-20250514"}
       backend = await resolve_backend(org_config=org_cfg)

       response = await backend.chat([{"role": "user", "content": "Hello!"}])


.. function:: _create_backend(provider, config=None)

   :param str provider: One of ``"ollama"``, ``"openai"``, ``"azure"``,
       ``"anthropic"``, ``"openrouter"``.
   :param config: Optional dict with provider-specific overrides.
       Required fields vary by provider (see individual backend docs).
   :type config: dict | None
   :rtype: LLMBackend
   :raises LLMConfigurationError: If a required config field is missing or
       empty.
   :raises ValueError: If *provider* is not recognised.

   Internal helper that instantiates an LLM backend for *provider*,
   extracting the necessary values from the config dict for each provider
   type.  All provider-specific values come **exclusively** from *config*
   — there is no env-var fallback and no hardcoded default at this layer.

   ====================== ==================================================
   Provider               Required config keys
   ====================== ==================================================
   ``"ollama"``           ``ollama_base_url``
   ``"openai"``           ``openai_api_key`` (``openai_model`` optional)
   ``"azure"``            ``azure_endpoint``, ``azure_api_key``,
                          ``azure_deployment``
   ``"anthropic"``        ``anthropic_api_key`` (``anthropic_model`` optional)
   ``"openrouter"``       ``api_key``, ``model``
   ====================== ==================================================


Concrete Implementations — ``core.llm_backends``
--------------------------------------------------

Module: ``core.llm_backends``

All backends register themselves with :class:`LLMBackendRegistry` at the
bottom of the module via :meth:`LLMBackendRegistry.register`.  This makes
them discoverable by :func:`resolve_backend` without explicit imports.


OllamaBackend
~~~~~~~~~~~~~

.. class:: OllamaBackend(base_url="http://localhost:11434")

   LLM backend powered by a local Ollama instance.  Requires no API key.
   Connects to the Ollama REST API at the configured base URL.

   .. attribute:: DEFAULT_CHAT_MODEL

      :type: ClassVar[str]
      :value: ``"llama3.2:3b"``

   .. attribute:: DEFAULT_EMBED_MODEL

      :type: ClassVar[str]
      :value: ``"nomic-embed-text"``

   .. attribute:: DEFAULT_EMBED_DIM

      :type: ClassVar[int]
      :value: ``768``

   .. rubric:: Constructor

   :param str base_url: Ollama server URL (default ``http://localhost:11434``).
       The system does **not** read this from an environment variable;
       it must be provided via per-org config or explicitly.

   Model defaults are class constants — there is no env-var fallback.

   .. rubric:: Provider Overrides

   ``_chat`` supported kwargs:

   * ``model`` — override the chat model (default: ``DEFAULT_CHAT_MODEL``).
   * ``temperature``
   * ``top_p``
   * ``max_tokens``
   * ``stream``

   ``embed`` supported kwargs:

   * ``model`` — override the embedding model (default:
     ``DEFAULT_EMBED_MODEL``).

   .. rubric:: Caching

   Prompt caching is **not supported** by the Ollama API.  If
   ``cache_config.enabled`` is ``True``, a debug-level log message is
   emitted and the request proceeds without caching headers.

   .. rubric:: Endpoints Called

   * Chat: ``POST {base_url}/api/chat``
   * Embed: ``POST {base_url}/api/embeddings`` (also handles the plural
     ``/api/embed`` response format transparently)

   .. rubric:: HTTP Client

   Uses an ephemeral :class:`httpx.AsyncClient` with a 120-second timeout
   for each request.  Error handling covers HTTP status errors and timeout
   exceptions, both logged with structured context.

   Usage::

       backend = OllamaBackend(base_url="http://localhost:11434")
       response = await backend.chat([
           {"role": "user", "content": "Hello from Ollama!"},
       ])
       print(response.content)

       embed_response = await backend.embed(["Hello world"])
       print(embed_response.dim)  # → 768


OpenAIBackend
~~~~~~~~~~~~~

.. class:: OpenAIBackend(api_key, model=None)

   LLM backend for the OpenAI API.  Uses the official ``openai`` library
   with :class:`openai.AsyncOpenAI`.

   Supports GPT-4o, GPT-4o-mini, GPT-4-turbo, and all OpenAI chat models.
   Embeddings use ``text-embedding-3-small`` by default.

   .. attribute:: DEFAULT_MODEL

      :type: ClassVar[str]
      :value: ``"gpt-4o-mini"``

   .. attribute:: DEFAULT_EMBED_MODEL

      :type: ClassVar[str]
      :value: ``"text-embedding-3-small"``

   .. attribute:: DEFAULT_EMBED_DIM

      :type: ClassVar[int]
      :value: ``1536``

   .. attribute:: MAX_RETRIES

      :type: ClassVar[int]
      :value: ``3``

   .. rubric:: Constructor

   :param str api_key: OpenAI API key (required).  Can be obtained from
       per-org config (``openai_api_key``) or provided directly.  Raises
       :class:`ValueError` if empty.
   :param model: OpenAI model name.  Falls back to :attr:`DEFAULT_MODEL`
       if ``None``.
   :type model: str | None

   .. rubric:: Provider Overrides

   ``_chat`` supported kwargs:

   * ``model`` — override the chat model.
   * ``temperature`` (default: ``0.0``)
   * ``max_tokens``
   * ``top_p``
   * ``frequency_penalty``
   * ``presence_penalty``
   * ``stop``

   ``embed`` supported kwargs:

   * ``model`` — override the embedding model.

   .. rubric:: Rate Limiting & Retries

   Handles HTTP 429 (rate limit) and 5xx server errors with exponential
   backoff (``wait = 2^attempt`` seconds — 2s, 4s, 8s).  Maximum 3
   retries set by :attr:`MAX_RETRIES`.  Non-retryable errors (client
   errors other than 429, auth failures) are raised immediately.

   All retry and error events are logged with structured context
   (attempt number, status code, wait time, model).

   .. rubric:: Tool Call Extraction

   When ``response.choices[0].message.content`` is ``None`` but tool calls
   are present, the first tool call's ``function.arguments`` (JSON string)
   is returned as the content.  This allows structured-output callers to
   receive function-call arguments without additional parsing.  A log
   event ``llm.tool_call_extracted`` is emitted with the function name and
   argument length.

   .. rubric:: Caching

   OpenAI supports automatic prompt prefix caching — no explicit headers
   are needed.  The backend reads ``prompt_tokens_details.cached_tokens``
   and ``prompt_tokens_details.cache_write_tokens`` from the API response
   and surfaces them in the :class:`TokenUsage`.

   Usage::

       backend = OpenAIBackend(
           api_key="sk-...",
           model="gpt-4o",
       )
       response = await backend.chat(
           messages=[{"role": "user", "content": "Hello!"}],
           temperature=0.7,
       )
       print(response.content, response.usage.total_tokens)


AzureBackend
~~~~~~~~~~~~

.. class:: AzureBackend(endpoint, api_key, deployment)

   LLM backend for Azure OpenAI Service.  Uses the
   :class:`openai.AsyncAzureOpenAI` client from the ``openai`` library.

   .. attribute:: DEFAULT_EMBED_DIM

      :type: ClassVar[int]
      :value: ``1536``

   .. attribute:: MAX_RETRIES

      :type: ClassVar[int]
      :value: ``3``

   .. rubric:: Constructor

   :param str endpoint: Azure OpenAI endpoint URL (e.g.
       ``https://my-resource.openai.azure.com``).  Required.
   :param str api_key: Azure OpenAI API key.  Required.
   :param str deployment: Azure OpenAI deployment name.  Required.  This
       is the name of the deployed model in the Azure AI Studio portal,
       and is used as both the chat model identifier and the embedding
       model identifier.

   All three parameters raise :class:`ValueError` if empty.

   The API version is hardcoded to ``"2024-08-01-preview"``.  This is the
   API version that supports prompt token detail reporting.

   .. rubric:: Provider Overrides

   ``_chat`` supported kwargs: ``temperature`` (default ``0.0``),
   ``max_tokens``, ``top_p``, etc.  The ``model`` kwarg overrides the
   deployment name (defaults to the deployment passed at construction).

   ``embed`` supported kwargs: ``model`` — override the deployment name
   for embeddings (defaults to the same deployment as chat).

   .. rubric:: Differences from OpenAIBackend

   * Uses :class:`openai.AsyncAzureOpenAI` instead of
     :class:`openai.AsyncOpenAI`.
   * The API key and endpoint are passed as separate parameters instead of
     a single base URL.
   * The deployment name is the model identifier — Azure does not accept
     standard OpenAI model names like ``"gpt-4o"`` without a deployment
     mapping.
   * Same retry, caching, and tool-call extraction logic as
     :class:`OpenAIBackend`.

   Usage::

       backend = AzureBackend(
           endpoint="https://my-resource.openai.azure.com",
           api_key="...",
           deployment="gpt-4o-deployment",
       )
       response = await backend.chat([
           {"role": "user", "content": "Hello from Azure!"},
       ])


AnthropicBackend
~~~~~~~~~~~~~~~~

.. class:: AnthropicBackend(api_key, model=None)

   LLM backend for the Anthropic API (Claude models).

   Uses the official ``anthropic`` library with
   :class:`anthropic.AsyncAnthropic`.

   .. warning::

      Embeddings are **not supported** — calling :meth:`embed` raises
      :class:`NotImplementedError`.

   .. attribute:: DEFAULT_MODEL

      :type: ClassVar[str]
      :value: ``"claude-sonnet-4-20250514"``

   .. attribute:: MAX_RETRIES

      :type: ClassVar[int]
      :value: ``3``

   .. rubric:: Constructor

   :param str api_key: Anthropic API key (required).  Raises
       :class:`ValueError` if empty.
   :param model: Anthropic model name.  Falls back to
       :attr:`DEFAULT_MODEL` if ``None``.
   :type model: str | None

   .. rubric:: System Message Handling

   Anthropic requires a separate ``system`` parameter distinct from the
   messages array.  The backend handles this transparently:

   1. If the first message has ``role == "system"``, it is extracted and
      passed as the ``system`` parameter to the API.
   2. The system message is removed from the messages array sent to
      Anthropic.
   3. If caching is enabled (see below), the system content is wrapped in a
      list-of-blocks format with ``cache_control`` markers.

   .. rubric:: Provider Overrides

   ``_chat`` supported kwargs:

   * ``model`` — override the model.
   * ``max_tokens`` (default: ``4096``)
   * ``temperature`` (default: ``0.0``)
   * ``top_p``
   * ``top_k``
   * ``stop_sequences``

   .. rubric:: Prompt Caching (Anthropic-specific)

   When ``cache_config.enabled`` is ``True`` **and** the system text is
   long enough (estimated via ``len(text) // 4 >= anthropic_min_tokens``),
   the system block is sent with a ``cache_control`` marker:

   * ``{"type": "ephemeral"}`` — standard caching (default).
   * ``{"cache_control": {"type": "ephemeral", "ttl": "1h"}}`` — extended
     TTL when ``anthropic_cache_ttl == "1h"``.

   Proxy requests to remaining message content blocks for cache_control
   markers are **not** implemented — only the system block receives
   caching.  TODO: needs author clarification — is this intentional, or
   should message-level cache_control be added?

   .. rubric:: Retries

   Rate limits (429) and server errors (5xx) are retried with exponential
   backoff (``2^attempt`` seconds).  The SDK's
   ``anthropic.RateLimitError`` and ``anthropic.APIStatusError`` are
   both handled.  Non-retryable errors are raised immediately.

   Usage::

       backend = AnthropicBackend(
           api_key="sk-ant-...",
           model="claude-sonnet-4-20250514",
       )
       response = await backend.chat([
           {"role": "system", "content": "You are a helpful assistant."},
           {"role": "user", "content": "Hello Claude!"},
       ])
       print(response.content)
       print(response.usage.cache_read_input_tokens)


OpenRouterBackend
~~~~~~~~~~~~~~~~~

.. class:: OpenRouterBackend(api_key, model)

   LLM backend powered by `OpenRouter <https://openrouter.ai/>`_'s unified
   API.  Uses an OpenAI-compatible client pointed at
   ``https://openrouter.ai/api/v1``.

   .. warning::

      Embeddings are **not supported** — calling :meth:`embed` raises
      :class:`NotImplementedError` and :attr:`embedding_dim` raises
      :class:`NotImplementedError`.

   .. attribute:: BASE_URL

      :type: ClassVar[str]
      :value: ``"https://openrouter.ai/api/v1"``

   .. attribute:: MAX_RETRIES

      :type: ClassVar[int]
      :value: ``3``

   .. rubric:: Constructor

   :param str api_key: OpenRouter API key (required).  Raises
       :class:`LLMConfigurationError` if empty.
   :param str model: OpenRouter model identifier (required).  Examples:
       ``"openai/gpt-4o"``, ``"anthropic/claude-sonnet-4"``,
       ``"google/gemini-1.5-pro"``.  Raises :class:`LLMConfigurationError`
       if ``None`` or empty.

   Unlike the other backends, both parameters **must** be provided —
   there is no default model and no env-var fallback.

   The client includes HTTP headers identifying the app to OpenRouter:

   * ``HTTP-Referer`` → ``"https://github.com/rohnsha0/openzync"``
   * ``X-OpenRouter-Title`` → ``"OpenZync - Agent Memory Platform"``

   .. rubric:: Provider Overrides

   ``_chat`` supported kwargs:

   * ``model`` — override the model.
   * ``temperature`` (default: ``0.1``)
   * ``max_tokens`` (default: ``4096``)

   .. rubric:: Session Stickiness

   When ``cache_config.enabled`` is ``True`` and ``cache_config.session_id``
   is set, a ``session_id`` field is included in the request body
   (via ``extra_body``).  This encourages OpenRouter to route consecutive
   requests to the same upstream provider, improving cache hit rates.

   .. rubric:: Empty Response Handling

   When the response content is ``None``:

   * If ``finish_reason`` is ``"content_filter"`` or ``"length"``, a
     :class:`ValueError` is raised immediately — these are deterministic
     failures.
   * Otherwise, the request is retried with exponential backoff.  If all
     retries are exhausted, a final :class:`ValueError` is stored as
     ``last_exception``.

   .. rubric:: Retries

   In addition to HTTP 429/5xx retries (same pattern as
   :class:`OpenAIBackend`), the backend also retries on:

   * :class:`openai.APITimeoutError` — network-level timeout.
   * :class:`openai.APIConnectionError` — connection refused, DNS
     resolution failure.
   * OpenAI error objects that carry a ``code`` attribute.

   .. rubric:: Cache Discount Tracking

   OpenRouter may return a ``cache_discount`` field in the response
   (accessed via ``model_extra`` since the OpenAI SDK strips unknown
   fields).  This is logged but not surfaced in :class:`TokenUsage`.

   Usage::

       backend = OpenRouterBackend(
           api_key="sk-or-...",
           model="openai/gpt-4o",
       )
       response = await backend.chat([
           {"role": "user", "content": "Hello via OpenRouter!"},
       ])
       print(response.content)


Auto-Registration
~~~~~~~~~~~~~~~~~

The following registrations occur at the bottom of
:mod:`core.llm_backends` (module load time)::

   LLMBackendRegistry.register("ollama", OllamaBackend)
   LLMBackendRegistry.register("openai", OpenAIBackend)
   LLMBackendRegistry.register("azure", AzureBackend)
   LLMBackendRegistry.register("anthropic", AnthropicBackend)
   LLMBackendRegistry.register("openrouter", OpenRouterBackend)

The import chain is deliberately lazy:

1. :mod:`core.llm` imports :mod:`core.llm_backends` via
   ``importlib.import_module("core.llm_backends")`` (line 533).
2. :mod:`core.llm_backends` imports names from :mod:`core.llm` at the top
   of the file.
3. The lazy import avoids a circular import deadlock — by the time
   ``importlib`` has finished importing ``llm_backends``, the ``LLMBackend``
   class and registry are available because Python has already created the
   module object (albeit with some names still being populated).


Design Patterns & Architecture
------------------------------

Strategy / Registry Pattern
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The LLM abstraction is a textbook **strategy pattern** with a **registry**
for provider discovery:

* :class:`LLMBackend` defines the strategy interface (``chat``, ``embed``,
  ``model_name``, ``embedding_dim``).
* Each concrete class in :mod:`core.llm_backends` implements the strategy.
* :class:`LLMBackendRegistry` acts as the provider registry, mapping string
  names to backend classes.
* :func:`resolve_backend` is the factory that selects a strategy based on
  configuration.

This design means:

* Adding a new provider requires no changes to existing backends.
* The resolution logic (config → provider) is fully decoupled from provider
  implementation.
* Callers depend only on the abstract interface, not on any specific
  provider.


Backend Selection Flow
~~~~~~~~~~~~~~~~~~~~~~~

The full backend selection flow at runtime is:

1. Caller (typically a service layer) calls
   :func:`resolve_backend(org_config=...)`.
2. :func:`resolve_backend` reads ``org_config.get("llm_backend")`` to
   determine the provider name (e.g. ``"openai"``).
3. ``_create_backend`` looks up the class via
   :meth:`LLMBackendRegistry.get(provider)`.
4. The class is instantiated with provider-specific parameters extracted
   from the org config (API keys, model names, endpoints).
5. The caller uses the returned :class:`LLMBackend` instance polymorphically.

For callers that already know the provider at coding time, the explicit
parameter form can be used::

   backend = await resolve_backend(provider="openai", org_config=org_cfg)

This skips the org-level ``llm_backend`` key lookup and goes straight to
backend instantiation.


API Key Resolution
~~~~~~~~~~~~~~~~~~

API keys are resolved **exclusively** from per-org configuration, **not**
from environment variables.  The per-org config dict is stored in OpenBao
under each organisation's namespace and fetched at runtime via
:func:`core.org_config.get_org_config`.

The mapping of config keys to provider constructors is:

====================== =================== ====================
Provider               Config key          Constructor parameter
====================== =================== ====================
OpenAI                 ``openai_api_key``  ``api_key``
Azure                  ``azure_api_key``   ``api_key``
Anthropic              ``anthropic_api_key`` ``api_key``
OpenRouter             ``api_key``         ``api_key``
Ollama                 *(none)*            *(none)*
====================== =================== ====================

Ollama is the exception — it requires no API key; only a ``base_url``
pointing to the local instance.

There is **no** env-var fallback anywhere in the LLM backend construction
pipeline.  If a required key is missing from the config,
:class:`LLMConfigurationError` is raised with a message directing the
operator to ``PATCH /admin/org/config``.

.. note::

   The actual storage of these API keys in OpenBao is encrypted using the
   Transit engine (see :class:`core.transit.TransitManager`).  The
   :func:`core.org_config.get_org_config` function returns decrypted values
   at runtime, so the LLM backends always receive plaintext API keys.


Prompt Caching Architecture
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Prompt caching is controlled at three levels:

**Level 1 — Global kill switch** (``OZ_PROMPT_CACHING_ENABLED``)

   Set via :class:`~core.config.Settings.PROMPT_CACHING_ENABLED`.  When
   ``False``, :func:`build_cache_config` immediately returns
   ``PromptCachingConfig(enabled=False)`` regardless of per-org settings.
   This allows operators to disable caching globally without touching every
   org's configuration.

**Level 2 — Per-org overrides**

   Each organisation can set ``prompt_caching.enabled``,
   ``prompt_caching.anthropic_min_tokens``, and
   ``prompt_caching.anthropic_cache_ttl`` in its org config.  These
   override the global defaults.

**Level 3 — Per-call configuration**

   Callers can pass a :class:`PromptCachingConfig` instance to
   :meth:`LLMBackend.chat` for fine-grained control per request.

Provider-specific caching behaviour:

* **OpenAI / Azure**: Automatic prompt prefix caching — no request headers
  needed.  Cache metrics are read from ``prompt_tokens_details`` in the
  response.
* **Anthropic**: Explicit ``cache_control`` markers on system blocks.
  Only the system prompt is cached; conversation messages are not.  Cache
  metrics are read from ``cache_read_input_tokens`` and
  ``cache_creation_input_tokens``.
* **OpenRouter**: Session stickiness (``session_id`` in
  ``extra_body``) so subsequent requests hit the same upstream cache.
  Cache metrics are passed through from the underlying provider.
* **Ollama**: No caching support.  Requests proceed normally with a debug
  log message indicating caching was ignored.


Connection Management
~~~~~~~~~~~~~~~~~~~~~

Each backend manages its own connection:

+------------------+----------------------+-----------+
| Backend          | HTTP client          | Timeout   |
+==================+======================+===========+
| OllamaBackend    | ``httpx.AsyncClient``| 120s      |
|                  | (ephemeral per call) |           |
+------------------+----------------------+-----------+
| OpenAIBackend    | ``openai.AsyncOpenAI``| SDK      |
|                  | (reusable client)    | default   |
+------------------+----------------------+-----------+
| AzureBackend     | ``openai.AsyncAzure- | SDK       |
|                  | OpenAI`` (reusable)  | default   |
+------------------+----------------------+-----------+
| AnthropicBackend | ``anthropic.Async-   | SDK       |
|                  | Anthropic`` (reusable)| default  |
+------------------+----------------------+-----------+
| OpenRouterBackend| ``openai.AsyncOpenAI``| SDK       |
|                  | (reusable, custom    | default   |
|                  | ``base_url``)        |           |
+------------------+----------------------+-----------+

The ephemeral ``httpx.AsyncClient`` in :class:`OllamaBackend` is a
potential performance concern — a new client (including TCP connection) is
created for every ``_chat`` and ``embed`` call.  TODO: needs author
clarification — was this intentional to keep the backend stateless, or
should a persistent client be used?

All backends share a common retry pattern for HTTP 429 (rate limit) and
5xx (server error) responses:

* **Exponential backoff**: ``wait = 2^attempt`` seconds (2s, 4s, 8s).
* **Maximum retries**: 3 (set via ``MAX_RETRIES`` class variable).
* **Non-retryable errors**: Client errors (4xx except 429) are raised
  immediately.
* **Logging**: Every retry and failure is logged with structured context
  (attempt number, status code, wait time, model).


Exception Handling
~~~~~~~~~~~~~~~~~~

Exceptions raised by the LLM abstraction layer are part of the
:class:`core.exceptions.AppError` hierarchy — see
:ref:`core.rst <exception-hierarchy-label>` for the full list.

=============== =============== ===========================================
Exception       HTTP Status     When raised
=============== =============== ===========================================
:class:`LLMConfigurationError`  502 (triggers ``502``)  ``502``  502     502
(actually maps as ``502``)       ``502``   Missing or invalid LLM config
``LLMStructuredOutputError``     ``502``   LLM output failed schema validation
``NotImplementedError``          N/A (Python   ``embed()`` called on a
                                 built-in)    backend that does not support it
``ValueError``                   N/A           Invalid constructor parameters
                                              (missing API key, etc.)
=============== =============== ===========================================

All infrastructure failures (connection refused, DNS failure, timeout)
propagate as the relevant Python exception (``httpx.HTTPStatusError``,
``openai.APIConnectionError``, etc.) and are logged with structured
context for observability.  There is **no** silent degradation — if a
provider is unreachable, the error propagates and the caller sees it.


Dependency Groups
~~~~~~~~~~~~~~~~~

The LLM backends have optional dependency requirements:

* **OpenAI / Azure / OpenRouter**: Depend on ``openai`` library — bundled
  with the base ``openzync`` package.
* **Anthropic**: Requires the ``anthropic`` library.  Install via::

      pip install openzync[llm]

  Or explicitly::

      pip install anthropic>=0.45.0

* **Ollama**: Uses ``httpx`` (already a core dependency) — no additional
  install required.

The ``AnthropicBackend`` module does not fail at import time if the
``anthropic`` library is missing — the import is inside the constructor,
so the error only surfaces when the class is instantiated.  This means
the registry can still list ``"anthropic"`` as an available provider even
when the SDK is not installed.  TODO: needs author clarification — should
this be guarded with a more informative error message?


Adding a New Provider
---------------------

To add a new LLM provider (e.g. Google Gemini, Cohere, Mistral AI):

1. **Create the backend class** in
   ``openzync-core/core/llm_backends.py`` (or a new file imported by it):

   .. code-block:: python

       class GeminiBackend(LLMBackend):
           \"\"\"LLM backend for Google Gemini.\"\"\"

           DEFAULT_MODEL: ClassVar[str] = "gemini-1.5-pro"
           MAX_RETRIES: ClassVar[int] = 3

           def __init__(self, api_key: str, model: str | None = None) -> None:
               if not api_key:
                   raise ValueError("Gemini API key is required")
               self._api_key = api_key
               self._chat_model = model or self.DEFAULT_MODEL

           @property
           def model_name(self) -> str:
               return self._chat_model

           @property
           def embedding_dim(self) -> int:
               return 0  # or 768 if embeddings are supported

           async def _chat(
               self,
               messages: list[dict],
               cache_config: PromptCachingConfig | None = None,
               **kwargs: Any,
           ) -> ChatResponse:
               # Provider-specific implementation
               ...

           async def embed(self, texts: list[str], **kwargs: Any) -> EmbeddingResponse:
               raise NotImplementedError("...")

2. **Register the backend** in the auto-registration block at the bottom
   of ``llm_backends.py``:

   .. code-block:: python

       LLMBackendRegistry.register("gemini", GeminiBackend)

3. **Add config key support** in ``_create_backend`` in ``core/llm.py``.
   Add a new ``elif provider == "gemini":`` block that extracts the
   required parameters from the config dict:

   .. code-block:: python

       elif provider == "gemini":
           if config is None or not config.get("gemini_api_key"):
               raise LLMConfigurationError(
                   "Gemini backend requires gemini_api_key in per-org "
                   "configuration.  Set it via PATCH /admin/org/config."
               )
           api_key = config["gemini_api_key"]
           model = config.get("gemini_model")
           instance = backend_cls(api_key=api_key, model=model)

4. **Add the enum member** (optional) to :class:`LLMProvider` in
   ``core/llm.py`` if you want the new provider to be represented in the
   enum for referential integrity:

   .. code-block:: python

       class LLMProvider(str, Enum):
           OLLAMA = "ollama"
           OPENAI = "openai"
           AZURE = "azure"
           ANTHROPIC = "anthropic"
           GEMINI = "gemini"  # new

   If you skip this step, the provider will still work via the registry but
   will not have an enum member — this is the current state of OpenRouter.

5. **Check optional dependencies** — if the new provider requires a
   third-party SDK, make sure the import is inside the constructor
   (lazy), not at the top of the module, so the registry loads
   successfully even when the SDK is not installed:

   .. code-block:: python

       class GeminiBackend(LLMBackend):
           def __init__(self, api_key: str, model: str | None = None) -> None:
               if not api_key:
                   raise ValueError("Gemini API key is required")
               from google.genai import AsyncClient  # lazy import
               self._client = AsyncClient(api_key=api_key)
               ...

6. **Test the new backend** by running at least:

   * Unit tests for the constructor (valid/invalid API keys).
   * Unit tests for ``_chat`` (mock the SDK client).
   * Integration test with :func:`resolve_backend` and the registry.
   * End-to-end test with the actual provider (requires credentials).

The backend will then be selectable via per-org config by setting
``llm_backend`` to ``"gemini"`` and providing the required config keys.
