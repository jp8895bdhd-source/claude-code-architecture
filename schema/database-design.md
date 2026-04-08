# Database Architecture

## Overview

This document describes the database layer of a Claude Code personal automation platform. The system uses a single SQLite database as its central data store, queried directly by agents and skills via the MCP SQLite integration.

- **Data Store:** SQLite (single-file, serverless)
- **Table Count:** 40+ tables organized across 9 functional domains
- **Access Pattern:** Read/write via MCP SQLite, with agents and skills executing complex queries and updates
- **Design Philosophy:** Schema optimized for fast lookups, trend analysis, and intelligent recommendations built on user feedback

---

## Domain 1: Sales Intelligence (9 tables)

This domain tracks account health, deal momentum, and engagement patterns. It forms the backbone of relationship intelligence across a portfolio.

### account_health

Primary scoring table for real-time account status tracking and prioritization.

| Column | Type | Purpose |
|--------|------|---------|
| account_slug | TEXT | Unique account identifier (lowercase, hyphen-separated) |
| status | TEXT | RED / YELLOW / GREEN health status |
| health_score | REAL | Composite score (0-100), weighted average of component scores |
| recency_score | INTEGER | Days since last meaningful contact, normalized to 0-100 scale |
| momentum_score | INTEGER | Deal velocity indicator: positive=moving forward, negative=stalled (0-100) |
| engagement_score | INTEGER | Interaction frequency and quality assessment (0-100) |
| fulfillment_score | INTEGER | Delivery on commitments and follow-through quality (0-100) |
| last_updated | TEXT | ISO 8601 timestamp of last calculation |

**Purpose:** Agents running `/pipeline` commands consume this table for rapid status display. Thresholds (RED < 40, YELLOW < 70) drive alert urgency.

---

### account_health_history

Time-series snapshots of health scores, enabling trend analysis and anomaly detection.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_slug | TEXT | FK to account_health |
| health_score | REAL | Score at snapshot time |
| recency_score | INTEGER | Recency component at time of snapshot |
| momentum_score | INTEGER | Momentum component at time of snapshot |
| engagement_score | INTEGER | Engagement component at time of snapshot |
| fulfillment_score | INTEGER | Fulfillment component at time of snapshot |
| snapshot_date | TEXT | ISO 8601 date of this snapshot (usually daily) |

**Purpose:** Tracks "is this account improving or deteriorating?" Useful for predicting churn risk or identifying dormant accounts awakening.

---

### followups

Urgency-ranked pending actions tied to accounts, extracted from email, meetings, and manual entry.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_slug | TEXT | FK to account_health |
| action_type | TEXT | email_response / meeting / demo / proposal / payment_follow / other |
| action_description | TEXT | What needs to happen (extracted or user-written) |
| urgency | TEXT | critical / high / medium / low |
| due_date | TEXT | ISO 8601 due date |
| created_at | TEXT | When this follow-up was created |
| created_by | TEXT | source: email / meeting_extraction / manual_entry |
| status | TEXT | pending / completed / skipped |
| completed_at | TEXT | ISO 8601 timestamp if completed (null otherwise) |
| notes | TEXT | Additional context or completion details |

**Purpose:** Agents use this for the `/followup` command to surface overdue or high-urgency items. Completed rows are kept for audit and to understand completion velocity per account.

---

### meetings

Meeting records with extracted context, commitments, and next steps.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_slug | TEXT | FK to account_health |
| meeting_date | TEXT | ISO 8601 date and time |
| meeting_type | TEXT | discovery / demo / negotiation / update / stakeholder_alignment / other |
| attendees | TEXT | Comma-separated list of participant names (no PII) |
| summary | TEXT | Key discussion points extracted from notes or email |
| takeaways | TEXT | Specific learnings or strategic insights from the conversation |
| next_steps | TEXT | Agreed-upon next actions |
| commitments_made | TEXT | Specific promises or deliverables from your side |
| sentiment | TEXT | positive / neutral / negative (captured from tone and outcomes) |
| created_at | TEXT | When this record was created |

**Purpose:** Supports account context retrieval via `/account` command. Also feeds into engagement_score calculations (frequency and quality).

---

### drafts

Generated email drafts with feedback tracking and performance metrics.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_slug | TEXT | FK to account_health |
| draft_type | TEXT | outreach / response / followup / proposal / other |
| recipient | TEXT | Recipient email address (hashed for privacy) |
| subject_line | TEXT | Email subject |
| body | TEXT | Draft content |
| generated_at | TEXT | When the draft was created by the agent |
| sent | BOOLEAN | Whether this draft was actually sent |
| sent_at | TEXT | ISO 8601 timestamp if sent (null otherwise) |
| response_received | BOOLEAN | Whether a response came back |
| response_date | TEXT | ISO 8601 timestamp of response |
| response_sentiment | TEXT | positive / neutral / negative (inferred from reply) |
| feedback | TEXT | User feedback on draft quality ("too formal", "needs data", etc.) |

