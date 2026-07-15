# Grounded Chat Contract

## 1. Scope

This contract is normative for SV-US-011. It fixes the public chat request, provider-neutral
generation boundary, server-owned tools, LangGraph turn flow, grounding validation, failure
behavior, and audit behavior. The existing retrieval, ranking, conversation, and installation
authentication contracts remain authoritative for their own boundaries.

Ask Sunny returns one complete response. It does not stream partial model output and it does not
use provider-hosted conversation state.

## 2. Public Request Boundary

`POST /chat` requires an active installation credential with `chat:write`. The JSON body is a strict
object with only:

- optional `conversation_id`: UUID;
- exactly one of `external_user_id` or `anonymous_session_id`, with the identity grammar from
  [`CONVERSATION_CONTEXT_CONTRACT.md`](CONVERSATION_CONTEXT_CONTRACT.md);
- `channel`: `web`, `mobile`, or `admin_test`;
- `message`: normalized non-empty text no longer than `MAX_CHAT_INPUT_CHARS`;
- optional `context`: a strict object containing optional IANA `timezone` and optional absolute
  HTTPS `page_url`, with no credentials, query, or fragment.

The request rejects unknown top-level and context members with `400 validation_error`. In
particular, it never accepts a provider, model, API key, system prompt, raw SQL, embedding,
tool/tool name, tool arguments, retrieval limit, `allowed_data_source_keys`, or ranking policy.
Instructions embedded in `message` remain untrusted user text and cannot change server policy.

The route passes the authenticated request correlation ID to `beginTurn`. A supplied conversation
must belong to the exact request identity; ownership misses remain the non-disclosing
`404 conversation_not_found` behavior defined by the conversation contract.

## 3. Successful Response

A successful turn returns `200` and exactly:

```json
{
  "conversation_id": "uuid",
  "message_id": "uuid",
  "answer": "Complete user-facing text.",
  "recommendations": [],
  "citations": [],
  "follow_up_questions": []
}
```

`recommendations` and `citations` use the complete public shapes from
[`RANKING_AND_CITATION_CONTRACT.md`](RANKING_AND_CITATION_CONTRACT.md). The arrays are always
present. `follow_up_questions` is always present, contains at most three unique non-empty strings,
and is empty when no next question is useful. `message_id` is the persisted assistant message ID,
not a provider response ID.

A clarification is a successful turn with one useful question in `answer`, empty recommendations
and citations, and the same question as the only `follow_up_questions` member. A no-evidence result
is also successful: it states that approved site content did not provide enough evidence, contains
no unsupported factual claim, and returns empty grounded arrays.

## 4. Provider-Neutral Generation Boundary

The provider registry resolves `AI_PROVIDER` once during startup. Only `openai` and `groq` are
registered. Orchestration, tools, HTTP routes, and persistence receive the selected adapter through
the common boundary and never branch on its name.

The internal request contains only:

- server instructions;
- bounded provider-neutral conversation messages in chronological order;
- the current user message and sanitized page/timezone context;
- registered strict tool definitions;
- prior provider output items and server-generated tool results for the current bounded loop;
- the strict final-response schema.

The normalized adapter result is one of:

```json
{ "kind": "tool_calls", "calls": [{ "call_id": "opaque-current-turn-id", "name": "search_content", "arguments": {} }], "usage": {} }
```

```json
{ "kind": "final", "output": { "answer": "...", "citation_ids": [], "recommendation_ids": [], "follow_up_questions": [] }, "usage": {} }
```

Tool call IDs exist only in memory for the active request and in provider-neutral tool audit data
when needed for ordering. They are not treated as provider conversation state. Usage normalizes
optional input, output, and total token counts to non-negative integers or `null`. Adapter errors
normalize to stable timeout, authentication, rate-limit, unavailable, invalid-response, and
permanent-request categories without including response bodies, credentials, or provider IDs in
public errors or durable records.

### OpenAI Responses adapter

