---
name: create-jira-ticket
description: Create a Jira ticket using the Atlassian CLI (acli). Uses git worktrees for code research so working directories are not disturbed. Use when the user says "create a ticket", "make a Jira ticket", "new Jira issue", "/create-jira-ticket", or similar.
version: 2.0.0
---

# Create Jira Ticket Skill

Creates a Jira ticket by writing a JSON file and running `acli jira workitem create --from-json`. Supports multiple boards/projects via a per-user config.

**Config file:** `~/.claude/skills/create-jira-ticket/config.json`

---

## Step 0: Onboarding Check

Every time this skill runs, first read `~/.claude/skills/create-jira-ticket/config.json`.

If the file is empty (`{}`) or missing the `boards` key, run the **onboarding flow** below. Otherwise, skip to Step 1.

### Onboarding Flow

#### 0a. Check for acli
Run this exact command:
```bash
which acli
```
See if the Atlassian CLI is installed.


- If **not found**, tell the user it's required and offer to install it:
  ```bash
  brew tap atlassian/homebrew-acli && brew install acli
  ```
  After install, verify with `acli --version`. If it still fails, link the user to: https://developer.atlassian.com/cloud/acli/guides/install-macos/

#### 0b. Jira URL

Ask the user: "What's your Jira site URL? (e.g. `https://your-company.atlassian.net`)"

Normalize the answer: strip any trailing slash and any path after the host. Save it as `jiraUrl` in the config. It will be used for the `acli` login site and for constructing ticket links.

#### 0c. Check authentication

Run this exact command:
```bash
acli jira auth status
```

- If the user **is authenticated**, note their email from the output and continue to 0d.
- If **not authenticated**, tell the user they need to log in and run (substituting the `jiraUrl` from 0b):
  ```bash
  acli jira auth login --web --site <jiraUrl>
  ```
  This will open a browser. Tell the user to click **"Accept"** in the browser to complete authentication. After they confirm, re-run `acli jira auth status` to verify and get their email.

#### 0d. Discover boards

First, auto-detect the user's most active boards from their recent tickets.

**Step 1:** Get the user's recent ticket keys and extract project prefixes. Use this exact command (substituting the authenticated email from step 0c):
```bash
acli jira workitem search --jql "reporter = '<email>'" --json \
  | jq -r '.[].key' \
  | grep -oE '^[A-Z]+' \
  | sort | uniq -c | sort -rn
```

This outputs lines like `30 MP`, ranked by frequency. Take the top prefixes (up to 4).

**Step 2:** For each unique project prefix, find the matching scrum board. Use this exact command — do not substitute alternative jq patterns:
```bash
acli jira board search --json | jq '.values[] | select(.location | contains("<PREFIX>")) | select(.type | contains("scrum"))'
```

If no scrum board exists for a prefix, try without the type filter to find any board:
```bash
acli jira board search --json | jq '.values[] | select(.location | contains("<PREFIX>"))'
```

**Step 3:** Present the discovered boards as the top suggestions using `AskUserQuestion` (max 4 options). The last option should be "Other — show all boards" so the user can pick from the full list if needed.

Example:
> "Based on your recent tickets, here are your most active boards. Which ones would you like to create tickets on?"
>
> 1. Messaging Platform Scrum (MP) — scrum [id: 59]
> 2. ENG board (ENG) — scrum [id: 63]
> 3. Other — show all boards

If the user picks "Other", display the full board list as a table (from `acli jira board search`) and let them reply with their selections.

Use `multiSelect: true` on the `AskUserQuestion` so the user can pick multiple boards at once.

#### 0e. Assignee preference

Look up the user's Jira account ID automatically. Using the email obtained from Step 0c, run this exact command (substituting the authenticated email):

```bash
acli jira workitem search --jql "assignee = '<email>'" --json --limit 1 | jq '.[].fields.assignee.accountId'
```

Get the `accountId` from the output. Do not ask the user for this — it should be discovered automatically.

Then ask:

> "Would you like tickets to be automatically assigned to you by default?"

- If **yes**, save `"autoAssign": true` and `"assigneeAccountId": "<accountId>"` in the config.
- If **no**, save `"autoAssign": false` and `"assigneeAccountId": "<accountId>"` in the config. The account ID is still saved in case the user wants to assign a specific ticket to themselves later, but tickets will be created unassigned by default.

