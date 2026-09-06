---
name: dify-chat-app-conversation
description: >-
  Run a conversation against a published Dify Chatflow, Chatbot or Agent app over the Service API —
  send a message, consume the SSE stream, keep one end user's history separate from another's, and
  stop a runaway generation.
api: Dify Service API
base_url: https://api.dify.ai/v1
auth: 'Authorization: Bearer {APP_API_KEY}'
operations:
  - sendChatMessage
  - stopChatMessageGeneration
  - getSuggestedQuestions
  - getConversationsList
  - getConversationHistory
  - renameConversation
  - deleteConversation
  - uploadChatFile
  - postChatMessageFeedback
  - getChatAppInfo
generated: '2026-09-06'
method: generated
source: openapi/_original/dify-service-api-openapi.json, conventions/dify-conventions.yml, errors/dify-problem-types.yml
---

# Hold a conversation with a Dify chat app

## Before you start

- The key is an **app API key**. It is scoped to one published app and it serves every end user of
  that app. Keep it server-side.
- Confirm the key and learn which app it belongs to with `getChatAppInfo` (`GET /info`). It returns
  `name`, `mode` and `description`.
- Pick a stable `user` value per end user and never change it. It scopes conversations, files and
  run records. A mismatched `user` on a later call returns `404`, not someone else's data.

## Send a message

`sendChatMessage` (`POST /chat-messages`) with `query`, `inputs`, `user`, and `response_mode`.

- `response_mode: blocking` returns one JSON body. Use it only for short, non-interactive calls —
  the Dify Cloud edge proxy may cut a long blocking request.
- `response_mode: streaming` returns `text/event-stream`. Use it for anything user-facing.
- Omit `conversation_id` to start a new conversation; pass the one you were given to continue it.
- To attach a file, upload it first with `uploadChatFile` (`POST /files/upload`, multipart) and pass
  the returned reference in `files`.

## Consume the stream

Every event except `ping` arrives as a `data: ` line holding one JSON object. Skip any line that is
not a `data: ` line — the keep-alive arrives as a bare `event: ping` with no payload, roughly every
ten seconds. Set the client read timeout comfortably above ten seconds.

Concatenate in order:

- `message` events for Chatbot and Chatflow apps.
- `agent_message` events for Legacy Agent and Agent apps. For Agent apps a single closing `message`
  event repeats the complete answer — treat it as the final answer, not extra text to append.

Close on the right terminal event:

- `message_end` for Chatbot, Legacy Agent and Agent apps.
- `message_end` then `workflow_finished` for Chatflow apps — both arrive, in that order.

Save `task_id` from the first event. Chatflow streams also carry `workflow_run_id`; save it too.

## Stop a generation

`stopChatMessageGeneration` (`POST /chat-messages/{task_id}/stop`) with the same `user`. This is the
only reversal available on a chat message and it works only while the generation is in flight — it
cancels further output, it does not remove what was already produced.

## Read history

- `getConversationsList` (`GET /conversations`) pages with `last_id` and `limit`.
- `getConversationHistory` (`GET /messages`) pages with `first_id` and `limit`.

Note the two idioms differ. A single paging loop will not serve both.

## Handle failures

Errors are `{code, message, status}` as `application/json` — not RFC 9457.

- Retry with backoff: `too_many_requests`, `500`, network failures.
- Do not retry as-is: `invalid_param`, `bad_request`, `unauthorized`, `forbidden`,
  `rate_limit_error`.
- Fix, do not retry: `provider_not_initialize`, `provider_quota_exceeded`,
  `model_currently_not_support`, `completion_request_error` — these point at the app's model
  configuration, not at your request.

Once the stream is open the HTTP status is already `200`. A later failure arrives as an `error`
event carrying `status`, `code` and `message`, and ends the stream. Treat it as terminal.

## Retry safely

**There is no idempotency mechanism in this API.** No `Idempotency-Key` header, no client-supplied
request key. A retried `sendChatMessage` creates a second message. Before re-firing a call whose
connection dropped, check what actually landed with `getConversationHistory` for that `user`.

## Clean up

`deleteConversation` (`DELETE /conversations/{conversation_id}`) is permanent. There is no trash and
no restore endpoint. Confirm with a human before calling it.
