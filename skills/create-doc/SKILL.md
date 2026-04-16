---
name: create-doc
description: Use when the user invokes /create-doc, "create a discovery doc", "create a technical design", "start a discovery", "write a design doc", or any variation. Creates a Google Doc from a template (Technical Discovery or Technical Design) and opens it in the browser.
version: 1.0.0
disable-model-invocation: true
---

# Create Doc Skill

Creates a formatted Google Doc from a template. Supports:
- **Technical Discovery** — research-focused; spawns Explore agents per repo to answer specific questions
- **Technical Design** — design-focused; user-driven with agent validation of code paths

Templates: `~/.claude/skills/create-doc/templates/`

---

## Phase 1: Collect Inputs

Ask (in sequence):

1. **Doc type** — Technical Discovery or Technical Design?

2. **Title** — Short name (e.g. "MessageRouter", "Subscription Pause Flow")

3. **Ticket number** — e.g. `PS-1234`. Optional.

4. **Background** — Why is this needed? What problem or initiative?

Then branch based on doc type:

### If Technical Discovery:

5. **Purpose & Scope** — What should this doc accomplish? Ask for:
   - Overall purpose (1-2 sentences)
   - What's **in scope** for this discovery
   - What's **out of scope** (often things to address in a future phase)

6. **Objectives & Questions** — List the specific questions the resulting technical design must answer. Framed as: "The technical design resulting from this discovery must answer these questions." These drive the research.

7. **Activities & Approach** — How will research be done? (code review, team meetings, etc.) Skip = auto-generated.

8. **Repos** — Scan and show numbered list:
   ```bash
   for d in ~/example/*/; do [ -d "${d}.git" ] && basename "${d%/}"; done | sort
   ```
   Ask for space-separated numbers or `all`.

### If Technical Design:

5. **Related discovery doc** — Link to the discovery doc if one exists (paste URL or "N/A").

6. **Goals & Non-Goals** — What are the concrete outcomes? What's explicitly out of scope?

7. **Proposed approach** — Ask the user to describe the design at a high level. They can paste from a discovery doc or describe from scratch.

8. **Repos** — Same scan as above. Used to validate code paths and gather exact field names/types.

---

## Phase 2: Research (Both Types)

### Pull repos

For each selected repo:
```bash
DEFAULT=$(git -C ~/example/REPO symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||')
DEFAULT=${DEFAULT:-main}
git -C ~/example/REPO checkout "$DEFAULT" && git -C ~/example/REPO pull
```

Skip failed repos; note them in the doc.

### Spawn Explore agents (parallel, model: haiku)

One agent per repo. Tailor the prompt per doc type:

**Discovery agent prompt:**
```
Discovery: {{TITLE}}
Background: {{BACKGROUND}}
Objectives to answer: {{OBJECTIVES_LIST}}
Repo: /Users/connor.marchand/example/REPO

Research this repo and return:
1. Role of this service in this context (or "not relevant" if so)
2. Answers to each objective from this repo's code — be specific: function names, struct fields, file locations
3. Key files (as GitHub links: https://github.com/Stodge-Inc/REPO/blob/main/PATH#Lnn)
4. Data structures / payload shapes with field names and types
5. API/proto/event contracts relevant to this topic
6. DB/storage details if relevant
7. Observations worth flagging (not solutions)
8. Open questions this repo raises
```

**Design agent prompt:**
```
Design: {{TITLE}}
User's proposed approach: {{APPROACH}}
Repo: /Users/connor.marchand/example/REPO

Validate and enrich the proposed design for this repo:
1. Does this repo need changes? If not, say so.
2. Exact files and functions that need to change
3. Current field names/types for any structs or protos being modified (link to source)
4. Any constraints, existing patterns, or gotchas to be aware of
5. GitHub links for all relevant code (https://github.com/Stodge-Inc/REPO/blob/main/PATH#Lnn)
```

---

## Phase 3: Compile the Document

Use the template at `~/.claude/skills/create-doc/templates/technical-discovery.md` or `technical-design.md` as the structural guide. Output is HTML.

### Technical Discovery — fill in each section:

- **Objectives & Questions**: Mark ✅ if research found a clear answer, 💬 if still open
- **Findings & Insights**: Organize by topic (not by repo). Synthesize across agents. For payload comparisons use `<table>` with ✅/❌ per producer. Include `<a href="...">` GitHub links inline.
- **Open Questions**: Aggregate from all agents + user. Unresolved = bullet list.
- **Possible Solutions**: Only if research surfaced clear architectural options. Rough options, no formal proposals.
- **Other Considerations**: Approaches considered but rejected/deferred.

### Technical Design — fill in each section:

- **Overview**: 2-3 sentences. What is being built and why.
- **Service Changes**: One subsection per affected service. Use agent findings for exact file/function references.
- **API/Proto/Event Changes**: Full field tables (name, type, required, description) from agent findings.
- **DB Changes**: Schema details from agent findings.
- **Implementation Plan**: Phases that are independently deployable where possible.
- **Testing Strategy**: Unit, integration, manual.
- **Rollout**: Deployment order, feature flags, rollback.
- **Risks**: Table with severity (H/M/L) and mitigation.

---

## Phase 4: Create the Google Doc

### Write HTML to /tmp/doc.html

Use the Write tool. Full HTML with semantic tags per the template structure.

### Upload via curl

```bash
export GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE=/Users/connor.marchand/.config/gws/credentials.json

ACCESS_TOKEN=$(gws auth export --unmasked 2>/dev/null | python3 -c "
import sys, json, urllib.request, urllib.parse, ssl
creds = json.load(sys.stdin)
ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
data = urllib.parse.urlencode({
    'client_id': creds['client_id'],
    'client_secret': creds['client_secret'],
    'refresh_token': creds['refresh_token'],
    'grant_type': 'refresh_token'
}).encode()
req = urllib.request.Request('https://oauth2.googleapis.com/token', data=data)
print(json.loads(urllib.request.urlopen(req, context=ctx).read())['access_token'])
")

RESULT=$(curl -s -X POST \
  "https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&fields=id,name,webViewLink" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -F "metadata={\"name\":\"DOC_TITLE\",\"mimeType\":\"application/vnd.google-apps.document\"};type=application/json" \
  -F "file=@/tmp/doc.html;type=text/html")

LINK=$(echo "$RESULT" | python3 -c "import sys,json; print(json.load(sys.stdin)['webViewLink'])")
echo "$LINK"
open "$LINK"
rm /tmp/doc.html
```

Note: `gws --upload` doesn't set the media content-type so Drive rejects HTML→Doc conversion. The curl approach above sets `type=text/html` on the file part, which fixes this.

---

## Phase 5: Report

Show:
- Doc type and title
- For discovery: which objectives are ✅ vs 💬, count of open questions
- For design: which services are affected
- The Google Doc link

Ask: "Would you like to refine any section or dig deeper into anything?"

If yes, spawn targeted Explore agents, update the HTML, and re-upload as a new doc (same title).
