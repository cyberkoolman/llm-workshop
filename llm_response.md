# Understanding Azure OpenAI API Response

## HTTP Response Headers

### Rate Limiting Headers
| Header | Value | Explanation |
|--------|-------|-------------|
| `x-ratelimit-key` | `gpt-4o` | The model/deployment the rate limit applies to |
| `x-ratelimit-limit-requests` | `180` | Max requests allowed per renewal period |
| `x-ratelimit-limit-tokens` | `30,000` | Max tokens (input + output) allowed per renewal period |
| `x-ratelimit-remaining-requests` | `179` | Requests remaining before hitting the limit |
| `x-ratelimit-remaining-tokens` | `29,986` | Tokens remaining before hitting the limit |
| `x-ratelimit-renewalperiod-requests` | `10` | Seconds until request quota resets |
| `x-ratelimit-renewalperiod-tokens` | `60` | Seconds until token quota resets |
| `x-ratelimit-reset-requests` | `0` | Seconds until request limit resets (0 = not throttled) |
| `x-ratelimit-reset-tokens` | `0` | Seconds until token limit resets (0 = not throttled) |
| `x-ratelimit-abusepenalty-active` | `False` | Whether the caller is currently penalized for abuse |

> **Key takeaway:** When `remaining-requests` or `remaining-tokens` hits 0, you get a **429 Too Many Requests** error. The `renewalperiod` tells you how often the quota refills.

### Request Routing & Metadata
| Header | Value | Explanation |
|--------|-------|-------------|
| `x-ms-region` | `South Central US` | Azure region that served the request |
| `x-ms-deployment-name` | `gpt-4o` | The deployment name used |
| `x-ms-served-model` | `gpt-4o-2024-11-20` | Actual model version behind the deployment |
| `x-request-id` | `fd43d712-...` | Unique ID for this request (useful for support tickets) |
| `apim-request-id` | `0b0134a4-...` | API Management layer request ID |

### Content Safety & Streaming
| Header | Value | Explanation |
|--------|-------|-------------|
| `x-ms-rai-invoked` | `true` | Responsible AI content filters were applied |
| `azureai-fe-is-streaming` | `False` | Whether this is a streaming response |
| `x-ms-is-spilled-over` | `false` | Whether request spilled over to another region for load balancing |

---

## JSON Response Body

### `choices[]` — The Model's Response

```json
"choices": [{
  "index": 0,
  "finish_reason": "stop",
  "message": { "role": "assistant", "content": "..." }
}]
```

| Field | Explanation |
|-------|-------------|
| `index` | Which choice this is (relevant when `n > 1`) |
| `finish_reason` | Why the model stopped: `stop` (natural end), `length` (hit max_tokens), `content_filter` (blocked) |
| `message.content` | The actual generated text |
| `message.refusal` | If the model refused to answer, the reason appears here |

### `content_filter_results` — Per-Choice Safety Scores

Applied to **both** the prompt (input) and each choice (output):

| Category | What it detects |
|----------|----------------|
| `hate` | Hate speech or discriminatory content |
| `self_harm` | Self-harm related content |
| `sexual` | Sexual content |
| `violence` | Violent content |
| `protected_material_code` | Copyrighted code |
| `protected_material_text` | Copyrighted text |
| `indirect_attack` | Prompt injection / jailbreak attempts (input only) |
| `jailbreak` | Direct jailbreak attempts (input only) |

Each has `severity` (safe/low/medium/high) and `filtered` (true = blocked).

### `usage` — Token Consumption

```json
"usage": {
  "prompt_tokens": 14,
  "completion_tokens": 24,
  "total_tokens": 38
}
```

| Field | Explanation |
|-------|-------------|
| `prompt_tokens` | Tokens consumed by your input message |
| `completion_tokens` | Tokens generated in the response |
| `total_tokens` | Sum of both — **this is what counts toward your rate limit and billing** |
| `cached_tokens` | Tokens served from prompt cache (cheaper, faster) |

### `usage.completion_tokens_details` — Breakdown of Output Tokens

| Field | Explanation |
|-------|-------------|
| `reasoning_tokens` | Tokens used for chain-of-thought (o1/o3 models) |
| `audio_tokens` | Tokens for audio output (multimodal) |
| `accepted_prediction_tokens` | Predicted tokens that were accepted (speculative decoding) |
| `rejected_prediction_tokens` | Predicted tokens that were rejected |

### `usage.latency_checkpoint` — Performance Metrics

| Field | Explanation |
|-------|-------------|
| `engine_ttft_ms` | Time to first token at the model engine |
| `engine_tbt_ms` | Time between tokens at the model engine |
| `engine_ttlt_ms` | Time to last token at the model engine |
| `service_ttft_ms` | Time to first token including service overhead |
| `service_tbt_ms` | Time between tokens including service overhead |
| `service_ttlt_ms` | Time to last token including service overhead |
| `pre_inference_ms` | Time spent before inference (routing, content filtering) |
| `user_visible_ttft_ms` | Time to first token as perceived by the caller |
| `total_duration_ms` | Total end-to-end request duration |

### Top-Level Metadata

| Field | Value | Explanation |
|-------|-------|-------------|
| `id` | `chatcmpl-E1g1X...` | Unique completion ID |
| `model` | `gpt-4o-2024-11-20` | Model version that generated the response |
| `created` | `1784068575` | Unix timestamp of when the response was created |
| `service_tier` | `default` | Service tier (default vs provisioned throughput) |
| `system_fingerprint` | `fp_40854b0bd2` | Model configuration snapshot (changes when model updates) |

---

## Rate Limiting in Practice

```
Your deployment: 30,000 TPM (tokens per minute) / 180 RPM (requests per minute)
```

**How 429 happens:**
1. Each request consumes `total_tokens` from your token budget
2. Each request consumes 1 from your request budget
3. Budgets refill every `renewalperiod` seconds
4. When either budget hits 0 → **HTTP 429 Too Many Requests**
5. The 429 response includes `Retry-After` header telling you how long to wait

**Best practices:**
- Implement exponential backoff with jitter
- Monitor `x-ratelimit-remaining-*` headers proactively
- Use lower `max_tokens` when you don't need long responses
- Avoid `n > 1` unless necessary (multiplies token usage)
