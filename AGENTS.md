# AGENTS.md — automatic-weight-recording

n8n workflow + Python helper that logs weight readings from a Wyze Scale to a health-tracking repo. Reads via `wyze_sdk` (HMAC-signed API), writes structured rows to a git-committed markdown log. Runs on OVH VPS via SSH-called Python script.

## What this is

n8n workflow + Python helper that logs weight readings from a Wyze Scale to a health-tracking repo. Reads via `wyze_sdk` (HMAC-signed API), writes structured rows to a git-committed markdown log. Runs on OVH VPS via SSH-called Python script.

## Where work lives (RULE — non-negotiable)

**Every task on this repo is a ticket on the [Personal Projects board](https://github.com/users/Godimas101/projects/2).** YOU (the agent) create the ticket BEFORE touching anything. No exceptions for "small" work.

Concrete rules — same as everywhere:

- **Starting work?** Open a ticket, add to the board, set Status = **In Progress**, then start.
- **Have an idea for later?** Ticket in **Backlog**. Not in memory, not in a README, not in NOTES.md.
- **Need Chris to check something before closing?** Move to **In QA** and comment what he needs to look at. Do NOT set to Done — that's Chris's call after review.
- **Finished + verified yourself?** Close the ticket with a closing summary (what you did / problems + solutions / anything NOT done).
- **Same-session micro-work?** Open + close in the same session — but the ticket exists.
- **Older than 30 days in Done?** The weekly cron moves it to Archived. The closed ticket persists.

Ticket body shape: see memory `[[feedback-ticket-body-shape]]` — What/Why → Acceptance → Related → Notes. Priority defaults to P2, Kind defaults to Feature.

## How to verify (before flagging In QA or closing)

- Test with a real Wyze reading (step on the scale) — the log entry should include all fields (weight, body_fat, bmi, bmr, muscle, water, bone, vfr, protein, metabolic_age).
- If wyze_sdk returns `body_fat: null`, log it as null — don't drop the row (that's the scale not capturing impedance, common with wet feet).
- Verify the n8n workflow can still call the SSH endpoint after any script change (the auth token refresh flow is fragile).

## MUST NOT

- Multiply weight by 2.20462 — `wyze_sdk` already returns lbs.
- Store Wyze credentials in n8n's credential store — tokens need to be updated on every run (see memory `[[project-tcs-fb-ig-token]]` for a similar pattern).

## Related

- Feeds: `personal-projects/health-tracking/` (weight log lives there)
- Similar Chris utility: [`table-to-chart`](https://github.com/Godimas101/table-to-chart) (turns the weight-log markdown into a chart)

---

*Part of Chris's `Godimas101` personal repos. Companion guide: `personal-docs/git-infrastructure.md` (private companion repo) covers the full infrastructure.*