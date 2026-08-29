---
name: tailor-executor-webhook-integration
description: Wire an external system into Tailor with an incoming webhook trigger, and send events back out with a webhook operation, respecting the published rate limits and retry semantics.
api: Tailor Executor Incoming Webhook API
operations:
  - CreateExecutorExecutor
  - ListExecutorExecutors
  - GetExecutorExecutor
  - ListExecutorIncomingWebhooks
  - GetExecutorIncomingWebhook
  - TriggerExecutor
generated: '2026-08-29'
method: generated
source: https://docs.tailor.tech/guides/executor/incoming-webhook-trigger
---

# Integrate an external system through Executor webhooks

## Inbound: let an external system trigger Tailor

1. **Define the executor** with an `incomingWebhookTrigger`:

   ```ts
   import { createExecutor, incomingWebhookTrigger } from "@tailor-platform/sdk";

   export default createExecutor({
     name: "incoming-webhook-based-executor",
     description: "exposes an endpoint",
     trigger: incomingWebhookTrigger<{
       body: { event: string };
       headers: Record<string, string>;
     }>(),
     operation: { /* tailorGraphql | webhook | function | jobFunction */ },
   });
   ```

   `name` must match `^[a-z0-9][a-z0-9-]{1,61}[a-z0-9]$`.

2. **Deploy**, then get the endpoint with `tailor executor webhook list`. It has the
   shape:

   ```
   https://api.erp.dev/v1/workspaces/{WORKSPACE_ID}/executors/{EXECUTOR_NAME}/invokeIncomingWebhook/{WEBHOOK_SECRET}
   ```

   The secret is in the path. Treat that URL as a credential.

3. **Read the payload.** It arrives wrapped:

   ```json
   { "args": { "method": "POST", "headers": { … }, "body": { … } } }
   ```

   Headers are lowercase with hyphens and may be arrays. `application/json` and
   `application/x-www-form-urlencoded` are both accepted; form values arrive as strings,
   repeated values as arrays.

### Rate limit — plan for it

100 requests/sec **per workspace**, counted across all incoming webhook endpoints, in
**fixed** one-second windows. Over the limit the gateway returns `429` and the executor
is never invoked — no platform-side retry.

Read these on every response:

- `RateLimit-Limit`
- `RateLimit-Remaining`
- `RateLimit-Reset` (seconds until the window resets)

Because windows are fixed, sustained traffic at exactly 100 req/sec still sees some
`429`s at boundaries. Target comfortably below it, retry with exponential backoff and
jitter, or batch events into one request.

## Outbound: let Tailor call an external system

Use an executor `operation` of `kind: "webhook"` with a `url`, `headers` and a
`requestBody` function. Only `POST` is supported.

- Timeout is 60 seconds; failure triggers a retry, up to 10 attempts. Delivery is
  therefore **at-least-once** — the receiving endpoint must be idempotent, because
  Tailor sends no delivery id or signature you can de-duplicate on.
- No webhook signing scheme is documented. Authenticate the callback yourself with a
  header you set in `headers`.
- Allowlist Tailor's published egress IPs if your endpoint is firewalled:
  asia-northeast `34.84.184.32`, `34.146.182.34`, `34.84.246.225`, `34.146.1.96`,
  `34.146.215.65`; us-west `34.127.79.202`, `34.168.21.128`, `34.82.25.72`
  (https://docs.tailor.tech/reference/platform/outgoing-ip-addresses).

## Concurrency ceiling

Executor job function operations are capped at 100 concurrently running per workspace.
Excess executions are **queued oldest-first, never dropped**. Tailor gives the drain
formula: `(queued ÷ 100) × average execution duration`. Do not build an integration
that assumes a fixed completion time for a burst — chain downstream steps on completion
events instead of on a schedule.

## Control-plane RPCs behind this

The executor surface is managed through `tailor.v1.OperatorService`. The RPC names carry
a doubled segment because the service is Executor and the resource is also Executor:

- `CreateExecutorExecutor`, `UpdateExecutorExecutor`, `GetExecutorExecutor`,
  `DeleteExecutorExecutor`, `ListExecutorExecutors`
- `ListExecutorIncomingWebhooks`, `GetExecutorIncomingWebhook`
- `TriggerExecutor`
- `GetExecutorJob`, `ListExecutorJobs`, `ListExecutorJobAttempts`

Only the `Get*`/`List*` members declare `idempotency_level = NO_SIDE_EFFECTS`. Do not
blind-retry `CreateExecutorExecutor` or `TriggerExecutor`.
