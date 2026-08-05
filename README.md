# Task Dashboard

A personal task dashboard. You write tasks into it; an hourly cloud agent picks each one up,
matches it to a team member (a Claude Code subagent), runs it, and writes the result back.

**Live page:** https://marukoprivate.github.io/dashboard/

There is no server, no database and no build step. The page is plain HTML/CSS/JavaScript, and
`tasks.json` in this repo is the single source of truth. When you add a task in the browser, the
page commits the updated `tasks.json` straight to GitHub using your personal access token.

---

## First-time setup

Three things, in this order: push the repo, turn on Pages, create a token.

### 1. Push this repo

The scaffold is committed locally but not pushed. From the project folder:

```
git push -u origin main
```

Until this is done the dashboard will load but report that `tasks.json` does not exist yet.

### 2. Enable GitHub Pages

1. Go to the repo on github.com: **marukoprivate/dashboard**
2. **Settings** → **Pages** (left sidebar)
3. Under **Source**, choose **Deploy from a branch**
4. Branch: **`main`**, folder: **`/ (root)`**
5. Click **Save**

Give it a minute or two, then open https://marukoprivate.github.io/dashboard/

### 3. Create the fine-grained personal access token

The dashboard needs a token to write to `tasks.json`. Without one it still loads and displays
tasks — it just cannot save.

1. github.com → click your avatar → **Settings**
2. **Developer settings** (bottom of the left sidebar)
3. **Personal access tokens** → **Fine-grained tokens**
4. **Generate new token**
5. Give it a name and an expiry date
6. **Repository access**: choose **Only select repositories** → select **`marukoprivate/dashboard`**
7. **Repository permissions**, set these two:
   - **Contents: Read and write** — lets the dashboard save `tasks.json`
   - **Issues: Read and write** — used by the cloud agent for reminders, idea write-ups and
     "new team member needed" notices
8. **Generate token** and copy it (GitHub shows it only once)
9. Open the dashboard, click **Settings**, paste the token into the box, click **Save token**

The dashboard verifies the token immediately and tells you whether it worked.

---

## Security

- **This site is public.** Anyone can open the page and read `tasks.json`, because the repo is
  public. Do not put passwords, keys or anything private into a task.
- **The token lives only in your browser**, in `localStorage`, on the device where you pasted it.
  It is never written into `tasks.json`, never committed, never put in the URL and never appears
  in the page source.
- **Never paste the token on a shared or public computer.** Anyone using that browser profile
  afterwards can use your token.
- Use **Clear token** in Settings to remove it from a device.
- If a token might have leaked, **revoke it**: github.com → Settings → Developer settings →
  Personal access tokens → Fine-grained tokens → the token → **Revoke**. Then generate a new one.
- The token is only as powerful as the permissions you gave it. Scoped to this one repo with
  Contents and Issues, the worst case is damage to this repo — which is fully recoverable from
  git history.

Reading works without a token, but unauthenticated requests are limited to **60 per hour per IP
address**. If you hit that, the page says so and tells you when the limit resets. With a token
the limit is 5,000 per hour.

---

## The five task types

| Type | What it means |
|---|---|
| `do_now` | Action it on the next sweep. Top priority. |
| `do_later` | Wait until `due_at` has passed, then action it. |
| `idea` | A topic to debate or brainstorm. Produces a written write-up posted as a GitHub Issue — not executed work. |
| `monitor` | Re-check repeatedly, forever, every `check_interval_hours` hours. |
| `reminder` | No agent work at all. Just alert you (as a GitHub Issue) when `due_at` arrives. |

Notes:

- `do_later` and `reminder` are the only types that use `due_at`. Without a due date the agent
  will not pick them up on its own.
- `monitor` is the only type that uses `check_interval_hours` and `last_checked_at`. The default
  interval is 1 hour.
- `do_now` and `idea` need neither.

---

## tasks.json schema

`tasks.json` is the contract between this page and the cloud agent. **Do not rename or
restructure these fields** — both sides depend on them.

Top level:

```json
{
  "version": 1,
  "updated_at": "2026-08-05T00:00:00Z",
  "tasks": []
}
```

| Field | Type | Meaning |
|---|---|---|
| `version` | number | Schema version. Currently `1`. |
| `updated_at` | string | ISO-8601 UTC. Refreshed on every write, by the page and by the agent. |
| `tasks` | array | The tasks. Empty array when there are none. |

Each entry in `tasks` has exactly these fields:

