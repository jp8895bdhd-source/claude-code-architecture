# Command Pattern: 35+ Slash Commands by Domain

## Overview

Slash commands are the user interface to Claude Code. Each command is a markdown file in `~/.claude/commands/` that maps a keyword to a skill, agent, or direct action.

Commands are thin wrappers — they parse arguments and dispatch to skills or agents that do the real work.

```
~/.claude/commands/
  ├── morning.md              # /morning — daily briefing
  ├── eod.md                  # /eod — end-of-day summary
  ├── pipeline.md             # /pipeline — account health dashboard
  ├── account.md              # /account [name] — account brief
  ├── contact.md              # /contact [name] — person intelligence
  ├── finance.md              # /finance — financial dashboard
  ├── amex.md                 # /amex — credit card tracking
  ├── miles.md                # /miles — airline rewards tracking
  ├── jobs.md                 # /jobs — job market monitoring
  ├── draft.md                # /draft [context] — email drafting
  ├── prep.md                 # /prep [context] — meeting prep
  ├── followup.md             # /followup — unreplied emails
  └── [30 more...]
```

Commands typically:
1. Parse arguments (account name, date range, filter)
2. Dispatch to a skill or agent
3. Return formatted results

---

## Command Organization by Domain

### Daily Workflow (11 commands)

**`/morning`**
```
Usage: /morning
Purpose: Aggregated daily briefing
Dispatches: daily-briefing agent (async, Haiku)
Returns:
  - Today's calendar with prep notes
  - Inbox status (unread count by type)
  - Action items (overdue, due today, at-risk)
  - Financial snapshot (balances, budget status)
  - Travel alerts (flights, upcoming reservations)
Runs at: 07:00 AM automatically (async hook)
```

**`/eod`**
```
Usage: /eod
Purpose: End-of-day wrap-up and tomorrow planning
Dispatches: daily-briefing agent
Returns:
  - What happened today (emails sent, meetings, actions completed)
  - Tomorrow's calendar (key meetings, prep notes)
  - Unfinished action items (move to tomorrow?)
  - Deal motion summary (deals advanced/closed/stalled)
  - Spending today
  - Recommendations (3 things to prioritize tomorrow)
```

**`/inbox`**
```
Usage: /inbox [count]
Purpose: Top emails ranked by urgency
Dispatches: Email search + custom ranking logic
Returns:
  - Urgent (responses needed, flagged, from key contacts)
  - Meetings (invites, confirmations)
  - Followup needed (you replied but no response yet)
  - FYI (bulk subscriptions, low priority)
Default: Top 10
```

**`/prep [context]`**
```
Usage: /prep "ClientName" or /prep "1:1 with SalesLead"
Purpose: Meeting preparation brief
Dispatches: account-intelligence agent (if company name detected) or contact agent
Returns:
  - Account/person background (recent activity, engagement)
  - Topics to discuss (current opportunities, blockers, changes)
  - Recent context (emails, meetings, deal status)
  - Key questions to ask
Timing: Takes ~10 seconds
```

**`/draft [context]`**
```
Usage: /draft "Follow up on proposal" or /draft "Networking email"
Purpose: Generate email draft
Dispatches: email-drafter subagent (Sonnet)
Returns:
  - Short version (1 paragraph, direct)
  - Long version (2-3 paragraphs, detailed)
  - Optional subject line variations
User must: Copy/paste to email client (no auto-send)
```

**`/followup`**
```
Usage: /followup
Purpose: Find emails needing attention
Dispatches: Email search + filtering
Returns:
  - Emails you sent but haven't received reply (>3 days)
  - Meetings that need scheduling
  - Proposals awaiting feedback
  - Conversations that went cold
```

**`/quick [question]`**
```
Usage: /quick "What's the GDP of France?" or /quick "Current BTC price"
Purpose: Fast answer via web search
Dispatches: Web search + Haiku
Returns: Single, direct answer (not exhaustive research)
Model: Haiku (fastest)
Timing: <5 seconds
```

**`/note [text]`**
```
Usage: /note "Follow up with ClientName on budget" or /note "Remember to book flights"
Purpose: Quick capture to Apple Notes
Dispatches: Apple Notes MCP directly
Creates: New note in "Claude" folder with timestamp
```

**`/remind [text]`**
```
Usage: /remind "Call main contact at 2pm" or /remind "Review proposal by Friday"
Purpose: Create reminder
Dispatches: AppleScript (not Apple Reminders MCP — it hangs)
Creates: New reminder in Reminders app
Timing: Instant
```