#### 0f. Repos directory

Ask the user: "Where are your repos located? (e.g. `~/example`)"

List the subdirectories at that path to discover available service/repo names. These will be used for service name inference when creating tickets.

#### 0g. Save config

Write the config to `~/.claude/skills/create-jira-ticket/config.json` in this format:

```json
{
  "jiraUrl": "https://your-company.atlassian.net",
  "reposDir": "~/example",
  "autoAssign": true,
  "assigneeAccountId": "712020:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "boards": [
    { "id": 42, "name": "MP Board", "projectKey": "MP" },
    { "id": 87, "name": "PLAT Board", "projectKey": "PLAT" }
  ]
}
```

Extract the board `id`, `name`, and `projectKey` from the output of `acli jira board search`. The board ID is needed for commands like sprint operations that require a board reference.

#### 0h. Allowlist cleanup command

Ask the user: "Would you like to auto-allow the following so they don't prompt every time?
- `Write(/tmp/jira-ticket.json)` — writing the ticket draft
- `Bash(rm -f /tmp/jira-ticket.json)` — cleaning up after creation"

If they say yes, read `~/.claude/settings.json`, add both `"Write(/tmp/jira-ticket.json)"` and `"Bash(rm -f /tmp/jira-ticket.json)"` to the `permissions.allow` array, and write the file back.

Tell the user: "Onboarding complete! You can re-run onboarding anytime by clearing `~/.claude/skills/create-jira-ticket/config.json`."

---

## Step 1: Collect Inputs

**Be autonomous.** Extract as much as possible from the user's initial message. Only ask about what's truly missing, and combine all remaining questions into a **single** `AskUserQuestion` call (max one round of questions before researching).

The inputs needed are:

1. **Board / Project** — If the user has **more than one board** in their config, ask which board this ticket should go on. If they have **only one board**, use it automatically without asking.
2. **Service** — The user will usually mention this in their request (e.g. "create a ticket for the router"). To resolve the service name:
   - List subdirectories in the `reposDir` from config to get available repo/service names.
   - If the user gave an exact match (e.g. "send-service"), use it directly.
   - If the user gave a partial/informal name (e.g. "router", "inbound", "streamer"), fuzzy-match against the repo names. If there's exactly one match, use it (no need to confirm). If there are multiple matches, include a question in the batch.
   - If no match, include a question in the batch.
   - This will be used as the `[service-name]` prefix in the summary.
3. **Description** — Use whatever the user provided. If they gave enough detail, proceed directly to research. If the description is missing or too vague to write acceptance criteria, ask for it.
4. **Summary** — **Never ask for this.** Auto-generate it from the description after research. It should be a concise imperative phrase (e.g. "Add retry logic to failed sends"). Format it as `[service-name] <generated summary>`.
5. **Sprint** — If not mentioned, default to the **current active sprint**. Fetch the active sprint automatically:
   ```bash
   acli jira board list-sprints --state active --id <BOARD_ID> --json
   ```
   Use the active sprint. Only ask if there are multiple active sprints or the user needs to choose between active/future/backlog.

**Key principle:** If you have enough information to research and draft the ticket, do it. Don't ask questions you can answer yourself. The user will review the final preview and can request changes there.

---

## Step 2: Research

**Always research.** Every ticket benefits from grounding in real code. Research informs the description, acceptance criteria, and surfaces relevant implementation details the user may not have mentioned.

### 2a. Determine repos to research

Start from the primary service identified in Step 1. Then expand based on the task:

- **Shared contracts**: If the task touches a gRPC or Kafka boundary, include `~/grpc` (proto definitions) and any consumer/producer services.
- **API tasks**: If the task involves an HTTP endpoint exposed to clients, include `example-api` (the monolith).
- **Cross-service data flow**: If the task involves data moving between services (e.g. a field added to an event), include all services that produce or consume that data.
- **Related services**: If the user's description implies interaction with another service (e.g. "send-service calls message-router"), include that service too.
- **External APIs**: If the task involves a third-party provider (Infobip, Twilio, Klaviyo, etc.), use `WebFetch` to pull relevant API documentation. Search for the provider's official API docs and fetch the specific endpoint or webhook reference.

