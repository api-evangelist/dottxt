---
name: Extract structured data with dottxt
description: >-
  Use the dottxt (.txt) OpenAI-compatible API to turn unstructured text into
  JSON that is guaranteed to satisfy a JSON Schema, using constrained decoding.
api: openapi/dottxt-platform-openapi.yml
operations:
  - createChatCompletion
  - listModels
---

# Extract structured data with dottxt

dottxt guarantees the model output matches your JSON Schema **by construction** —
no retries, no validation loops, no defensive parsing.

## Auth
- Base URL: `https://api.dottxt.ai/v1`
- Header: `Authorization: Bearer $DOTTXT_API_KEY` (keys start with `sk-dottxt-`)
- Also send `Content-Type: application/json`

## Steps

1. **Pick a model** — call `listModels` (`GET /models`) to see models available to
   your key (e.g. `openai/gpt-oss-20b`).
2. **Define the target schema** — write a JSON Schema for the object you want.
   Set `additionalProperties: false` and mark the fields you require. The schema
   is a *generation program*, not a formatting hint.
3. **Call `createChatCompletion`** (`POST /chat/completions`) with:
   - `model`: the chosen model id
   - `messages`: your prompt (system + user)
   - `response_format`: `{ "type": "json_schema", "json_schema": { "name": "...", "schema": { ... } } }`
   The returned `choices[0].message.content` is JSON that satisfies your schema.
4. **(Optional) Stream** — set `stream: true` for token deltas (SSE), or
   `stream: "patch"` to receive RFC 6902 JSON Patch events field-by-field as the
   object is built (NDJSON, or SSE with `Accept: text/event-stream`).

## Example

```bash
curl https://api.dottxt.ai/v1/chat/completions \
  -H "Authorization: Bearer $DOTTXT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-oss-20b",
    "messages": [{ "role": "user", "content": "Extract: John Smith <john@acme.com>, VP Engineering" }],
    "response_format": {
      "type": "json_schema",
      "json_schema": {
        "name": "contact",
        "schema": {
          "type": "object",
          "properties": {
            "name":  { "type": "string", "minLength": 1 },
            "email": { "type": "string", "pattern": "^[^@]+@[^@]+$" },
            "role":  { "type": "string" }
          },
          "required": ["name", "email"],
          "additionalProperties": false
        }
      }
    }
  }'
```

## Rules & error handling
- Because output is schema-guaranteed, **do not** add JSON-repair or retry loops.
- Use `seed` for deterministic sampling.
- Errors use the OpenAI envelope `{ "error": { code, message, type, param } }`:
  `401` invalid key, `402` insufficient credits, `403` no model access,
  `404` model not found, `429` rate limited, `400` bad request. See
  `errors/dottxt-problem-types.yml`.