**Purpose:** Tracks which draft styles generate responses. Enables A/B learning over time. Feedback column feeds preference learning algorithms.

---

### emails_received

Inbound email log with context and urgency classification.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_slug | TEXT | FK to account_health |
| from_address | TEXT | Sender email (hashed) |
| subject | TEXT | Email subject line |
| received_at | TEXT | ISO 8601 timestamp |
| urgency | TEXT | auto-classified: critical / high / medium / low / promotional |
| action_required | BOOLEAN | Whether this email requires follow-up |
| summary | TEXT | AI-extracted summary of email content |
| message_id | TEXT | Unique message ID for threading |
| thread_id | TEXT | Group multiple emails in a conversation |

**Purpose:** Enables email triage and urgency alert generation. Feeds into recency_score calculation.

---

### commitments

Explicit commitments made to accounts (extracted from emails, meetings, and chat).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_slug | TEXT | FK to account_health |
| commitment_text | TEXT | What was promised |
| commitment_date | TEXT | ISO 8601 when promise was made |
| due_date | TEXT | ISO 8601 promised delivery date |
| status | TEXT | pending / delivered / delayed / not_met |
| delivery_date | TEXT | ISO 8601 when actually delivered (null if pending) |
| notes | TEXT | Context or explanation |

**Purpose:** Directly impacts fulfillment_score. Missing commitments tank health scores and trigger high-urgency alerts.

---

### opportunity_tracking

Pipeline opportunities and deal stage progression.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_slug | TEXT | FK to account_health |
| opportunity_name | TEXT | Deal identifier (project name or generic "Opportunity 1") |
| stage | TEXT | prospect / qualification / demo / proposal / negotiation / won / lost |
| est_value | REAL | Estimated opportunity value (null if uncertain) |
| probability | INTEGER | Win probability 0-100 |
| created_at | TEXT | ISO 8601 when opportunity was created |
| last_stage_change | TEXT | ISO 8601 of most recent stage update |
| notes | TEXT | Deal context or blockers |

**Purpose:** Feeds into momentum_score. Rapid stage progression = positive momentum; stalled deals = negative.

---

### conversation_history

Persistent, queryable conversation logs for context retrieval by agents.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_slug | TEXT | FK to account_health |
| date | TEXT | ISO 8601 date of conversation |
| medium | TEXT | email / call / meeting / video / chat / other |
| topic | TEXT | Main subject discussed |
| participants | TEXT | Comma-separated participant identifiers |
| summary | TEXT | Conversation summary |
| key_insights | TEXT | Learnings that impact strategy |

**Purpose:** Provides continuity. When an agent retrieves account context for a call, it pulls recent conversations to avoid repeat questions.

---

## Domain 2: Credit Card Tracking (8 tables)

This domain automates credit card spend analysis, benefit tracking, and rewards optimization. Built around automated scraping (with manual fallback for MFA-heavy platforms).

### transactions

Complete transaction ledger across all tracked cards.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| card_id | TEXT | Card identifier (masked: "****1234") |
| transaction_date | TEXT | ISO 8601 date |
| merchant | TEXT | Merchant name |
| category | TEXT | Standardized category (dining, gas, groceries, travel, other) |
| amount | REAL | Transaction amount in USD |
| status | TEXT | posted / pending |
| description | TEXT | Merchant description from statement |
| posting_date | TEXT | ISO 8601 when transaction posted (for pending → posted tracking) |

**Purpose:** Core data for spending analysis. Queries group by category/merchant/date for spending summaries and anomaly detection.

---

### benefits_usage

Tracking utilization of card-specific perks: monthly credits, subscription reimbursements, travel protections, lounge access, etc.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| card_id | TEXT | FK to card (benefits are card-specific) |
| benefit_name | TEXT | e.g., "dining credit", "travel credit", "streaming reimbursement" |
| period | TEXT | YYYY-MM (month/year of benefit window) |
| claimed | BOOLEAN | Whether the benefit was used in this period |
| claimed_date | TEXT | ISO 8601 when claimed (null if not claimed) |
| benefit_value | REAL | Dollar value of the benefit |
| notes | TEXT | How it was used (e.g., "StockX purchase" for credit) |

**Purpose:** Alerts for "use it or lose it" benefits expiring at month-end. Answers: "Am I getting full value from this card?"

---

### benefits_dismissed

User-suppressed benefit reminders (prevents alert spam for intentionally skipped benefits).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| card_id | TEXT | FK to card |
| benefit_name | TEXT | Which benefit was dismissed |
| dismissed_at | TEXT | ISO 8601 timestamp |
| dismissed_until | TEXT | ISO 8601 when to resume nagging (usually next month) |
| reason | TEXT | User explanation (e.g., "don't use this service") |

