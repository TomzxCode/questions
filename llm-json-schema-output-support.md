# Which LLM harnesses support returning a JSON matching a provided schema?

## Answer

Support falls into two mechanisms: constrained decoding, where tokens are masked to guarantee schema-valid output, and prompt plus validate plus retry, which is reliable but not guaranteed. Almost all harnesses accept a JSON Schema directly or a typed model (Pydantic, Zod) compiled to one.

### Managed APIs with native JSON Schema enforcement

- OpenAI and Azure OpenAI: Structured Outputs via `response_format: {type: "json_schema", strict: true}`, plus strict schemas on tool/function calls.
- Anthropic: `output_config.format` with `type: "json_schema"` in the Messages API (GA since the `structured-outputs-2025-11-13` beta), plus `strict: true` tool use, enforced through constrained decoding.
- Google Gemini: controlled generation via `response_mime_type: "application/json"` plus `response_schema`.
- Mistral: `response_format: {type: "json_schema"}`.
- Cohere: JSON Schema mode via `response_format: {type: "json_object", schema: ...}`, plus `strict_tools` for tool calls.
- xAI (Grok): `json_schema` response format.
- Groq: `json_schema` response format.
- Together AI: `response_format: {type: "json_schema"}` on supported models, plus regex mode.
- Fireworks AI: `response_format: json_schema` backed by a grammar engine.
- OpenRouter: forwards `json_schema` to whichever upstream provider supports it.

### Local inference engines

- vLLM: structured outputs (`guided_json` / `response_format`), with backends including xgrammar, outlines, and guidance.
- SGLang: JSON-schema-constrained decoding.
- llama.cpp: GBNF grammars, convertible from a JSON Schema.
- Ollama: structured outputs by passing a JSON Schema in `format`.
- LM Studio: JSON Schema structured output option.

### Client libraries and frameworks (multi-provider)

- Instructor (Python, TypeScript, and more): Pydantic or Zod models with validation and retries over many providers.
- LangChain: `with_structured_output`, enforcement strength depends on the provider.
- LlamaIndex: Pydantic structured output programs.
- LiteLLM: normalizes `response_format` json_schema across providers.
- PydanticAI: typed agent outputs.
- BAML: schema-first prompt language with validation and retries.
- Mirascope: typed structured outputs.
- DSPy: typed output fields on predictors.
- OpenAI Agents SDK: `output_type` structured final outputs.
- Microsoft Semantic Kernel: structured output support across model connectors.

### Token-level constrained generation libraries (local models)

- Outlines: compiles a JSON Schema into a finite state machine for logit masking.
- XGrammar: efficient JSON-Schema-to-grammar compiler, used as a backend by vLLM and SGLang.
- Guidance: grammar and template-based generation.
- LM Format Enforcer: JSON Schema and regex constrained decoding wrapper.

### Caveats

- Enforcing providers support only a JSON Schema subset, commonly excluding `allOf`, `oneOf`, `not`, and numeric range keywords.
- Some OpenAI-compatible endpoints (for example DeepSeek) only offer `json_object` mode, valid JSON but not schema-matched.
- Validation-retry libraries (Instructor, BAML) work with any model but do not guarantee validity, while constrained decoding does.

## Follow-up: opencode and pi

Neither coding-agent harness offers schema-constrained final responses; support exists only around tool calls.

### opencode

- No structured output feature. There is no way to ask the server, SDK, or `opencode run` for a response guaranteed to match a provided schema.
- `opencode run --format json` and the server API return the harness's own fixed JSON event/message format, not schema-constrained model output.
- Custom tools (including MCP tools) are defined with JSON Schemas, so tool arguments are schema-described, but that does not constrain the assistant's reply.
- Workarounds: prompt for JSON and parse/validate yourself, or use a plugin/custom agent that calls a provider SDK or a library like Instructor directly.

### pi (badlogic/pi-mono, `@earendil-works/pi-ai` / `pi-agent-core`)