Use your judgment — for a small scoped change, one or two repos is fine. For a larger task, cast a wide net.

### 2b. Create worktrees

For each repo you need to research, fetch latest and create a temporary worktree from `origin/main`. This gives Explore agents a real filesystem they can use with normal tools (Glob, Grep, Read) — no git plumbing required.

**First, resolve the absolute path to the repo.** The `reposDir` from config may contain `~`, which won't expand inside a constructed string. Resolve it with:

```bash
REPO_PATH=$(eval echo "<reposDir>/<repo>")
```

For example, if `reposDir` is `~/example` and the repo is `send-service`:
```bash
REPO_PATH=$(eval echo "~/example/send-service")
# → /Users/connor.marchand/example/send-service
```

Then use `$REPO_PATH` (the absolute path) for all git commands — this works regardless of your current working directory.

Worktrees live at `<repo>/.worktrees/research` and are **persistent** — they are reused across ticket creations, never deleted. Before using, ensure `.worktrees/` is gitignored:

```bash
grep -q '\.worktrees' "$REPO_PATH/.gitignore" || echo '.worktrees/' >> "$REPO_PATH/.gitignore"
```

Then bring the worktree up to date. If it doesn't exist yet, create it. If it already exists, just fetch and reset to pick up any new commits on `origin/main`:

```bash
# Fetch latest
git -C "$REPO_PATH" fetch origin

# If worktree doesn't exist, create it
if [ ! -d "$REPO_PATH/.worktrees/research" ]; then
  git -C "$REPO_PATH" worktree add "$REPO_PATH/.worktrees/research" origin/main
else
  # Already exists — reset to latest origin/main
  git -C "$REPO_PATH/.worktrees/research" reset --hard origin/main
fi
```

Prepare all worktrees before spawning any agents.

### 2c. Spawn Explore agents in parallel

Spawn one **Explore** agent per repo (subagent_type: `Explore`, model: `sonnet`). All agents run in parallel.

Each agent works entirely within its `/tmp/research-<repo>` worktree using standard tools (Glob, Grep, Read, Bash). No git plumbing commands needed.

Give each agent a thorough, specific research brief. Tell it exactly what you need to understand to write a good ticket:

```
Agent(
  subagent_type: "Explore",
  model: "sonnet",
  description: "Research <repo>",
  prompt: "Thoroughly research <REPO_PATH>/.worktrees/research for the following task: <task description>

  Specifically, I need to understand:
  - <specific question 1 — e.g. what struct holds the callback payload and what fields it has>
  - <specific question 2 — e.g. where is the gRPC call made and what data it returns>
  - <specific question 3 — e.g. which Kafka topic is produced to, and what does the message schema look like>
  - <any other relevant details for writing precise acceptance criteria>

  Use Glob, Grep, and Read freely. Go deep — follow imports, read handler implementations, find the actual data structures. Return:
  - Exact file paths and line numbers for all relevant code
  - Struct/type definitions with field names
  - Function signatures and their call sites
  - Any gotchas, constraints, or related areas that would affect implementation"
)
```

If you also need to fetch external API docs, do that with `WebFetch` in parallel alongside the Explore agents.

### 2d. Synthesize findings

Before writing the ticket, synthesize what you learned across all repos:
- How do the pieces connect? (e.g. proto field → gRPC response → service struct → Kafka event)
- What are the exact field names, types, and locations?
- Are there multiple places that need to change?
- Are there any constraints or edge cases the user should know about?

Use this synthesis to write a ticket that is specific, accurate, and actionable — with real field names, real function names, and real file references rather than vague descriptions.

**Important:** When referencing code locations in the ticket description or Additional Information, always format them as GitHub links:
```
https://github.com/Stodge-Inc/{repo}/blob/main/{path/to/file}#L{line}
```
Never use bare file paths. All repos are in the `Stodge-Inc` GitHub org.

---

## Step 3: Determine Type

Based on the user's description:
- If it sounds like a bug/defect (e.g. "broken", "not working", "error", "fix"), set type to `Defect`
- Otherwise, default to `Task`

---

## Step 4: Write the JSON File

Write `/tmp/jira-ticket.json` with the following structure. The description must be in Atlassian Document Format (ADF) with sections for **Description**, **Acceptance Criteria**, and optionally **Additional Information**.