**`/research [topic]`**
```
Usage: /research "Market trends in hospitality" or /research "CompanyName funding"
Purpose: Deep research with auto-save
Dispatches: Web search + synthesis agent
Saves to: ~/research/[category]-[topic].md
Returns: Markdown document with findings, sources, action items
```

**`/weekly`**
```
Usage: /weekly
Purpose: Weekly rollup across all domains
Dispatches: Multiple agents in parallel
Returns:
  - Deals closed, advanced, stalled (pipeline summary)
  - Spending this week (vs budget, vs last week)
  - Emails sent, replied-to count (responsiveness)
  - Meetings held (count by type)
  - Travel miles earned
  - Action items completed vs outstanding
  - Key wins, blockers, learnings
Runs: Sunday evening (optional scheduling)
```

---

### Sales Intelligence (3 commands)

**`/pipeline`**
```
Usage: /pipeline [view] [filter]
Views:
  (default) — Dashboard: counts by status, top at-risk
  detail — Full details on all accounts
  red — RED accounts only (>45 days without contact)
  yellow — YELLOW accounts (weak engagement)
Filters: team, region, product, owner
Purpose: Account health dashboard
Dispatches: pipeline skill
Returns: Buckets (RED/YELLOW/GREEN) with health scores
```

**`/account [name]`**
```
Usage: /account "ClientName" or /account "CompanyName"
Purpose: Full account brief
Dispatches: account-intelligence agent (Sonnet)
Returns:
  - Health score (0-100)
  - Key contacts + recent interactions
  - Deal status and timeline
  - Recent activity (emails, meetings)
  - Alerts (gaps, changes, opportunities)
  - Next steps recommendation
Timing: ~10 seconds
```

**`/contact [name]`**
```
Usage: /contact "JohnSmith" or /contact "Sarah Chen"
Purpose: Person intelligence
Dispatches: Contact search + synthesis
Returns:
  - Title, company, email, phone
  - Last interaction (when, what about)
  - Meeting frequency
  - Email responsiveness
  - Common topics
  - LinkedIn profile if available
```

---

### Financial Tracking (12 commands)

**`/finance`**
```
Usage: /finance [view] [period]
Views:
  (default) — Dashboard: balances, spending, alerts
  spend — Spending breakdown by category
  upcoming — Due dates and payment alerts
  summary — Year-to-date overview
  sync — Manual refresh from banks
Period: this-week, this-month, ytd, last-30
Purpose: Financial dashboard
Dispatches: finance skill
Returns: Account balances, spending vs budget, alerts
```

**`/spend [period]`**
```
Usage: /spend (defaults to this-month) or /spend this-week
Purpose: Detailed spending breakdown
Dispatches: finance skill with "spend" view
Returns:
  - Category breakdown (groceries, dining, travel, etc.)
  - Budget comparison (amount / limit)
  - Trends (vs last period)
  - Largest transactions
```

**`/amex`**
```
Usage: /amex [view]
Views:
  (default) — Dashboard: balance, utilization, points, due date
  rewards — Point balance + earning rate per category
  benefits — Active benefits summary
  transactions — Recent charges
  refresh — Manual sync with issuer
Purpose: Primary credit card tracking
Dispatches: amex skill
Returns: Balance, limit, utilization %, rewards, next due date
Sync: Hourly automatic
```

**`/miles`**
```
Usage: /miles [airline] [view]
Airlines: united, american, delta, southwest, jal, virgin, etc.
Views:
  (default) — Dashboard: balance, elite status, expiration
  upcoming — Next scheduled flights
  alerts — Expiring miles, benefits, status
  elite — Status progress and benefits
Purpose: Frequent flyer tracking
Dispatches: miles skill
Returns: Account balance, elite status, upcoming trips, alerts
```

**`/bills`**
```
Usage: /bills
Purpose: Upcoming payment due dates
Dispatches: finance skill with "upcoming" view
Returns: List of known recurring bills, due dates, amounts
Also shows: Estimated vs actual (if multiple years of data)
```

**`/history [merchant]`**
```
Usage: /history "Airline" or /history "Grocery" (optional filter)
Purpose: Transaction history search
Dispatches: Database query
Returns: Last 20 transactions matching filter, with dates and amounts
```

**`/budget`**
```
Usage: /budget [category]
Purpose: Budget review and adjustment
Returns: Current month spending vs budget, recommendations
Can update: YAML config to adjust budget numbers
```

**`/audit`**
```
Usage: /audit
Purpose: Financial compliance review
Returns: All transactions in last month, categorized, flagged for review
Checks: Duplicate charges, unusual amounts, missing categorization
```

**`/investments`**
```
Usage: /investments
Purpose: Portfolio dashboard
Returns: Holdings, allocation, performance YTD
```