```json
{
  "id": "t-20260805-001",
  "title": "short one-line summary",
  "details": "longer free text, may be empty string",
  "type": "do_now | do_later | idea | monitor | reminder",
  "status": "pending | in_progress | done | blocked | needs_new_agent | cancelled",
  "created_at": "ISO-8601 UTC",
  "created_by": "web | cloud | chat",
  "due_at": "ISO-8601 UTC or null",
  "check_interval_hours": null,
  "last_checked_at": null,
  "assigned_agent": "agent name or null",
  "agent_rationale": "why that agent, or null",
  "repo_url": "https://github.com/... or null",
  "result": "summary of outcome, or null",
  "issue_number": null,
  "history": [{ "at": "ISO-8601 UTC", "event": "created via web" }]
}
```

| Field | Type | Meaning |
|---|---|---|
| `id` | string | `t-<YYYYMMDD>-<3-digit sequence>`, e.g. `t-20260805-001`. Unique within the file. The date part is the UTC date it was created. |
| `title` | string | Short one-line summary. Shown as the card heading. |
| `details` | string | Longer free text for the agent. May be an empty string. |
| `type` | string | One of the five types above. |
| `status` | string | One of `pending`, `in_progress`, `done`, `blocked`, `needs_new_agent`, `cancelled`. |
| `created_at` | string | ISO-8601 UTC timestamp of creation. |
| `created_by` | string | `web` (this dashboard), `cloud` (the hourly agent) or `chat`. |
| `due_at` | string \| null | ISO-8601 UTC. For `do_later` it is when to start; for `reminder` it is when to alert. `null` for other types. |
| `check_interval_hours` | number \| null | `monitor` only — how many hours between checks. Defaults to 1. `null` for other types. |
| `last_checked_at` | string \| null | `monitor` only — ISO-8601 UTC of the last check. `null` until first checked. |
| `assigned_agent` | string \| null | Which team member the agent matched to this task. `null` until matched. |
| `agent_rationale` | string \| null | Why that team member was chosen. `null` until matched. |
| `repo_url` | string \| null | An extra repo the agent may work in for this task. `null` if none. |
| `result` | string \| null | Summary of the outcome, written by the agent. `null` until there is one. |
| `issue_number` | number \| null | GitHub Issue number, when the agent opened one. The dashboard links to it. |
| `history` | array | Append-only audit log. Each entry is `{"at": ISO-8601 UTC, "event": "what happened"}`. |

### Status meanings

| Status | Meaning |
|---|---|
| `pending` | Not started. Waiting for the next sweep, or for `due_at` to pass. |
| `in_progress` | The agent is working on it right now. |
| `done` | Finished. Look at `result`. |
| `blocked` | The agent could not proceed — `result` says why. Needs you. |
| `needs_new_agent` | No existing team member fitted this task. The agent opened an issue proposing a new role. **This one needs you to act:** add a new agent definition under `.claude/agents/`, then set the task back to `pending`. |
| `cancelled` | You called it off. No further work. |

Tasks created from the dashboard always start as `status: "pending"`, `created_by: "web"`,
`assigned_agent: null`, with one `history` entry.

---

## Using the dashboard

- **Add a task** — fill in the form at the top. The due-date field appears only for `do_later`
  and `reminder`; the check-interval field only for `monitor`.
- **Mark done / Cancel** — buttons on each task card. Both write straight back to `tasks.json`.
- **Search** filters by title, details, result or id. **Show done & cancelled** brings closed
  tasks back into view.
- **Overdue** `reminder` and `do_later` tasks are outlined in red. Tasks flagged
  `needs_new_agent` get a banner at the top of the page.
- The page works on a phone and follows your system light/dark setting.
- Task text can be written in **Thai or English**. The page encodes and decodes UTF-8 correctly
  in both directions, and self-tests that on every load — see **Diagnostics** in Settings.

### If two writes collide

The hourly cloud agent commits to the same `tasks.json`. If it commits while you have the page
open, your save would normally fail with a stale-`sha` conflict. Instead the page automatically
re-reads the latest `tasks.json`, re-applies your change on top of it, and retries (up to three
attempts total). Your input is never silently dropped; if all attempts fail, the entry stays in
the form and an error explains why.

---

## Repo layout

```
index.html          the dashboard (self-contained: HTML + CSS + JS, no dependencies)
tasks.json          source of truth - the task list
CLAUDE.md           instructions for the hourly cloud agent
README.md           this file
.gitignore          OS/editor cruft
.claude/agents/     the 12 team-member definitions the cloud agent chooses between
```

## Editing tasks.json by hand

You can, but be careful: it must stay valid JSON. If it is not, the dashboard refuses to save
rather than overwrite it, and tells you to fix it on GitHub first. Bump nothing else — just keep
the schema above and refresh `updated_at`.
