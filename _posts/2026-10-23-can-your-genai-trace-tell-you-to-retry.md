---
layout: post
title: "Can Your GenAI Trace Tell You Whether to Retry?"
date: 2026-10-23 09:00:00 +0800
categories: [engineering, observability]
tags: [opentelemetry, genai, tracing, error-handling, retries, streaming, llm]
excerpt: "When an LLM call fails mid-stream, does the error.type on your span tell you whether to retry? For non-streaming calls, mostly yes. For streaming, it depends on whether your instrumentation has a resolver for your provider."
lang: en
---

In [my last post](/2026/10/20/your-rate-limit-errors-might-be-auth-failures/), a failure classifier filed missing credentials under rate limiting. That made me wonder whether the traces we rely on can even tell those failures apart.

When an LLM call fails, the first decision is boring and important: retry or not. A 401 means stop — your credentials are wrong, and retrying only burns time. A 429 means wait and try again. A 529 from Anthropic means the provider is overloaded, which is a different kind of waiting. If your observability pipeline can't tell these apart, your retry logic is guessing.

So I asked a narrow question: if you instrument LLM calls with an off-the-shelf OpenTelemetry GenAI library, does the `error.type` on a failed span let you distinguish an auth failure from a rate limit?

## What the spec says

The OTel GenAI semantic conventions don't define an error taxonomy. `error.type` should carry an error code or the exception's fully qualified class name. That's a reasonable default, but it means distinguishability isn't decided by the spec. It's decided by the instrumentation: either it maps the provider's own error code into `error.type`, or it falls back to the class name of whatever exception the SDK happened to raise.

## The non-streaming path: it mostly works

I looked at three instrumentation libraries that claim GenAI semconv support (the OpenTelemetry contrib OpenAI instrumentation, OpenLLMetry, OpenLIT). All three set `error.type` to the exception class name. None of them set `http.response.status_code`.

That turns out fine for the ordinary case, because the major Python provider SDKs already split errors by status. openai-python raises `AuthenticationError` for 401 and `RateLimitError` for 429. anthropic-sdk-python does the same and adds `OverloadedError` for 529. LiteLLM's mapping keeps those class names too. So on a request that fails at the HTTP layer, your trace says `RateLimitError` or `AuthenticationError`, and you can branch on that.

Note what's doing the work, though: the provider SDK's type system. The instrumentation just passes the class name through. Put a raw HTTP client or a generic OpenAI-compatible gateway in front, and you lose it — with no status-code attribute to fall back on.

## The streaming path: it depends on the provider

Agents mostly stream. And streaming has a failure mode the type system doesn't cover: the server returns HTTP 200, starts the stream, and *then* sends an error event mid-stream — `overloaded_error`, `rate_limit_error`, whatever.

Public issues on the Anthropic SDKs describe what happens next: anthropic-sdk-python #1258 (closed as completed in March, but the behavior is reported to still reproduce on 1.11.0 and main; see the still-open PR #1921) and TypeScript SDK #1262. The SDK builds the exception from the stream's original response, which has status 200. Status-based dispatch never matches, so you get a bare `APIStatusError` (Python) or base `APIError` (TypeScript) instead of `OverloadedError` or `RateLimitError`. openai-python's Chat streaming path is similar: a mid-stream error comes out as the base `APIError`.

My first draft stopped there and called it an ecosystem-wide problem. That was one provider's evidence stretched over everyone, and the next provider I checked broke it. google-genai, on a mid-stream `{"error": ...}` chunk, dispatches on the *body* code: 4xx raises `ClientError`, 5xx raises `ServerError`, and `.code` stays `429`. On top of that, the official OpenTelemetry google-genai instrumentation has an error-type resolver that writes `error.type = "429"` onto the span. The official OpenAI Responses path has a resolver too.

So the SDKs aren't throwing the meaning away. All three keep it somewhere — `.code`, `.type`, or the error body. Whether it reaches your trace depends on whether the instrumentation has a resolver for that provider. The official anthropic and openai-chat instrumentations don't have one yet. I checked this against a mock server that sends mid-stream errors: on their spans, `error.type` falls back to the class name, and 429, 529 and 500 look identical. Adding `http.response.status_code` wouldn't help, because it would say 200. One caveat on the Anthropic side: the span the anthropic SDK emits on its own (`anthropic.messages.create`, as of 1.15.0) does carry the body error type, such as `rate_limit_error`. So for Anthropic the gap is in the official instrumentation's span, not in every span of the trace.

## So: can your trace tell you to retry?

- **Non-streaming, standard provider SDK:** yes, by accident of good SDK design.
- **Streaming, mid-stream error:** it depends on whether the instrumentation has a resolver for your provider. With google-genai or OpenAI Responses under the official instrumentation, yes: `error.type` carries the provider code. With OpenAI Chat streaming, not yet. The trace knows something failed, but not what kind of failure it was. With anthropic, the official instrumentation's span has the same problem, but the SDK's own span carries the provider code.

That's narrower than "streaming breaks error classification", and the fix is smaller too. The mechanism already exists. Two instrumentations just need a resolver that reads `body.error.type` / `err.code`. Also, if you run OpenLLMetry instead of the official packages, its behavior may differ. I haven't checked it on this path.

## What I'd do

1. Classify failures by type and status, never by matching error-message text. Text matching is how an auth failure ends up filed as a rate limit.
2. For streaming, read the provider's body-level error type and record it as its own span attribute.
3. Keep 429 and 529 in separate buckets. "You're sending too much" and "the provider is full" call for different backoff behavior and different alerts.
4. Treat a bare `APIStatusError` with status 200 as "unclassified", not as "retryable".

Your trace is only as specific as the instrumentation lets it be. The provider usually still knows what went wrong. The open question is whether anyone wrote the resolver that carries that onto your span.
