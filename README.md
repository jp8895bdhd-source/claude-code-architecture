# Claude Code Architecture Showcase

## The Story

This repository documents a personal automation platform built entirely within Claude Code, Anthropic's CLI tool. The builder brings 15+ years in enterprise SaaS sales and zero formal coding background. No separate development environment. No framework setup. No language switching. Every component was built through conversational prompts with Claude.

The system now runs daily operations across six functional domains: sales pipeline management, financial tracking and portfolio automation, travel logistics and frequent flyer management, job market intelligence, household and kitchen automation, and personal workflow optimization. It demonstrates what enterprise-grade adoption looks like when a non-technical power user fully commits to the platform.

This isn't a toy. The system makes real decisions, moves real money, sends real emails, and has been in continuous operation for over a year. It reveals both the ceiling of what a non-engineer can build with Claude and the kinds of problems a serious adoption platform needs to solve.

---

## System at a Glance

| Metric | Count |
|--------|-------|
| Custom AI agents | 6 |
| Domain-specific skills | 14 |
| Slash commands | 35 |
| SQLite tables | 40+ |
| Functional domains | 9 |
| MCP server integrations | 12+ |
| Scheduled automation triggers | 9 (5 cron + 4 LaunchAgents) |
| Custom scripts | 36+ |
| Hook points | 7 |
| **Total system size** | ~1GB (Claude Code harness), with domain-specific modules |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│         Claude Code Harness (~1GB)                          │
│  Settings • Hooks • Plugins • Scripts • Commands • Agents   │
└─────────────────────────────────────────────────────────────┘
              ↓              ↓              ↓              ↓
        ┌─────────┐  ┌──────────────┐  ┌─────────────┐  ┌──────────┐
        │ Sales   │  │ Investment & │  │ Household  │  │ Calendar │
        │ Intel   │  │ Financial    │  │ Automation │  │ & Email  │
        │(175MB)  │  │ (216MB)      │  │            │  │          │
        └─────────┘  └──────────────┘  └─────────────┘  └──────────┘
              ↓              ↓              ↓              ↓
        ┌──────────────────────────────────────────────────────────┐
        │         SQLite (9 Domains, 40+ Tables)                   │
        │  Sales • Credit Cards • Travel • Retirement • Jobs •    │
        │  Tax • Personal Finance • Subscriptions • System        │
        └──────────────────────────────────────────────────────────┘
              ↓              ↓              ↓              ↓
        ┌──────────────────────────────────────────────────────────┐
        │         MCP Integrations (12+)                           │
        │  Gmail • Google Calendar • Apple (Mail/Calendar/         │
        │  Messages/Notes/Contacts/Maps) • Playwright •           │
        │  Perplexity • GitHub • Plaid • AppleScript • Photos     │
        └──────────────────────────────────────────────────────────┘
