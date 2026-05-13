**The Build**

| Deadline: Sunday, 17 May 2026, 11:59 PM IST. Submit: Reply to the email with your GitHub repo link. Stuck? Reply at any point. We'd rather hear than have you blocked. |
| :---- |

## **The problem**

We want to build AI employees for D2C brands.

A D2C founder runs their business across a long tail of SaaS tools. To answer one cross-tool question, they spend half an hour stitching exports in Excel. Most of the time they don't bother. They run on vibes.

## **What we want you to build**

A working v0. Five hard requirements.

* **At least 3 proper connectors** to different SaaS sources, behind one shared abstraction. You pick which three.

* **A universal data model.** Normalise across sources. Provenance on every row.

* **A chat layer.** Tool-use loop. Reads and writes over the data. Every numerical claim in the answer carries a citation back to the source rows. Uncited numbers don't survive to the user.

* **At least one autonomous agent.** Your "AI employee". Watches the data, proposes a ₹-saving or ops-saving action. Don't actually send anything; we want the run log and the reasoning. You pick what it does.

* **A scalability layer.** Works for one merchant today, needs to hold for ten thousand. Build the harness, or sketch it in the README. What breaks first. What you've built to absorb it.

## **What we're looking at**

Two things matter equally: **how well you build, and what you choose to build.** The first is craft. The second is judgment. We want to see both.

Why these three connectors. Why this schema. Why this agent. Why these chat tools. Why this harness. The "why" should be in your README, not just the "what".

## **What we score on**

| Axis | What good looks like |
| :---- | :---- |
| **Judgment** | What you chose to build, why those connectors, why this agent. Reasoning visible in the README. |
| **Speed** | How fast you got from start to working v0. We'll see it in your commit history. |
| **Connector abstraction** | One interface, three real implementations, swappable. |
| **Schema discipline** | Source-agnostic. Provenance on every row. |
| **Chat grounding** | Citation contract is real. Every number traceable. No hallucinated values. |
| **Agent design** | Trigger, data, decision, action all explicit. Failure modes called out before we ask. |
| **Scale / harness thinking** | Honest about what breaks at 10k merchants. Real engineering, not a paragraph of buzzwords. |
| **Eval honesty** | You tell us where it breaks before we find it. |

## **What you submit**

Reply to the email with your GitHub repo link. The README should answer:

1. What you built. 5-line architecture summary.

2. Connectors. Which 3\. Why these 3\.

3. Schema. Why this shape.

4. Chat. The tool schema you exposed. How citation works.

5. Agent. What it does. Why this one.

6. Scale. How this goes from 1 merchant to 10,000. What breaks. What you've built to absorb it.

7. Eval. Where it breaks.

8. Hours spent. Across how many days or sessions.

9. What you'd do with another week.

## **A note on AI tools**

Use them. We expect you to. Be honest in the README about what you wrote vs. what the LLM wrote. That's signal, not stigma.

## **If you get stuck**

Reply to the email. Tell us where you're stuck, what you've tried, what you'd ship if we said "ship what you have". We'd rather adjust than have you submit nothing.