**Purpose:** Reduces notification fatigue. Prevents the same "you have an unclaimed credit" alert appearing repeatedly.

---

### rewards

Points, miles, and status tracking across multiple reward programs per card.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| card_id | TEXT | FK to card |
| reward_type | TEXT | points / miles / cashback / status_points |
| balance | REAL | Current balance |
| currency | TEXT | USD / miles / points |
| last_updated | TEXT | ISO 8601 timestamp of last balance update |
| expiration_date | TEXT | ISO 8601 date if rewards expire (null if no expiration) |
| scraped_from | TEXT | URL or source where data was retrieved |

**Purpose:** Enables reward health monitoring and expiration alerts. Stores multiple reward types per card.

---

### payments

Payment history and due date tracking for account management.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| card_id | TEXT | FK to card |
| due_date | TEXT | ISO 8601 date payment is due |
| statement_balance | REAL | Total balance on this billing cycle |
| minimum_payment | REAL | Minimum payment required |
| payment_made | BOOLEAN | Whether payment has been made for this cycle |
| payment_date | TEXT | ISO 8601 when payment was submitted (null if not paid) |
| payment_amount | REAL | Amount paid (null if not paid) |
| payment_method | TEXT | auto-pay / manual_online / check / other |

**Purpose:** Ensures no missed payments. `/upcoming_payments` command queries due_date fields across all cards.

---

### account_meta

Card-level metadata: balances, credit limits, account status.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| card_id | TEXT | Unique card identifier |
| cardholder | TEXT | Family member name (no PII beyond what card issuer shows) |
| card_network | TEXT | Visa / Mastercard / Amex / Discover |
| issuer | TEXT | Bank/issuer name |
| card_type | TEXT | Personal / Business |
| current_balance | REAL | Current statement balance |
| available_credit | REAL | Credit available to use |
| credit_limit | REAL | Total credit limit |
| apr | REAL | Annual percentage rate |
| account_status | TEXT | active / closed / frozen |
| last_four | TEXT | Last 4 digits of card number |
| opened_at | TEXT | ISO 8601 account open date |

**Purpose:** Account health monitoring. Flags high balances relative to limit, closed accounts, frozen accounts.

---

### scrape_log

Execution history for automated card data scrapers (with success/failure detail).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| card_id | TEXT | FK to card |
| scrape_started_at | TEXT | ISO 8601 start time |
| scrape_completed_at | TEXT | ISO 8601 end time (null if in progress) |
| status | TEXT | success / failure / mfa_required / session_expired |
| rows_imported | INTEGER | Number of transactions/balance updates imported |
| error_message | TEXT | Failure reason (if status != success) |
| requires_manual_login | BOOLEAN | Whether scraper hit MFA and needs human intervention |
| scrape_method | TEXT | automated / manual_screenshot / manual_download |

**Purpose:** Debugging automation failures. Identifies when scrapers need credential refresh or manual intervention.

---

## Domain 3: Travel / Frequent Flyer (7 tables)

Multi-airline frequent flyer program tracking with expiration alerts and booking optimization.

### accounts

Airline loyalty program membership records.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| airline_code | TEXT | Standard airline code (e.g., "UA", "DL", "AA") |
| family_member | TEXT | Family member name |
| membership_id | TEXT | Loyalty program number |
| elite_tier | TEXT | Status tier (e.g., Silver, Gold, Platinum, Diamond) |
| elite_expiration | TEXT | ISO 8601 when elite status expires |
| tsa_known_traveler | TEXT | Known Traveler Number if available |
| account_status | TEXT | active / closed / dormant |

**Purpose:** Enables multi-person household tracking. Queries pull all family members' accounts.

---

### balances

Snapshot-based balance tracking for trend analysis and expiration prediction.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_id | INTEGER | FK to accounts |
| miles_balance | INTEGER | Current miles balance |
| snapshot_date | TEXT | ISO 8601 timestamp of when balance was captured |
| scraped_from | TEXT | Source URL for audit trail |

**Purpose:** Each scrape creates a new row, not an update. Enables trend analysis: "Are my miles being redeemed faster than earned?" Supports expiration prediction based on historical burn rate.

---

### expiration

Expiring miles schedule with urgency alerts.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_id | INTEGER | FK to accounts |
| expiration_date | TEXT | ISO 8601 date miles expire |
| miles_expiring | INTEGER | Number of miles at risk |
| alert_sent | BOOLEAN | Whether user has been notified |
| alert_sent_at | TEXT | ISO 8601 when alert was sent |
| resolution | TEXT | redeemed / forfeited / reinstated |
| notes | TEXT | How miles were handled |

**Purpose:** Prevents expiration forfeitures. Alerts trigger 30/60/90 days before expiration.

---

### flights