```

---

## Agent Architecture

The system runs six specialized agents, each configured with a model tier optimized for its function:

### 1. Daily Briefing Agent (Fast Model)
Aggregates morning and end-of-day summaries from five independent data sources: calendar events, unread email and critical threads, personal reminders, investment portfolio snapshot, and financial health indicators. Runs on a fast model to minimize latency for time-critical morning use.

### 2. Account Intelligence Agent (Mid-Tier Model)
Generates full account briefs synthesizing live database records, recent email threads, calendar activity, research notes, and relationship memory. Used before calls or important email drafts.

### 3. Email Drafting Agent (Best Model)
Composes emails with personality consistency and audience calibration. Analyzes historical email threads to match tone and voice, then generates two versions: one straightforward, one with more personality. Runs on the best model because tone and relationship quality matter more than speed.

### 4. Meeting Prep Agent (Mid-Tier Model)
Creates one-page pre-meeting briefs including calendar context, attendee research, recent communications, account health status, and talking points drawn from the intelligence database.

### 5. Research Agent (Mid-Tier Model)
Conducts multi-source web research and automatically categorizes findings into organized markdown files for long-term reference and pattern detection.

### 6. Home Lab Manager Agent (Mid-Tier Model)
Manages infrastructure, scripts, and file organization with explicit caution modes that prevent accidental changes to live systems.

---

## Skill Architecture

Skills are domain-specific automation modules, each self-contained with its own database schema, validation logic, and output templates. Fourteen active skills:

### Sales & Customer Intelligence
**Pipeline Manager**
- Real-time account health scoring (RED/YELLOW/GREEN status)
- Engagement metrics across four dimensions: recency, momentum, engagement, and fulfillment (0-100 scale each)
- Automatic follow-up ranking by urgency and days-since-last-contact
- Email thread analysis to extract tone and sentiment

**Email Templates & Drafting**
- Context-aware draft generation pulling account history and deal status
- Tone matching from historical thread analysis
- Automatic version generation (formal + casual)
- Link and template library integration

### Financial Tracking & Optimization
**Credit Card Tracker**
- Playwright-based browser automation scraping live transaction data
- Multi-table schema tracking transactions, benefits, rewards, and payments
- Benefit utilization ROI calculation with expiration alerts
- Scrape history logging for debugging and compliance

**Retirement Fund Analyzer**
- Automated login with MFA code extraction from Messages database
- Fund comparison engine: fee analysis and return benchmarking against category benchmarks
- Verdict engine: fund-level recommendations (hold, upgrade, reduce)
- Monthly automated re-analysis via cron with report distribution

**Tax Projection Engine**
- Multi-person wage and withholding tracking
- Year-end federal liability projection with gap analysis
- W-4 adjustment recommendations with tax bracket modeling
- Quarterly update workflow with spreadsheet generation

**Personal Finance Dashboard**
- Plaid-powered bank account and credit card sync
- Spending categorization with merchant intelligence
- Upcoming payment due date alerts and reminders
- Month-over-month trend analysis with anomaly detection

### Travel & Logistics
**Frequent Flyer Manager**
- Seven airline integrations (each requires manual login due to anti-bot detection)
- Family member tracking across household accounts
- Expiring miles alerts and elite status monitoring
- Award availability tracking and award flight search assistance

### Job Market Intelligence
**Job Market Monitor**
- 320+ target company tracking across multiple ATS platforms (Greenhouse, Lever, Ashby, SmartRecruiters)
- 400-line skill module with fit scoring algorithm
- ATS-optimized document generation with keyword analysis
- Preference learning from user feedback and application outcomes
- Weekly automated outreach and application tracking

**Career Research**
- Role and company research with salary benchmarking
- Interview preparation guides
- Compensation negotiation talking points

### Household & Lifestyle
**Subscription Tracker**
- Use-it-or-lose-it benefit alerts (airline status, hotel elite benefits)
- Credential management and vault integration
- Utilization logging and cost-per-use analysis

**Smart Home Control**
- Mesh network device management and monitoring
- Device dashboard with status and control
- Natural language command routing (lights, temperature, appliances)

**Kitchen AI**
- Meal planning with dietary preference learning
- Shopping list generation with store-specific optimization
- Receipt processing and cookbook integration

---

## Command System (35 Slash Commands)

Commands are organized by domain and provide rapid access to the most common operations:

**Sales Intelligence** (8 commands)
- `/pipeline` — Current deal status and urgency ranking
- `/account [company]` — Full account brief from intelligence database
- `/contact [name]` — Person profile and relationship history
- `/draft [type]` — Email draft generation
- `/prep` — Meeting preparation brief
- `/followup` — Next critical followups by urgency
- `/competitor [name]` — Competitive intelligence
- `/summary` — Weekly pipeline summary

**Financial & Investments** (9 commands)
- `/portfolio` — Investment holdings and performance
- `/roth` — Retirement fund analysis and fund comparison
- `/tax` — Year-to-date tax projection
- `/spend` — Spending summary by category
- `/cards` — Credit card balances and benefits
- `/bills` — Upcoming payments due within 30 days
- `/miles` — Frequent flyer balances and expiring miles
- `/net` — Net worth snapshot
- `/gain` — RTK token savings analytics

**Job Market & Career** (5 commands)
- `/jobs` — New opportunities matching preferences
- `/applyfor [role]` — Application checklist and tailored resume
- `/track` — Job application pipeline status
- `/salary [role]` — Compensation benchmarking
- `/research [company]` — Company and role research

**Household & Lifestyle** (6 commands)
- `/wifi` — Network device status and control
- `/meal` — Meal planning and shopping list
- `/subscriptions` — Active subscriptions and benefits
- `/calendar` — Personal calendar and busy times
- `/morning` — Daily briefing
- `/eod` — End-of-day summary

**Admin & System** (7 commands)
- `/discover` — Analyze usage patterns for missed automation opportunities
- `/agents` — Current agent status and model assignments
- `/audit` — System health check (disk, memory, sync status)
- `/backup` — Trigger manual backup to GitHub
- `/export [domain]` — Export domain data as CSV or JSON
- `/logs` — Recent activity logs
- `/proxy [cmd]` — Execute raw command without token optimization

---

## Hook System

Seven hook points enable event-driven automation and system-wide observability:

**SessionStart**
- Initialize browser automation profiles
- Load custom binaries and environment variables
- Validate data source connectivity
- Warm cache with critical data

**PreToolUse**
- Token optimization rewrite (Rust CLI proxy intercepts Bash commands, rewrites for 60-90% token savings)
- Validate tool permissions
- Enrich prompts with recent context

**PostToolUse**
- Log execution time and resource usage
- Extract and persist structured data from tool output
- Trigger downstream notifications

**Stop**
- Auto-emailer sends task summary to configured recipients
- Trigger GitHub backup of local changes
- Archive session context for future reference

**TaskCompleted**
- macOS native notification
- Update database timestamps
- Log completion metrics

**PreCompact / PostCompact**
- Context compression logging and analytics
- Identify frequently accessed information for pre-loading

**UserPromptSubmit**
- Custom prompt enrichment (inject relevant historical context automatically)
- Validate prompt for sensitive information leakage
- Route to appropriate agent based on domain detection

---

## Database Architecture

SQLite serves as the single source of truth across nine functional domains with 40+ tables:

### Sales Intelligence
- `accounts` — Account master records with health status
- `account_health_history` — Time-series snapshots for trend analysis
- `follow_ups` — Next action items with priority and due dates
- `meetings` — Calendar-integrated meeting records
- `email_drafts` — Draft templates and generated drafts

### Credit Card
- `transactions` — Individual purchase records with merchant categorization
- `benefits` — Annual benefits and terms by card
- `rewards` — Reward accrual and redemption tracking
- `payments` — Payment history and due dates
- `scrape_logs` — Debugging and compliance audit trail

### Travel
- `airline_accounts` — Login credentials and family member mapping
- `balances` — Current mileage balances across accounts
- `expiration_schedule` — Miles expiring within 12 months
- `flights` — Booked flights and award flight searches
- `claims` — Disputed transactions and service recovery tracking

### Retirement Planning
- `fund_menu` — Available funds with fees and asset allocation
- `holdings` — Current holdings and cost basis
- `analysis_verdicts` — Fund-level recommendations
- `performance` — Fund returns vs benchmarks
- `scrape_logs` — MFA login timestamps and success/failure

### Job Search
- `opportunities` — Job postings by company and role
- `feedback` — Application outcomes and interview notes
- `preferences` — Fit criteria and target companies
- `outreach` — Application history with dates and personalization

### Tax Planning
- `employers` — W-2 employer records and withholding
- `paychecks` — Detailed paycheck records
- `deduction_estimates` — Estimated tax deductions by category
- `projection_logs` — Year-end projection runs and adjustments

### Personal Finance
- `accounts` — Bank and credit card accounts
- `transactions` — All account transactions
- `institutions` — Connected financial institutions
- `liabilities` — Loans and credit balances
- `investments` — Brokerage holdings and valuations
- `recurring_bills` — Detected recurring charges
- `sync_tracking` — Plaid sync status and last update

### Subscriptions
- `subscriptions` — Service list with monthly cost
- `benefit_usage` — Use-it-or-lose-it benefit logs

### System
- `automation_timestamps` — Last run times for cron jobs
- `session_logs` — Daily session summaries
- `token_usage` — Command-level token accounting (via RTK)

---

## MCP Integrations

Twelve MCP servers provide read and write access to external data sources:

| Server | Purpose |
|--------|---------|
| Gmail | Search, read, and draft emails |
| Google Calendar | Create, modify, and query events |
| Apple Calendar | Calendar event integration |
| Apple Mail | Draft composition and send |
| Apple Messages | SMS and iMessage access |
| Apple Notes | Personal note storage |
| Apple Contacts | Contact lookup and enrichment |
| Apple Maps | Location research and directions |
| Playwright | Full-page browser interaction and scraping |
| Perplexity | Multi-source web research |
| GitHub | Repo management, PRs, issues, and backups |
| Plaid | Bank and credit card sync |
| AppleScript | System automation and macro execution |
| Photos | Photo library access and organization |
| Context7 | Documentation and API reference lookup |

---

## Scheduled Automation

### Cron Jobs (5)
- **Weekly receipt processing** — Extract transaction data from email receipts
- **Weekly task summary emailer** — Compile accomplishments and send digest
- **Weekly GitHub backup** — Version control and disaster recovery
- **Monthly retirement fund analysis** — Fund comparison and recommendation update
- **Daily network health check** — Mesh network monitoring and alerts

### macOS LaunchAgents (4)
- **Weekly system status report** — Disk, memory, and application health
- **Token usage audit** — RTK analytics and cost tracking
- **Trending features research alert** — Monitor for new Claude Code capabilities
- **Trending features email digest** — Weekly digest of product improvements

---

## Token Optimization: RTK (Rust Token Killer)

The system includes a custom Rust-based CLI proxy that intercepts all Bash commands via the PreToolUse hook and rewrites them to strip verbose output. Typical savings: 60-90% tokens per command.

**How it works:**
1. Command submitted (e.g., `git status`)
2. RTK intercepts and analyzes
3. Rewrites to minimal-output variant (e.g., `git status --porcelain`)
4. Executes rewritten command
5. Returns compact output

**Analytics:**
- `rtk gain` — Show cumulative token savings
- `rtk gain --history` — Command usage history with savings per command
- `rtk discover` — Analyze Claude Code history for missed optimization opportunities

**Example savings:**
- `ls -la` → `ls -1` (60% reduction)
- `git log` → `git log --oneline` (75% reduction)
- `npm list` → `npm list --depth=0` (80% reduction)

---

## Design Principles

### Model Tier Optimization
Each agent is assigned a specific model tier based on function. Fast models for latency-critical summary work, best models for quality-critical output like email composition and relationship advice.

### Modular Skill Architecture
Skills are self-contained modules: each owns its database schema, validation logic, scraping logic, and output templates. A new skill can be added without touching existing code.

### Database-First Approach
SQLite is the single source of truth. All tools write to the database first, then generate output. This enables time-series analysis, anomaly detection, and historical comparison.

### Automation Bias
If a task can reasonably be automated (recurring, multi-step, data-driven, time-sensitive), it gets automated. The system asks "Why would I do this manually?" rather than "Should I automate this?"

### Voice Consistency
All written output (emails, briefs, notes) maintains consistent personality and tone. The email agent learns historical voice patterns and applies them consistently across all drafts.

### Self-Improving Loops
The system learns from user feedback: job application outcomes improve matching accuracy, email feedback improves voice modeling, financial projections improve with validation against actual outcomes.

---

## What This Demonstrates

**For Product Teams:** This is what deep adoption looks like in a non-technical user. The system spans multiple problem domains, runs continuously, makes real decisions, and generates business value. The architectural choices reflect how a real user prioritizes: cost (model tier selection), usability (slash commands), reliability (database-first design), and long-term maintainability (modular skills, hooks, automation bias).

**For Sales & Customer Success:** This demonstrates the ceiling of what a motivated user can achieve without hiring engineers. It shows that non-technical professionals can build sophisticated systems when the tools are designed for natural language interaction. It also reveals common adoption patterns: every domain started with a simple script, then evolved into a skill, then became fully integrated with the broader system.

**For Engineering:** This reveals where the platform excels and where friction still exists. The system required custom workarounds in four areas: frequent flyer login (anti-bot detection requires manual intervention), email draft handling (had to use AppleScript compose rather than native draft operations), MFA code extraction (pulled from Messages database), and schedule flexibility (needed custom hook implementations). The token optimization layer exists because verbose output from common tools wastes tokens.

**For Recruitment:** This candidate has demonstrated:
- Deep product fluency across Claude Code, MCP, agents, hooks, plugins, and the broader product
- Ability to design systems that solve real business problems
- Comfort with ambiguity and problem-solving when documentation is thin
- Pragmatic engineering mindset (build what works, optimize later)
- Understanding of cost, reliability, and user experience
- Ability to communicate technical decisions in business language

---

## Getting Started with This Repository

This repo documents the architecture and philosophy. The actual implementation lives in Claude Code settings and the user's local filesystem. To explore:

1. Read through this README to understand the overall structure
2. Review the `/architecture` folder for detailed domain diagrams
3. Check `/examples` for sample skills, commands, and hook implementations
4. See `/database` for the complete SQLite schema

---

## License

This documentation is provided as-is for educational and reference purposes. The code and implementations are personal infrastructure and not available for reuse.

---

**Built with Claude. Powered by persistence. No engineers required.**
