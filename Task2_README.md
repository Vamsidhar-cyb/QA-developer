# Task 2 — n8n Workflow README

**Author:** Sriharsha  
**Date:** 2 June 2025  
**Workflow name:** Morning Brief — GitHub Trending + Discord Digest

---

## APIs Used & Why

| API | Endpoint | Why |
|-----|----------|-----|
| **GitHub REST API v3** | `GET /search/repositories` | Free, no auth required for basic queries (PAT increases rate limit to 5000/hr). Returns rich repo metadata including stars, forks, language, and topics — ideal for a "trending" digest. |
| **GitHub REST API v3** | `GET /repos/{owner}/{repo}/readme` | Enriches each repo with its README (base64-encoded). Decoding a 280-char snippet gives readers a real flavour of the project without clicking through. |
| **Discord Webhook** | `POST {webhook_url}` | Zero setup for the recipient — no bot registration needed. Supports rich embeds (colors, fields, timestamps). The same webhook is reused by the Bonus uptime monitor, keeping credentials centralised. |

---

## Workflow Steps

```
Schedule (every 1h)
  → GitHub search repos          [First HTTP Request]
  → Transform: top 5, reshape    [Code node]
  → Enrich: fetch README         [Second HTTP Request — per item]
  → Merge README snippet         [Code node — base64 decode]
  → IF stars ≥ 1000?             [Conditional branch]
      YES → badge = "🔥 HOT"
      NO  → badge = "📈 RISING"
  → Merge branches
  → Build Discord payload        [Code node]
  → Discord POST                 [HTTP Request]
  → Delivery check / log         [Code node]

Error Trigger → Error Handler    [Parallel — catches any node failure]
```

---

## Transformation Logic

The **Transform — Top 5 Repos** Code node does the following:

1. Checks for missing `items` key (upstream GitHub failure) and returns a structured error object instead of crashing.
2. Sorts the 10 results by `stargazers_count` descending (GitHub's API should already do this, but the extra sort is a defensive guarantee).
3. Slices the top 5.
4. Maps each raw GitHub object to a clean flat shape: `{ rank, name, description, stars, forks, language, url, owner, defaultBranch, updatedAt, openIssues, topics }`.

The **Merge README Snippet** Code node:

- Decodes the base64 README content from the enrichment response.
- Strips Markdown symbols (`#`, `*`, `` ` ``, `>`, `[`, `]`) to produce readable plain text.
- Collapses newlines and trims to 280 characters with an ellipsis.
- Handles three failure cases: missing `content` field, base64 decode error, or 404 (repo has no README).

---

## Conditional Branch

**Threshold:** `stars >= 1000`

- **≥ 1000 stars** → item is tagged `badge: "🔥 HOT"` and `tier: "hot"`. These are established, popular repos.
- **< 1000 stars** → item is tagged `badge: "📈 RISING"` and `tier: "rising"`. Newer repos worth watching.

Both branches feed into a **Merge** node (SQL UNION mode) that re-combines items ordered by star count, so the Discord message still shows a ranked list.

---

## Error Handling Strategy

| Mechanism | Where | Behaviour |
|-----------|-------|-----------|
| `continueOnFail: true` | GitHub Search, Enrich README, Discord POST | Node failure is caught and passed downstream as an error object rather than halting the execution. The downstream Code node inspects the response and returns a structured `{ error: true, … }` item. |
| Guard clause in Transform | Code node | If `items` is missing (GitHub returned an error body), the node immediately returns `{ error: true, message: '…' }` and the rest of the chain propagates gracefully. |
| Error Trigger + Handler | Workflow-level | The **Error Trigger** node fires if *any* node throws an unhandled exception. The **Error Handler** Code node logs the error object to `console.error` (visible in n8n execution logs) and returns a `{ handled: true, … }` record. In production, this would also POST to a Slack `#alerts` channel or send an email. |
| Delivery check | Post-Discord | Checks the Discord HTTP response for error codes and logs the outcome. |

**The workflow will never silently succeed when data is missing** — every failure path either routes to the Error Trigger or produces an explicit error object in the execution log.

---

## Credentials (never hard-coded)

| Credential name | Type | Used by |
|-----------------|------|---------|
| `GitHub PAT` | HTTP Header Auth (`Authorization: Bearer <token>`) | GitHub Search, Enrich README |
| `Discord Webhook URL` | HTTP Query Auth (URL field) | Discord POST |

Both are stored in n8n's encrypted Credentials store and referenced by name from each node — no secrets appear in the workflow JSON.

---

## Testing

To trigger manually instead of waiting for the schedule:

```bash
# Activate the workflow, then execute once from the UI, OR
# Use the Webhook trigger variant and hit it with curl:
curl -X POST https://<your-n8n-host>/webhook/<webhook-id>
```

Expected output in Discord: a single message with a rich embed containing 5 repo entries, each with README snippet, star count, language, topics, and a direct GitHub link.
