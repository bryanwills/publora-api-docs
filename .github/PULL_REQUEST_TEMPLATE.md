<!-- What & why: -->


## ⚠️ Two-repo pipeline checklist — merging THIS PR does NOT update docs.publora.com

The live site is served from the GitLab mirror (`rnd-experiments/post-actor`). After merging here, someone must run the sync (Alex/Max have the runbook — ping them, or do it yourself):

- [ ] Every changed file copied byte-identically to `post-actor/sites/docs.publora.com/content/…`
- [ ] If `schema/openapi.yaml` changed: `npm run openapi:generate-json` re-run (never hand-edit `openapi.json`)
- [ ] Offline gates pass in the site dir: `npm run docs:check-all`
- [ ] GitLab MR opened **and merged**
- [ ] Site deployed after the MR merge (`npm run pages:build` + `wrangler deploy`)
- [ ] Edge cache purged for every changed URL (page + `/raw/<slug>` + `<slug>.md`; `llms.txt`/`llms-full.txt` if titles/descriptions changed) — the cache survives deploys
- [ ] Live spot-check: `curl https://docs.publora.com/raw/<slug>` shows the new content

Skipping these leaves the live site contradicting this repo (it has happened).
