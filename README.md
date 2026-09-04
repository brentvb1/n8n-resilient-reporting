# Resilient Web Scraper — n8n demo

A small n8n workflow that scrapes a listing page, and — more importantly —
knows the difference between *"there was nothing to find"* and *"I broke."*
Most scrapers can't tell those apart, and that gap is where silent failures
live.

It runs against [books.toscrape.com](https://books.toscrape.com), a sandbox
built specifically for scraper practice, so it imports and executes with no
credentials and nothing to configure. Import `resilient-scraper-workflow.json`
into any n8n instance and run it.

## What it does

Trigger → fetch the page (with retries) → parse the listings → **validate the
result** → branch: output the data if the scrape worked, or raise an alert if
it didn't. The parsing is deliberately ordinary. The validation branch is the
point.

## The lesson this is built around

I built a scraper like this for real, ran it, and it immediately did the most
dangerous thing a scraper can do: it reported success and returned nothing.
No error. No crash. A green checkmark and an empty result.

The cause was mundane — my parser assumed the page's titles sat on the same
line as their links, and the real markup put them on separate, indented lines,
so the pattern matched zero items. The *failure* was mundane too. What wasn't
mundane was the trap it would have set: had the workflow simply emailed
"0 results" and moved on, I'd have assumed the source had gone quiet and never
known the scraper was broken. Days could pass. The report would look fine.
It would just be silently, confidently wrong.

The fix wasn't a better regex — it was refusing to let "found nothing" and
"broke" share a code path. This demo bakes that in:

1. **The fetch retries before giving up**, so a transient blip doesn't read as
   a failure.

2. **A hard fetch failure routes to an alert**, not into a swallowed error.

3. **The parser reports what it found without judging it** — its job is to
   extract, not to decide whether the extraction is trustworthy.

4. **A dedicated validation step makes that judgment**, and it treats an empty
   result as a probable breakage — because on a page that normally has dozens
   of items, "zero" almost always means the markup moved, not that the world
   went quiet. Breakage takes the alert path. Success takes the data path.
   They never converge.

That is the whole idea: **a scraper that fails loudly is worth ten that fail
silently**, because you can fix the one that tells you it's broken.

## Running it

Import the JSON, open the workflow, and click execute. The `Output parsed data`
node shows the books it found; the `Raise alert` node is what fires if parsing
ever comes back empty. To watch the alert path work, point the `Fetch page`
node at a URL with no matching markup and run it again — you'll get the alert
instead of silent nothing.

## What a production version adds

The two end nodes (`Output parsed data`, `Raise alert`) stand in for delivery
steps kept out of a public demo: a Slack/email digest on success, and a real
alert to a person on failure. In production I'd also add run logging, a record
of the last-good result count to catch a *partial* break (e.g. 40 items
dropping to 3), and — for any real site rather than a practice sandbox — a
check of that site's terms and a polite request cadence.

## The honest caveat

HTML scraping is brittle by nature. Any scraper, including this one, will break
when the target site changes its markup. That is not an argument against
scraping — it's an argument for building scrapers that *announce* their own
breakage instead of hiding it. This one announces.
