---
title: ngrok AI Gateway
description: Call models through the ngrok AI Gateway with the LiteLLM Python SDK or Gateway. Configure your access key, catalog model IDs, provider pinning, and models you run yourself.
---

# ngrok AI Gateway

The [ngrok AI Gateway](https://ngrok.ai) gives you one access key for models from many providers, with failover between providers and keys, spend tracking, and routing to models you run yourself (Ollama, vLLM, LM Studio). Use the `ngrok/` prefix with a model ID from the [ngrok model catalog](https://ngrok.com/docs/ai-gateway/reference/model-catalog).

Provider support was added in [PR #PENDING](https://github.com/BerriAI/litellm/pulls). Use a build that includes it.

## Overview

| Property | Details |
|----------|---------|
| Provider prefix | `ngrok/` |
| API key environment variable | `NGROK_AI_GATEWAY_KEY` |
| Default API base | `https://gateway.ngrok.ai/v1` |
| Optional API base override | `NGROK_AI_GATEWAY_BASE_URL` or the `api_base` parameter |
| Upstream endpoints | `/chat/completions`, `/responses`, `/messages`, `/embeddings` |
| Provider documentation | [ngrok AI Gateway docs](https://ngrok.com/docs/ai-gateway/overview) |

## Setup

Create an access key at [app.ngrok.ai/keys](https://app.ngrok.ai/keys) and export it before you start the SDK or proxy:

```bash
export NGROK_AI_GATEWAY_KEY="ng-xxxxx-g1-xxxxx"
```

Model IDs keep the gateway's catalog format after the `ngrok/` prefix. `ngrok/anthropic/claude-sonnet-5-5` sends `anthropic/claude-sonnet-5-5` to the gateway, and the gateway picks a provider that serves it. To pin one provider, use `provider:model`, for example `ngrok/openai:gpt-5.5`. `GET https://gateway.ngrok.ai/v1/models` lists the models your access key can call.

The ngrok platform already uses `NGROK_API_KEY` for its own API, so this provider reads `NGROK_AI_GATEWAY_KEY` instead. Don't put an ngrok platform API key there.

## LiteLLM Python SDK

```python
from litellm import completion

response = completion(
    model="ngrok/anthropic/claude-sonnet-5-5",
    messages=[{"role": "user", "content": "Say hello in five words"}],
)

print(response.choices[0].message.content)
```

For streaming, pass `stream=True` and iterate over the chunks:

```python
from litellm import completion

response = completion(
    model="ngrok/openai/gpt-5.5",
    messages=[{"role": "user", "content": "Say hello in five words"}],
    stream=True,
)

for chunk in response:
    print(chunk.choices[0].delta.content or "", end="")
```

Embeddings use the same prefix:

```python
from litellm import embedding

response = embedding(
    model="ngrok/amazon/titan-embed-text-v2",
    input=["ngrok AI Gateway"],
)
```

## LiteLLM Gateway

Save this configuration as `config.yaml`:

```yaml
model_list:
  - model_name: claude-sonnet
    litellm_params:
      model: ngrok/anthropic/claude-sonnet-5-5
      api_key: os.environ/NGROK_AI_GATEWAY_KEY
  - model_name: gpt-5.5
    litellm_params:
      model: ngrok/openai:gpt-5.5
      api_key: os.environ/NGROK_AI_GATEWAY_KEY
```

Start the proxy with the access key exported in its environment:

```bash
litellm --config config.yaml
```

Send a request using a configured model alias:

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet",
    "messages": [{"role": "user", "content": "Say hello in five words"}]
  }'
```

If your proxy requires authentication, also send `Authorization: Bearer <your-litellm-key>`. This is your LiteLLM key, not your ngrok access key.

Responses API and Anthropic Messages requests for `ngrok/` models go to the gateway's own `/responses` and `/messages` endpoints. Those endpoints only reach providers that support them, so check the catalog's surfaces for a model before you use them. Chat completions work for every chat model.

## Models you run yourself

If you add your own model to the gateway, such as Ollama on a machine behind an ngrok endpoint, call it with the provider ID you gave it in the gateway, as `provider:model`, for example `ngrok/my-ollama:llama3.2`. See [Use a model you run yourself](https://ngrok.com/docs/ai-gateway/guides/use-a-model-you-run-yourself).
