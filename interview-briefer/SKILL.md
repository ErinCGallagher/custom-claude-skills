---
name: interview-briefer
description: Auto-research and generate a company intelligence briefing document to prepare for a job interview. Use this skill whenever the user mentions an upcoming interview, wants to prepare for a company interview, or asks for a briefing/overview of a company in the context of a job. Trigger for phrases like "prep me for my interview at X", "brief me on [company] before my interview", "I have an interview at [company]", "help me prepare for [company]", "what should I know about [company] interview", or any request combining company research + job interview prep. Also trigger when the user provides a company name and a job description link together. Do NOT trigger for general company research with no interview context.
---

# Company Intelligence Briefer

You're helping the candidate walk into an interview feeling fully prepared. Your job is to research the target company and role, then produce a polished briefing document they can read beforehand — personalised to their specific background and stories.

## Before You Begin

Read both resource files first:

- `references/about_candidate.md`
- `references/behavioural_stories.md`

A file is considered empty if it contains only template placeholders (e.g. `[Candidate Name]`, `[Strength]`, comment blocks, or blank section bodies) and no real candidate data.

**If either file is empty, stop immediately.** Do not proceed with research or ask the user any other questions. Instead, tell them:

> "Before I can build your briefing, I need your background and story bank filled in. Please update the following file(s) and then run the skill again:
> - `references/about_candidate.md` — your background, pitch, strengths, comp history, and what you're looking for
> - `references/behavioural_stories.md` — your STAR stories with a theme → story mapping table
>
> See the `example_*` files in the same folder for reference."

These files are what make the Talking Points and Behavioural Story Mapping sections genuinely useful rather than generic.

## Inputs

- **Company name** (required — ask if not provided)
- **Job description link** (strongly encouraged — pass directly to the jd_analysis agent if provided; do not fetch or analyze it yourself)
- **Interview date/role** (optional context — use to calibrate urgency and focus)
- **Location:** Default to **Toronto, Canada** unless explicitly stated otherwise. This affects office expectations, team structure, and compensation context.

---

## Research Process

**Do not do any web searching, URL fetching, or external research yourself.** All research is delegated entirely to the agents below — that is their whole purpose. Your only pre-synthesis work is reading the candidate's resource files and the agent instruction files, then spawning the agents.

**Before spawning any agents**, ask the user two questions:

> "Do you want me to research the hiring team and likely hiring manager on LinkedIn as well? It adds a couple of minutes but gives you specific people to reference."

> "Do you want a Hiring Manager Round prep section — specific likely questions mapped to your stories, with the key beats to land for each?"

Wait for their answers, then spawn all applicable agents in parallel.

| Agent                                                       | Instruction File                | When to Run                                          |
| ----------------------------------------------------------- | ------------------------------- | ---------------------------------------------------- |
| Company Research (overview, news, culture, interview, comp) | `agents/company_research.md`    | Always                                               |
| Team & People Intelligence                                  | `agents/team_linkedin.md`       | Only if user says yes to the LinkedIn question       |
| Job Description Analysis                                    | `agents/jd_analysis.md`         | Only if a JD URL was provided                        |
| Hiring Manager Round Prep                                   | `agents/hm_prep.md`             | Only if user says yes to the HM prep question        |

The hm_prep agent does not do external research — pass it the contents of `references/about_candidate.md`, `references/behavioural_stories.md`, the JD analysis output, and the company research output (interview themes + common question sections) directly in its prompt.

Before spawning each agent, read its instruction file from the `agents/` directory and pass the full contents as the agent's prompt (substituting the company name, team, role, and JD URL as appropriate). Collect all results, then synthesize them yourself into the final brief.

**If the jd_analysis agent returns `STATUS: BLOCKED`:** Stop and ask the user:
> "The job description link wasn't accessible. Could you tell me the job title and team name? I'll search for it directly."

Once the user responds, re-run the jd_analysis agent with the role name and team name so it can do a targeted fallback search. Do not proceed to synthesis until you have either JD content or confirmation from the user that they don't have those details.

---

## Output

