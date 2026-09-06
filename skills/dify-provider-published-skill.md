---
name: Dify
description: Use when building AI applications with workflows, chatflows, or agents; integrating knowledge bases and RAG; calling APIs; managing model providers and tools; publishing web apps; or monitoring application performance and logs.
metadata:
    mintlify-proj: dify
    version: "1.0"
---

# Dify Skill

## Product Summary

Dify is a low-code platform for building, deploying, and monitoring AI applications. Agents use it to orchestrate workflows and chatflows (visual node-based apps), ground them in knowledge bases via RAG, integrate LLM providers and tools, and publish them as APIs or web apps. Key entry points: the visual builder at `cloud.dify.ai` or self-hosted instances, the REST API at `https://api.dify.ai/v1` (or your instance URL), and the CLI tool `difyctl` for running apps from scripts and terminals. Core concepts: **Workflows** (single-run batch processing), **Chatflows** (multi-turn conversation), **Nodes** (LLM, knowledge retrieval, code, agents, tools, logic), **Knowledge Bases** (RAG-powered document storage), and **Integrations** (model providers, plugins, tools).

## When to Use

Reach for this skill when:
- Building or modifying workflows, chatflows, or agent apps in the Dify builder
- Integrating knowledge bases or RAG into an application
- Configuring model providers (OpenAI, Anthropic, local models, etc.)
- Calling Dify apps via REST API from backend services
- Running Dify apps from CLI scripts, CI pipelines, or as agent tools
- Publishing apps as web apps, embedding them, or exposing them as APIs
- Debugging workflow execution, checking logs, or inspecting node outputs
- Setting up tools, plugins, or custom integrations
- Monitoring app usage, performance, and user feedback

## Quick Reference

### App Types

| Type | Use Case | Starts With | Ends With | Conversation |
|------|----------|-------------|-----------|--------------|
| **Workflow** | Batch processing, reports, data pipelines | User Input or Trigger | Output (optional) | No |
| **Chatflow** | Interactive assistants, Q&A, guided flows | User Input | Answer (required) | Yes |
| **Agent** | Autonomous tool use, multi-step reasoning | User Input | Answer | Yes |
| **Chatbot** | Simple chat with optional knowledge | User Input | Answer | Yes |

### Core Nodes

| Node | Purpose | Key Config |
|------|---------|-----------|
| **User Input** | Collect variables from users | Field types: text, file, select, etc. |
| **LLM** | Call language models | Model, temperature, system prompt, vision |
| **Knowledge Retrieval** | Search knowledge bases (RAG) | Knowledge base, retrieval method, top-k |
| **Agent** | Autonomous tool use with reasoning | Model, tools, max iterations, strategy |
| **Code** | Run Python or JavaScript | Code block, input/output variables |
| **If/Else** | Branch on conditions | Condition expression (Jinja2) |
| **Loop/Iteration** | Repeat over arrays | Input array, output variable |
| **HTTP Request** | Call external APIs | URL, method, headers, body |
| **Template** | Format output with Jinja2 | Template string |
| **Output** | Return workflow results | Output variables |
| **Answer** | Return chatflow response | Answer text (required for chatflows) |
| **Parameter Extractor** | Extract structured data from text | LLM-powered extraction rules |
| **Doc Extractor** | Convert documents to text | Input files |

### API Endpoints by App Type

| App Type | Key Endpoints |
|----------|---------------|
| **Workflow** | `POST /workflows/run`, `POST /workflows/{id}/run`, `GET /workflows/{id}/logs` |
| **Chatflow** | `POST /chat-messages`, `GET /conversations`, `POST /conversations/{id}/messages` |
| **Agent** | `POST /chat-messages`, `GET /conversations` |
| **Knowledge Base** | `POST /datasets/{id}/retrieve`, `POST /documents`, `GET /chunks` |

### CLI Commands

```bash
difyctl auth login              # Sign in with OAuth
difyctl apps list               # List workspace apps
difyctl apps run <app-name>     # Run an app interactively
difyctl apps run <app-name> --input key=value  # Pass inputs
difyctl apps run <app-name> --output json      # Get JSON output
```

### Key File Paths & Config

- **Model providers**: Integrations > Model Provider (UI) or `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` (env vars)
- **Knowledge bases**: Knowledge section in UI; API at `/datasets`
- **Plugins**: Integrations > Plugins; marketplace or local file upload
- **API keys**: Create per-app in app settings, or workspace-level for knowledge APIs
- **Environment variables** (self-hosted): `.env` file; key ones: `OPENAI_API_KEY`, `ENABLE_OAUTH_BEARER`, `WORKFLOW_LOG_RETENTION_DAYS`

## Decision Guidance

### When to Use Workflow vs. Chatflow

| Scenario | Choose |
|----------|--------|
| One-shot processing (report, data transform, batch job) | **Workflow** |
| Multi-turn conversation, user asks follow-ups | **Chatflow** |
| Scheduled or webhook-triggered execution | **Workflow** (with Trigger node) |
| Real-time user interaction expected | **Chatflow** |
| Complex logic, many branches, no conversation needed | **Workflow** |

### When to Use Agent vs. LLM Node

| Scenario | Choose |
|----------|--------|
| LLM decides which tools to call dynamically | **Agent** |
| Fixed sequence of steps, no tool selection needed | **LLM** + explicit tool calls |
| Research, troubleshooting, multi-step reasoning | **Agent** |
| Simple prompt-response, no tools | **LLM** |
| Model lacks native function calling | **Agent** with ReAct strategy |

### When to Use Knowledge Retrieval vs. Prompt Context

