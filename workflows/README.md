# n8n Workflows Setup

This folder contains starter workflows for your multi-agent orchestration.

## Included Workflows

- `whatsapp.json`: End-to-end intake and orchestration (Planner -> Developer -> Reviewer -> Deployer)
- `planner.json`: Single-purpose workflow to run Planner from a webhook
- `developer.json`: Developer + Reviewer + conditional Deployer chain
- `mcp.json`: MCP bridge workflow that routes `planner`, `full_pipeline`, and `ping` actions
- `mcp-ping.json`: Minimal ping workflow used by the MCP bridge for health checks
- `jira-task-manager.json`: Webhook-based Jira task manager (create/list/comment/transition/link_run)
- `whatsapp-message-templates.json`: Registry of Meta-approved WhatsApp templates for outbound status updates
- `whisper-transcription.json`: Dedicated webhook flow for audio detection and Whisper self-hosted transcription

## run_id Propagation

Workflows propagate a `run_id` across all agent calls.

- If a request includes `run_id`, it is reused.
- If it is missing, workflows generate one and forward it.
- The same `run_id` is sent to Planner, Developer, Reviewer, and Deployer.

Use this to correlate executions with backend state endpoints:

- `GET /runs/{run_id}`
- `GET /runs`

## Import Steps

1. Open n8n UI.
2. Go to Workflows.
3. Use Import from File.
4. Import each JSON from this folder.
5. Save and Activate workflows.

## Connection and Communication Flow

1. Incoming event enters one of the webhook nodes in n8n.
2. n8n sends HTTP requests to your multiagent API using `MULTIAGENT_API_BASE_URL`.
3. If your backend enforces API-key auth, set `MULTIAGENT_API_KEY` in n8n so the workflows forward `X-API-Key` automatically.
4. Agent chain expected by the workflows:
  - `POST /agents/planner`
  - `POST /agents/developer`
  - `POST /agents/reviewer`
  - `POST /agents/deployer`
5. n8n returns a JSON response to the webhook caller.

## Suggested Webhook Paths

- `/webhook/whatsapp-intake`
- `/webhook/planner-entry`
- `/webhook/developer-review-deploy`
- `/webhook/mcp-bridge`
- `/webhook/jira-task-manager`
- `/webhook/whisper-transcribe`

## Jira Task Manager Workflow

`jira-task-manager.json` expects `POST` payloads with an `action` field and Jira credentials.

Supported actions:

- `create_issue`
- `list_issues`
- `add_comment`
- `transition_issue`
- `link_run`

Minimal payload example:

```json
{
  "action": "create_issue",
  "jira_base_url": "https://your-company.atlassian.net",
  "jira_email": "you@company.com",
  "jira_api_token": "<jira-api-token>",
  "project_key": "MAS",
  "summary": "Implement retry in n8n",
  "description": "Add backoff and stage-level retries in workflows"
}
```

## Automatic run_id -> Jira Link (full pipeline)

`whatsapp.json` now auto-links the `run_id` to Jira when the request includes Jira context.

Send these optional fields in `/webhook/whatsapp-intake` or `/webhook/mcp-bridge` with `action=full_pipeline`:

- `jira_issue_key` (or `issue_key`)
- `jira_base_url`
- `jira_email`
- `jira_api_token`

When `jira_issue_key` is present, the workflow calls `jira-task-manager` with `action=link_run` and appends the link result in `jira_link` of the webhook response.

## Quick Test Payloads

### 1) Planner

```json
{
  "task": "Create a FastAPI endpoint for booking CRUD"
}
```

### 2) Developer Review Deploy

```json
{
  "plan": "Build booking CRUD endpoints with validation and tests"
}
```

### 3) WhatsApp Intake (raw text fallback)

```json
{
  "text": "Create a new backend service with auth and deployment"
}
```

## Notes

- These workflows are intentionally minimal for fast bootstrap.
- Add retries and richer error handling before production rollout.
- Ensure your backend deployment is updated so agent responses include `run_id`.

## WhatsApp Channel Completion

`whatsapp.json` now returns a `whatsapp_response` block in both success and rejection paths:

- `task_status`
- `pr_url`
- `deploy_url`
- `error_message`

This structure is designed for direct forwarding to Meta Cloud API send-message handlers.

`whisper-transcription.json` supports audio-first intake with `WHISPER_BASE_URL` and model marker `whisper-large-v3`.

## Smoke Tests

Use your n8n URL from Render and run these tests from any terminal.

### 1) Planner endpoint through n8n

```bash
curl -X POST "https://<your-n8n-host>/webhook/planner-entry" \
  -H "Content-Type: application/json" \
  -d '{"task":"Create booking CRUD API"}'
```

### 2) Full pipeline through n8n

```bash
curl -X POST "https://<your-n8n-host>/webhook/whatsapp-intake" \
  -H "Content-Type: application/json" \
  -d '{"text":"Build backend service with auth and deployment"}'
```

If these fail with connection errors, validate `MULTIAGENT_API_BASE_URL` and confirm your API is publicly reachable.