The OpenAI adapter uses only `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `OPENAI_CHAT_MODEL`, and
`AI_REQUEST_TIMEOUT_MS`. It calls the configured base URL's `/responses` endpoint, supplies the
complete server-owned input, uses strict function schemas and a strict JSON-schema text format, and
sets `store: false`. It never sends or persists `previous_response_id`, a provider conversation ID,
or a provider response ID.

### Groq Responses adapter

The Groq adapter uses only `GROQ_API_KEY`, `GROQ_BASE_URL`, `GROQ_CHAT_MODEL`, and
`AI_REQUEST_TIMEOUT_MS`. It calls the configured base URL's `/responses` endpoint and supplies the
same complete server-owned history. It omits unsupported state and request parameters, including
`store`, `previous_response_id`, `conversation`, `truncation`, `include`, `prompt`,
`prompt_cache_key`, and `safety_identifier`.

Both adapters retain the provider's raw current-turn output only in memory long enough to continue
the tool loop. Neither adapter logs a key, authorization header, raw provider body, provider name,
model, or provider identifier as conversation audit data.

Current capability references checked for SV-US-011:

- OpenAI Responses migration and storage behavior:
  https://developers.openai.com/api/docs/guides/migrate-to-responses
- OpenAI function calling:
  https://developers.openai.com/api/docs/guides/function-calling
- OpenAI structured outputs:
  https://developers.openai.com/api/docs/guides/structured-outputs
- Groq Responses API and unsupported parameters:
  https://console.groq.com/docs/responses-api
- Groq OpenAI compatibility:
  https://console.groq.com/docs/openai

Model values remain explicit deployment settings. Documentation verification does not silently
replace an operator-selected model.

## 5. Server-Owned Tools

The only launch tools are `list_data_sources`, `search_content`, and `get_content_detail`. The
registry rejects any other name without dispatch. Tool arguments are parsed as strict JSON objects;
raw SQL, URLs, provider/model settings, embeddings, authorization, and undeclared fields fail tool
validation.

### `list_data_sources`

Input is exactly `{}`. Output contains the current allowlist version and a bounded array of stored,
enabled source descriptors. Each descriptor contains only `data_source_key`, `source_kind`, `label`,
and compact public context useful for source selection. Sources outside the persisted allowlist are
not returned. Missing or unavailable policy fails closed to an empty array.

### `search_content`

Input contains `query`, optional `data_source_keys`, optional validated `filters`, and optional
`limit`. Query and limit cannot exceed server configuration. The service intersects selected keys
with the freshly loaded persisted allowlist before every execution, then delegates to the
SV-US-008 retrieval boundary. An omitted key list means the stored allowlist; an invalid or empty
intersection returns no results. Both retrieval branches receive the same validated filters.

Output is bounded and compact. It includes allowlist version, retrieval mode/degradation, and
results with stable source identity, title, direct HTTPS URL, normalized score, and approved public
evidence. It excludes embeddings, private metadata, arbitrary SQL fragments, credentials, and raw
database rows.

### `get_content_detail`

Input contains exactly `data_source_key` and `source_id`. The key is intersected with the freshly
loaded stored allowlist before the SV-US-008 detail boundary runs. An unauthorized, missing, or
mismatched result is `null`; it does not disclose whether content exists outside policy. Output uses
the bounded public detail shape and never becomes a new recommendation target by itself.

Each execution records normalized arguments, bounded result, status, latency, and stable error code
through the conversation audit boundary. Stored tool names are registry names, not provider-native
tool types.

## 6. LangGraph Turn Flow

SV-US-011 uses `@langchain/langgraph` `StateGraph`. The durable conversation's
`langgraph_thread_id` is the graph thread identity. Application conversation tables remain the
audit source of truth; the local PostgreSQL checkpoint boundary remains separate and stores only
provider-neutral serializable workflow state.

The graph has these named nodes in order, with conditional edges where stated:

1. `load_context`: call `beginTurn`, load chronological bounded history, conversation summary,
   thread identity, and the current persisted source policy.
2. `classify_intent`: create a bounded intent classification from current text and history.
3. `extract_constraints`: normalize dates, timezone, location, category, budget, amenity,
   accessibility, rating, and approved metadata constraints without creating missing values.
4. `decide_tools`: decide whether one material clarification is required; otherwise prepare only
   registered tool definitions.
5. `select_data_sources`: list stored sources and select only concrete keys present in that result.
6. `retrieve_content`: execute the bounded registered tool loop. Re-intersect policy per call and
   stop after `MAX_TOOL_ITERATIONS` total calls.
7. `rank_and_filter`: pass retrieved candidates to the SV-US-009 ranker and retain its canonical
   recommendations, citations, disclosures, volatile details, and uncertainties.
8. `generate_answer`: return a deterministic clarification/no-evidence response when applicable;
   otherwise request a strict final provider response against the ranked evidence.
9. `persist_turn`: validate grounding, write the user and assistant messages, tool audits, citations,
   normalized usage, status, and provider-neutral graph checkpoint through `completeTurn` and the
   checkpoint boundary.

Each graph invocation supplies `{ configurable: { thread_id } }`. The workflow does not use
provider memory as a checkpointer. Checkpoints exclude API keys, authorization, provider/model
names, provider response/conversation IDs, raw provider bodies, raw SQL, and private source fields.

Intent and constraint state is advisory. It cannot create source authority or bypass retrieval
validation. A clarification is required only when a missing material constraint prevents reliable
retrieval or when the user's reference is ambiguous after bounded history is loaded. The server
asks exactly one concise question and never invents the missing value.

## 7. Grounding Gate

The provider final schema contains only:

- `answer`: bounded non-empty user-facing text;
- `citation_ids`: unique IDs drawn from the current ranker output;
- `recommendation_ids`: unique stable identities drawn from the current ranker output;
- `follow_up_questions`: at most three bounded strings.

The server, not the model, constructs public citation and recommendation objects. Unknown IDs are
invalid. The answer must not introduce a URL outside the selected current citations and must not
assert an item or volatile fact that the selected ranked evidence labels uncertain. A malformed
response, unknown reference, unsupported URL, or missing evidence for a factual recommendation
fails the grounding gate; it is never returned as a successful answer.

The graph may retry a schema-invalid final response once while still inside the global tool/model
iteration and timeout bounds. It must not repair factual content by guessing. Exhaustion produces
the stable failed-turn behavior below.

## 8. Failure And Persistence Behavior

Validation and authentication failures occur before `beginTurn` and create no conversation audit
records. After `beginTurn`, every terminal path attempts exactly one `completeTurn`:

- success stores the user and assistant messages and marks the turn `succeeded`;
- retrieval unavailable, provider timeout/unavailable, invalid provider response, grounding
  failure, or iteration exhaustion stores the user message, a stable non-factual assistant fallback,
  audited tools/usage available so far, a stable `error_code`, and marks the turn `failed`.

After a failed turn is durably completed, the route returns a stable `503 chat_unavailable` for
transient retrieval/provider/timeout failures or `502 chat_generation_invalid` for invalid schema or
grounding. Public messages contain no upstream body or provider/model identity. If the database
itself prevents completion, the existing `503 conversation_unavailable` applies; no implementation
can claim a failed audit write succeeded.

An empty allowlist, zero retrieval results, or evidence insufficient for a supported answer is not
an infrastructure failure. It returns the successful no-evidence/clarification behavior from
section 3 and persists it normally.

## 9. Persistence Prohibitions

Conversation, message, tool-call, usage, and checkpoint JSON must pass the existing recursive
provider-key prohibition. They additionally must not store authorization headers, API keys, model
names, base URLs, provider response bodies, or raw database/query state. `usage_events.metadata`
may contain provider-neutral counts, outcome class, graph node, tool iteration count, allowlist
version, retrieval mode, and grounding outcome.

## 10. Verification Gate

SV-US-011 requires tests for strict request validation, exact-one identity and ownership, anonymous
and logged-in creation/continuation, chronological server-owned history, clarification,
no-evidence, allowlist intersection on every tool call, unknown tool rejection, instruction/SQL
injection, tool bounds, timeout abort, provider failure mapping, malformed structured output,
unknown evidence references, unsupported URLs, persistence of success/failure, provider-neutral
records/checkpoints, OpenAI `store: false`, Groq unsupported-parameter omission, and equivalent
normalized adapter results.

The normal formatting, lint, type, contract, unit, integration, security, Docker, and live pinned
database gates remain mandatory.
