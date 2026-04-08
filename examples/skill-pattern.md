# Skill Pattern: Domain-Specific Automation Modules

## Overview

Skills are self-contained modules that own a domain. Each skill lives in `~/.claude/skills/[name]/` with this structure:

```
~/.claude/skills/[skillname]/
  ├── skill.md              # Main instruction file
  ├── data/
  │   ├── config.yaml       # Thresholds, API endpoints, constants
  │   └── reference/        # Lookup tables, static data
  ├── scripts/
  │   ├── sync.py           # Data fetching/refresh logic
  │   ├── dashboard.py      # Formatting for display
  │   └── utils.py          # Shared functions
  └── README.md             # Quick reference (optional)
```

A skill combines:
- **Instructions** (skill.md) — how to think about the domain
- **Configuration** (YAML) — thresholds, API keys, data sources
- **Scripts** (Python/Bash) — automation and data transformation
- **State** (SQLite, optional) — cached data, historical tracking

Skills are invoked via slash commands. The command framework passes arguments to skill.md, which orchestrates scripts and outputs results.

---

## Example 1: Sales Pipeline Intelligence Skill

**File:** `~/.claude/skills/pipeline/skill.md`

### Slash Command

```
/pipeline [view] [filter]

Views:
  (default)     — Dashboard with status counts and at-risk accounts
  detail        — Full details on every account in each bucket
  red           — Just the RED accounts, sorted by contact gaps
  yellow        — YELLOW accounts, sorted by engagement risk
  
Filters:
  (optional)    — Region, team, product line, etc.
  
Examples:
  /pipeline                 # Full dashboard
  /pipeline detail team:abc  # All accounts owned by team ABC
  /pipeline red             # RED accounts only
```

### Configuration

**File:** `~/.claude/skills/pipeline/data/config.yaml`

```yaml
# Health scoring thresholds
health_scoring:
  engagement_threshold_days: 30
  months_at_risk: 3
  momentum_boost: 
    closed: 25
    advanced: 15
    stalled: -20
  risk_penalties:
    no_contact_3mo: -30
    ceo_change: -20
    budget_cut: -15

# Status bucket thresholds
buckets:
  green: 70-100
  yellow: 40-69
  red: 0-39

# Contact tracking
alerts:
  urgent_if_days_since_contact: 45
  monitor_if_days_since_contact: 30
  stale_if_days_since_contact: 60
```

### Instructions

> You are the pipeline dashboard engine. Your job: show the user exactly what's healthy, what needs attention, and where to focus today.
>
> **Input:** The user specifies a view (dashboard, detail, red, yellow) and optional filters.
>
> **Process:**
> 1. Load all accounts from the database
> 2. Calculate health score for each using the scoring rules (see config.yaml)
> 3. Bucket into RED/YELLOW/GREEN
> 4. Sort by: risk severity (RED first, worst to best), then by recency of contact
> 5. Format for the requested view
>
> **Calculation details:**
> - **Engagement score** = (message count in 30 days * 10) + (meeting count in 60 days * 5). Clamp 0-50.
> - **Momentum score** = Start at 0. Apply bonuses/penalties based on last deal movement. Clamp 0-50.
> - **Health score** = Engagement + Momentum - Risk Penalties. Clamp 0-100.
>
> **Views:**
> 
> **Dashboard** (default):
> ```
> PIPELINE SUMMARY | [Date]
> 
> RED       [count]  •••• [brief % at risk]
> YELLOW    [count]  •••  [brief % at risk]  
> GREEN     [count]  ✓    [brief % healthy]
> 
> [Then list top 5 at-risk accounts]
> ```
>
> **Detail** view:
> ```
> RED ACCOUNTS (sorted by contact gap, worst first)
> • Account Name | Health 25 | Last contact 72 days ago | Status: [deal status]
> 
> YELLOW ACCOUNTS (sorted by health score descending)
> • Account Name | Health 58 | Last contact 18 days ago | Status: [deal status]
> 
> GREEN ACCOUNTS (sorted by health score descending)
> • Account Name | Health 82 | Last contact 5 days ago | Status: [deal status]
> ```
>
> **Red/Yellow only** views:
> Same as detail but filtered to one bucket.
>
> **Constraints:**
> - Never make assumptions about a score if the data is incomplete. If an account has no email data, note it.
> - Sort stability: within a bucket, always sort by contact recency first, then health score. This surfaces "been too long" before "might be weak".
> - Do NOT send emails, do NOT change deal status, do NOT contact anyone.

### Scripts

**File:** `~/.claude/skills/pipeline/scripts/sync.py`