**`/taxes`**
```
Usage: /taxes [year]
Purpose: Tax tracking and estimation
Returns: YTD wages, withholding, deductions, estimated liability
```

**`/balance`**
```
Usage: /balance
Purpose: Quick account balance snapshot
Dispatches: finance skill
Returns: All accounts and current balances (fastest view)
```

---

### Job Market Monitoring (8 commands)

**`/jobs`**
```
Usage: /jobs [filter]
Filters: company, location, seniority, keywords
Purpose: Daily job market scan
Dispatches: jobs skill (runs on schedule, 06:00 AM)
Returns:
  - New postings matching your profile
  - Company match score (0-100)
  - Salary range if disclosed
  - Work location verification (not just keywords)
  - Fit analysis vs current role
Summary: "3 good matches this week"
```

**`/company [name]`**
```
Usage: /company "TechCorp" or /company "SaaSCompany"
Purpose: Deep dive on specific company
Dispatches: Web search + synthesis agent
Returns:
  - Company background (size, funding, stage)
  - Recent news (funding, leadership changes, layoffs)
  - Job openings + roles
  - Estimated salary ranges
  - Glassdoor/interview feedback if available
  - Decision: "Worth exploring?" assessment
```

**`/linkedin [query]`**
```
Usage: /linkedin "VP Sales at TechCorp" or /linkedin "ML Engineers in SF"
Purpose: LinkedIn search for people/companies
Dispatches: LinkedIn Sales Navigator MCP
Returns: Matching profiles with current roles, companies, connection suggestions
```

**`/scan [frequency]`**
```
Usage: /scan daily (default) or /scan weekly
Purpose: Set automatic job market scan frequency
Configures: Cron job for /jobs command
Returns: Confirmation of schedule
```

**`/saved [view]`**
```
Usage: /saved (default: all) or /saved shortlist or /saved applied
Purpose: Manage saved job postings
Views: all saved, "shortlist" favorites, already applied-to
```

**`/feedback [company] [outcome]`**
```
Usage: /feedback "CompanyName" "offered" or /feedback "CompanyName" "rejected"
Purpose: Track interview/offer outcomes
Logs: Company name, outcome, date, notes
Updates: Job matching algorithm (learns what works)
```

**`/applyfor [URL]`**
```
Usage: /applyfor "https://jobs.company.com/posting/123"
Purpose: Quick-apply guidance
Dispatches: applyfor agent
Returns:
  - Fit analysis (match score)
  - Tailored resume (if applicable)
  - Cover letter suggestions
  - Application checklist
Timing: Takes ~30 seconds
```

**`/insights`**
```
Usage: /insights
Purpose: Job market intelligence
Returns: Trends in hiring, salary movements, emerging roles
Based on: Historical application and offer data
```

---

### Travel & Airline Rewards (8 commands)

**`/flights`**
```
Usage: /flights [timeframe]
Timeframe: today, week, month, upcoming
Purpose: Upcoming flights dashboard
Dispatches: miles skill
Returns: All scheduled flights with:
  - Dates, times, routes
  - Confirmation numbers
  - Seat assignments
  - Alerts (weather, changes, boarding)
```

**`/elite [airline]`**
```
Usage: /elite (all airlines) or /elite "United"
Purpose: Elite status tracking
Returns:
  - Current status (Gold, Platinum, etc.)
  - Progress to next tier
  - Benefit expiration dates
  - Upcoming requalification deadlines
```

**`/balance [airline]`**
```
Usage: /balance (all) or /balance "United" or /balance "Japanese Airlines"
Purpose: Miles balance snapshot
Returns: Points available per program, expiration dates
```

**`/awards [airline]`**
```
Usage: /awards "United" or /awards (all)
Purpose: Award availability search
Returns: Upcoming flights available for redemption with points
```

**`/claim`**
```
Usage: /claim [transaction] or /claim all (to audit)
Purpose: Track airline reimbursement claims
Returns:
  - Claimed expenses
  - Reimbursement status
  - Pending approvals
```

**`/upgrade [booking]`**
```
Usage: /upgrade (check all bookings) or /upgrade "United DEN-ORD"
Purpose: Check upgrade eligibility and availability
Returns: Status, cost if paid upgrade available
```

**`/expiring`**
```
Usage: /expiring [airline]
Purpose: Alert on expiring miles or elite status
Returns: What expires in next 90 days with recommended action
```

---

### Smart Home (3 commands)

**`/wifi`**
```
Usage: /wifi [action]
Actions: status, restart, optimize, details
Purpose: Mesh network dashboard and control
Returns: Network speed, connected devices, signal strength
```

