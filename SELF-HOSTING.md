# Self-Hosting the Profile Cards

The GitHub stats, trophy and activity-graph cards in [`README.md`](README.md) are rendered by
third-party services. The shared public instances are heavily rate-limited, and as of
**1 September 2026** four of the cards were failing:

| Card | Public host | Status |
|---|---|---|
| GitHub Stats | `github-readme-stats.vercel.app` | `503` |
| Top Languages | `github-readme-stats.vercel.app` | `503` |
| Trophies | `github-profile-trophy.vercel.app` | `402` — Vercel quota exhausted |
| Activity Graph | `github-readme-activity-graph.vercel.app` | `402` — Vercel quota exhausted |

Deploying your own instances fixes this permanently: your Vercel project has its own quota, and
your own GitHub token has its own 5,000 req/hr API budget that nobody else is spending.

> **Already done for you:** the stats and top-languages cards have been repointed to
> `github-stats-extended.vercel.app` (see [Background](#background)), which is currently serving
> `200`. Those two cards work right now. Sections 3 and 4 below are what remain broken.

---

## Background

`anuraghazra/github-readme-stats` — the project that powered the stats cards — now carries this
notice in its own readme:

> This repository is no longer maintained. Please use the successor project **GitHub Stats
> Extended** instead.

So this guide targets **`stats-organization/github-stats-extended`**, the actively maintained fork.
It is a drop-in replacement: identical `/api` and `/api/top-langs/` paths and identical query
parameters, so switching is a hostname swap and nothing more.

---

## Prerequisites

- A GitHub account (you have one).
- A [Vercel](https://vercel.com) account — the Hobby tier is free and sufficient. Sign in with
  GitHub so Vercel can see your forks.
- 20–30 minutes.

---

## Step 1 — Create a Personal Access Token

All three services read the GitHub API on your behalf and need a token to do it.
**One token works for all three.**

1. Go to <https://github.com/settings/tokens> → **Generate new token** → **classic**.

   > A *classic* token is the safe default: all three projects accept one, and it is the only
   > kind that can include **private** contributions.
   >
   > github-stats-extended also accepts a fine-grained token (Metadata, Contents, Issues,
   > Pull requests, Commit statuses — all read-only, "All repositories"), but that limits it
   > to **public repositories only**. Use classic if you want `count_private=true` to work.

2. Name it something you'll recognise later, e.g. `profile-cards`.
3. Set an expiry. If you pick anything other than "No expiration", **put a reminder in your
   calendar** — the cards will silently start failing on the day it expires.
4. Select scopes:
   - `read:user` — **required**. Lets the services read your profile and contribution data.
   - `repo` — **only if** you want private repositories counted (this is what makes
     `count_private=true` in the README actually do something). Skip it if you don't.
     The github-stats-extended docs list `repo` alongside `read:user` as their standard pair.

   Select nothing else. These services never need write access.

5. Click **Generate token** and copy it. GitHub shows it exactly once.

> **Handling the token:** paste it only into Vercel's environment-variable fields. Never commit it
> to this repo, never put it in a URL, and never paste it into an issue or chat. If it leaks,
> revoke it immediately at <https://github.com/settings/tokens> — revoking is instant and you can
> just generate a new one.

---

## Step 2 — Deploy the Stats + Top Languages card

This one is **optional** — the cards already work via the public instance. Self-host it if you want
guaranteed uptime, or if you want `count_private=true` to function (the public instance cannot see
your private repos no matter what parameters you pass).

> [!IMPORTANT]
> This project is a **pnpm/turbo monorepo**, not a flat serverless project. You must set the
> Root Directory to `apps/backend`, or the build fails with:
>
> ```
> No Output Directory named "dist" found after the Build completed.
> ```
>
> Vercel only reads `vercel.json` from the Root Directory, and the build command lives in
> `apps/backend/vercel.json`. Point the root at the repo top and Vercel never sees it, falls back
> to guessing a framework, and looks for a `dist/` folder this repo never produces.

1. Go to <https://github.com/stats-organization/github-stats-extended> and click **Fork**.

   **Untick "Copy the `master` branch only."** You need the `release` branch in step 7.

2. Go to <https://vercel.com/new>, find your fork, and click **Import**.

3. **Set Root Directory to `apps/backend`.** It's under the "Root Directory" heading on the import
   screen — click **Edit** and pick the `apps/backend` folder. This is the step that matters.

4. Leave the build and output settings alone. Once the root directory is right, `vercel.json`
   supplies the build command.

5. Expand **Environment Variables** and add:

   | Name | Value |
   |---|---|
   | `PAT_1` | the token from Step 1 |

   Optionally also set `TURBO_PLATFORM_ENV_DISABLED` to `true`, which silences a harmless
   build-time warning from turbo about variables missing from `turbo.json`.

   To keep the token out of build logs: add it under Project Settings → Environment Variables,
   untick **Deployment** under Environments, then tick **Sensitive**.

6. Click **Deploy**.

7. **Switch the project to the `release` branch.** `master` carries unreleased and potentially
   unstable code; `release` is the latest stable tag.

   1. Project Settings → **Environments → Production**, change the tracked branch from `master`
      to `release`.
   2. **Deployments → ⋯ → Create Deployment**, pick `release`, then **Deploy to production**.

8. Copy your deployment hostname — something like `github-stats-extended-abc123.vercel.app`.

**Test it before touching the README:**

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  "https://YOUR-HOST.vercel.app/api?username=Surajvthakur&show_icons=true&theme=tokyonight"
```

`200` means you're good. See [Troubleshooting](#troubleshooting) for anything else.

> **Avoid the one-click "Deploy to Vercel" button** in the upstream docs. It sets the root
> directory for you, but the repo it creates is not registered as a fork, so GitHub's
> **Sync fork** button won't work and you lose the easy update path in [Maintenance](#maintenance).

### Optional environment variables

| Name | Effect |
|---|---|
| `CACHE_SECONDS` | Cache lifetime for generated cards. Defaults to 86400 (24h). Lower it to refresh faster. |
| `WHITELIST` | Comma-separated usernames allowed to use your instance. Set it to `Surajvthakur` to stop strangers burning your API quota. |
| `EXCLUDE_REPO` | Comma-separated repos to leave out of stats and top-languages. |
| `FETCH_MULTI_PAGE_STARS` | `true` for accurate star totals when you have >100 repos. Slower. |
| `UPDATE_AFTER_HOURS` | Hours before a previously requested card is proactively regenerated. Default 11. |
| `TURBO_PLATFORM_ENV_DISABLED` | `true` silences the turbo build warning. Cosmetic. |

Setting `WHITELIST=Surajvthakur` is worth doing — a public URL pointed at your token is otherwise
free API budget for anyone who finds it.

## Step 3 — Deploy the Trophy card

Currently returning `402`. This is the one you actually need.

1. Fork <https://github.com/ryo-ma/github-profile-trophy>.
2. Import the fork at <https://vercel.com/new>, defaults again.

   > This project runs on **Deno**, via the `vercel-deno` runtime declared in its `vercel.json`.
   > That's handled automatically — just don't change the framework preset.

3. Add the environment variable:

   | Name | Value |
   |---|---|
   | `GITHUB_TOKEN1` | the token from Step 1 |

   **Note the trailing `1`.** The variable is `GITHUB_TOKEN1`, not `GITHUB_TOKEN`. Getting this
   wrong is the single most common failure here — the deploy succeeds and every card renders as an
   error. You may optionally add `GITHUB_TOKEN2` with a second token; the service rotates between
   them on rate-limit.

4. Deploy, then copy the hostname and test:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  "https://YOUR-TROPHY-HOST.vercel.app/?username=Surajvthakur&theme=radical"
```

Note the `/?username=` — the trophy service serves from the **root path**, unlike the other two.

---

## Step 4 — Deploy the Activity Graph card

Also currently `402`.

1. Fork <https://github.com/Ashutosh00710/github-readme-activity-graph>.
2. Import the fork at <https://vercel.com/new>, defaults.
3. Add the environment variable:

   | Name | Value |
   |---|---|
   | `TOKEN` | the token from Step 1 |

   **Just `TOKEN`** — not `GITHUB_TOKEN`, not `PAT_1`. Each of these three projects uses a
   different name; there is no consistency between them.

4. Deploy, copy the hostname, and test:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  "https://YOUR-GRAPH-HOST.vercel.app/graph?username=Surajvthakur&theme=tokyo-night&hide_border=true"
```

The path is `/graph`.

---

## Step 5 — Point the README at your instances

Every URL that needs changing is tagged with a `SELF-HOST` comment. Find them all:

```bash
grep -n "SELF-HOST" README.md
```

Then swap each hostname, leaving every query parameter exactly as it is:

| # | Replace this host | With your host |
|---|---|---|
| 1 | `github-stats-extended.vercel.app` | your Step 2 host (skip if you didn't do Step 2) |
| 2 | `github-profile-trophy.vercel.app` | your Step 3 host |
| 3 | `github-readme-activity-graph.vercel.app` | your Step 4 host |

You can do all of them with `sed` — substitute your real hostnames:

```bash
sed -i \
  -e 's|github-profile-trophy\.vercel\.app|YOUR-TROPHY-HOST.vercel.app|g' \
  -e 's|github-readme-activity-graph\.vercel\.app|YOUR-GRAPH-HOST.vercel.app|g' \
  README.md
```

Then commit and push:

```bash
git add README.md && git commit -m "chore: point profile cards at self-hosted instances" && git push
```

---

## Step 6 — Verify

Check that every image in the README resolves. This is the same sweep that found the original
breakage:

```bash
grep -oP 'src(?:set)?="\K[^"]+' README.md | grep '^https' | while read -r u; do
  printf "%s  %s\n" "$(curl -s -o /dev/null -w '%{http_code}' -L --max-time 15 "$u")" "$u"
done
```

Every line should read `200`.

Finally, open <https://github.com/Surajvthakur> and toggle GitHub's appearance setting between
Light and Dark to confirm the cards render in both themes.

> **Note on caching:** GitHub proxies all README images through its Camo cache, so a card can keep
> showing a stale or broken version for a few minutes after you've fixed it. Confirm with `curl`
> first — that bypasses Camo entirely and tells you the truth.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `No Output Directory named "dist" found` | Root Directory not set to `apps/backend` (Step 2) | Project Settings → Build & Deployment → Root Directory → `apps/backend`, then Redeploy. Vercel reads `vercel.json` only from the Root Directory. |
| Build succeeds but cards are unstable | Project is tracking `master` | Switch the Production branch to `release` (Step 2.7). |
| `402` | Vercel account quota exhausted | You're still pointing at the shared public instance, not your own. Re-check the hostname in `README.md`. |
| `403` / "Maximum retries exceeded" | Token missing, misnamed, or revoked | Check the env var name character-for-character: `PAT_1`, `GITHUB_TOKEN1`, `TOKEN`. Redeploy after fixing — Vercel does **not** apply env var changes to an existing deployment. |
| `404` on the trophy card | Wrong path | Trophy serves from `/?username=`, not `/api?username=`. |
| `500` on the activity graph | Usually the `TOKEN` var | Confirm it's named `TOKEN` exactly, then redeploy. |
| Card shows "Something went wrong" | Token expired | Generate a new one (Step 1) and update it in all three Vercel projects. |
| Private repos not counted | Token lacks `repo` scope | Regenerate the token with `repo` ticked; `count_private=true` needs it. |
| Changed an env var, nothing happened | Vercel needs a redeploy | Deployments → latest → ⋯ → **Redeploy**. |
| Stats card stale after pushing commits | `CACHE_SECONDS` (24h default) | Lower `CACHE_SECONDS`, or just wait. |

---

## Maintenance

- **Token expiry is the thing that will break this.** If you set an expiry in Step 1, put a
  calendar reminder a week before. All three projects fail the same silent way.
- **Keep the forks current** — GitHub's UI has a **Sync fork** button on each fork's main page.
  Vercel redeploys automatically when the fork updates. Worth doing a couple of times a year.
- **If a card breaks again**, run the Step 6 sweep first. The HTTP status tells you which of the
  three deployments is at fault, and the table above tells you why.
