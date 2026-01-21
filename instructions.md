# Copilot Instructions — CorreoRoutingDemo (Azure Logic Apps + Azure OpenAI)

## Big picture
- This repo manages an **Azure Logic App (Consumption)** that classifies incoming emails and routes them using **Azure OpenAI** (chat/completions).
- Main flow: **Outlook trigger (When a new email arrives V2)** → HTML-to-text conversion → build `email_text` → HTTP call to Azure OpenAI → Parse JSON → routing actions.
- The Outlook trigger + connectors are **owned by `$connections`**; do not break existing connector bindings.

## Key directories / files (expected)
- `scripts/` — Azure CLI automation (export/audit/patch).
- `exports/` — generated outputs (`logicapp.json`, `definition.json`, `connections.json`).
- `definitions/` — edited Logic App definitions (`*.definition.json`, `*.patched.json`).
- `prompts/` — prompt text for classification (system message) and agent prompts.

## Critical conventions (Logic Apps Consumption)
- **Do not change `triggers`** when patching an existing workflow. Only update:
  - `properties.definition.parameters`
  - `properties.definition.actions`
- In Logic Apps **Consumption**, do not use `"type": "SecureString"` inside `properties.definition.parameters`.
  - Use `"type": "String"` and **never commit defaultValue secrets**.
- Normalize endpoint inputs: always wrap endpoint with `trim()` and avoid newline/space issues:
  - `@concat(trim(parameters('aoai_endpoint')), '/openai/deployments/', ...)`

## Email fields mapping
- Always read from the trigger (Outlook V2 payload):
  - `email_subject` from `triggerBody()?['subject']`
  - `email_from` from `triggerBody()?['from']?['emailAddress']?['address']`
  - `email_body_html` from `triggerBody()?['body']?['content']`
- Build `email_text` with context:
  - `FROM`, `SUBJECT`, `BODY` (BODY should come from `body('Html_a_texto')`)

## Connectors / actions patterns
- HTML conversion action uses Conversion Service connector:
  - `Html_a_texto` must pass the real HTML: `@variables('email_body_html')`
  - Extract output robustly: `coalesce(body('Html_a_texto')?['text'], body('Html_a_texto'), '')`
- Azure OpenAI HTTP call:
  - Path: `/openai/deployments/{deployment}/chat/completions?api-version={apiVersion}`
  - Headers: `Content-Type: application/json`, `api-key: <parameter>`
  - Use `retryPolicy` + `timeout` (e.g., exponential retries, 30s)

## Model output contract (must match Parse JSON)
- The assistant must return JSON with exactly:
  - `intent` (string)
  - `products` (array of strings)
  - `country` (string)
  - `urgency` (string: P1/P2/P3)
  - `summary` (string)
  - `confidence` (number 0..1)
- Keep `Parse JSON` schema aligned to this contract.

## Safety / ops
- Never print or commit API keys. Use Azure CLI to **read keys only into env vars**, not echoed.
- Avoid email-loop in Catch scope:
  - If sending error emails via Outlook, add an early condition to ignore subjects starting with `[ERROR]`.

## Dev workflows (CLI)
- Export workflow:
  - `az logic workflow show -g <RG> -n <WF> -o json > exports/logicapp.json`
  - `jq '.properties.definition' exports/logicapp.json > exports/definition.json`
  - `jq '.properties.parameters."$connections"' exports/logicapp.json > exports/connections.json`
- OpenAI discovery:
  - `az resource list --resource-type Microsoft.CognitiveServices/accounts --query "[?name=='testIAyovoy']|[0]" -o json`
  - deployments: `az cognitiveservices account deployment list ...` (fallback `az rest` if needed)

## When making changes
- Prefer generating a patched `definition.patched.json` from exported `definition.json`.
- Apply changes via Portal Code View (safe) or via `az rest` PATCH only if triggers are preserved.
