---
name: dify-knowledge-base-ingest
description: >-
  Build and maintain a Dify knowledge base over the Service API — create it, ingest documents from
  text or file, wait out asynchronous indexing, manage chunks and metadata, and retrieve against it.
api: Dify Service API
base_url: https://api.dify.ai/v1
auth: 'Authorization: Bearer {KNOWLEDGE_API_KEY}'
operations:
  - createDataset
  - listDatasets
  - getDatasetDetail
  - updateDataset
  - deleteDataset
  - createDocumentFromText
  - createDocumentFromFile
  - getDocumentIndexingStatus
  - listDocuments
  - updateDocument
  - batchUpdateDocumentStatus
  - createSegments
  - listSegments
  - updateSegment
  - retrieveSegments
  - createMetadataField
  - batchUpdateDocumentMetadata
generated: '2026-09-06'
method: generated
source: openapi/_original/dify-service-api-openapi.json, rate-limits/dify-rate-limits.yml, conventions/dify-conventions.yml
---

# Ingest and retrieve with a Dify knowledge base

## Understand the credential first

This flow uses a **knowledge base API key**, minted under Knowledge → Service API. It is not an app
key, and it is deliberately broader: it can reach **every knowledge base visible to the account that
created it**. Dify's own specification flags this as a data-security concern. Do not hand this key
to an agent that only needs one knowledge base — the API has no per-knowledge-base scoping.

## Respect the rate limit before you start

Knowledge operations are governed by a **workspace-wide** limit measured per minute, not per key:

- Sandbox: 10/min
- Professional: 100/min
- Team: 1,000/min

Creating, deleting and updating knowledge bases; uploading, deleting, updating, disabling, enabling,
archiving and restoring documents; adding, deleting, updating and bulk-importing segments; hit
tests; and queries from apps and workflows all count. A bulk ingest at Sandbox tier will throttle
the whole workspace, including live app traffic. Pace the loop.

No rate-limit headers are returned, so you cannot see how close you are. Budget by the number of
calls you make.

## Create the knowledge base

`createDataset` (`POST /datasets`). Read it back with `getDatasetDetail`
(`GET /datasets/{dataset_id}`) to see its embedding model, retrieval configuration and document
statistics.

## Ingest documents

- From raw text: `createDocumentFromText`
  (`POST /datasets/{dataset_id}/document/create-by-text`).
- From a file: `createDocumentFromFile`
  (`POST /datasets/{dataset_id}/document/create-by-file`) — PDF, TXT, DOCX and similar.

Both return a `batch` id. **Indexing is asynchronous.** Do not query the knowledge base until it
finishes.

## Wait for indexing

Poll `getDocumentIndexingStatus`
(`GET /datasets/{dataset_id}/documents/{batch}/indexing-status`) until every document reaches
`completed` or `error`. Status advances `waiting` → `parsing` → `cleaning` → `splitting` →
`indexing` → `completed`. Each poll counts against the workspace rate limit — back off between
polls.

## Manage chunks

`listSegments`, `getSegmentDetail`, `createSegments`, `updateSegment` and `deleteSegment` work
within one document; `createChildChunk` / `getChildChunks` / `updateChildChunk` /
`deleteChildChunk` work within one segment. Updating a chunk re-triggers indexing for it.

## Metadata and tags

- `createMetadataField` (`POST /datasets/{dataset_id}/metadata`) defines a custom field;
  `getBuiltInMetadataFields` and `toggleBuiltInMetadataField` control the built-in ones.
- `batchUpdateDocumentMetadata` (`POST /datasets/{dataset_id}/documents/metadata`) sets values
  across many documents in one call — prefer it to a per-document loop, both for speed and for the
  rate limit.
- Tags are many-to-many with knowledge bases via explicit `bindTagsToDataset` and
  `unbindTagFromDataset`.

## Retrieve

`retrieveSegments` (`POST /datasets/{dataset_id}/retrieve`) with a `query` and an optional
`retrieval_model`. This is the same call the docs describe as test retrieval and as production
retrieval — there is no separate test surface.

## Turning content off, and deleting it

Prefer the reversible path:

- `batchUpdateDocumentStatus` (`PATCH /datasets/{dataset_id}/documents/status/{action}`) with
  `action=disable` removes documents from retrieval; `action=enable` restores them.
- `action=archive` moves them to the archive; `action=un_archive` restores them. No time limit is
  published on un-archiving.

`deleteDocument` and `deleteDataset` are documented as **permanent** — `deleteDataset` takes every
document with it. There is no restore endpoint and no retention window. Archive first, delete only
on an explicit human instruction.

## Retry safely

No idempotency key exists. A retried `createDocumentFromText` creates a second document. Before
re-firing, call `listDocuments` and check whether the document already landed.
