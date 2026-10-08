# fleeex parameter surface (`POST /v1/chat/completions`)

Rule: a parameter that would change the answer, the bill, or an implied guarantee is a
`400` **naming the parameter**. A parameter that changes nothing observable is accepted and
ignored. Authoritative source: the route's OpenAPI (https://docs.fleeex.dev/api-reference/parameters).

## Honored

| Parameter | Notes |
|---|---|
| `model` | Catalog id or alias (prefer aliases, e.g. `nova-lite`). |
| `messages` | Roles `system`, `user`, `assistant`, `tool`. `content`: string or `text` / `image_url` parts. `tool_call_id`, replayed `tool_calls`. |
| `tools`, `tool_choice` | `'auto'`, `'required'`, or `{ type: 'function', function: { name } }`. |
| `response_format` | `{ type: 'json_schema', json_schema: { name, schema } }`. |
| `max_completion_tokens` / `max_tokens` | 1 to 32,768, default 1024. Also sizes the wallet **reservation**: an oversized value can cause a `402` on a wallet that would cover the real answer. |
| `temperature` | 0 to 2. |
| `top_p` | 0 to 1. |
| `stop` | String or up to 4 strings. |
| `stream`, `stream_options.include_usage` | SSE, iso-OpenAI chunks; usage chunk last. |

## Accepted, ignored

`user`, `metadata`, `store`, `n: 1`, `logit_bias: {}`, `frequency_penalty: 0`,
`presence_penalty: 0`, `logprobs: false`, `response_format: { type: 'text' }`,
`image_url.detail: 'auto'`, `json_schema.strict`. An explicit `null` reads as unset.

## Rejected (400), and the fix

| Sent | Fix |
|---|---|
| `seed` | Remove it. |
| `n` > 1 | Make several requests. |
| non-empty `logit_bias`, non-zero penalties | Remove them. |
| `logprobs: true` | Remove it. |
| `tool_choice: 'none'` | Send the request without `tools`. |
| `tool_choice` without `tools` | Remove `tool_choice`. |
| `function.strict` or any undeclared tool sub-field | Remove it. |
| `response_format: { type: 'json_object' }` | Use `json_schema` with an explicit schema. |
| `image_url.url` not a `data:` URI | Fetch server-side and send `data:image/<png\|jpeg\|gif\|webp>;base64,...`. |
| `image_url.detail` other than `'auto'` | Remove it. |
| `stream_options` without `stream: true` | Remove it. |
| unsupported model | Pick from `GET /v1/models`. |
| anything else not listed | Remove it. |

## Bounds (400 when exceeded)

| | Bound |
|---|---|
| Body | 1 MB (`413` above), nesting 32 levels |
| `messages` | 1 to 200 |
| `content` | 256,000 chars per message or text part, 20 parts |
| Images | 8 per request, `user` messages only, 512 KB decoded each |
| `tools` | 128; `name` 64 chars `[A-Za-z0-9_-]`; `description` 4,096; `parameters` 16,384 chars, 10 levels |
| `tool_calls` | 32 per message; `arguments` 32,768 chars; `tool_call_id` 128 |
| `json_schema` | `name` 64 chars `[A-Za-z0-9_-]`; `schema` 16,384 chars, 10 levels; `description` 4,096 |
