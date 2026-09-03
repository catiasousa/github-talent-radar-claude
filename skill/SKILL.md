---
name: github-talent-radar
description: Source and score GitHub candidates for a role and maintain a live, editable candidate tracker — the Claude-native replacement for an n8n + Airtable GitHub sourcing workflow. Use when the user wants to source candidates for a role, run a talent/candidate search on GitHub, refresh an existing role's tracker, or mentions "talent radar", GitHub sourcing, or replacing n8n/Airtable recruiting automation.
---

# GitHub Talent Radar

Replaces an n8n + Airtable pipeline that nightly ran 5 parallel GitHub searches, scored candidates with Claude, and wrote qualified profiles to Airtable. This skill does the same job natively: search GitHub, score candidates against a real job posting, and keep the results in a live tracker the user can filter, sort, and check off as they work outreach — without n8n or Airtable.

## 0. Gather inputs before doing anything

Ask (conversationally, or via AskUserQuestion if the session supports it) for whatever isn't already given:

- **The role**: a job posting URL (fetch it with WebFetch and extract title, seniority, required skills, nice-to-haves, domain context) or a pasted job description.
- **Search angles**: at least one of the 5, each with a concrete value:
  1. Location — a city/country string
  2. Programming language — a language name
  3. Repo contributor — an `owner/repo`
  4. Topic/library — a GitHub topic tag
  5. Company org — a GitHub org handle
- **New run vs. refresh**: if the user says this is a follow-up for a role you've already built a tracker for, ask for that tracker's Artifact URL (or find it via `Artifact` `list` action, matching title "<Role> Talent Radar") instead of creating a duplicate. Each role gets its OWN tracker (one Artifact per role) — never merge roles into one shared tracker unless the user explicitly asks for that.

Don't silently assume defaults for missing search criteria — ask. Do proceed without asking if the user has clearly already supplied everything needed.

## 1. Pick your GitHub access path

Check whether this session has a linked computer (look for `mcp__remote-devices__*` tools, e.g. via ToolSearch for "remote-devices device_bash"):

- **Linked device available (preferred)**: run GitHub API calls via `device_bash` using the user's real GitHub token (ask for one once if not already available in that shell; never write it into any file inside the connected folder — keep it in the device shell's own scratch space or pass it inline per-call). This gives 5,000+ requests/hour and avoids flaky rate limits. Note: `device_bash` requires at least one connected folder to run at all, even for pure network calls — if none is connected yet, call `device_request_folder_access` for a small folder (e.g. Documents) and explain why.
- **No linked device**: this cloud sandbox's direct `curl`/Bash access to `api.github.com` is blocked by an egress proxy ("sessions are bound to their configured repositories" — it only allows repo-scoped calls for repos explicitly attached to the coding session, not general API browsing). The working fallback is the `WebFetch` tool pointed straight at `api.github.com/...` JSON endpoints — it fetches successfully but unauthenticated and on a shared IP, so expect occasional 403s under load. Retry failed calls once or twice before giving up on that one profile, and don't parallelize more than ~8 WebFetch calls at once. Tell the user up front this run may be slower/less complete than with a linked device, and suggest linking one for reliability.

If the user supplies a GitHub personal access token but no device is linked, saving it to a local sandbox file for use via `curl` in Bash will NOT work (the proxy blocks it regardless of auth) — don't bother; use WebFetch instead and tell the user the token can't be used here without a linked device.

## 2. Run the search angles

For each angle the user specified, hit the corresponding endpoint via WebFetch or `device_bash` curl:

- **Location**: `GET /search/users?q=location:"<location>"&sort=followers&order=desc&per_page=50`
- **Programming language**: GitHub's user-search has no reliable `language:` qualifier — instead `GET /search/repositories?q=language:<lang>&sort=stars&order=desc&per_page=20`, then pull contributors from the top repos, OR search users by a language-associated keyword in bio if the user gives one. Note this workaround to the user.
- **Repo contributor**: `GET /repos/{owner}/{repo}/contributors?per_page=100&anon=false`
- **Topic/library**: `GET /search/repositories?q=topic:<topic>&sort=stars&order=desc&per_page=20`, then pull contributors from the top 3-5 matching repos (bound the cost — don't enrich hundreds of people).
- **Company org**: `GET /orgs/{org}/public_members?per_page=100`

Merge all results, dedupe by GitHub login (case-insensitive), and record which angle(s) found each person as their `source`.

## 3. Filter before enriching

Drop bot accounts (`[bot]` suffix). If the merged list is large (>40 people), prioritize by whatever signal each angle provides (follower count, contribution count) rather than enriching everyone — enrichment is the expensive step.

## 4. Enrich

For each remaining candidate: `GET /users/{login}` (name, company, location, bio, blog, followers, public_repos, hireable). When the bio is empty or thin, also check `GET /users/{login}/repos?sort=pushed&per_page=10` (or fetch their `github.com/{login}` profile page via WebFetch, which surfaces pinned repos and richer bio context than the raw API) to find real signal — notable projects created/maintained, primary languages, evidence relevant to the role.

## 5. Exclude people who are already "hired"

Before scoring, check each candidate's `company` field and any org affiliation against: (a) the hiring company itself — they may already work there, (b) the maintaining team of the source repo/project, if a repo-contributor or topic search was used — a top contributor is often a founder or core employee of the project, not an external candidate. Don't silently drop these — mark them `status: "excluded"` with an `exclude_reason` and keep them visible in the tracker's excluded section for transparency.

## 6. Score the rest

Score each remaining candidate yourself, directly, as a technical recruiter would — no external API call needed, you ARE the scoring engine. Use the job's actual requirements (not generic criteria) and produce fields matching the Airtable schema this replaces exactly:

- `fit_score` (0-100)
- `key_strengths` (1-2 sentences, cite specific, verifiable evidence — real projects, real contributions, real bio text, never invented)
- `key_gaps` (1-2 sentences, honest about what's unverified or missing)
- `priority_action` (a concrete next step, banded roughly: ≥70 "reach out now" energy, 50-69 "warm, verify first", <50 "deprioritize")
- `outreach_hook` (one specific, personalized opening line referencing their actual work — omit/empty string if there's nothing genuine to point to; never generate a generic hook)
- `profile_summary` (one-line description)

Also carry through: `full_name`, `github_url`, `location_city`, `current_company`, `source`, `source_role` (the role title), `date_sourced` (today's date), `followers`, `signal` (their raw contribution/signal count from the sourcing angle).

## 7. Create or update the tracker

Each role's tracker is a separate Artifact-published HTML page declaring `capabilities: {db: {}}`, with two collections:

- `meta/info` (one doc): `{role_title, job_url, search_summary, last_run_at}`
- `candidates` (collection, one doc per candidate, **doc_id = GitHub login**): all the fields from steps 5-6, plus manual tracking fields that default on creation only — `contacted: false, contact_date: null, replied: false, reply_sentiment: null, do_not_contact: false, notes: ""` — and never overwritten on a refresh (see below).

**New role** (no existing tracker): build the page (a live, sortable, filterable card grid reading from `db.collection("candidates")` via `onSnapshot`, with checkboxes for contacted/replied/do-not-contact and a notes field wired to `db.doc(id).update(...)`, plus a collapsible excluded-profiles section) — load the `artifact-design` and `artifact-capabilities` skills first. Publish it, then seed `meta/info` and `candidates` via the `Artifact` tool's `write_db` action (batch writes, up to 50 at once) — never hardcode candidate data inside the page's own JS, the page must read live from the store.

**Refreshing an existing role's tracker**: don't republish the HTML (it doesn't need to change). Re-run the search/enrich/score pipeline, then `write_db` with `db_op: "update"` (merge, not `set`/replace) for each candidate — this updates `date_sourced`, `fit_score`, etc. while preserving whatever the user has already checked off or written in `notes`. Only use `set` for genuinely new candidates not yet in the collection. Update `meta/info` with the new `last_run_at` and `search_summary`.

## 8. Report back

Tell the user: how many profiles were found/enriched/excluded/scored, the top 2-3 candidates and why, any GitHub-access caveats from step 1, and that the tracker is live and editable (checkboxes/notes save automatically, visible to anyone with the link). Don't paste the URL in text — the Artifact publish already surfaces it as a card.

## Notes

- This whole flow works as a genuine on-demand replacement for the n8n + Airtable workflow. True unattended nightly scheduling additionally requires a linked device bound to a scheduled task (`create_trigger` with `requires_local_device: true`) so GitHub auth is available at fire time — don't promise scheduling works without that piece in place, and don't build it unless the user asks.
- Never store a GitHub token in any file inside a user's connected folder, in the published Artifact's HTML/JS, or in the `db` store (it's shared, unencrypted data) — only ever in the sandbox's own private scratch space for the duration of a session, or used inline in a device-shell command.
