# emailer-dashboard

A status dashboard for [tuongphantrue](https://github.com/tuongphantrue)'s fleet of GitHub Actions emailer bots — live at:

**https://tuongphantrue.github.io/emailer-dashboard/**

Static site, no backend. Everything on the page is either a browser-side call to the GitHub API or a plain fetch of a file already sitting in one of the bot repos.

## What it shows

- **Run status** for each bot — last result, run number, trigger type, a 10-run sparkline history
- **Last email sent** — subject + preview, expandable per bot
- **Fleet overview** — bots tracked / healthy / failing, at a glance
- **Search** (`Ctrl+K`) and **category filters** (Price / Finance / Social) in the sidebar
- **Run now** — manually trigger any bot's workflow, if a token is configured
- Auto-refreshes every 5 minutes

## How it works

Two files, no build step, no framework:

| File | Purpose |
|---|---|
| `index.html` | The whole app — markup, styling, and logic in one file |
| `config.json` | Which repos to track, fetched fresh on every page load |

`index.html` never needs editing to add, remove, or relabel a bot — that all lives in `config.json`.

## Deploying

1. Push `index.html` and `config.json` to the root of this repo
2. **Settings → Pages → Source → Deploy from a branch → `main`**
3. Give it a minute, then visit the Pages URL above

## `config.json` reference

```json
{
  "owner": "tuongphantrue",
  "repos": [
    { "name": "gold-price-emailer", "workflow": "run.yml", "category": "finance" }
  ]
}
```

| Field | Required | Notes |
|---|---|---|
| `owner` | yes | GitHub username or org the repos live under |
| `repos[].name` | yes | Exact repo name |
| `repos[].workflow` | yes | Workflow filename under `.github/workflows/` — used for both status lookups and "Run now" |
| `repos[].category` | no | `price`, `finance`, or `social` — drives the sidebar filter and the colored tag. Omit it and the bot just won't get a tag or match any category filter (it'll still show under "All bots") |

**Adding a 4th category:** the sidebar/tag colors for `price`/`finance`/`social` are hardcoded in `index.html` (search for `.tag.price` / `.avatar.price` / `.nav-item.price` and their siblings). Add a matching set of rules plus a new `<a class="nav-item">` in the sidebar for a new category to get the same treatment.

## Enabling "Run now"

Viewing status and emails needs no auth — the repos are public. Triggering a run does:

1. GitHub → your profile picture → **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**
2. Set an expiration (90 days is reasonable), **Repository access → Only select repositories** → pick the bot repos
3. **Permissions → Repository permissions → Actions → Read and write**
4. Generate, copy the token, paste it into the dashboard's token box → **save locally**

Stored in that browser's `localStorage` only — never in a file, never sent anywhere but `api.github.com`.

If "Run now" 401s/403s despite this, fall back to a **classic** token instead: **Tokens (classic) → Generate new token** → check the **`repo`** scope. Some fine-grained tokens get rejected on this specific endpoint even with the right permission set.

## Enabling email previews

The dashboard reads a `latest.json` file from each bot repo's `main` branch. That file doesn't exist until the bot itself is updated to write it. **This step is per-repo and still outstanding as of writing** — every row currently shows "no latest.json yet" until it's done.

For each of the 9 emailer repos:

**1. In the script**, right after it sends the email:
```python
import json, datetime
with open("latest.json", "w") as f:
    json.dump({
        "subject": subject,     # swap in the actual subject variable
        "preview": body[:300],  # swap in the actual body variable
        "sent_at": datetime.datetime.utcnow().isoformat() + "Z"
    }, f)
```

**2. In the workflow**, one new step after the run step, plus a permissions block up top:
```yaml
permissions:
  contents: write

steps:
  # ...existing steps...
  - run: python main.py
    env: # ...unchanged...
  - name: Save latest result
    run: |
      git config user.name "github-actions[bot]"
      git config user.email "github-actions[bot]@users.noreply.github.com"
      git add latest.json
      git diff --staged --quiet || git commit -m "Update latest result"
      git push
```

No new accounts or services either way — just GitHub.

## Privacy / security notes

- No server exists. Every request is either the visitor's browser calling `api.github.com` directly, or a plain fetch of a public file on `raw.githubusercontent.com`
- The only credential involved is the optional GitHub token above, and it never leaves `localStorage` in whichever browser it was entered into
- `config.json` and `latest.json` are both plain public files by design — don't put anything in either that isn't fine to be public