**`/appliances`**
```
Usage: /appliances [action]
Purpose: Smart home device status
Returns: Power status, schedules, recent activity
```

**`/thermostat [action]`**
```
Usage: /thermostat (check status) or /thermostat "72F"
Purpose: Temperature control and history
Returns: Current temp, schedule, recent changes
```

---

### Kitchen & Household (3 commands)

**`/meals`**
```
Usage: /meals [timeframe]
Timeframe: today, week, upcoming
Purpose: Meal plan and shopping prep
Returns: Planned meals, ingredients, recipes
```

**`/pantry`**
```
Usage: /pantry [view]
Views: inventory, expiring, shopping-list
Purpose: Kitchen inventory tracking
Returns: What's in stock, what's expiring, what to buy
```

**`/shopping`**
```
Usage: /shopping [store]
Store: grocery, costco, etc. (or default: all)
Purpose: Shopping list aggregation
Returns: Items to buy, organized by store, with quantities
```

---

### Meta & System (5 commands)

**`/cheatsheet`**
```
Usage: /cheatsheet (or /help)
Purpose: Show command cheat sheet
Returns: Full table of 35+ commands with short descriptions
Similar to Unix `man` pages
```

**`/audit`**
```
Usage: /audit
Purpose: Audit all custom Claude Code configuration
Returns:
  - All commands (count, functionality)
  - All agents (available models)
  - All skills (domains covered)
  - All hooks (event triggers)
  - Settings (token limits, tool access, etc.)
```

**`/loop [interval] [command]`**
```
Usage: /loop 5m "/finance" or /loop 30m "/inbox"
Purpose: Run a command on a recurring schedule
Configures: In-session loop (doesn't persist past session)
Example: Monitoring mode during work day
```

**`/trending`**
```
Usage: /trending
Purpose: New Claude Code tools, tricks, or community patterns
Returns: GitHub stars, discussions, new MCP servers
Helps: Stay current with ecosystem
```

**`/backup`**
```
Usage: /backup (on-demand) or /backup dry-run
Purpose: Sync Claude Code config to GitHub
Returns: Confirmation of backup, what changed
```

---

## Key Design Patterns

### The No-Args Default Pattern

Every command has a useful default behavior when called with no arguments:
- `/pipeline` → Full dashboard (not `/pipeline help`)
- `/finance` → Balances + spending (not `/finance help`)
- `/miles` → All airlines status (not `/miles help`)

This makes the interface frictionless. Users don't need to remember subcommands.

### Consistent Subcommand Structure

Most domains follow: `/[domain]` (dashboard) and `/[domain] [view]` (specialized view)

```
/finance             # Dashboard (balances + spending)
/finance spend       # Spending detail
/finance upcoming    # Due dates
/finance sync        # Manual refresh

/miles               # All airlines status
/miles united        # Just United
/miles upcoming      # My upcoming flights
/miles alerts        # Expiring miles and benefits
```

### Dispatch Pattern

Commands dispatch to either:
- **Skills** (for domain-specific dashboards with state/config)
- **Agents** (for analytical tasks requiring synthesis)
- **MCP directly** (for quick actions like creating notes)

```markdown
/pipeline → pipeline skill (self-contained, has DB)
/account → account-intelligence agent (synthesizes multiple sources)
/note → Apple Notes MCP directly (simple passthrough)
```

### Argument Parsing Strategy

Arguments are space-separated, optional, and contextual:

```
/account "Client Name Inc"     # Name (quoted because multi-word)
/prep "1:1 with JohnSmith"     # Context description
/finance spend this-month      # View + period
/miles united upcoming         # Airline + view
```

Command parser handles:
- Quoted strings with spaces
- Hyphenated periods (this-month vs thisMonth)
- Optional arguments with sensible defaults

### Output Standardization

Most commands return scannable dashboards with:
1. **Summary line** (top)
2. **Status indicators** (✓, ⚠, →, ○)
3. **Data tables or structured lists**
4. **Next actions** (at bottom)

This makes output predictable and readable even when skimming.

---

## Growth Strategy

Starting setup: ~15 commands (morning, eod, inbox, draft, prep, pipeline, account, contact, finance, amex, miles, jobs, quick, note, remind)

Expansion:
- **Add 3-4 domain-specific commands** when a new skill is built
- **Consolidate** when multiple commands do similar things
- **Retire** when a command is never used (collect metrics)

Over 2 years, typical setup grows to 35-50 commands, organized by domain. The `/cheatsheet` command becomes the reference.

Every command should earn its place. If it's not used in a month, consider retiring it.
