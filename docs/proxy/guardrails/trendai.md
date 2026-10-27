# TrendAI Guard

Use TrendAI Vision One AI Guard to check prompt and response text that passes through LiteLLM Proxy. TrendAI policies decide whether to allow, block, or redact the text.

:::note Availability

The built-in integration is proposed in [LiteLLM PR #41254](https://github.com/BerriAI/litellm/pull/41254). Use a LiteLLM build or image that includes it. A Helm chart upgrade alone does not add the integration to an older proxy image.

:::

This page uses the built-in provider, `guardrail: trendai`. The [upstream plugin README](https://github.com/trendmicro/tm-v1-ai-guard-litellm-plugin#readme) describes the separate `trendai_guard.TrendAIGuardrail` plugin. The built-in provider does not need a plugin file mount or the plugin's Helm overrides.

## Prerequisites

### Create a Vision One API token

In the Vision One console, open **Workflow and Automation > Third-Party Integrations > LiteLLM**, then select **Generate Token**. Store the token as `TMV1_API_KEY`, or use a configured [LiteLLM secret manager](../../secret.md).

Use a token and endpoint from the same Vision One region. Confirm that the policies you want to enforce are assigned to the application name you send as `app_name`.

### Find your AI Guard endpoint

For a TrendAI-hosted deployment, copy **Endpoint URL** from the LiteLLM third-party integration page in Vision One.

For a self-hosted AWS deployment, open the AI Guard CloudFormation stack, select **Outputs**, and copy `GuardAPIEndpoint`.

Set the value as `TRENDAI_AI_GUARD_BASE_URL`. A hosted endpoint can include the product path, for example `https://api.xdr.trendmicro.com/v3.0/aiSecurity`. LiteLLM adds `/applyGuardrails` when needed, so a URL that already ends with that path is also accepted.

## Quick start

### Set credentials

This example uses an OpenAI model. Use the credentials required by your model provider if you choose a different model.

```bash
export OPENAI_API_KEY="<your-model-provider-key>"
export TMV1_API_KEY="<your-Vision-One-token>"
export TRENDAI_AI_GUARD_BASE_URL="https://api.xdr.trendmicro.com/v3.0/aiSecurity"
export LITELLM_MASTER_KEY="sk-$(openssl rand -hex 32)"
```

Replace the example base URL with the endpoint for your deployment and region. Keep credentials out of configuration files and version control.

### Configure the proxy

Save the following as `config.yaml`. Change `app_name` if your Vision One policies use a different application name.

```yaml title="config.yaml"
model_list:
  - model_name: guardrail-model
    litellm_params:
      model: openai/{{openai_small}}
      api_key: os.environ/OPENAI_API_KEY

guardrails:
  - guardrail_name: trendai-guard
    litellm_params:
      guardrail: trendai
      mode: [pre_call, post_call]
      api_key: os.environ/TMV1_API_KEY
      api_base: os.environ/TRENDAI_AI_GUARD_BASE_URL
      app_name: litellm
      default_on: true
      fallback_on_error: block
      timeout: 5.0
      streaming_buffer_until_moderated: true

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

`pre_call` checks the input before the model call. `post_call` checks the model response. With `default_on: true`, requests do not need to select the guardrail explicitly.

### Start the proxy

```bash
litellm --config config.yaml --port 4000
```

### Send a model request

This request calls the configured model and can incur provider charges.

```bash
curl -i http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "guardrail-model",
    "messages": [{"role": "user", "content": "Explain an AI guardrail in one sentence."}]
  }'
```

If both checks allow the request, LiteLLM returns the model response. To exercise a block or redaction, use a test input that matches an enabled policy for your application in Vision One. A prompt that looks unsafe does not by itself establish that the configured policy will block it.

## Verify the guardrail

### Confirm registration

Open **Guardrails** in the [LiteLLM Admin UI](../ui.md) and confirm that `trendai-guard` is listed with the expected modes. You can also inspect the configured guardrails through the API.

```bash
curl -sS http://localhost:4000/guardrails/list \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

### Test without a model call

Use the [Guardrail Test Playground](./test_playground.md), or send text directly to the guardrail endpoint. This endpoint calls TrendAI without calling the configured model.

```bash
curl -i http://localhost:4000/guardrails/apply_guardrail \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "guardrail_name": "trendai-guard",
    "input_type": "request",
    "text": "Hello, explain what a guardrail does."
  }'
```

An allowed scan returns the processed text in `response_text`. To test output scanning, set `input_type` to `response`. Redacted text and detection results depend on your TrendAI policies and scanners.

## Deployment

### Docker

Use an image that contains the built-in integration. Set `LITELLM_IMAGE` to that image reference, then mount the configuration from the quick start. No separate `trendai_guard.py` mount is required.

```bash
docker run --rm \
  -p 4000:4000 \
  -e OPENAI_API_KEY \
  -e TMV1_API_KEY \
  -e TRENDAI_AI_GUARD_BASE_URL \
  -e LITELLM_MASTER_KEY \
  -v "$PWD/config.yaml:/app/config.yaml:ro" \
  "${LITELLM_IMAGE:?Set LITELLM_IMAGE to an image containing the TrendAI integration}" \
  --config /app/config.yaml --port 4000
```

### Helm

For the monolithic `litellm-helm` chart, place the complete quick-start configuration under `proxy_config` in your values file. Set `image.repository` and `image.tag` to an image that contains the built-in integration.

Store the model credentials, `TMV1_API_KEY`, `TRENDAI_AI_GUARD_BASE_URL`, and `LITELLM_MASTER_KEY` in a Kubernetes Secret. List that Secret under `environmentSecrets` so its keys become proxy environment variables.

```yaml title="values.yaml"
environmentSecrets:
  - litellm-credentials
```

The Secret must exist in the same namespace as the proxy. See the [LiteLLM Helm chart README](https://github.com/BerriAI/litellm/tree/main/helm/litellm-helm#readme) for Secret creation and installation examples, or the [deployment guide](../deploy.md) for other deployment layouts. The external plugin's override file is not required for `guardrail: trendai`.

## Supported modes

| Mode | What it checks | Effect on the request |
| --- | --- | --- |
| `pre_call` | Input text before model inference. | Can block or redact the prompt before it reaches the model. |
| `during_call` | Input text in parallel with model inference. | Can reject the call, but inference can already be in progress. Use `pre_call` when input redaction is required. |
| `post_call` | Model response text. | Can block or redact non-streamed output. Streamed output uses the buffering behavior below. |
| `logging_only` | Logged input and output text. | Records scan results without changing or rejecting the client response. |

Use a list to select several modes, as in `mode: [pre_call, post_call]`. For per-request or per-key selection instead of `default_on`, see the [guardrails quick start](./quick_start.md).

## Parameters and environment variables

| Parameter | Default or fallback | Description |
| --- | --- | --- |
| `guardrail` | Set to `trendai`. | Selects the built-in integration. |
| `mode` | Set explicitly in the configuration. | One supported mode or a list of modes. |
| `api_key` | `TMV1_API_KEY`. | Vision One integration token. The fallback uses LiteLLM's secret resolution, including configured secret managers. |
| `api_base` | `TRENDAI_AI_GUARD_BASE_URL`. | AI Guard base URL for the deployment and region. Required unless supplied explicitly. |
| `app_name` | `TMV1_APPLICATION_NAME`, then `litellm`. | Application name for policy assignment and tracking. Sent as `TMV1-Application-Name`. |
| `default_on` | `false`. | Applies the guardrail without a per-request selection when enabled. |
| `fallback_on_error` | `block`. | Rejects calls when TrendAI cannot complete a check. `allow` permits calls on service failures, but does not override explicit policy blocks. |
| `timeout` | `5.0` seconds. | Timeout for each TrendAI API request. |
| `streaming_buffer_until_moderated` | `true`. | Holds streamed output until moderation finishes. Set to `false` for incremental delivery. |
| `response_content_chunk_size_bytes` | `49500`. | Target UTF-8 bytes per response scan window. Must be greater than zero. |
| `stream_overlap_size` | `256` characters. | Overlap between response scan windows. Must be non-negative. This does not control when buffered output is released. |

You can omit `api_key` to use the `TMV1_API_KEY` fallback. An explicit key takes precedence over that fallback. An explicit `api_base` or `app_name` also takes precedence over its environment fallback.

Large response texts are scanned in overlapping windows. Each window is a separate TrendAI request. These window settings apply to response scanning, including non-streamed responses, rather than to the client-visible chunk size.

## Streaming

On supported `post_call` streaming paths, responses are fully buffered until moderation finishes by default. A block or fail-closed error withholds the original model chunks. Successful moderation releases the buffered response, so the first model chunk arrives only after generation and moderation finish. Memory use grows with the buffered response.

Set `streaming_buffer_until_moderated: false` to allow incremental delivery. Earlier chunks can then reach the client before a later block or error. This opt-out removes the zero-original-chunk guarantee.

A failure can appear as an HTTP error before streaming starts or as a protocol-specific stream error. For example, the Responses API can return a `response.failed` event under HTTP 200 with no original model output.

Do not rely on streamed output for PII redaction. Use `pre_call` for input redaction or non-streamed `post_call` requests for response redaction. Buffering controls delivery before a verdict and does not select a streaming text-rewrite mode.

## Errors and policy outcomes

On a non-streaming request, an explicit TrendAI policy block returns HTTP 400. The message contains the reason returned by TrendAI, or a fallback policy-violation message.

With `fallback_on_error: block`, service failures such as rejected credentials, timeouts, and invalid API responses reject the call with HTTP 503. With `fallback_on_error: allow`, those failures let the call proceed. This setting does not turn an explicit policy block into an allow.

TrendAI can also return redactions for allowed content. The integration requests PII redaction from the API and applies returned text rewrites on supported non-streaming paths. Configure the relevant policies and scanners in Vision One rather than relying on an example prompt to trigger a particular result.

## Scan scope and current limits

On structured Chat Completions, Responses, and Anthropic Messages inputs, the integration selects the latest user turn. Earlier messages and system instructions remain outside that request scan. The Gemini input translation supplies flat text that also includes system instructions and earlier messages.

Tool-call arguments supplied separately from text are not scanned. The integration does not inspect image, audio, or video content. Do not enable `scan_only_tool_results` for this guardrail, since tool-result-only scanning is rejected at initialization.

Do not rely on streamed `/v1/completions` responses for output moderation. The current shared translation does not invoke the post-call scan on that legacy streaming endpoint.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| The guardrail is absent from the configured list. | Confirm that the proxy image includes the built-in integration, uses `guardrail: trendai`, and loaded the intended configuration. Inspect startup errors for missing credentials or an unsupported provider. |
| Requests fail with `Security Guard Error`. | Check the token, endpoint region, API reachability, and `timeout`. A rejected token is a service failure, not evidence of a content-policy block. |
| Unsafe-looking text is allowed. | Confirm that the relevant scanners and policies are enabled for the configured `app_name`. Use a policy-specific test case in Vision One. |
| The first streamed token is delayed. | Full buffering is enabled by default. Check generation time and moderation latency before opting out. |
| A stream returns HTTP 200 but contains an error. | Read the stream's terminal error event. HTTP status alone does not establish that moderation passed. |

For Vision One onboarding and the external plugin deployment, see the [upstream plugin README](https://github.com/trendmicro/tm-v1-ai-guard-litellm-plugin#readme). For shared LiteLLM controls, see the [guardrails quick start](./quick_start.md).
