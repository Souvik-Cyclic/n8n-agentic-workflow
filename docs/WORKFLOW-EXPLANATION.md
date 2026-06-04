# Workflow Explanation — Vanguard

One n8n workflow, driven by the **CI-completed** event of a pull request. A deterministic router
splits on the CI result into two AI-assisted branches.

## Architecture
```
        GitHub Webhook  (workflow_run = CI finished)
                    │
        Only PR Check (Filter)        ← deterministic: keep name=="PR Check" & completed
                    │
        Route by Result (Switch)      ← deterministic: conclusion success / failure
        ┌───────────┴───────────────┐
   success                       failure
        ▼                            ▼
  Get PR Diff                  Get Run Jobs → Pick Failed Job
  Build Review Prompt          → Get Job Log → Extract Log URL → Fetch Log
  PR Reviewer (Gemini)  [AI]   Build Log Prompt
  Parse Review                 Failure Analyzer (Gemini)  [AI]
  Post Review Comment          Parse Diagnosis
  Send Approval Email          Post Build Comment
  Wait for Human (form) [HITL]
  Approved? (Switch)
   ├ approve → Add Label → Approve Comment
   └ reject  → Reject Comment (with the note)
```

## 1. Major workflow steps
1. **Trigger** — a GitHub webhook fires when a CI run (`workflow_run`) finishes.
2. **Filter** — keep only completed runs of the "PR Check" workflow; drop everything else.
3. **Route** — switch on the CI result: success or failure.
4. **On success** — fetch the PR diff, have an AI review it, post the review, email a maintainer
   an approval form, and wait for their decision.
5. **On failure** — find the failed job, download its log, have an AI diagnose it, and post the
   diagnosis on the PR.

## 2. AI nodes
Only two nodes use AI, both powered by **Google Gemini 2.5 Flash** and used where reasoning over
unstructured text is needed. Each is prompted to return **strict JSON** so the rest of the
workflow can rely on the output.

- **PR Reviewer** (Gemini 2.5 Flash) — reads the diff → `{summary, highlights, concerns, secrets}`.
- **Failure Analyzer** (Gemini 2.5 Flash) — reads the failed log → `{error_type, root_cause, file, suggested_fix}`.

## 3. Deterministic nodes
Everything that isn't reasoning is plain, predictable logic — no model involved:

- **Only PR Check** (Filter) and **Route by Result** / **Approved?** (Switch) — control flow.
- **Build Review Prompt**, **Build Log Prompt** — assemble the prompts.
- **Parse Review**, **Parse Diagnosis** — parse the AI JSON and build the comment / email.
- **Pick Failed Job**, **Extract Log URL** — pick the failed job and read the signed log URL.
- All **GitHub API calls** (diff, jobs, logs, comments, label) and the **email** call.

## 4. Branches
- **CI result branch** (`Route by Result`): success → review path · failure → diagnosis path.
- **Build-failed guard** is implicit in the filter/route — only failed runs reach the analyzer.
- **Human-decision branch** (`Approved?`): approve → add `approved` label + comment ·
  request changes → post the maintainer's note.

## 5. Final output
- A GitHub **PR comment** — a review, a failure diagnosis, or the recorded human decision.
- An **email** with a one-click Approve / Request-changes **form** (the human-in-the-loop gate).
- A GitHub **label** (`approved`) once a human approves.

---

## Node-by-node (reference)
| Node | Type | Role |
|------|------|------|
| Webhook | trigger | receives GitHub `workflow_run` events |
| Only PR Check | Filter | keep only completed *PR Check* runs · **deterministic** |
| Route by Result | Switch | success vs failure · **deterministic** |
| Get PR Diff | HTTP (GitHub) | fetch the PR diff · tool |
| Build Review Prompt | Code | assemble the review prompt · **deterministic** |
| **PR Reviewer** | HTTP (Gemini) | **AI** — review → JSON |
| Parse Review | Code | parse JSON → comment + email · **deterministic** |
| Post Review Comment | HTTP (GitHub) | post the review on the PR · tool |
| Send Approval Email | HTTP (email) | email the maintainer a review form · HITL |
| Wait for Human | Wait (form) | pause for Approve / Request changes · **HITL** |
| Approved? | Switch | branch on the human's decision · **deterministic** |
| Add Label | HTTP (GitHub) | add the `approved` label · tool |
| Approve Comment | HTTP (GitHub) | post the approval (+ note) · tool |
| Reject Comment | HTTP (GitHub) | post requested changes + note · tool |
| Get Run Jobs | HTTP (GitHub) | list jobs to find the failed one · tool |
| Pick Failed Job | Code | select the failed job · **deterministic** |
| Get Job Log | HTTP (GitHub) | request the log (returns a redirect) · tool |
| Extract Log URL | Code | read the signed log URL from the header · **deterministic** |
| Fetch Log | HTTP | download the raw log · tool |
| Build Log Prompt | Code | assemble the diagnosis prompt from the log tail · **deterministic** |
| **Failure Analyzer** | HTTP (Gemini) | **AI** — diagnose → JSON |
| Parse Diagnosis | Code | parse JSON → comment · **deterministic** |
| Post Build Comment | HTTP (GitHub) | post the diagnosis on the PR · tool |

## Agentic practices demonstrated
Agent roles (Reviewer, Failure Analyzer) · structured JSON output · tool use (GitHub API,
Google Gemini 2.5 Flash, email) · routing on event/result/decision · deterministic control &
parsing · human-in-the-loop approval · fallback handling (empty arrays, the log redirect,
Gemini retry).

## Setup
See **[SETUP.md](SETUP.md)** to run this workflow yourself.