- Tools are defined with TypeBox schemas and validated via `validateToolCall`, with partial-JSON streaming of tool arguments.
- `constrainedSampling: { type: "json_schema", strict: "prefer" | "require" }` opts tool calls into provider-side strict schema enforcement (OpenAI, Anthropic, Bedrock Converse, Mistral, Gemini 3); `strict: "require"` fails when unsupported, `"prefer"` falls back.
- Also supports OpenAI grammar-constrained custom tools (Lark or regex variants) on GPT-5+ class models.
- No `generateObject`-style API constraining the final assistant message to a schema; the common pattern is defining a no-op tool whose arguments carry the structured payload, enforced via `constrainedSampling`.

## Follow-up: single-choice questions and multi-question collection

The proposed pattern: decompose a structured output into N questions, each "pick one option from a list", then aggregate the answers into the final output. Two building blocks: choice-constrained generation, and app-level aggregation.

### Choice from a list as a dedicated primitive

- vLLM: `guided_choice` parameter, exactly this use case.
- Outlines: `generate.choice(model, ["A", "B", ...])`.
- Guidance: `select([...])` within a template.
- Together AI: regex mode, `response_format: {type: "regex", pattern: "(a|b|c)"}`.
- llama.cpp: GBNF grammar listing the alternatives.
- SGLang: regex-constrained decoding.
- LM Format Enforcer: character-level parsers, including regex.

### Choice via `enum` inside a JSON schema

Works on every schema-enforcing provider from the main answer:

- OpenAI, Anthropic (enum supported; note it does not guarantee capitalization of values, compare case-insensitively), Google Gemini, Cohere (enum confirmed supported), Mistral, Groq, xAI, Fireworks, Ollama, LM Studio.
- So the minimal schema `{"type": "object", "properties": {"answer": {"type": "string", "enum": [...]}}}` gets constrained single-choice everywhere.

### Multi-question collection

- No harness provides this as a built-in feature; it is application logic: one constrained call per question, then aggregate. Every harness above supports it.
- Guidance is the exception in that one template can chain multiple selections and text fills in a single generation.
- Instructor, PydanticAI, and LangGraph help structure the loop and validate the aggregated final object.
- Crucially, this pattern also works on harnesses with no schema support at all, including opencode and pi's agent layer: ask each question as plain text ("answer with exactly one of: a, b, c"), match the reply against the allowed set, and re-ask on mismatch. Single-choice answers are trivially validated in application code, so no constrained decoding is needed.

## Follow-up: collecting output via a script/tool call

The proposed pattern: expose a script as a tool, and treat its call arguments as the structured output. The schema on the tool parameters does the constraining, and the script receives validated arguments.

### Who supports it

- **Every harness with JSON-schema tool calling**, which is nearly all of them:
  - Managed APIs: OpenAI (strict tool mode), Anthropic (`strict: true` tool use), Gemini, Cohere (`strict_tools`), Mistral, Groq, xAI, Together, Fireworks, OpenRouter (forwards tools upstream).
  - Local: vLLM, Ollama, llama.cpp server, LM Studio (tool support varies by model).
  - Agent frameworks: LangChain, LlamaIndex, OpenAI Agents SDK, PydanticAI, Semantic Kernel, smolagents.
- **pi**: this is the idiomatic pattern. `AgentTool` with a TypeBox schema, argument validation, execution, and optional `constrainedSampling` strict enforcement. A no-op tool whose execute just records args equals structured output.
- **opencode**: expose the script as a custom tool or wrap it in an MCP server; tool arguments arrive schema-described, and the harness (or your MCP client) executes the script with them.

### Caveats

- Without strict mode, tool arguments are schema-described but not guaranteed; validate and return errors so the model retries (pi's `validateToolCall` plus `isError` tool results is the canonical loop).
- Strict tool mode (OpenAI, Anthropic, Cohere `strict_tools`, pi `constrainedSampling`) closes that gap: arguments are constrained-decoded to match the schema.
- This is not a hack, it is the substrate: OpenAI's and Anthropic's structured outputs are internally implemented as strict tool calling.
