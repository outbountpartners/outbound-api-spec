# ChatGPT Custom GPT — setup guide

How to wire this API into a ChatGPT Custom GPT in 4 minutes.

## What you'll get

A Custom GPT that can:
- Read the leaderboard, commission, meetings, clients, campaigns, users
- Create + update meetings, clients, campaigns
- Invite + manage users (if your key has `users:write`)

The OpenAPI spec is annotated with `x-openai-isConsequential: true` on every write operation, so ChatGPT **prompts the user for confirmation** before any POST or PATCH executes. Reads run silently.

## Prerequisites

1. ChatGPT Plus, Pro, Team, or Enterprise subscription (Custom GPTs require a paid plan)
2. A Portal API key with the scopes you want the GPT to use. Issue one from `portal.outboundpartners.com/api` (admin/super_admin only).

## Setup steps

1. Go to **https://chatgpt.com/gpts/editor** (or "Explore GPTs" → "Create").

2. **Configure** tab:
   - **Name:** "Outbound Partners Assistant" (or whatever)
   - **Description:** "Query and update the Outbound Partners portal — meetings, clients, campaigns, leaderboard, commission."
   - **Instructions** (paste verbatim, edit to taste):

     ```
     You're an assistant for the Outbound Partners portal. You have actions
     wired up that let you read and write meetings, clients, campaigns,
     users, leaderboard, and commission data.

     Conventions:
     - Always resolve names to UUIDs via list/search endpoints before
       writes. Never fabricate IDs.
     - For writes, confirm with the user in one block before invoking the
       action. The system will prompt for confirmation on its own for
       consequential operations, but a clear plain-English confirmation
       from you helps the user know what's about to happen.
     - When showing leaderboard, commission, or meeting lists, summarize
       conversationally — don't dump raw JSON. Pull out the headline (#1
       performer, behind-target SDRs, etc.).
     - Periods like "this week" / "last month" map to the period enum
       values exposed by leaderboard_get.
     - For commission, UK SDRs default to 12 meetings/month, SA SDRs to 6.
       The endpoint computes everything else.
     - For meetings, sub_status values starting with not_qualified_*
       exclude the meeting from commission — flag this to the user before
       applying.

     If a tool call returns an error, surface the error code and message
     clearly. Don't retry blindly.
     ```

3. **Actions** section → **Create new action**:

   - **Authentication:** API Key
     - **Auth Type:** Custom
     - **Custom Header Name:** `x-api-key`
     - **API Key:** paste your `opk_live_...` key

   - **Schema:** click **Import from URL** and paste:

     ```
     https://sirhkzqpdgarrcyqnjtl.supabase.co/functions/v1/api/openapi.json
     ```

     Or paste the OpenAPI YAML directly from this repo's [`openapi.yaml`](../openapi.yaml).

4. **Privacy Policy URL** (required by ChatGPT for published GPTs):
   - Use `https://portal.outboundpartners.com/privacy` (or whatever your privacy page is)

5. **Save**. Test by asking your GPT:
   - "Show me this week's leaderboard"
   - "How's commission looking this month?"
   - "Create a test meeting for client X" *(your GPT should ask for confirmation before posting)*

## What the consequential marker does

When ChatGPT sees `x-openai-isConsequential: true` on an operation:

- It displays a confirmation dialog before invoking
- "Always allow" is disabled — every call needs a click
- The action still goes through your API key (so scope enforcement is unaffected)

Operations marked consequential in this spec:

| Operation | Why |
|---|---|
| `POST /v1/meetings` | Creates a meeting record (affects SDR commission) |
| `PATCH /v1/meetings/{id}` | Can change outcome → affects commission and feedback emails |
| `POST /v1/clients` | Creates a client |
| `PATCH /v1/clients/{id}` | Can flip Active/Inactive |
| `POST /v1/campaigns` | Creates a campaign |
| `PATCH /v1/campaigns/{id}` | Can adjust targets |
| `POST /v1/users` | Sends an invitation email |
| `PATCH /v1/users/{id}` | Can deactivate users (logs them out) |

## Limits & caveats

- ChatGPT Custom GPTs cap at **30 operations** per Action. We currently expose **22 operations across 14 paths**, comfortably under the limit.
- ChatGPT doesn't currently support `multipart/form-data` uploads in Actions — fine for us, we don't expose any.
- Rate limits are enforced at the Portal API key (default 600 req/min), not at the OpenAI layer.
- The OpenAPI doc lists the live Supabase URL (`sirhkzqpdgarrcyqnjtl.supabase.co/functions/v1/api`) as **server[0]**. ChatGPT's Action importer picks server[0] by default, so imports work out-of-the-box. The planned custom domain (`api.outboundpartners.com`) is listed as server[1] for forward compatibility — once the CNAME is provisioned, swap the order.

## Want write access via MCP instead?

ChatGPT Custom GPT Actions are the simplest integration. If you need agentic flows that compose multiple tools, the MCP server is a better fit — see [`outbound-mcp`](https://github.com/outbountpartners/outbound-mcp).

## Troubleshooting

| Symptom | Fix |
|---|---|
| "Schema validation failed" on import | Check the spec is valid: `redocly lint openapi.yaml` — **zero errors expected** (warnings about missing description fields on individual schemas are fine). |
| 401 on every call | API key not set under Action → Authentication, OR header name wrong (must be exactly `x-api-key`). |
| 403 on write actions | Your key lacks the required write scope. Issue a new key with the right scopes via `/api`. |
| 403 `forbidden_super_admin_required` on `/v1/users` writes | The `users:write` scope alone is not enough — the issuing key must also be flagged as super-admin. Re-issue from `/api` while signed in as a super_admin. |
| 422 `validation_failed` with `details` array | The body or query failed schema validation. The `details` array tells you exactly which fields and why. |
| 429 Too Many Requests | You're over the per-key rate limit (default 600 rpm). Either back off or request a higher limit on the key. |
| 409 `idempotency_conflict` | You re-used an `Idempotency-Key` with a different request body. Pick a new key per logical request. |
| 409 `idempotency_in_flight` | A previous request with this key is still being processed. Wait a few seconds and retry. |
| Action invocation blocked / confirmation dialog | You're hitting the consequential confirmation — that's working as designed. Confirm in the dialog. |
