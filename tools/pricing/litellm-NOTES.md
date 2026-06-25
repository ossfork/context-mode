# LiteLLM Catalog — Adapter Notes

Vendored from BerriAI/litellm as context-mode's comprehensive pricing base, so
unknown/unseen models still resolve a price instead of failing.

- **Source:** `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json`
- **Local file:** `litellm-catalog.json`
- **Size:** ~1.53 MB (1,570,159 bytes)
- **Top-level keys:** 2,901 (2,900 model entries + 1 `sample_spec` template — skip `sample_spec`)
- **Entries with numeric `input_cost_per_token`:** 2,432
- **Distinct `litellm_provider` values:** 120 (top: fireworks_ai, bedrock, openai, azure, gemini, mistral, openrouter …)

## Shape

The file is a flat JSON object. **Each key is a model id**; each value is a metadata object.
There is no wrapper array. One special key, `sample_spec`, is a documentation template
(not a real model) and must be excluded.

```jsonc
{
  "sample_spec": { /* template — ignore */ },
  "gpt-4o": { "input_cost_per_token": 0.0000025, ... },
  "claude-sonnet-4-20250514": { ... }
}
```

## Cost fields — verified present in the fetched JSON

> All `*_cost_per_token` values are **PER TOKEN** (USD), not per-million.
> To convert to per-Mtok (context-mode's internal unit): **multiply by `1e6` (×1,000,000).**
> e.g. `gpt-4o` `input_cost_per_token = 0.0000025` → `0.0000025 × 1e6 = $2.50 / Mtok`.

Core fields (use these; coverage count in parentheses):

| Field | Meaning | Count |
|-------|---------|-------|
| `input_cost_per_token` | prompt cost per token | 2,433 |
| `output_cost_per_token` | completion cost per token | 2,430 |
| `cache_read_input_token_cost` | cached-prompt read cost per token | 645 |
| `cache_creation_input_token_cost` | cache-write/creation cost per token | 210 |

Real example (`claude-sonnet-4-20250514`, provider `anthropic`):
`input_cost_per_token: 0.000003` (= $3.00/Mtok), `output_cost_per_token: 0.000015` (= $15.00/Mtok),
`cache_read_input_token_cost: 3e-7` (= $0.30/Mtok), `cache_creation_input_token_cost: 0.00000375` (= $3.75/Mtok).

### Schema surprises / gotchas

- **Tiered & variant cost fields exist** — e.g. `input_cost_per_token_above_200k_tokens`,
  `cache_read_input_token_cost_above_200k_tokens`, `_priority`, `_batches`, `_flex`,
  `_above_1hr`, `_above_272k_tokens`. Treat these as optional overrides; fall back to the
  base `input_cost_per_token` / `output_cost_per_token`.
- **Non-token cost units also appear** and are NOT per-token: `input_cost_per_second`,
  `input_cost_per_character`, `input_cost_per_image`, `input_cost_per_pixel`,
  `output_cost_per_second`, `output_cost_per_reasoning_token`,
  `search_context_cost_per_query` (the last can be an object keyed by search-context size).
  Do not blindly ×1e6 these — only the `*_per_token` family is per-token.
- **2,366 of 2,900 keys contain `/`** — namespaced ids like `bedrock/...`,
  `1024-x-1024/dall-e-2`, image/resolution-prefixed entries. Match on the full key.
- Metadata fields used for context limits: `max_tokens`, `max_input_tokens`,
  `max_output_tokens` (and `mode` distinguishes `chat`, `embedding`, image, etc.).
- Some entries carry `deprecation_date` (81 entries).

## Lookup strategy (for the adapter)

Given an incoming `model_id`:

1. **Exact match** — `catalog[model_id]`. Fastest; covers the common case.
2. **Provider-stripped / namespaced fallback** — many ids are `provider/model`.
   Try stripping or adding a known provider prefix:
   - if `model_id` has no `/`, try `catalog[provider + "/" + model_id]`;
   - if `model_id` is `provider/model`, also try the bare `model` segment.
3. **Provider + model match** — scan entries whose `litellm_provider` matches the resolved
   provider and whose key endsWith the model segment.
4. **Skip `sample_spec`** in every scan.
5. From the matched entry, read `input_cost_per_token` / `output_cost_per_token`
   (+ optional `cache_read_input_token_cost`, `cache_creation_input_token_cost`),
   then **× 1e6** to get per-Mtok rates.
6. If nothing matches or `input_cost_per_token` is absent (some entries price only by
   second/character/image), the model has no usable per-token price — fall through to
   context-mode's own default.
