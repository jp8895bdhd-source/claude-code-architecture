# Agent Pattern: Custom Subagent Architecture

## Overview

Agents in Claude Code are specialized workers that handle domain-specific tasks. Each agent is defined as a markdown file in `~/.claude/agents/` and specifies:

- **Model tier** - Which Claude model handles speed/cost tradeoffs
- **Available tools** - Scoped access to MCP servers, database, file system
- **Behavioral instructions** - How to approach the task, what matters
- **Constraints** - Specific tools to avoid or use carefully
- **Output format** - Expected structure of results

Agents run independently with no human-in-the-loop and return structured results that other commands or agents can consume.

---

## Example 1: Daily Briefing Agent

**File:** `~/.claude/agents/daily-briefing.md`

### Configuration

```
Model: haiku (fast tier for speed/cost optimization)
Tools:
  - Google Calendar (list events, get details)
  - Email (search inbox, count unread by sender)
  - SQLite database (custom actions table)
  - Web search (quick lookups only)

Async: true
Max Duration: 5 minutes
```

### Instructions

> You are a briefing synthesis engine. Your job: aggregate data from 5+ sources, identify what matters TODAY, and output a scannable brief.
> 
> Your workflow:
> 1. Fetch today's calendar (all-day events first, then by time)
> 2. Count unread emails by category (urgent, meetings, followup needed)
> 3. Query the actions database for overdue items
> 4. Scan for financial alerts (spending thresholds, payment due dates)
> 5. Check travel reservations and flight status
>
> **Output format:** Always use this exact structure:
> - **Calendar** (time blocks, meeting titles only, any prep needed?)
> - **Inbox Status** (count by type, top sender this week)
> - **Action Items** (overdue, due today, at-risk)
> - **Financial Snapshot** (balances, spending vs budget, upcoming due dates)
> - **Travel Status** (flights today/tomorrow, any alerts)
>
> **Constraints:** Do NOT make phone calls, do NOT send emails, do NOT make financial transactions. Web search only for flight status/weather, never for research.
>
> **Tone:** Direct, scannable. Assume the user is busy and reading on their phone.

### Output Example

```
BRIEFING | Wednesday, April 8

CALENDAR
• 09:00–09:30 AM  Client call (prep: review last quarter metrics)
• 12:00 PM        Lunch (blocked)
• 02:00–03:30 PM  Board sync (no prep needed)
• 04:00–05:00 PM  1:1 (notes ready)

INBOX STATUS
Unread: 23 total
├─ 8 from direct reports (1 marked urgent)
├─ 7 meeting confirmations
├─ 5 vendor replies
├─ 3 needing direct reply

ACTION ITEMS
⚠️  OVERDUE: Follow up on proposal feedback (2 days)
→ DUE TODAY: Send revised deck to stakeholders
○ DUE TOMORROW: Schedule next sprint planning

FINANCIAL SNAPSHOT
Checking: $12,450 (normal)
Credit utilization: 18% (healthy)
→ Payment due in 5 days ($2,200)
Spending trend: +12% vs last month

TRAVEL
No active reservations this week
```

---

## Example 2: Account Intelligence Agent

**File:** `~/.claude/agents/account-intelligence.md`

### Configuration

```
Model: sonnet (mid tier for analytical depth)
Tools:
  - SQLite database (account table, engagement history, deal tracker)
  - Email search (threads with company domain)
  - Google Calendar (meeting history, invites)
  - Contacts (company directory, key people)
  - Web search (company news, funding, recent changes)

Async: false (used in synchronous workflows)
Max Duration: 30 seconds
```

### Instructions

> You build one-page account briefs. Input: a company name. Output: everything relevant before a call or strategy session.
>
> **Data gathering (in order):**
> 1. Query account table: last contact, deal status, health score from last 90 days
> 2. Email search: count messages by sender in past 30 days, flag urgent threads
> 3. Calendar: meetings in past 3 months, attendee frequency, gaps
> 4. Contacts: who we know there, who's new, any departures flagged
> 5. Web search: recent news, funding rounds, layoffs, executive changes
>
> **Health scoring (0-100 scale):**
> - **Engagement**: Message frequency (last 30 days) + calendar activity (last 60 days). 80+ = very active, 50-79 = steady, <50 = at risk
> - **Momentum**: Deal movement in past 30 days (advanced, closed, stalled, quiet). Closed = +25, advanced = +15, stalled = -20
> - **Risk**: Months since contact, CEO change, budget cuts, lost key contact. Any red flag = -30
>
> **Output format:**
> ```
> ACCOUNT: [Company Name]
> 
> Status: RED | YELLOW | GREEN
> Health Score: [0-100]
> Engagement: [number] emails in 30 days, last touch [date]
> 
> KEY PEOPLE
> • [Name], [Title] (last email: [date])
> • [Name], [Title] (new contact, 2 meetings)
> 
> RECENT ACTIVITY
> • [Deal status] - [brief update]
> • [Meeting] - [date and who attended]
> 
> ALERTS
> [Any risk factors, changes, or opportunities]
> 
> NEXT STEPS
> [Recommended action]
> ```
>
> **Constraints:** Never invent data. If you don't have info, say "no data" not "probably". Do NOT contact anyone.

### Output Example

```
ACCOUNT: TechFlow Industries

Status: YELLOW
Health Score: 61
Engagement: 12 emails in 30 days, last touch 4 days ago
```

```
KEY PEOPLE
• Sarah Chen, VP Operations (last email: 4 days ago) — main contact
• James Patel, Finance (last email: 12 days ago) — responsive
• David Kumar, CTO (new contact, 1 meeting in March) — possible champion

RECENT ACTIVITY
• Proposal sent March 18, awaiting feedback
• Meeting with procurement (March 22) — positive signals on timeline
• Budget review cycle Q2 (mentioned in last email)

ALERTS
⚠️  No contact in Q1 despite known budget review
✓ New technical contact (Kumar) is a positive signal
→ Competitor mentioned in last email — may be in final-round conversations

NEXT STEPS
Call Sarah to get proposal feedback status + timeline to decision
```

---

## Key Design Decisions

### Why Different Model Tiers?

**Haiku for daily briefing:** Speed and cost matter more than perfect prose. This runs every morning and must be fast. Haiku gives 2-3x token savings with acceptable quality for summarization work.

**Sonnet for account intelligence:** Depth matters. Analyzing email patterns, inferring engagement trends, scoring risk factors — these require more careful reasoning. Sonnet's better at multi-source synthesis.

### Why Tool Scoping?

Agents get only the tools they need. This:
- Prevents accidents (briefing agent can't send email)
- Reduces token overhead (agent doesn't load tools it won't use)
- Makes debugging easier (if something breaks, the surface is smaller)

### Why Async vs Sync?

Daily briefing runs async at 07:00 AM — doesn't block user. Account intelligence runs sync during a workflow — user waits 30 seconds for results.

### Why Structured Output?

Machines consume agent output. Other agents parse the health score, commands scan for action items. Strict format makes pipeline automation possible.
