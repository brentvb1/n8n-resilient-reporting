# Resilient Scheduled Reporting — n8n demo

A small, self-contained n8n workflow that shows how I build automations to
**fail loudly instead of silently**. It runs against a public demo API
(`dummyjson.com`), so it imports and executes with no credentials — the point
is the reliability pattern, not the data.

Import `resilient-reporting-workflow.json` into any n8n instance to inspect it.

## What it does

Daily trigger → fetch data → validate the response → branch on validity →
build a report (or raise an alert). A generic stand-in for the kind of
scheduled "pull data, analyze it, deliver a report" workflow businesses rely
on every morning.

## The part that matters: four reliability decisions

Most workflows are built for the happy path and quietly break the first time
reality doesn't cooperate. The four things below are what separate an
automation you can trust from one you have to babysit:

1. **Transient failures are retried, not fatal.** The fetch step retries with
   a backoff before giving up, so a momentary blip or a rate-limit hiccup
   doesn't kill the whole run.

2. **Hard failures route to an alert, not into the void.** The fetch step's
   error output is wired to an alert path — a failed call surfaces, it doesn't
   vanish.

3. **The response is validated before it's trusted.** The most dangerous
   failure isn't an error — it's a call that "succeeds" but returns empty or
   malformed data that then flows downstream and silently corrupts the report.
   The workflow checks the shape of the data explicitly and flags it.

4. **Bad data and failures are loud.** Both the API-error output and the
   failed-validation branch converge on a single alert step, where a real
   build fires a Slack/email notification — so the team finds out immediately
   instead of discovering a stale or wrong report days later.

## What a production version adds

The two placeholder nodes (`Write to Destination`, `Raise Alert`) stand in for
credentialed steps I keep out of a public demo: a Google Sheets / Excel writer
and a Slack or email alert. In real builds I also add webhook auth with a
negative test to confirm rejection, run logging/state tracking, and stagger
scheduled runs to avoid resource collisions.

---

## Other work (sanitized)

Public demos can't show client work, so here are three real builds described
without any identifying detail:

**Paginated sync that silently truncated.** A workflow syncing 15,000+ records
from a desktop accounting system into a spreadsheet kept stopping partway
through with no error. Root cause was three stacked failures: a server-side
pagination cursor that expired within seconds, an out-of-memory condition from
holding full record objects, and a destination step reporting success while
writing nothing due to a malformed header row. Rebuilt as a decoupled
fetch/accumulate loop with slimmed payloads and a direct API write that
bypassed the unreliable node.

**Multi-step LLM pipeline that stays in its lane.** An n8n workflow where a
language model generates the narrative section of a report from computed
numerical analysis — designed so the model writes *only* prose and never
recomputes or invents the underlying figures, keeping the output grounded in
the deterministic calculation upstream. Draft-not-send by default until
validated.

**Self-hosted scheduled reporting with real error handling.** Scheduled
workflows that pull business data on a cron, transform it into multi-tab Excel
reports split by category, and email them automatically — with header-token
auth on the webhooks (confirmed with a wrong-token rejection test) and
staggered scheduling to avoid concurrent-run collisions.
