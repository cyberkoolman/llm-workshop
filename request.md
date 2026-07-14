# Azure OpenAI Chat Completion — Request Parameters

## Basic Request Structure

```json
{
  "messages": [...],
  "max_tokens": 1000,
  "temperature": 0.7,
  "top_p": 1.0,
  "n": 1,
  "stream": false,
  "stop": null,
  "presence_penalty": 0,
  "frequency_penalty": 0,
  "seed": null,
  "response_format": { "type": "text" }
}
```

---

## Messages Array

The conversation history. Each message has a `role` and `content`.

| Role | Purpose | Example |
|------|---------|---------|
| `system` | Sets the AI's behavior, persona, and constraints | "You are a helpful assistant that responds in JSON." |
| `user` | The human's input/question | "What is rate limiting?" |
| `assistant` | Previous AI responses (for multi-turn context) | "Rate limiting is..." |

**Why it matters:** The model is stateless — you must send the full conversation each time. This is why token costs grow with conversation length.

---

## Core Parameters

### `max_tokens`
| | |
|--|--|
| **What** | Maximum number of tokens the model can generate in its response |
| **Default** | Model-dependent (often 4096) |
| **Range** | 1 to model's context limit |
| **Use when** | You want to control cost/length. Lower = cheaper & faster. |
| **Tip** | Set it just high enough for your use case. A summarizer might need 200, a code generator might need 4000. |

### `temperature`
| | |
|--|--|
| **What** | Controls randomness/creativity of the output |
| **Default** | 1.0 |
| **Range** | 0.0 to 2.0 |
| **Low (0.0–0.3)** | Deterministic, focused, consistent — good for factual Q&A, code, data extraction |
| **Medium (0.5–0.8)** | Balanced — good for general conversation |
| **High (1.0–2.0)** | Creative, diverse, unpredictable — good for brainstorming, poetry, creative writing |
| **Tip** | Use 0 for production where consistency matters. Use higher for ideation. |

### `top_p` (nucleus sampling)
| | |
|--|--|
| **What** | Only considers tokens within the top P probability mass |
| **Default** | 1.0 |
| **Range** | 0.0 to 1.0 |
| **Example** | `top_p: 0.1` = only consider the top 10% most likely tokens |
| **Tip** | Alter `temperature` OR `top_p`, not both. They serve similar purposes. |

### `n`
| | |
|--|--|
| **What** | Number of completions to generate per request |
| **Default** | 1 |
| **Use when** | You want multiple candidates to pick the best one |
| **Impact** | Multiplies token usage and cost by n. A request with `n: 5` and `max_tokens: 1000` could consume up to 5000 completion tokens. |
| **Real-world uses** | Majority voting for accuracy, generating content variants, picking best code solution |
| **Tip** | Most production apps use `n: 1`. Use higher n for quality-critical one-off tasks. |

---

## Control Parameters

### `stop`
| | |
|--|--|
| **What** | Up to 4 sequences that will stop generation when encountered |
| **Default** | null |
| **Example** | `"stop": ["\n\n", "END", "###"]` |
| **Use when** | You need the model to stop at a specific boundary (e.g., generating one function at a time) |

### `stream`
| | |
|--|--|
| **What** | Whether to stream partial responses as server-sent events |
| **Default** | false |
| **Use when** | Building a chat UI — users see tokens appear in real-time instead of waiting for the full response |
| **Tip** | Improves perceived latency dramatically. The actual total time is similar, but users feel it's faster. |

---

## Repetition & Diversity Controls

### `presence_penalty`
| | |
|--|--|
| **What** | Penalizes tokens that have already appeared, encouraging new topics |
| **Default** | 0 |
| **Range** | -2.0 to 2.0 |
| **Positive** | Model avoids repeating any topic it already mentioned |
| **Use when** | You want broad, diverse responses that cover many topics |

### `frequency_penalty`
| | |
|--|--|
| **What** | Penalizes tokens proportional to how often they've appeared |
| **Default** | 0 |
| **Range** | -2.0 to 2.0 |
| **Positive** | Model avoids repeating the same words/phrases |
| **Use when** | You want to reduce verbatim repetition (e.g., "the the the") |

> **Difference:** `presence_penalty` = "don't revisit topics", `frequency_penalty` = "don't repeat words"

---

## Determinism & Reproducibility

### `seed`
| | |
|--|--|
| **What** | Integer seed for deterministic output |
| **Default** | null (random) |
| **Use when** | You need reproducible results for testing, debugging, or compliance |
| **Tip** | Same seed + same parameters + same prompt = same output (mostly). Check `system_fingerprint` in response — if it changes, the model was updated and outputs may differ. |

---

## Output Format

### `response_format`
| | |
|--|--|
| **What** | Constrains the output format |
| **Options** | `{"type": "text"}` (default), `{"type": "json_object"}`, `{"type": "json_schema", "json_schema": {...}}` |
| **json_object** | Forces valid JSON output — model will always return parseable JSON |
| **json_schema** | Forces output to match a specific schema (structured outputs) |
| **Use when** | You're parsing the response programmatically. Eliminates JSON parsing errors. |
| **Tip** | When using `json_object`, include "respond in JSON" in your system message. |

---

## Advanced Parameters

### `tools` / `tool_choice`
| | |
|--|--|
| **What** | Define functions the model can call (function calling) |
| **Use when** | The model needs to interact with external systems (APIs, databases, calculations) |
| **Example** | Weather lookup, database query, calculator |

### `logprobs`
| | |
|--|--|
| **What** | Returns the log probability of each output token |
| **Use when** | You need confidence scores, building classifiers, or debugging model behavior |

### `logit_bias`
| | |
|--|--|
| **What** | Manually adjust the likelihood of specific tokens appearing |
| **Range** | -100 (ban) to 100 (guarantee) |
| **Use when** | You want to ban certain words or force specific tokens |

---

## Parameter Combinations for Common Use Cases

| Use Case | temperature | max_tokens | n | Other |
|----------|-------------|------------|---|-------|
| **Factual Q&A** | 0 | 500 | 1 | `seed` for reproducibility |
| **Code generation** | 0–0.2 | 4000 | 1 | `stop: ["\n\n\n"]` |
| **Creative writing** | 1.0–1.5 | 2000 | 1 | `presence_penalty: 0.6` |
| **Data extraction** | 0 | 1000 | 1 | `response_format: json_schema` |
| **Brainstorming** | 1.2 | 1000 | 3 | Pick best from 3 options |
| **Chat UI** | 0.7 | 1000 | 1 | `stream: true` |
| **Classification** | 0 | 10 | 1 | `logprobs: true` for confidence |

---

## Cost & Rate Limit Impact

| Parameter | Impact on tokens billed |
|-----------|------------------------|
| `max_tokens` | Caps completion tokens (but actual usage may be less) |
| `n` | Multiplies completion tokens by n |
| Long `messages` history | Increases prompt tokens every turn |
| `stream` | No cost difference — same tokens, just delivered incrementally |
| `response_format: json` | Slightly more tokens (JSON syntax overhead) |

> **Remember:** Both prompt tokens AND completion tokens count toward your TPM rate limit. The response header `x-ratelimit-remaining-tokens` shows your remaining budget.