Upcoming flight bookings with confirmation codes and metadata.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_id | INTEGER | FK to accounts |
| confirmation_code | TEXT | Booking reference |
| route | TEXT | Origin/destination (e.g., "LAX-JFK") |
| departure_date | TEXT | ISO 8601 date and time |
| cabin_class | TEXT | economy / premium_economy / business / first |
| airline_code | TEXT | Carrier |
| booked_via | TEXT | miles_award / revenue_ticket / other |
| booking_date | TEXT | ISO 8601 when booked |
| notes | TEXT | Seat assignments, special requests, seat upgrades |

**Purpose:** Trip planning. Enables context around "use these miles before they expire" recommendations.

---

### claims

Tracking for missing miles claims (manual adjustment requests with resolution status).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_id | INTEGER | FK to accounts |
| flight_date | TEXT | ISO 8601 of the flight in question |
| missing_miles | INTEGER | Miles not credited |
| claim_submitted_date | TEXT | ISO 8601 when claim was filed |
| claim_status | TEXT | pending / approved / denied / investigation |
| resolution_miles | INTEGER | Miles awarded (if approved) |
| resolution_date | TEXT | ISO 8601 when claim was resolved |
| notes | TEXT | Airline correspondence or internal notes |

**Purpose:** Tracks manual recovery efforts. Helps identify which airlines honor claims readily vs. resist them.

---

### scrape_log

Airline portal scraper execution history (includes auth failures, MFA challenges).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_id | INTEGER | FK to accounts |
| scrape_started_at | TEXT | ISO 8601 start time |
| scrape_completed_at | TEXT | ISO 8601 end time (null if in progress or failed) |
| status | TEXT | success / failure / mfa_required / website_changed |
| miles_captured | INTEGER | Rows inserted into balances table |
| error_message | TEXT | Failure reason |
| requires_manual_login | BOOLEAN | MFA or CAPTCHA blocked automation |
| browser_used | TEXT | headed / headless (headless usually fails on MFA) |

**Purpose:** Diagnostics for scraper failures. Identifies when manual login is necessary vs. bot-blocking prevention.

---

## Domain 4: Retirement Fund Analysis (5 tables)

Quarterly/monthly fund performance tracking and rebalancing recommendations.

### fund_menu

Investment options available in the retirement plan.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| fund_ticker | TEXT | Fund identifier (e.g., "VTSAX", "BND") |
| fund_name | TEXT | Full fund name |
| asset_class | TEXT | US equities / international / fixed income / other |
| expense_ratio | REAL | Annual fee as percentage (e.g., 0.03 for 0.03%) |
| ytd_return | REAL | Year-to-date return percentage |
| one_year_return | REAL | 1-year trailing return |
| five_year_return | REAL | 5-year trailing return |
| inception_date | TEXT | ISO 8601 when fund was created |
| last_updated | TEXT | ISO 8601 of latest performance data |

**Purpose:** Reference table for all available investment options. Updated monthly/quarterly to capture performance changes.

---

### holdings

Current portfolio allocation and position-level data.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| fund_ticker | TEXT | FK to fund_menu |
| shares | REAL | Number of shares held |
| share_price | REAL | Current share price |
| position_value | REAL | shares × share_price |
| allocation_percentage | REAL | Position value as % of total portfolio |
| last_scraped | TEXT | ISO 8601 when this holding was last updated from platform |

**Purpose:** Enables quick portfolio health queries. Feeds into rebalancing recommendations.

---

### analysis

Fund verdict engine output (hold, watch, swap recommendations).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| fund_ticker | TEXT | FK to fund_menu |
| verdict | TEXT | HOLD / WATCH / SWAP |
| reasoning | TEXT | Why this verdict (e.g., "expense ratio too high", "strong 5yr return", "lagging benchmark") |
| suggested_alternative | TEXT | If SWAP, which fund to replace this with |
| confidence | INTEGER | Confidence level 1-100 |
| analyzed_at | TEXT | ISO 8601 when analysis was performed |
| analyst_notes | TEXT | Additional context for user review |

**Purpose:** Supports `/roth` command. Verdicts are computed monthly, compared to previous month to highlight changes.

---

### performance

Historical tracking data for graphing and trend analysis.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| fund_ticker | TEXT | FK to fund_menu |
| report_date | TEXT | ISO 8601 (typically month-end) |
| one_month_return | REAL | Return over last month |
| three_month_return | REAL | Return over last 3 months |
| one_year_return | REAL | Return over last year |
| benchmark_comparison | REAL | Over/under benchmark return |

**Purpose:** Enables trend visualization. "Is this fund underperforming lately?" queries.

---

### scrape_log

