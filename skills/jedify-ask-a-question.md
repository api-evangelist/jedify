---
name: Ask Jedify a question and get the answer
description: >-
  Ask a natural-language analytics question against your connected data through
  Jedify's asynchronous session/inquiry REST API, then poll for the grounded answer,
  SQL, and chart. Grounds an agent in the correct auth, async flow, and polling rules.
api: https://be.jedify.com/api
docs: https://docs.jedify.com/api-reference/overview
method: generated
source: https://docs.jedify.com/api-reference/endpoints
operations:
  - "POST /session/new"
  - "POST /session/{session_id}/inquiry/agent"
  - "GET /inquiry/{inquiry_id}/status"
  - "GET /inquiry/{inquiry_id}/result"
mcp_alternative:
  server: https://be.jedify.com/mcp/message
  tools: [ask_a_single_question, check_question_status, wait_for_completion, get_questions_data]
---

# Ask Jedify a question and get the answer

Use this to run one natural-language analytics question against a Jedify account's
connected data warehouse and retrieve the grounded answer.

## Authentication
Every REST request carries the header `X-API-Key: <YOUR_API_KEY>`. Create the key at
`app.jedify.com/settings/api-keys`. The key is bound to a user's identity and
permissions — results are scoped to what that user may see.

## Steps

1. **Create a session.**
   `POST /session/new` → returns a `session_id`. A session groups related questions
   so follow-ups share context.

2. **Submit the question.**
   `POST /session/{session_id}/inquiry/agent` with the natural-language question in the
   body → returns an `inquiry_id`. Processing is asynchronous; do not expect the answer
   in this response.

3. **Wait for the result.** Prefer the server-side long-poll:
   `GET /inquiry/{inquiry_id}/result`. If it returns HTTP **408**, the inquiry is still
   processing — retrying is safe; just call it again. Alternatively poll
   `GET /inquiry/{inquiry_id}/status` client-side.

4. **Read the outcome from `status.general`.** Terminal states: `done`, `failed`,
   `timeout`, `stopped`, `out_of_domain`, `no_data`. Only `done` carries an answer;
   `out_of_domain` means the question falls outside the connected data's scope and
   `no_data` means the query returned nothing — surface those to the user rather than
   retrying blindly.

5. **Use the result fields.** On `done` the response includes `answer` (natural language),
   `sql_query` (the generated SQL), `data` (structured values), `chart` (visualization
   metadata), and `explanation` (the reasoning). Show `answer` + `explanation`; offer
   `sql_query` for transparency.

## Rules
- There is **no idempotency key** — do not blindly resubmit a question on a network error;
  first check the existing `inquiry_id` status.
- Respect the terminal-state semantics above; `out_of_domain`/`no_data` are answers, not failures.
- To cancel a long-running inquiry, use the MCP `stop_question` tool (no documented REST stop endpoint).

## MCP alternative
The same flow is available agent-natively via the Jedify MCP server
(`https://be.jedify.com/mcp/message`, Asker mode): `ask_a_single_question` →
`check_question_status` / `wait_for_completion` → `get_questions_data`. Authenticate the
proxy with browser login or `JEDIFY_API_KEY` via `npx -y @jedify/mcp-auth`.
