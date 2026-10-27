# TrendAI Guard

TrendAI Guard scans prompt and response text sent through LiteLLM Proxy. It can allow, block, or redact text according to the policies configured in your TrendAI deployment.

:::note Availability

This page covers the integration proposed in [LiteLLM PR #41254](https://github.com/BerriAI/litellm/pull/41254). Use a build that includes that integration.

:::

## Configuration

Add the guardrail to your existing proxy configuration. Set `TMV1_API_KEY` and `TRENDAI_AI_GUARD_BASE_URL` before starting the proxy.

```yaml title="config.yaml"
guardrails:
  - guardrail_name: trendai
    litellm_params:
      guardrail: trendai
      mode: [pre_call, post_call]
      default_on: true
      api_key: os.environ/TMV1_API_KEY
      api_base: os.environ/TRENDAI_AI_GUARD_BASE_URL
      fallback_on_error: block
      streaming_buffer_until_moderated: true
```

`pre_call` scans the prompt before the model call. `post_call` scans the model response. With `default_on: true`, the guardrail applies without a per-request guardrail selection.

## Environment variables

| Variable | Purpose |
| --- | --- |
| `TMV1_API_KEY` | TrendAI API key. The fallback uses LiteLLM's secret resolution, including a configured secret manager. |
| `TRENDAI_AI_GUARD_BASE_URL` | Base URL for your TrendAI deployment. Required unless `api_base` is set explicitly. |
| `TMV1_APPLICATION_NAME` | Application name sent to TrendAI. Defaults to `litellm` unless `app_name` is set explicitly. |

You can omit `api_key` to use the `TMV1_API_KEY` fallback. An explicitly supplied key takes precedence.

## Streaming

With `post_call`, streamed responses are fully buffered until moderation finishes by default. A block or fail-closed error withholds the original model chunks. The client can receive an HTTP error or a protocol-specific stream error instead.

Set `streaming_buffer_until_moderated: false` to allow incremental delivery. Earlier chunks can then reach the client before a later block or error.

Full buffering delays the first model chunk until generation and moderation finish. Memory use grows with the buffered response. Buffering alone does not add streamed redaction support.

## Error handling

`fallback_on_error` defaults to `block`. Set it to `allow` to let requests proceed when TrendAI cannot complete a check. This setting controls service failures, not policy blocks returned by TrendAI.

## Current limits

Tool-call arguments supplied separately from text are not scanned. Structured chat input scans select the latest user turn, while the Gemini input translation also supplies system and earlier message text.

Do not rely on streamed `/v1/completions` responses for output moderation. The current shared translation does not invoke the post-call scan on that legacy streaming endpoint.