| Scenario | Choose |
|----------|--------|
| Large document corpus (100+ pages) | **Knowledge Retrieval** (RAG) |
| Small, fixed context (< 5 pages) | **Prompt context** (paste in system prompt) |
| User-uploaded documents | **Knowledge Retrieval** |
| Frequently updated reference material | **Knowledge Retrieval** |
| Need metadata filtering or reranking | **Knowledge Retrieval** |

## Workflow

### Building a Workflow or Chatflow

1. **Create the app**: Studio > Create from Blank > Workflow or Chatflow
2. **Add User Input node**: Define input fields (text, file, select, etc.)
3. **Add processing nodes**: LLM, knowledge retrieval, code, agents, logic
4. **Connect nodes**: Drag edges to pass variables downstream
5. **Configure each node**: Click to open panel, set model, prompts, parameters
6. **Test**: Click Test Run, fill inputs, check output and logs
7. **Debug**: Click Last Run on a node to see raw prompt, response, errors
8. **Publish**: Click Publish > Publish Update to make live
9. **Share**: Copy web app URL, API key, or embed code

### Running a Workflow via API

1. **Get API key**: Open app > API > Create API Key
2. **Get input schema**: `GET /apps/{app_id}/parameters` to see required fields
3. **Make request**: `POST /workflows/run` with `inputs` object and `user` identifier
4. **Stream or block**: Use `response_mode: streaming` for live events, `blocking` for final result
5. **Handle response**: Check `workflow_run_id` for logs, `outputs` for results
6. **Stop if needed**: `POST /workflows/tasks/{task_id}/stop` (streaming only)

### Integrating a Knowledge Base

1. **Create knowledge base**: Knowledge > Create Knowledge Base
2. **Upload documents**: Drag files or paste text; wait for indexing
3. **Test retrieval**: Click Retrieval Testing, try queries
4. **Add to workflow**: Drag Knowledge Retrieval node, select knowledge base
5. **Configure retrieval**: Set search method (semantic, keyword, hybrid), top-k, reranking
6. **Use in LLM**: Pass retrieved chunks to LLM prompt via `{Knowledge Retrieval/result}`

### Setting Up a Model Provider

1. **Go to Integrations > Model Provider**
2. **Click Install** on desired provider (OpenAI, Anthropic, etc.)
3. **Enter API key** and any custom endpoints
4. **Set as default** (top-right > Default Models) for easy access
5. **Use in nodes**: LLM and Agent nodes auto-populate with available models

### Publishing and Sharing

1. **Publish**: Click Publish > Publish Update (required before sharing)
2. **Share as web app**: Copy public URL from Publish panel
3. **Embed on website**: Use iframe or chat widget code from Embed section
4. **Expose as API**: Create API key, document endpoints in API Reference
5. **Connect to agents**: Use `difyctl` to register app as a tool for AI agents

## Common Gotchas

- **Variables not found in downstream nodes**: Ensure upstream node is connected and variable name matches exactly (case-sensitive). Use `{` or `/` in text fields to autocomplete.
- **Knowledge base not indexing**: Check model provider is configured and has valid credentials. Indexing happens async; wait and refresh.
- **LLM node returns empty or error**: Verify model is selected, API key is valid, and prompt is valid Jinja2. Check Last Run logs for exact error.
- **Agent loops infinitely**: Set `max_iterations` to a reasonable number (3–15 depending on task). Check tool descriptions are clear so agent knows when to stop.
- **API calls fail with 403 Forbidden**: Ensure API key is scoped to the correct app and workspace. Knowledge API keys are broader; app keys are narrower.
- **Workflow runs but output is wrong**: Test each node individually with Test Run. Use variable inspect to see intermediate values. Check node-level logs in Last Run.
- **Chatflow doesn't remember context**: Chatflows are stateless per message; use conversation variables or external storage if you need multi-turn memory.
- **Published app doesn't reflect changes**: Always click Publish > Publish Update after editing. Web app caches; hard-refresh browser if needed.
- **File uploads fail**: Check file size limits (usually 10–100 MB), supported types, and that User Input node has file field enabled.
- **Trigger node doesn't fire**: Verify webhook URL is correct, payload matches expected format, and trigger is enabled in node config.

## Verification Checklist

Before submitting or publishing:

- [ ] All required input fields are defined in User Input node
- [ ] All downstream nodes have upstream connections (no orphaned nodes)
- [ ] Variable names are spelled correctly and match references (case-sensitive)
- [ ] Model provider is configured and API key is valid
- [ ] Knowledge bases are indexed and retrieval test passes
- [ ] LLM prompts are valid Jinja2 syntax (test with `{variable}` references)
- [ ] Agent max_iterations is set to a reasonable limit
- [ ] Error handling is in place (If/Else for edge cases, or error branches)
- [ ] Test Run completes without errors; output is sensible
- [ ] Web app settings are configured (name, icon, description)
- [ ] API key is created if exposing as API
- [ ] Publish Update is clicked before sharing or deploying

## Resources

- **Comprehensive page listing**: https://docs.dify.ai/llms.txt
- **Quick Start (30 minutes)**: https://docs.dify.ai/en/quick-start
- **Workflow & Chatflow Overview**: https://docs.dify.ai/en/cloud/use-dify/build/workflow-chatflow
- **API Reference**: https://docs.dify.ai/en/api-reference/guides/get-started
- **Knowledge Base & RAG**: https://docs.dify.ai/en/cloud/use-dify/knowledge/readme
- **CLI (difyctl)**: https://docs.dify.ai/en/cli/overview

---

> For additional documentation and navigation, see: https://docs.dify.ai/llms.txt