Derive clear acceptance criteria from the user's description. If they didn't provide explicit criteria, infer reasonable ones.

### ADF Formatting Rules (description body only, NOT the summary)

**Inline code** - Use the `code` mark for field names, service names, function names, struct names, etc.:
```json
{ "type": "text", "text": "ShopId", "marks": [{ "type": "code" }] }
```

**Hyperlinks** - Use the `link` mark for GitHub URLs and any other links. Never paste raw URLs as plain text:
```json
{ "type": "text", "text": "infobip.go#L217", "marks": [{ "type": "link", "attrs": { "href": "https://github.com/Stodge-Inc/message-webhooks/blob/main/cmd/message-webhooks-worker/worker/infobip.go#L217" } }] }
```

**Combining marks** - You can combine code + link or other marks in the same `marks` array.

**Summary field** - The `summary` is plain text only. No ADF formatting, no code marks, no links.

### Example `/tmp/jira-ticket.json`:

```json
{
  "assignee": "<assigneeAccountId from config if autoAssign is true, otherwise omit this field>",
  "description": {
    "type": "doc",
    "version": 1,
    "content": [
      {
        "type": "heading",
        "attrs": { "level": 2 },
        "content": [{ "type": "text", "text": "Description" }]
      },
      {
        "type": "paragraph",
        "content": [
          { "type": "text", "text": "Update " },
          { "type": "text", "text": "send-service-infobip", "marks": [{ "type": "code" }] },
          { "type": "text", "text": " to include billing fields." }
        ]
      },
      {
        "type": "heading",
        "attrs": { "level": 2 },
        "content": [{ "type": "text", "text": "Acceptance Criteria" }]
      },
      {
        "type": "bulletList",
        "content": [
          {
            "type": "listItem",
            "content": [{ "type": "paragraph", "content": [
              { "type": "text", "text": "Callback data includes " },
              { "type": "text", "text": "ShopId", "marks": [{ "type": "code" }] },
              { "type": "text", "text": " and " },
              { "type": "text", "text": "CountryCode", "marks": [{ "type": "code" }] }
            ] }]
          }
        ]
      },
      {
        "type": "heading",
        "attrs": { "level": 2 },
        "content": [{ "type": "text", "text": "Additional Information" }]
      },
      {
        "type": "bulletList",
        "content": [
          {
            "type": "listItem",
            "content": [{ "type": "paragraph", "content": [
              { "type": "text", "text": "gRPC call: " },
              { "type": "text", "text": "infobip.go#L217", "marks": [{ "type": "link", "attrs": { "href": "https://github.com/Stodge-Inc/message-webhooks/blob/main/cmd/message-webhooks-worker/worker/infobip.go#L217" } }] }
            ] }]
          }
        ]
      }
    ]
  },
  "projectKey": "<selected board's projectKey from config>",
  "sprintId": "<sprint ID if user selected a sprint, omit if backlog>",
  "summary": "[send-service-infobip] Add billing fields to callback data",
  "type": "Task"
}
```

Only add an "Additional Information" section if there is relevant extra context (e.g. code links from the research step).

Use the Write tool to create `/tmp/jira-ticket.json` with the populated content.

---

## Step 5: Preview and Confirm

Show the user a preview before creating:

```
Ticket Preview:
  Project:  <projectKey from selected board>
  Type:     Task (or Defect)
  Sprint:   <sprint name, or "Backlog">
  Summary:  [service-name] Summary text
  Assignee: Connor Marchand

  Description:
    <description text>

  Acceptance Criteria:
    - criterion 1
    - criterion 2
```

Ask: "Does this look good? Reply 'yes' to create, or describe any changes."

---

## Step 6: Create the Ticket

```bash
acli jira workitem create --from-json /tmp/jira-ticket.json --json
```

---

## Step 7: Report, Open, and Sprint

Parse the output and report:
- The ticket key (e.g. `MP-1234`)
- Link: `<jiraUrl>/browse/<KEY>` (using `jiraUrl` from config)

Open the ticket in the browser:
```bash
open "<jiraUrl>/browse/<KEY>"
```

Clean up:
```bash
rm -f /tmp/jira-ticket.json
```

The sprint assignment is handled via the `sprintId` field in the JSON file, so no separate move command is needed.
