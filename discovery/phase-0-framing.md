# Phase 0 Framing: Side-by-Side Candidate Comparison

## Candidate A — Slack replacement where bots are first-class citizens

### User Profile

The primary user is a software engineer or technical team lead at a company that has meaningfully adopted automation: scheduled pipelines, CI/CD bots, alert integrations, or LLM-powered assistants posting into shared channels. `[observation]` They work in async-first teams of 5–50 people where Slack (or equivalent) is the primary coordination surface. `[observation]` They are sophisticated enough to have written or deployed at least one bot but frustrated enough that they've complained in a retro or Slack thread about bot noise. `[inference]` A secondary user is the platform or developer-experience engineer responsible for maintaining integrations — this person feels the pain most acutely because they own the bots and field the complaints. `[inference]`

### Strongest Pain

Today, bot messages in Slack are visually and structurally second-class: they carry a BOT badge, cannot reply to humans as peers, flood channels with unthreaded noise, and offer no native way to assign work to them or track their outputs conversationally. `[observation]` The moment of maximum friction occurs when a team is mid-incident: an alert bot fires into #ops, a human asks a follow-up question in thread, the bot cannot read the thread and re-fires the same alert, and someone manually copies state between the bot output and a human conversation in a different channel. `[inference]` Teams currently paper over this with elaborate channel taxonomies, Zapier duct tape, or just muting bots entirely — all of which are workarounds that signal the underlying model is broken. `[inference]`

### Smallest Validatable Wedge

Recruit five teams that run at least one Slack bot in a production channel. `[hypothesis]` In a 30-minute interview each, ask them to walk through the last time a bot message caused confusion or required manual follow-up. `[hypothesis]` Do not pitch a product — listen for whether the friction they describe maps to the first-class-bot framing (peer addressing, conversational continuity, shared context) or to something else entirely (e.g., alert fatigue, bad bot copy, permission problems). `[hypothesis]` If four of five interviews surface the same structural gap (bots can't participate in conversation, only emit into it), the pain is real enough to proceed. `[hypothesis]` This requires no code, no mockup, and no commitment beyond calendar time.

---

## Candidate B — Single pane of glass for engineering and product

### User Profile

The primary user is an engineering manager or senior individual contributor (staff+ engineer or tech lead) at a growth-stage company running three or more distinct tooling surfaces: a project tracker (Jira, Linear), a code host (GitHub, GitLab), and a product analytics or observability tool (PostHog, Datadog, Mixpanel). `[observation]` They have enough context obligations — sprint health, PR review queue, feature flag rollout status, error rates — that they maintain a set of browser tabs or a personal dashboard that they refresh throughout the day. `[observation]` They are not highly specialized in any one tool; they are generalists who need a quick read of overall system health rather than deep dives. `[inference]`

### Strongest Pain

The tax is not the tab-switching itself but the **re-orientation cost**: each tool has its own mental model, search syntax, and notification stream, so moving between them requires a context shift, not just a click. `[inference]` The moment of maximum friction is the morning standup or async status update, when the user must manually aggregate state across tools to answer "what's blocked, what shipped, what's on fire" — a synthesis that no individual tool provides and that currently lives in someone's head or in a hand-written doc that is immediately stale. `[inference]` Existing "integrations" (e.g., GitHub-Jira links, Slack digests) partially address this but produce data dumps rather than opinionated summaries, so the cognitive load of synthesis is not actually reduced. `[observation]`

### Smallest Validatable Wedge

Identify ten engineering managers or tech leads via LinkedIn or a community (e.g., Rands Leadership Slack, Engineering Enablement Discord) who manage teams using at least GitHub + one tracker. `[hypothesis]` Ask them to share a screen recording (or walk through live) of their first 15 minutes of work on a Monday morning, narrating what they are checking and why. `[hypothesis]` The goal is to measure two things without leading: (1) how many tool transitions occur before they feel "oriented," and (2) whether the synthesis step is explicit (they write something down) or implicit (they just know). `[hypothesis]` If the average is more than four tool transitions before orientation and the synthesis is always implicit, the pain is real and under-served. `[hypothesis]` This requires no code and produces a reusable research artifact regardless of outcome.

---

## Questions for G0

The following are decision criteria the human should weigh before selecting a candidate. No ranking is implied.

1. **Switching cost tolerance**: Candidate A requires displacing an entrenched communication tool (Slack); users must migrate contacts, history, and integrations. Candidate B adds a layer on top of existing tools without requiring replacement. How much behavior change is this team willing to ask of early users?

2. **Distribution surface**: Candidate A competes in a market with strong network effects — a messaging tool is only as valuable as the number of colleagues on it. Candidate B can be adopted by a single person on day one. Which growth model fits the team's go-to-market capability?

3. **Depth of wedge**: Candidate A's wedge (bots as peers) is a fundamentally different data model than Slack, which may require a greenfield build. Candidate B's wedge (unified read view) is a read-only aggregation problem that can be prototyped with existing APIs. How much technical risk is acceptable before G1?

4. **Market timing**: The LLM wave has produced a surge of "AI-native" messaging experiments. `[observation]` Is the team trying to catch this wave (urgency favors A) or does the saturation of AI tools make a focus-productivity play (B) more differentiated right now?

5. **Evidence base**: Both candidates above rest primarily on inferences and hypotheses because no primary research has been conducted yet. The G0 decision should account for how confident the team is in the stated pain before committing to discovery depth for one candidate over the other. Which candidate has stronger pre-existing signal from the team's own experience?