Retirement platform scraper execution history (tracks MFA status, platform availability).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| scrape_started_at | TEXT | ISO 8601 start time |
| scrape_completed_at | TEXT | ISO 8601 end time |
| status | TEXT | success / failure / mfa_blocked / website_unavailable |
| rows_imported | INTEGER | Number of holdings/prices updated |
| error_message | TEXT | Failure reason |
| mfa_status | TEXT | none / required / completed / failed |
| last_successful_scrape | TEXT | ISO 8601 of most recent successful scrape |

**Purpose:** Identifies authentication issues. Flags when analysis is stale (hasn't updated in 30+ days).

---

## Domain 5: Job Search Intelligence (4 tables)

Job opportunity surfacing, ranking, and application tracking with preference learning.

### opportunities

Available job opportunities with fit scoring and source tracking.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| company_name | TEXT | Hiring company |
| job_title | TEXT | Position title |
| location | TEXT | Location or "Remote" |
| work_location | TEXT | on-site / hybrid / remote (verified from posting, not inferred) |
| posting_url | TEXT | Application link |
| industry | TEXT | Categorized industry |
| fit_score | INTEGER | Algorithmic fit 1-10 (computed from preference learning) |
| posted_date | TEXT | ISO 8601 when opportunity appeared |
| application_status | TEXT | new / reviewing / applied / interviewing / offered / rejected / withdrawn |
| date_applied | TEXT | ISO 8601 when application submitted (null if not applied) |
| resume_variant_used | TEXT | Which resume template was used (e.g., "product-focused-v2") |
| notes | TEXT | Key fit points: "strong on X, weak on Y" |
| posting_source | TEXT | LinkedIn / Greenhouse / Indeed / direct / recruiter / other |

**Purpose:** Core table for job search automation. Fit scores update as preference model improves.

---

### feedback

User ratings on surfaced opportunities (feeds preference learning model).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| opportunity_id | INTEGER | FK to opportunities |
| rating | TEXT | yes / maybe / no / block |
| rated_at | TEXT | ISO 8601 when user rated this |
| reason | TEXT | User explanation (e.g., "not SaaS", "location red flag", "perfect fit") |

**Purpose:** Training data for fit_score model. "Block" ratings prevent similar opportunities from surfacing.

---

### preferences

Learned patterns from feedback history (drives fit scoring and filtering).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| pattern_type | TEXT | company_name / industry / location / title_keyword / company_size |
| pattern_value | TEXT | The specific pattern (e.g., "Healthcare", "San Francisco", "Head of") |
| sentiment | TEXT | positive / negative / blocked |
| weight | REAL | How strongly this pattern affects fit scoring (higher = stronger signal) |
| confidence | REAL | How confident is the system in this pattern (based on feedback frequency) |
| learned_from_count | INTEGER | Number of feedback instances supporting this pattern |

**Purpose:** Dynamically adjusts opportunity scoring. High-weight patterns heavily influence which jobs surface.

---

### outreach

Application tracking and follow-up status per opportunity.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | PRIMARY key |
| opportunity_id | INTEGER | FK to opportunities |
| application_submitted | BOOLEAN | Whether application was completed |
| submitted_date | TEXT | ISO 8601 when submitted |
| submitted_to | TEXT | Hiring email/portal where application went |
| follow_up_sent | BOOLEAN | Whether follow-up/inquiry email was sent |
| follow_up_date | TEXT | ISO 8601 when follow-up was sent |
| response_received | BOOLEAN | Response to application/follow-up |
| response_date | TEXT | ISO 8601 of response |
| response_type | TEXT | rejection / interview_scheduled / under_review / auto_confirmation |
| interview_scheduled | BOOLEAN | Interview booked |
| interview_date | TEXT | ISO 8601 of scheduled interview |
| outcome | TEXT | pending / offer / rejection / withdrew |
| notes | TEXT | Feedback from recruiters, interview results, etc. |

**Purpose:** Tracks application lifecycle. Identifies which sourcing channels yield interviews and offers.

---

## Domain 6: Tax Projection (4 tables)

Multi-person household wage tracking, withholding verification, and year-end tax liability projection.

### employers

Employer records for each household member.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| household_member | TEXT | Family member name |
| employer_name | TEXT | Company name |
| pay_frequency | TEXT | weekly / biweekly / semimonthly / monthly |
| employment_start | TEXT | ISO 8601 employment start date |
| employment_end | TEXT | ISO 8601 end date (null if currently employed) |
| gross_annual | REAL | Estimated annual gross (for verification purposes) |

**Purpose:** Reference table for paycheck parsing. Multiple employers per person supported.

---

### paychecks

Individual paycheck records with comprehensive deduction detail.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| employer_id | INTEGER | FK to employers |
| pay_date | TEXT | ISO 8601 when payment was received |
| pay_period_end | TEXT | ISO 8601 end date of pay period |
| gross_pay | REAL | Gross earnings this period |
| federal_withheld | REAL | Federal income tax withheld |
| state_withheld | REAL | State income tax withheld (if applicable) |
| social_security_withheld | REAL | FICA SS withholding (deducted from check) |
| medicare_withheld | REAL | FICA Medicare withholding (deducted from check) |
| pretax_401k | REAL | Traditional 401(k) contribution (pre-tax, reduces W-2 wages) |
| roth_401k | REAL | Roth 401(k) contribution (after-tax, still reportable) |
| roth_ira_contribution | REAL | Roth IRA contribution if tracked here |
| health_dental_vision | REAL | Insurance premiums (pre-tax) |
| fsa | REAL | Flexible Spending Account contribution |
| hsa | REAL | Health Savings Account contribution |
| net_pay | REAL | Take-home amount |
| paycheck_number | INTEGER | Sequence number for the year |

**Purpose:** Core data for tax projection. Cumulative sums per category drive year-end estimates. Enables "verify withholding is correct" checks.

---

### deduction_estimates

Itemized deduction projections for year-end planning and tax liability forecasting.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| deduction_type | TEXT | mortgage_interest / property_tax / charitable / medical / student_loan_interest / other |
| amount | REAL | Estimated total for the year |
| confidence | TEXT | high / medium / low (based on available data) |
| notes | TEXT | Source or assumption |
| projected_for_year | INTEGER | Tax year this is for |

**Purpose:** Supports "will we itemize or take standard deduction?" analysis. Helps predict tax liability.

---

### projection_log

Year-end tax liability projections with gap analysis and W-4 adjustment recommendations.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| projection_date | TEXT | ISO 8601 when projection was calculated |
| projected_tax_year | INTEGER | Tax year being projected (e.g., 2024) |
| household_member | TEXT | Which family member this projection is for |
| ytd_gross | REAL | Year-to-date gross earnings |
| ytd_federal_withheld | REAL | Year-to-date federal tax withheld |
| estimated_total_tax_liability | REAL | Estimated total federal tax owed |
| projected_refund_or_owed | REAL | Positive = refund, negative = owed |
| withholding_gap | REAL | Variance from proper withholding ("should have withheld $X more/less") |
| w4_recommendation | TEXT | Suggested W-4 changes to correct withholding |
| projection_confidence | TEXT | high / medium / low (depends on data completeness) |
| notes | TEXT | Additional context (bonus expected, spouse income changes, etc.) |

**Purpose:** Proactive tax planning. Identifies overpayment/underpayment risks before April.

---

## Domain 7: Personal Finance / Banking (8 tables)

Aggregated financial data from Plaid integration, supporting account monitoring, spending analysis, and net worth tracking.

### accounts

Linked bank/credit/investment accounts from Plaid.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_id_plaid | TEXT | Plaid account identifier (for API references) |
| institution_id | TEXT | Plaid institution ID |
| household_member | TEXT | Which family member owns this account |
| account_name | TEXT | Display name (e.g., "Checking", "Emergency Fund") |
| account_type | TEXT | depository / credit / investment / loan |
| account_subtype | TEXT | checking / savings / money_market / credit_card / brokerage / 401k / student_loan / mortgage |
| current_balance | REAL | Latest balance snapshot |
| last_balance_update | TEXT | ISO 8601 of most recent Plaid sync |
| account_status | TEXT | active / closed / dormant |

**Purpose:** Account registry. Enables multi-account queries and per-account filtering.

---

### transactions

All financial transactions aggregated from linked accounts via Plaid.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_id | INTEGER | FK to accounts |
| transaction_id_plaid | TEXT | Plaid transaction ID (for deduplication) |
| transaction_date | TEXT | ISO 8601 date of transaction |
| merchant | TEXT | Merchant name (normalized by Plaid) |
| merchant_category | TEXT | Plaid standard category (e.g., "FOOD_AND_DRINK") |
| custom_category | TEXT | User-overridden category (for refinements) |
| amount | REAL | Transaction amount |
| direction | TEXT | debit / credit (outflow / inflow) |
| pending | BOOLEAN | Whether transaction is still pending |
| authorized_date | TEXT | ISO 8601 when transaction was authorized (if available) |

**Purpose:** Spending queries, monthly summaries, anomaly detection.

---

### institutions

Bank/credit union metadata for account context.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| institution_id_plaid | TEXT | Plaid institution ID |
| institution_name | TEXT | Official institution name |
| institution_url | TEXT | Website URL |
| logo_url | TEXT | Institution logo URL (for UI) |

**Purpose:** Lookup table for display names and logos.

---

### liabilities

Loans, debts, and credit lines with payment status.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_id | INTEGER | FK to accounts (the loan/credit account) |
| liability_type | TEXT | student_loan / mortgage / personal_loan / auto_loan / line_of_credit |
| principal_balance | REAL | Outstanding balance |
| interest_rate | REAL | APR or interest rate |
| monthly_payment | REAL | Required monthly payment (if applicable) |
| next_payment_date | TEXT | ISO 8601 of next payment due |
| last_payment_date | TEXT | ISO 8601 of most recent payment |
| loan_term_months | INTEGER | Length of loan in months (null if open-ended) |
| months_remaining | INTEGER | Calculated from loan_term and start date |

**Purpose:** Debt tracking and payment schedule monitoring.

---

### investments

Brokerage and retirement portfolio holdings.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_id | INTEGER | FK to accounts (the investment account) |
| ticker | TEXT | Security ticker or identifier |
| security_name | TEXT | Full security name |
| quantity | REAL | Number of shares held |
| price | REAL | Most recent market price |
| position_value | REAL | quantity × price |
| cost_basis | REAL | Total amount invested (for gain/loss calculation) |
| unrealized_gain_loss | REAL | position_value - cost_basis |
| last_updated | TEXT | ISO 8601 of latest price |

**Purpose:** Net worth calculation. Position-level returns analysis.

---

### recurring

Auto-detected recurring bills and subscriptions from transaction patterns.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| merchant | TEXT | Recurring merchant name |
| frequency | TEXT | weekly / biweekly / monthly / quarterly / annual |
| estimated_next_date | TEXT | ISO 8601 predicted next charge date |
| average_amount | REAL | Average charge amount based on history |
| last_amount | REAL | Most recent charge |
| last_charge_date | TEXT | ISO 8601 of most recent charge |
| confidence | INTEGER | Plaid confidence in detection 1-100 |
| is_active | BOOLEAN | Whether transaction still recurring |

**Purpose:** Subscription tracking, budget forecasting, and "use it or lose it" reminder generation.

---

### sync_cursors

Incremental sync state for cursor-based pagination (enables efficient re-syncing).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| account_id | INTEGER | FK to accounts |
| cursor | TEXT | Plaid cursor token for next incremental update |
| last_synced | TEXT | ISO 8601 timestamp of most recent sync |
| next_available_cursor | TEXT | Plaid's next cursor (for next sync cycle) |

**Purpose:** Plaid sync optimization. Enables "pull only new transactions" rather than full refresh each time.

---

### sync_log

Sync execution history with detailed status tracking.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| sync_started_at | TEXT | ISO 8601 start of sync |
| sync_completed_at | TEXT | ISO 8601 end of sync (null if in progress) |
| status | TEXT | success / failure / partial |
| accounts_synced | INTEGER | Number of accounts updated |
| transactions_imported | INTEGER | Number of transaction rows created |
| errors | TEXT | Detailed error messages if status != success |

**Purpose:** Sync health monitoring. Identifies when Plaid is unreachable or returning errors.

---

## Domain 8: Subscriptions (2 tables)

SaaS subscription and recurring service tracking with cost analysis and benefit utilization.

### subscriptions

Active subscription list with renewal dates and cost tracking.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| service_name | TEXT | Service name |
| category | TEXT | streaming / productivity / cloud_storage / fitness / news / other |
| cost_per_month | REAL | Monthly cost (if billed monthly) |
| billing_frequency | TEXT | monthly / quarterly / annual |
| renewal_date | TEXT | ISO 8601 of next billing date |
| payment_method | TEXT | credit_card / bank_account / app_store |
| status | TEXT | active / cancelled / paused |
| household_member | TEXT | Who uses it |
| cancellation_deadline | TEXT | ISO 8601 date to cancel to avoid next charge |
| notes | TEXT | Login details, shared status, etc. |

**Purpose:** Monthly cost monitoring, "too many subscriptions?" analysis, renewal date alerts.

---

### benefit_usage

Monthly utilization logging for use-it-or-lose-it benefit tracking.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| subscription_id | INTEGER | FK to subscriptions |
| month | TEXT | YYYY-MM of tracking month |
| used | BOOLEAN | Whether service was actively used in this month |
| usage_detail | TEXT | How much/how often (e.g., "5 movies", "10 articles") |

**Purpose:** Identifies unused subscriptions that should be cancelled.

---

## Domain 9: System / Automation (3 tables)

Internal tracking for automation health, session analytics, and system-wide insights.

### automation_timestamps

Execution log for all scheduled tasks (cron jobs, LaunchAgents, manual skill runs).

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| task_name | TEXT | Name of the automation (e.g., "daily-scrape-amex", "weekly-tax-projection") |
| task_type | TEXT | scraper / analyzer / reporter / sync / other |
| scheduled_time | TEXT | ISO 8601 when task was supposed to run |
| started_at | TEXT | ISO 8601 when task actually started |
| completed_at | TEXT | ISO 8601 when task finished |
| status | TEXT | success / failure / timeout / skipped |
| duration_seconds | INTEGER | How long the task took |
| error_message | TEXT | Error details if status != success |
| rows_affected | INTEGER | Number of database rows created/updated |

**Purpose:** Automation health dashboard. Identifies failing tasks and performance regressions.

---

### session_logs

Claude Code session metadata for usage analytics and token cost tracking.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| session_id | TEXT | Unique session identifier |
| session_start | TEXT | ISO 8601 when session began |
| session_end | TEXT | ISO 8601 when session ended |
| duration_minutes | INTEGER | Total session time |
| primary_skill | TEXT | Which skill/command was primary focus |
| commands_executed | INTEGER | Number of CLI commands run |
| mcp_reads | INTEGER | Number of MCP read operations |
| mcp_writes | INTEGER | Number of MCP write operations |
| token_usage | INTEGER | Estimated tokens consumed in session |
| ai_calls | INTEGER | Number of Claude API calls |

**Purpose:** Session analytics, token usage trending, identifying expensive operations.

---

### insights

Aggregated patterns and analytics computed across all domains.

| Column | Type | Purpose |
|--------|------|---------|
| id | INTEGER | Primary key |
| insight_type | TEXT | spending_pattern / deal_momentum / automation_health / opportunity_trend / other |
| finding | TEXT | The insight discovered |
| supporting_data | TEXT | Metrics or context backing the insight |
| generated_at | TEXT | ISO 8601 when insight was computed |
| severity | TEXT | informational / warning / critical |

**Purpose:** High-level system insights. "You're spending 40% more on subscriptions than last month." "Pipeline momentum is positive."

---

## Design Principles

### Why SQLite?

- **Zero Infrastructure:** Single file, no server, no backups to manage. Copy the file and it's backed up.
- **MCP Integration:** Claude agents read and write directly to SQLite via the MCP SQLite integration. No translation layer.
- **Portability:** Moved across machines trivially. Shareable (read-only) with minimal privacy concern.
- **Adequate Performance:** Single-user analytics workloads don't require Postgres. Sub-100ms queries on 40+ tables is normal.
- **Schema Flexibility:** SQLite supports adding columns without downtime. No migration ceremonies.

### Schema Patterns

#### Scrape Logs Everywhere

Every external data pull (Plaid sync, card scraper, airline portal, tax platform) writes a row to a scrape_log table. Timestamps, success/failure, error messages, rows imported. This makes debugging trivial: "Why is my miles balance stale?" → Check scrape_log. "When did the last successful Plaid sync happen?" → One query.

#### Snapshot-Based Balance Tracking

Instead of `UPDATE balances SET miles_balance = ...`, the system `INSERT`s a new row each time. This preserves history for trend analysis without complex time-series databases. "Are my rewards growing or shrinking?" is a simple ORDER BY and trend comparison.

#### Verdict/Recommendation Tables Separate from Raw Data

Fund analysis stores raw fund data (prices, returns, expense ratios) in `fund_menu` and `performance`. Analysis results (HOLD/SWAP/WATCH recommendations) go into the `analysis` table. This enables "what changed since last analysis?" queries and prevents recommendations from being stale while data is fresh.

#### Preference Learning Tables

User feedback on opportunities and email drafts is stored with weights in preference tables. These feed scoring algorithms. As the system learns user preferences, it adjusts fit scores, draft styles, and account urgency rankings. The feedback table becomes an audit trail of "why did the system recommend that?"

#### Hashed PII / No Sensitive Data

Email addresses, names, and identifiers are either hashed or generic (e.g., "family_member_1"). Financial amounts and account numbers are stored but not in cleartext. This table schema could be shared or exposed with minimal privacy risk.

### Data Flow

```
External APIs (Plaid, Amex, Airline Portals, Tax Platforms)
    ↓
MCP Scrapers & Sync Agents
    ↓
SQLite Database (40+ tables across 9 domains)
    ↓
Skills, Commands, and Agents (read/write via MCP SQLite)
    ↓
User Output (/pipeline, /followup, /amex, /miles, /account, etc.)
```

Agents run on a schedule (daily, weekly, monthly) to keep data fresh. User commands query the database and present summaries, alerts, and recommendations. The entire system is built on queryable, time-stamped data with clear audit trails.

---

## Future Extensions

As the system evolves, these tables are designed to support:

- **Predictive Analytics:** Historical data in snapshot and performance tables enables trend forecasting (expiring miles, spending patterns, deal churn risk).
- **Multi-Currency Support:** Adding currency columns to transaction and balance tables enables international tracking.
- **Custom Rules Engine:** A new domain for rule definitions (e.g., "alert if spending in category X exceeds $Y per month") would integrate with existing transaction data.
- **Collaborative Filtering:** Feedback from multiple users in a household could improve recommendations across the system.
- **Integrations:** New external data sources (real estate, vehicle info, insurance, etc.) can each add a domain with the same design patterns.