```python
#!/usr/bin/env python3
"""
Fetch and calculate health scores for all accounts.
Called before rendering the dashboard to ensure fresh data.
"""

import sqlite3
import yaml
from datetime import datetime, timedelta

def calculate_health(account, config):
    """Calculate 0-100 health score for an account."""
    
    # Engagement: emails and meetings in recent windows
    engagement = account['emails_30d'] * 10 + account['meetings_60d'] * 5
    engagement = min(max(engagement, 0), 50)  # Clamp 0-50
    
    # Momentum: deal movement bonuses/penalties
    momentum = 0
    if account['last_deal_movement'] == 'closed':
        momentum += 25
    elif account['last_deal_movement'] == 'advanced':
        momentum += 15
    elif account['last_deal_movement'] == 'stalled':
        momentum -= 20
    momentum = min(max(momentum, 0), 50)  # Clamp 0-50
    
    # Risk penalties
    risk = 0
    days_since_contact = (datetime.now() - account['last_contact']).days
    if days_since_contact > config['alerts']['stale_if_days_since_contact']:
        risk -= 30
    elif days_since_contact > config['alerts']['urgent_if_days_since_contact']:
        risk -= 15
    
    if account['ceo_change_flag']:
        risk -= 20
    if account['budget_cut_flag']:
        risk -= 15
    
    health = engagement + momentum - abs(risk)
    return min(max(health, 0), 100)  # Clamp 0-100

def main():
    with open('data/config.yaml') as f:
        config = yaml.safe_load(f)
    
    conn = sqlite3.connect('/path/to/accounts.db')
    accounts = conn.execute('SELECT * FROM accounts').fetchall()
    
    for account in accounts:
        health = calculate_health(account, config)
        conn.execute(
            'UPDATE accounts SET health_score = ?, updated_at = ? WHERE id = ?',
            (health, datetime.now(), account['id'])
        )
    conn.commit()
    print(f"Updated health scores for {len(accounts)} accounts")

if __name__ == '__main__':
    main()
```

### Output Example

```
PIPELINE SUMMARY | Wednesday, April 8

RED       3   ••••  All have >45 days without contact
YELLOW    7   •••   Mixed engagement, at-risk trends
GREEN     12  ✓     Healthy engagement, active deals

AT-RISK (Top 5, contact gaps + weak momentum)
1. Enterprise Corp     | Health 22  | 87 days since contact | Stalled deal
2. Global Systems     | Health 31  | 62 days since contact | Awaiting feedback  
3. Metro Holdings     | Health 38  | 51 days since contact | Quiet
4. Tech Partners      | Health 45  | 39 days since contact | Budget cut flagged
5. Regional Inc       | Health 52  | 28 days since contact | Slow engagement

→ Run /pipeline detail to see all accounts
→ Run /pipeline red to focus on critical accounts only
```

---

## Example 2: Financial Dashboard Skill

**File:** `~/.claude/skills/finance/skill.md`

### Slash Command

```
/finance [view] [period]

Views:
  (default)     — Dashboard: balances, spending, upcoming due dates
  spend         — Spending by category (this month, with budget comparison)
  upcoming      — Payment due dates and alerts
  summary       — Year-to-date financial overview
  sync          — Refresh data from banks (manual trigger)

Period (optional): this-week, this-month, ytd, last-30

Examples:
  /finance                    # Dashboard
  /finance spend this-month   # Spending breakdown for current month
  /finance upcoming           # Due dates and alerts
  /finance sync               # Force refresh from bank
```

### Configuration

**File:** `~/.claude/skills/finance/data/config.yaml`

```yaml
accounts:
  checking:
    name: "Primary Checking"
    type: "depository"
    institution: "Bank A"
  savings:
    name: "Emergency Fund"
    type: "depository"
    institution: "Bank A"
  credit_primary:
    name: "Primary Credit Card"
    type: "credit"
    institution: "Card Issuer A"

spending_budget:
  groceries: 500
  dining: 300
  travel: 1000
  utilities: 250
  
alerts:
  high_utilization_threshold: 0.30  # Alert if credit > 30% of limit
  low_balance_checking: 1000        # Alert if checking < $1K
  payment_due_soon_days: 7          # Alert if payment due within 7 days

sync:
  frequency_minutes: 60
  api_source: "plaid"  # External bank connection
```

### Instructions

