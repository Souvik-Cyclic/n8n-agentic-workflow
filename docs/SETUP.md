# Setup — Vanguard

Step-by-step guide to run the workflow yourself. It runs on **self-hosted n8n** (n8n Cloud does
not expose custom environment variables, which this workflow uses).

---

## What you need before you start
- **Docker** (the repo ships a `docker-compose.yml` that runs n8n + a tunnel).
- A **GitHub repository** you control, with a CI workflow named **"PR Check"** that runs on pull
  requests (set up in Step 5).
- A **GitHub personal access token (PAT)** — fine-grained, scoped to that repo, with:
  **Pull requests: Read & write**, **Contents: Read**, **Actions: Read**, **Issues: Read & write**.
- An **AI endpoint** that speaks the OpenAI chat-completions format and serves **Gemini 2.5 Flash**,
  plus its **API key**.
- An **email service** with an HTTP endpoint that accepts a JSON body `{ to, subject, body, cc }`.

---

## Where config vs secrets live (read this first)
Two kinds of settings, in two different places — don't mix them:

| | Lives in | Examples |
|---|----------|----------|
| **Non-secret config** | `.env` (used by `docker-compose.yml`) | `GEMINI_URL`, `EMAIL_SERVER_URL`, `REVIEWER_EMAIL`, the public URL |
| **Secrets** | the **n8n Credentials UI** (encrypted in n8n's data volume) | the **GitHub PAT**, the **Gemini key** |

The PAT and Gemini key are **never** put in `.env` or the workflow JSON — n8n credentials are
entered once in the UI and stored encrypted.

---

## Step 1 — Configure `.env`
```bash
cp .env.example .env
```
Open `.env` and fill in the non-secret values:
```env
GEMINI_URL=https://<your-openai-compatible-endpoint>/v1/chat/completions
EMAIL_SERVER_URL=https://<your-email-service>/send
REVIEWER_EMAIL=you@example.com
# N8N_HOST / WEBHOOK_URL are set in Step 2 once you have a public URL
```

## Step 2 — Start n8n
```bash
docker compose up -d
```
This starts **n8n** plus a **cloudflared** tunnel that exposes it to the internet (so GitHub's
webhook and the email's approval-form links can reach it). Get the tunnel URL and put it in `.env`:
```bash
docker compose logs cloudflared | grep -o 'https://.*\.trycloudflare\.com'
```
Set in `.env`:
```env
N8N_HOST=<the-host-without-https>      # e.g. abcd-efgh.trycloudflare.com
WEBHOOK_URL=https://<the-host>/        # same host, with https:// and a trailing /
```
Apply it: `docker compose up -d n8n`. Then open **http://localhost:5678** and create the owner account.

> Quick-tunnel URLs are **ephemeral** — if the URL changes, update `.env` (and re-apply) and the
> GitHub webhook (Step 6). For a stable URL, use a named cloudflared tunnel or your own domain.

## Step 3 — Import the workflow
In n8n: **Workflows → ⋯ / Import from File →** select `workflow/vanguard.workflow.json`.

## Step 4 — Add the two credentials
**Credentials → New → Header Auth.** Create both:

| Name it | Used by nodes | Header name | Header value |
|---------|---------------|-------------|--------------|
| `GitHub` | Get PR Diff, Post Review Comment, Add Label, Approve/Reject Comment, Get Run Jobs, Get Job Log, Post Build Comment | `Authorization` | `Bearer <your GitHub PAT>` |
| `Gemini API` | PR Reviewer, Failure Analyzer | `Authorization` | `<the key your GEMINI_URL endpoint expects>` |

Then open `PR Reviewer`, `Failure Analyzer`, and each **GitHub** HTTP node and select the matching
credential from the dropdown. **Save.**

## Step 5 — Add the "PR Check" CI workflow to your repo
Vanguard only reacts to a CI workflow **named exactly "PR Check"**. Add
`.github/workflows/pr-check.yml` to your repo:
```yaml
name: PR Check
on:
  pull_request:
    branches: ["main"]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ... your build / test steps (e.g. npm test, go test, pytest) ...
```

## Step 6 — Point GitHub at the webhook
1. **Activate / Publish** the workflow in n8n, then open the **Webhook** node and copy its
   **Production URL** (e.g. `https://<your-tunnel>/webhook/github`).
2. In your repo: **Settings → Webhooks → Add webhook**
   - **Payload URL:** the production webhook URL
   - **Content type:** `application/json`
   - **Which events:** *Let me select individual events* → check **Workflow runs** only
   - **Active:** on → **Add webhook**

## Step 7 — Test it
- Open a PR whose CI **passes** → a review comment appears on the PR, and an **approval email**
  arrives. Click **Review & decide**, choose **Approve**, add a note, submit → the PR gets the
  `approved` label + an approval comment.
- Open a PR whose CI **fails** → a **"Build Failed"** comment appears with the error type, root
  cause, likely file, and a suggested fix.

---

## Troubleshooting
- **Webhook deliveries show 5xx/000** → the tunnel URL changed. Re-grab it (Step 2), update `.env`,
  `docker compose up -d n8n`, and update the GitHub webhook URL.
- **AI node fails with "access to env vars denied"** → ensure `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`
  is set (it's in `docker-compose.yml`).
- **Nothing triggers** → confirm the repo's CI workflow is named exactly **"PR Check"** and the
  webhook is subscribed to **Workflow runs**.
- **GitHub API node 401/403** → the PAT is missing a scope (see "What you need").

## n8n Cloud note
n8n Cloud doesn't allow custom env vars. There, set `GEMINI_URL` / `EMAIL_SERVER_URL` /
`REVIEWER_EMAIL` as **Variables** and change the expressions from `$env.X` to `$vars.X`, or inline
the values. Everything else is identical.
