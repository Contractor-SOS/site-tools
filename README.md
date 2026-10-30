# site-tools

Shared GitHub Actions workflows for every website CSOS hosts. Each site repo carries a small caller that points here, so a fix made in this repo reaches every site at once.

| Workflow | What it does | Called by |
| --- | --- | --- |
| `.github/workflows/claude-edit.yml` | Checks out the site's `preview` branch, has Claude Code make the requested change inside `site/`, commits it to `preview`, and reports the result to SEED | the site repo's `.github/workflows/claude-edit.yml` |
| `.github/workflows/archive-site.yml` | Takes full-page desktop and phone screenshots of an old website and copies listed files (logos, favicons), so nothing is lost before a move | the private `site-archive` repo |

## One-time setup for the organization

- This repo and every site repo are **public**, so they can share these workflows and read organization secrets on GitHub Free. Never commit passwords, API keys or tokens. Run logs are public too.
- Organization secret `CLAUDE_CODE_OAUTH_TOKEN` (**Settings → Secrets and variables → Actions**, repository access: all repositories). It comes from running `claude setup-token`, uses that person's Claude subscription, and lasts a year.
- Optional organization secret `SEED_WEBHOOK_SECRET`: when set, each result sent to SEED carries an `X-SEED-Signature: sha256=<HMAC of the body>` header that SEED checks.
- The Claude GitHub App is installed on the organization for all repositories.

## Setup for each site repo

- A `preview` branch, made from `main`.

## Running an edit by hand

Site repo → **Actions → Claude edit → Run workflow**. Type the change in "What to change" and press **Run workflow**. A few minutes later the change is on the `preview` branch, Cloudflare rebuilds preview.<domain>, and Claude's summary appears on the run's summary page.

## What SEED sends and gets back

SEED starts the site repo's `Claude edit` workflow with these inputs: `request_id`, `request`, `screenshots`, `attachments`, `history`, `callback_url`, `max_turns` and `model`. When `callback_url` is set, the workflow POSTs this JSON to it when the run ends:

```json
{
  "request_id": "REQ-123",
  "repository": "Contractor-SOS/cce-website",
  "status": "changed | no_changes | failed",
  "summary": "Claude's plain-English summary",
  "commit": "sha of preview after the run",
  "run_url": "https://github.com/.../actions/runs/...",
  "branch": "preview"
}
```

The commit message also carries the summary and a `SEED-Request:` line, so SEED can read results from GitHub if a callback is missed.

## Guard rails

- Claude can only read, search and edit files. It has no shell or web access, and stops at the turn limit.
- Anything changed outside `site/` is undone before the commit.
- The workflow pushes only to `preview`. `main`, the live site, changes only when SEED's Approve merges `preview` into it.
- One edit runs at a time per site; a run started meanwhile waits its turn.
- Runs must be started by a person (SEED starts them as the GitHub account its integration was connected with). The Claude action refuses bot-started runs.