> You are the financial dashboard. Your job: show balances, spending patterns, and upcoming obligations in a way that's actionable and honest.
>
> **Process:**
> 1. Query latest balances from bank sync (via Plaid MCP)
> 2. Fetch transactions for the requested period
> 3. Categorize spending (use merchant + description matching)
> 4. Compare against budget thresholds
> 5. Flag alerts (high utilization, low balances, due dates)
> 6. Format for the requested view
>
> **Views:**
>
> **Dashboard** (default):
> ```
> ACCOUNTS
> Checking: $[amount]  [green if > $1K]
> Savings:  $[amount]  
> Credit:   $[amount] of $[limit] ([% utilization])
> 
> THIS MONTH SPENDING
> Groceries:   $[spend] / $[budget]  [green if under, red if over]
> Dining:      $[spend] / $[budget]
> Travel:      $[spend] / $[budget]
> Utilities:   $[spend] / $[budget]
> 
> ALERTS
> [List any: high utilization, low balance, upcoming due dates, budget overages]
> ```
>
> **Spend** view:
> ```
> SPENDING BREAKDOWN | [Period]
> 
> Groceries        $245   ████░░░░░░  49% of $500 budget
> Dining           $112   ██░░░░░░░░  37% of $300 budget
> Travel           $800   ████████░░  80% of $1000 budget
> Utilities        $120   ████░░░░░░  48% of $250 budget
> Other            $56    
> 
> TOTAL            $1,333
> On budget: 4/5 categories
> ```
>
> **Upcoming** view:
> ```
> PAYMENT DUE DATES
> → 4 days   Primary Credit Card  $[amount]  (due date: [date])
> → 12 days  Loan Payment         $[amount]  (due date: [date])
> ○ 32 days  Utility Bill         $[amount]  (estimated, due date: [date])
> ```
>
> **Constraints:**
> - Do NOT make transactions, do NOT change account settings, do NOT auto-pay anything.
> - Spending data is always 1-2 days behind; note this in output.
> - If an alert exists, always surface it (high util, low balance, overdue).
> - Never invent account data. If a balance is unknown, say "no data" not "likely $0".

### Scripts

**File:** `~/.claude/skills/finance/scripts/dashboard.py`

```python
#!/usr/bin/env python3
"""
Format account balances and spending for display.
"""

import yaml
from datetime import datetime

def format_dashboard(balances, spending, alerts, config):
    """Generate dashboard view."""
    
    output = []
    output.append(f"ACCOUNTS | {datetime.now().strftime('%A, %B %d')}")
    output.append("")
    
    # Account balances
    for acct_id, balance in balances.items():
        acct_config = config['accounts'][acct_id]
        name = acct_config['name']
        
        # Add alert indicator if balance is low
        indicator = " ✓" if balance['amount'] > config['alerts']['low_balance_checking'] else " ⚠"
        output.append(f"{name:20} ${balance['amount']:>10,.2f}{indicator}")
    
    output.append("")
    output.append("THIS MONTH SPENDING")
    
    # Spending vs budget
    for category, budget in config['spending_budget'].items():
        spent = spending.get(category, 0)
        pct = (spent / budget * 100) if budget > 0 else 0
        status = "✓" if spent <= budget else "⚠"
        bar = "█" * int(pct / 10) + "░" * (10 - int(pct / 10))
        
        output.append(
            f"{category:15} ${spent:>7,.0f} / ${budget:>7,.0f}  {bar}  {status}"
        )
    
    output.append("")
    if alerts:
        output.append("ALERTS")
        for alert in alerts:
            output.append(f"  {alert}")
    
    return "\n".join(output)

if __name__ == '__main__':
    with open('data/config.yaml') as f:
        config = yaml.safe_load(f)
    print("Dashboard formatter ready")
```

### Output Example

```
ACCOUNTS | Wednesday, April 8

Checking              $3,450 ✓
Savings              $15,200 ✓
Credit               $2,840 of $10,000  (28%)  ✓

THIS MONTH SPENDING
Groceries               $245 / $500      ████░░░░░░  ✓
Dining                  $112 / $300      ██░░░░░░░░  ✓
Travel                  $800 / $1,000    ████████░░  ✓
Utilities               $120 / $250      ████░░░░░░  ✓

ALERTS
→ Payment due in 4 days: $2,840 on primary credit card
```

---

## Key Design Decisions

### Self-Contained Ownership

Each skill owns its domain end-to-end: data fetching, business logic, output formatting. This means:
- Skill can be updated without affecting others
- Debugging stays local
- Easy to grant/revoke access (user trusts one skill more than another)

### Configuration via YAML

Thresholds, API endpoints, budget numbers — these live in YAML, not hardcoded. When priorities change ("increase spending budget to $400"), you edit YAML, not Python. Non-technical users can do this.

### Scripts for Heavy Lifting

Complex calculations (health scoring), API calls (bank sync), data transformation (categorizing transactions) live in Python/Bash scripts. The skill.md file orchestrates but doesn't compute — it stays readable and instruction-focused.

### State via SQLite (Optional)

Some skills cache data locally (accounts table in pipeline skill, historical spending in finance skill). This enables:
- Fast dashboards (no API call needed every time)
- Historical trending
- Offline operation when banks are down

### Subcommand Pattern

Every domain has a no-args default view ("dashboard") and specialized views ("red", "detail", "spend", "upcoming"). Users learn one pattern: `/domain` shows the overview, `/domain [view]` shows details.

This scales: as new views are needed, the command interface doesn't change.
