---
name: tldr
description: Condense a long document, plan, message, or block of text into a short, skimmable summary built for fast review and go/no-go decisions. Use this whenever the user asks to "tl;dr", "summarize", "condense", "shorten", "give me the gist", "what's the key point", or wants a quick take on a plan, PRD, proposal, spec, PR description, email thread, meeting notes, or any wall of text — and especially whenever Claude has just produced a multi-step plan or long proposal that the user needs to approve, since a tight summary makes reviewing and signing off much easier. Trigger even when the user doesn't say "tldr" explicitly but signals they want to review or decide on something long ("is this good to go?", "what am I agreeing to?", "quickly, what does this do?").
---

# TL;DR

Turn long content into a summary someone can read in ~30 seconds and act on. The goal is not to compress evenly — it's to surface what a reviewer needs to decide: what will happen, what's risky, and what they should look at closely.

## When you're summarizing a plan of your own

If you (Claude) just produced a plan, proposal, or multi-step approach and the user wants the TL;DR to approve it, optimize for the approval decision. The reader already trusts the direction — they need to catch anything they'd regret waving through. Lead with the shape of the plan, then flag the parts that carry risk or are irreversible.

## Output format

Pick the format that fits the length and stakes. Default to the shortest one that works.

### For most things — the one-liner + bullets

```
**TL;DR:** [one sentence — what this is and what it does/asks for]

- [Key point 1]
- [Key point 2]
- [Key point 3]
```

Aim for 3–5 bullets. Each bullet is one idea, front-loaded with the thing that matters. Cut anything the reader could infer or wouldn't act on.

### For plans awaiting approval — add a decision line

```
**TL;DR:** [what the plan does, in one sentence]

**Does:**
- [concrete action 1]
- [concrete action 2]

**Watch:** [the 1–2 things worth a closer look — irreversible steps, assumptions, scope edges, anything that would be costly to get wrong]

**To approve, you're OK with:** [the key commitment(s) in plain terms]
```

Only include **Watch** if there's something real to flag. Don't manufacture risk to fill the section — an honest "nothing risky here" is more useful than invented caveats.

### For very short source content

If the input is only a few paragraphs, skip the scaffolding and give a single tight sentence or two. Structure that's heavier than the thing it summarizes defeats the purpose.

## Principles

**Front-load the decision, not the background.** A reviewer wants "this migrates the DB and deletes the old table" before they want why the migration exists. Lead with consequences and asks.

**Preserve specifics that change the decision.** Numbers, names, deadlines, irreversible actions, and money survive compression. "Costs some money" is useless; "costs ~£400/mo" is a decision input. Vague summaries force the reader back to the original, which defeats the point.

**Match length to stakes, not to source length.** A 40-page doc about a reversible, low-stakes change can be three bullets. A one-paragraph plan to delete production data deserves a Watch line. Compress by importance, not by ratio.

**Don't smuggle in new claims.** Summarize only what's in the source. If something critical is missing or ambiguous in the original, it's fair to note the gap ("plan doesn't say how rollback works") — but never invent detail to make the summary feel complete.

**Neutral on the merits unless asked.** The job is to represent the content faithfully so the user can decide, not to argue for or against it. Flagging risk is fair; editorializing on whether the plan is good is not, unless they ask for your take.

## Examples

**Example 1 — summarizing a long PRD**

Input: a 3-page product requirements doc for a new export feature.

Output:
> **TL;DR:** Adds CSV/JSON export to the reports page, gated behind a paid-tier flag, shipping in Q3.
>
> - Users on Pro+ can export any report; free tier sees an upsell
> - Export runs async with an email-when-ready flow (no blocking UI)
> - Requires a new `exports` service + S3 bucket; owned by the Data team
> - Success metric: 15% of Pro users export within 30 days

**Example 2 — summarizing Claude's own migration plan for approval**

Output:
> **TL;DR:** Migrates the users table to the new schema in three steps, with a backfill in the middle.
>
> **Does:**
> - Adds new columns, backfills from the old ones, then drops the old columns
> - Runs the backfill in batches overnight to avoid table locks
>
> **Watch:** Step 3 drops the old columns — irreversible once run, so it's gated behind a manual confirm after you've verified the backfill.
>
> **To approve, you're OK with:** a schema change to the production users table and an overnight backfill job.

## Anti-patterns

- **Restating the structure instead of the substance.** "The doc has three sections covering background, approach, and risks" tells the reader nothing. Summarize what those sections *say*.
- **Even compression.** Don't shrink every part proportionally; drop the unimportant parts entirely and keep the load-bearing ones sharp.
- **Hedging everything.** "May potentially involve some changes" is noise. Say what changes.
- **Burying the ask.** If the content is a request for a decision, that ask goes in the first line.
