# Compare Documentation Sources

Compare docs.publora.com (deployed site) against this repo's API documentation and the real API code.

## Instructions

Launch 10 parallel subagents to compare documentation across all sections. Each agent compares:
1. Remote docs at `apod01:~/p/post.actor/sites/docs.publora.com/content/`
2. Local docs in this repo at `./docs/`
3. Real API code at `../backend/src/app/routes/` (when applicable)

Use SSH to read remote files: `ssh apod01 "cat <path>"`

**IMPORTANT:** Run all 10 subagents in PARALLEL using a single message with 10 Task tool calls.

---

## Agent 1: Core Endpoints (create, get, delete, update, list)

Compare these files between remote and local:
- `endpoints/create-post.md`
- `endpoints/get-post.md`
- `endpoints/delete-post.md`
- `endpoints/update-post.md`
- `endpoints/list-posts.md`

For each file:
1. Read remote: `ssh apod01 "cat ~/p/post.actor/sites/docs.publora.com/content/endpoints/<file>"`
2. Read local: `./docs/endpoints/<file>`
3. Compare request/response schemas, examples, error codes
4. Check against API code: `../backend/src/app/routes/postRoutes*.js`

Report inconsistencies with file paths and line numbers.

---

## Agent 2: LinkedIn Endpoints

Compare these files:
- `endpoints/linkedin-statistics.md`
- `endpoints/linkedin-followers.md`
- `endpoints/linkedin-reactions.md`
- `endpoints/linkedin-profile-summary.md`
- `endpoints/linkedin-feed-retrieval.md`
- `endpoints/linkedin-comments.md`

Check against: `../backend/src/app/routes/linkedinRoutes*.js`

---

## Agent 3: Utility Endpoints

Compare these files:
- `endpoints/platform-connections.md`
- `endpoints/upload-media.md`
- `endpoints/webhooks.md`
- `endpoints/post-logs.md`
- `endpoints/test-connection.md`

Check against relevant routes in `../backend/src/app/routes/`

---

## Agent 4: Guides Part 1 (Scheduling & Posts)

Compare these files:
- `guides/scheduling.md`
- `guides/bulk-scheduling.md`
- `guides/update-scheduled-post.md`
- `guides/delete-scheduled-post.md`
- `guides/threading.md`
- `guides/threads-multi-post.md`
- `guides/twitter-threads.md`

---

## Agent 5: Guides Part 2 (Analytics & LinkedIn)

Compare these files:
- `guides/analytics.md`
- `guides/linkedin-analytics.md`
- `guides/linkedin-reactions.md`
- `guides/linkedin-comments.md`
- `guides/linkedin-mentions.md`

---

## Agent 6: Guides Part 3 (General)

Compare these files:
- `guides/cross-platform.md`
- `guides/error-handling.md`
- `guides/media-uploads.md`
- `guides/platform-limits.md`
- `guides/rate-limits.md`
- `guides/validation.md`
- `guides/workspace.md`
- `guides/list-connected-accounts.md`
- `guides/mcp-server.md`
- `guides/cursor-ai.md`

---

## Agent 7: Platform Documentation

Compare ALL platform files:
- `platforms/bluesky.md`
- `platforms/facebook.md`
- `platforms/instagram.md`
- `platforms/linkedin.md`
- `platforms/mastodon.md`
- `platforms/telegram.md`
- `platforms/threads.md`
- `platforms/tiktok.md`
- `platforms/x-twitter.md`
- `platforms/youtube.md`

---

## Agent 8: MCP Documentation

Compare ALL MCP files:
- `mcp/client-setup.md`
- `mcp/examples.md`
- `mcp/openclaw.md`
- `mcp/tools-reference.md`
- `mcp/troubleshooting.md`

---

## Agent 9: Examples Part 1 (JS/TS/Python)

Compare these files:
- `examples/javascript/quick-start.md`
- `examples/javascript/media-upload.md`
- `examples/javascript/scheduling-workflows.md`
- `examples/typescript/quick-start.md`
- `examples/python/quick-start.md`
- `examples/python/bulk-csv-import.md`

---

## Agent 10: Examples Part 2 (Other Languages & Auth)

Compare these files:
- `examples/curl/all-endpoints.md`
- `examples/go/quick-start.md`
- `examples/php/quick-start.md`
- `examples/ruby/quick-start.md`
- `examples/no-code/zapier-integration.md`
- `examples/no-code/n8n-integration.md`
- `examples/no-code/make-integration.md`
- `examples/frameworks/laravel.md`
- `examples/frameworks/django.md`
- `examples/frameworks/nextjs.md`
- `authentication.md`
- `getting-started.md`

---

## Output Format

Each agent should report findings as:

```
## [Section Name] Inconsistencies

### [filename.md]
| Issue | Remote (docs.publora.com) | Local (this repo) | API Code |
|-------|---------------------------|-------------------|----------|
| Description | What remote says | What local says | Actual behavior |

### Files Missing
- List any files that exist in one location but not the other
```

After all agents complete, consolidate findings into a single report.
