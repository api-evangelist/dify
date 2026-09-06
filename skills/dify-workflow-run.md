---
name: dify-workflow-run
description: >-
  Execute a published Dify Workflow app over the Service API, follow the run through its SSE event
  stream, survive a dropped connection, answer a Human Input pause, and stop a run in flight.
api: Dify Service API
base_url: https://api.dify.ai/v1
auth: 'Authorization: Bearer {APP_API_KEY}'
operations:
  - executeWorkflow
  - runWorkflowById
  - streamWorkflowEvents
  - stopWorkflowTaskGeneration
  - getWorkflowRunDetail
  - getWorkflowLogs
  - getChatflowHumanInputForm
  - submitChatflowHumanInputForm
  - getChatAppParameters
generated: '2026-09-06'
method: generated
source: openapi/_original/dify-service-api-openapi.json, asyncapi/dify-events.yml, conventions/dify-conventions.yml
---

# Run a Dify workflow and follow it to completion

## Start the run

`executeWorkflow` (`POST /workflows/run`) with `inputs`, `user` and `response_mode`. To run a
specific workflow version by id, use `runWorkflowById` (`POST /workflows/{workflow_id}/run`).

Discover what `inputs` the app expects with `getChatAppParameters` (`GET /parameters`) rather than
guessing field names.

## Two identifiers, and they are not interchangeable

- `task_id` controls the **in-flight generation**. It is what `stopWorkflowTaskGeneration` takes.
- `workflow_run_id` names the **persistent run record**. It is the only handle for reconnecting or
  for checking the outcome later.

Save `workflow_run_id` the moment it arrives. If the connection drops before you have it, the run
is unrecoverable from the client side.

## Follow the stream

A Workflow stream opens with a bare `event: ping` frame. Treat the first `data:` event —
`workflow_started` — as the signal the run was accepted, not the first frame.

Order you will normally see:

1. `workflow_started` — `data.inputs`
2. `node_started` / `node_finished` per node — `data.node_id`, `data.node_type`, `data.status`,
   `data.outputs`, `data.execution_metadata`
3. `node_retry` when a node retries after failure — `data.retry_index`
4. `iteration_*` and `loop_*` for Iteration and Loop node progress (informational)
5. `agent_log` for Agent node step logs (informational; note it carries no `workflow_run_id`)
6. `workflow_finished` — `data.status` is one of `succeeded`, `failed`, `partial-succeeded`,
   `stopped`, plus `data.outputs` and `data.total_tokens`

## Reconnect after a drop

Reopen with `streamWorkflowEvents` (`GET /workflow/{workflow_run_id}/events`), passing the same
`user` that started the run. A mismatched `user` returns `404`.

- `include_state_snapshot=true` replays the status of nodes that already ran.
- `continue_on_pause=true` keeps one stream open across multiple Human Input pauses.

After reconnecting to a still-running workflow, confirm completion with `getWorkflowRunDetail`
(`GET /workflows/run/{workflow_run_id}`) rather than trusting the reconnected stream's final event
alone.

## Answer a Human Input pause

When the run reaches a Human Input node the stream emits `human_input_required` carrying
`form_token`, `form_content` and `expiration_time`, then `workflow_paused` with `paused_nodes` and
`reasons` — and **this stream ends there**.

1. Read the form definition with `getChatflowHumanInputForm`
   (`GET /form/human_input/{form_token}`).
2. Submit it with `submitChatflowHumanInputForm` (`POST /form/human_input/{form_token}`) before
   `expiration_time`.
3. Pick the resumed run up on `streamWorkflowEvents`. The resumed stream carries
   `human_input_form_filled` or `human_input_form_timeout` through to `workflow_finished`.

## Stop a run

`stopWorkflowTaskGeneration` (`POST /workflows/tasks/{task_id}/stop`) with the same `user`. It works
only while the task is in flight, and the run finishes with `status: stopped`. Work a completed
workflow already performed — documents written, external systems called — is not rolled back.

## Failures

A node failure does not change the HTTP status: the connection stays `200` and you see
`node_finished` and `workflow_finished` with `status: "failed"`. Other failures end the stream with
an `error` event carrying `status`, `code` and `message`. Handle both; either is terminal.

On `429`, distinguish the two codes: `too_many_requests` is a concurrency ceiling worth retrying
with backoff, and `rate_limit_error` is a plan quota that will not clear on retry.

## Retry safely

There is no idempotency key on this API. A retried `executeWorkflow` starts a second run. If a
connection dropped and you hold the `workflow_run_id`, check `getWorkflowRunDetail` before firing
again; if you do not hold one, search `getWorkflowLogs` (`GET /workflows/logs`) before assuming the
run never started.
