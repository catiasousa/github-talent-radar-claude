# GitHub Talent Radar — Claude Edition

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A Claude-native fork of [Github-Talent-Radar-with-Claude-n8n-Airtable](https://github.com/catiasousa/Github-Talent-Radar-with-Claude-n8n-Airtable), same sourcing pipeline, rebuilt to run entirely inside Claude instead of n8n + Airtable.

## What this project does

This project builds a GitHub sourcing system you run through Claude. It searches GitHub using multiple discovery methods, evaluates each profile itself (no separate AI API call as Claude does the scoring in-context), ranks candidate fit for your target role and keeps high-signal candidates in a live tracker page, including a personalized outreach line for each one.

Ask Claude any time you want a fresh batch for a role. With a computer linked and a scheduled task set up, it can also run unattended on a cadence you choose, but that's opt-in, not required.

## Why GitHub

Manual GitHub sourcing is slow, repetitive, and difficult to scale consistently. This workflow turns sourcing into a repeatable system with consistent search patterns, structured AI evaluation, and clean candidate tracking.

## Workflow logic explained

This pipeline applies a consistent sourcing process to GitHub profiles, run whenever you ask.

It begins with multiple discovery paths to capture different engineering signals, then merges and de-duplicates profiles so each candidate is evaluated once.

Each profile is enriched with practical context from GitHub activity, then scored by Claude directly, structured fit score, strengths, gaps, and a priority action, with no separate scoring call or JSON round-trip needed.

Candidates are saved to a live tracker page with standardized fields and outreach hooks, so your review and contact process stays fast and repeatable — and the page itself is where you check off outreach as you go.

The result is a reliable shortlist generation loop based on real builder activity, not just keyword matching.

## Who this is for (and not for)

**This is for you if**

- You recruit software, platform, or AI engineering talent and want sourcing signals from real code activity.
- You want to identify candidates through practical indicators such as repositories, contribution patterns, and project ownership.
- You want an on-demand (or scheduled) workflow that scores profiles, prioritizes outreach, and writes structured candidate records to a live tracker.

**This is not for you if**

- You only hire roles that are not evaluated through technical output.
- You cannot use external APIs or automated profile analysis due to policy constraints.

## The search engines this workflow runs

This workflow runs up to 5 searches in parallel and merges results into one ranked candidate pipeline. You choose which ones to use for each run.

**Search 1 — By Location.** Find engineers in a city or country.
What it does: finds engineers who set a specific location in their GitHub profile, sorted by followers.
Why it matters: helps target geography-specific hiring needs.

**Search 2 — By Programming Language.** Find specialists in a specific language.
What it does: finds highly followed engineers associated with the language you need.
Why it matters: useful when language depth matters more than location.

**Search 3 — By Repository Contributor.** Find people who built the exact thing you need.
What it does: pulls contributors from a target repository.
Why it matters: very high-signal proof of hands-on experience in the domain.

**Search 4 — By Library or Topic.** Find engineers working in a specific domain.
What it does: finds topic-tagged repositories and pulls contributor profiles.
Why it matters: surfaces domain practitioners beyond one flagship repo.

**Search 5 — By Company Organisation.** Mine specific companies' engineering teams.
What it does: pulls public members from a company GitHub organisation.
Why it matters: supports competitor and peer-company sourcing.

Note: you give Claude your search criteria in plain language each time you ask it to source (see Customizing below). 

## Accounts you need to create

- **GitHub** (github.com), your source of candidates.
- **Claude** (claude.ai), with Cowork/Artifacts enabled. This is the automation engine, the scoring engine and the candidate database, all in one. It replaces n8n, the separate Anthropic API account and Airtable.

## How to get access set up

- **GitHub token (optional)**: Settings → Developer settings → Personal access tokens → generate one. Even a token with zero scopes works — it's only used to raise API rate limits for reading public data, nothing is written to GitHub.
- **Claude**: no separate API key to manage (you use your normal Claude login). When Claude needs your GitHub token, it'll ask for it in the conversation.

Never paste secrets into the skill file or any file in this repo. If Claude asks for your GitHub token, give it directly in the chat when prompted, it should never be written into a committed file.

## Quickstart — two ways to use it

**Option A — Install it as a skill (recommended)**

1. Open a conversation with Claude.
2. Paste the contents of `skill/SKILL.md`, or attach the file, and ask Claude to save it as a skill.
3. Review and save the skill when Claude shows you the confirmation card.
4. From then on, just tell Claude the role and your search criteria whenever you want a batch sourced.

**Option B — One-off, without installing anything**

Paste `skill/SKILL.md` into a conversation and ask Claude to follow it for a single run. Nothing is saved for next time, but it's a fast way to try it once.

## How the workflow operates

1. You tell Claude the role (a job posting link or description) and your search criteria.
2. Claude runs the GitHub discovery searches you specified.
3. Candidate data is normalized, deduped and filtered.
4. Claude evaluates role fit directly and writes summary notes (no separate evaluation step).
5. Qualified candidates are saved to a live tracker page Claude publishes.
6. You review candidates on that page and send outreach, checking off progress as you go.

## Tracker fields required by this workflow

Fields the workflow writes:

`full_name`, `github_url`, `location_city`, `current_company`, `source`, `source_role`, `date_sourced`, `fit_score`, `priority_action`, `key_strengths`, `key_gaps`, `outreach_hook`, `profile_summary`

Manual tracking fields (yours to fill in on the tracker page as you work outreach):

`contacted`, `contact_date`, `replied`, `reply_sentiment`, `do_not_contact`, `notes`

## Setup details

**1) Install the skill**

Give Claude `skill/SKILL.md` and ask it to save the skill (see Quickstart, Option A).

**2) Connect access**

No credentials to wire into a tool. Optionally link your computer to Claude for reliable GitHub API access, and give Claude a GitHub personal access token when it asks (never store it in a file). 

**3) Tell Claude the role you're hiring for**

Give it a job posting link or a pasted description and the role title. No config file to edit.

**4) Choose your search criteria**

Tell Claude which of the 5 search angles to use and their values (a location, a language, a repo, a topic, an org) and you can change this on every run.

**5) Run a test**

Ask Claude to source a small batch and review the tracker it publishes before relying on it further.

**6) Automate it (optional)**

Ask Claude to set up a scheduled task if you want it running on a cadence without you asking each time (this needs a linked computer so GitHub access is available when it fires). 

**7) Schedule**

If you set up scheduling, pick your own cadence when asking Claude to create it (the original template ran nightly at 02:00, match that or choose differently).

## Customizing

- Change your search criteria on any run to match new hiring goals, nothing to edit in a file.
- Ask Claude to adjust scoring strictness for different seniority or role profiles.
- The skill already excludes candidates already employed at the hiring company or at the source project's own team. Ask Claude to extend that exclusion logic further if needed.
- Ask Claude to add fields to the tracker if your process needs more than what's listed above.

## Repository structure

```
skill/
  SKILL.md
.github/
  workflows/
    gitleaks.yml
README.md
.gitleaks.toml
.gitignore
LICENSE
```

## License

MIT.