Save the briefing as `[Company]_interview_brief.md` in the outputs folder using this structure:

```
# Interview Brief: [Company Name]

**Role:** [Job title, or "Not specified"]
**Date Prepared:** [Today's date]
**Location:** Toronto, Canada (unless stated otherwise)

---

## TL;DR
[2–3 sentences. What the company does, where they are right now (momentum/challenges), and the single most important thing to know going into this interview. A busy person should be able to read just this and be 60% prepared.]

---

## Company Overview
[One solid paragraph, 3-4 sentences: what they do, business model, stage, scale, headquarters, key products. Name the actual products.]

## Recent News & Developments
[Bullet list, most recent first. 3–5 items with approximate dates. Focus on things the interviewer might bring up.]

## Funding & Financial Health
[Most recent round or stock status. For public companies: recent performance signal, any guidance changes. 2-3 sentences max]

---

## Engineering Culture & Tech Stack

[2–3 paragraphs on how they build. Use specific evidence — blog post titles, named technologies, real architectural decisions. No vague claims without a source.]

### Recent Engineering Blog Highlights
[2–3 bullets: title, approximate date, 1–2 sentence takeaway, link if available. If no blog found, say so and suggest where to look.]

---

## Team Intelligence

[What you found about the people on this team. Only insert this section if asked to do so.]

**Likely Hiring Manager:**
[Name and title if found, or "Could not identify — worth asking the recruiter"]

**Team Members Found:**
[Names, titles, LinkedIn links where available, any notable public content they've written or presented]

**What the Team is Working On:**
[Public signals about current focus, recent projects, or technical direction from team members' content]

*Note: LinkedIn requires login for full access — names surfaced via public Google results. Verify before the interview.*

---

## Interview Intelligence

**Typical Process:**
[Interview stages in order]

**Common Question Themes:**
[Topics that come up repeatedly]

**Compensation — Toronto / Canada:**
[Bands by level if found: Senior and Staff specifically. Note source and date. If no Canada-specific data found, note that and provide US figures with a conversion caveat.]

**Tips from Past Candidates:**
[3–5 specific, actionable pieces of advice]

**Red Flags / Things to Watch:**
[Honest patterns from negative reviews]

---

## Culture Signals

**Consistent Praise:** [What employees love]
**Recurring Concerns:** [What they complain about — don't soften this]
**Toronto Office:** [Any signals about the Toronto team specifically — size, vibe, RTO policy]
**Overall Vibe:** [1–2 sentences synthesising]

---

## Role Analysis
*(Only if a JD was provided)*

**What They're Actually Looking For:**
[The core problem they're hiring to solve]

**Must-Haves:**
[Non-negotiable requirements]

**Nice-to-Haves:**
[Preferred but not required]

**Likely Interview Focus Areas:**
[Technical and behavioral topics that will probably dominate, based on JD emphasis]

---

## Hiring Manager Round Behavioural Stories

*(Only include this section if the user requested HM prep — paste the hm_prep agent output here as-is)*

---

## Questions to Ask

[6–8 thoughtful questions. At least one should reference something specific from the Team Intelligence section — a blog post a team member wrote, a technical decision, or a question directed at the hiring manager if identified. Mix of: role/team, engineering culture, and forward-looking questions.]

---

*Sources: [List key URLs and search results used]*
```

---

## Quality Bar

**Be specific, not generic.** Don't write "they care about engineering quality" — write "their March 2026 eng blog post on their Kafka migration suggests event-driven architecture is a current focus."

**Make the HM prep feel personal.** The questions, stories, and beats should read like someone who knows the candidate's work wrote them. Reference their projects, platform work, any core technology they used, and failures they overcame — whatever actually connects to what this company needs. If none of their stories connect well, say so honestly.

**Cover gaps honestly.** If LinkedIn data was sparse, say so. If comp data for Toronto wasn't available, say that.

**Surface surprises.** A blog post from the hiring manager, a recent reorg, a controversy — make it visible.

**Keep the TL;DR genuinely useful.** Imagine the candidate reading it 10 minutes before the interview.
