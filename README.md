# Portfolio Intelligence — case study

A private dashboard and nightly email for my own brokerage account: positions, day moves, news on the
names I hold, and a short written summary. Built to replace the ten minutes each evening I spent
clicking between a broker app and a news feed.

This is a write-up. The app is private — it reads a live account — so the code isn't published.

## What it does

Every weekday evening it pulls positions from a brokerage-aggregation API, gets closing prices and
headlines from a market-data vendor, writes a snapshot to Postgres, and emails a digest: total value,
open P&L, the day's three best and worst positions, headlines on the largest holdings, and a
two-sentence summary. During the day there's a dashboard on top of the same data.

Stack: Next.js on Vercel, Supabase for Postgres, cron jobs for the nightly run, Claude for the written
summary, Resend for delivery.

## Three bugs worth writing down

**The nightly email never sent, and nothing said so.** The job looked up holdings using the wrong
identifier — it passed the application's client ID where the per-user ID belonged. The API returned
nothing, the code fell back to its cache, the cache lookup used the same wrong ID and also returned
nothing, and the job exited with "no positions found". Every night, for months.

Nobody noticed because a cron job that fails silently is indistinguishable from a cron job that isn't
scheduled. Nothing alerts you to an email that doesn't arrive. The fix was one line; the lesson was that
a scheduled job needs to report success somewhere a human looks, not just fail quietly.

**Positions could count twice.** Two code paths deduplicated holdings differently — one keyed on ticker,
the other on ticker plus account. With two connections to the same brokerage account, the same holding
appeared twice and the total inflated. The two paths had been written weeks apart, each sensible on its
own. There's now one definition, used by both.

**A silent fallback presented stale numbers as live.** When the live fetch failed, the job quietly used
the last cached holdings and sent an email that looked completely normal. That's worse than failing:
a missing email is obvious, but an email with yesterday's numbers and today's date is trusted. It now
labels the figures, dates them, and marks the subject line, and the endpoint reports whether it served
live or cached data.

## The news section was always about the same stock

The digest showed headlines for "the top 5 holdings", except the code took the first five positions in
whatever order the API returned them, over a 24-hour window. Most tickers have no news on a given day,
so the section usually showed whichever single name did — often the same small position, night after
night.

Now it sorts by market value, looks back three days, caps each ticker at two articles, and interleaves
them so one noisy name can't fill the section. Small fix, and the section went from decorative to useful.

## What it costs to run

Worth measuring rather than guessing, because "add an LLM to it" sounds expensive and mostly isn't.

The nightly email costs about **a tenth of a cent**. One call to a small model, a prompt of a few hundred
tokens, an output capped at 256. Everything else — prices, headlines, database, hosting, email delivery —
sits inside free tiers. Call it **three cents a month**.

The expensive part was never the automation. It was the on-demand "AI ideas" tab, which fanned out to a
larger model with web search across the whole portfolio: roughly a hundred times the cost of a digest,
per click. Caching the research results at two hours took the cost of repeated clicks to zero, and the
lesson generalises — the recurring job was cheap and the interactive feature was not, which is the
opposite of what I'd assumed.

## The part I removed

The app had a tab where a model produced buy / sell / hedge recommendations, each with a confidence
percentage. The prompt told it to use a 30–89% range and never to exceed 89%.

That number is invented. There's no model behind it, no calibration, nothing to check it against. It
looks quantitative, which is precisely the problem: a made-up number wearing a lab coat is worse than no
number, because it invites you to act on it.

What replaced it is a risk engine, [riskkit](https://github.com/omar-r21/riskkit), which I built
separately and open-sourced. It computes value-at-risk and expected shortfall, validates those models
against twenty years of history, and attributes risk to individual positions. The app is now a consumer
of it. Every weekday evening a scheduled job sends the engine the portfolio's weights (never dollar
amounts), runs it, and writes the results to Postgres. The Risk tab that took the old tab's place shows
95% and 99% VaR and ES, and which positions actually drive the risk as opposed to which are largest. The
digest says whether the day's move broke the previous evening's forecast. A separate monthly job
backtests the models on a decade of history for the holdings old enough to have one, and names the
ones it had to leave out.

The language model stays, in a smaller role: it writes the two or three sentences that summarise the
session. It no longer makes recommendations. It's handed only the computed figures and told not to
write any number, asset, or reason for a move that isn't in them. That's a constraint in the prompt, not
a structural guarantee, so the figures in the email's tables come from the code, never from the model.

That split also settles what can be public: the analytics are a general-purpose library with no personal
data in them, so they're open source. The app that reads a real account stays private.

---

Not investment advice. Copyright (c) 2026 Omar Rayan; the app's source isn't published.
