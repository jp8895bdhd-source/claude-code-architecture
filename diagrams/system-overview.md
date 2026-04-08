# Claude Code Personal Automation Platform - System Architecture

## Diagram 1: High-Level System Architecture

```mermaid
graph TB
    subgraph Core["Claude Code Harness"]
        Agents["Agents<br/>6 specialized agents"]
        Skills["Skills<br/>14 reusable skills"]
        Commands["Commands<br/>35 slash commands"]
        Hooks["Hooks<br/>Session lifecycle"]
        Plugins["Plugins<br/>RTK token rewrite"]
        Scripts["Scripts<br/>Auto-execution"]
    end

    subgraph Sales["Sales Intelligence Platform"]
        Accounts["Account Health<br/>Tracking"]
        Deals["Deal Pipeline<br/>Management"]
        FollowUps["Follow-up<br/>Scheduling"]
        Meetings["Meeting<br/>Preparation"]
    end

    subgraph Invest["Investment & Financial Tracking"]
        Portfolio["Portfolio<br/>Monitoring"]
        Funds["Fund Analysis<br/>Monthly Reports"]
        Spending["Spending<br/>Analytics"]
        Banking["Banking Data<br/>Integration"]
    end

    subgraph Home["Household Automation"]
        Meals["Meal<br/>Planning"]
        IoT["IoT Device<br/>Control"]
        Network["Network<br/>Management"]
        Schedules["Task<br/>Scheduling"]
    end

    subgraph Data["Data Layer"]
        SQLite["SQLite Database<br/>40+ tables<br/>9 domains"]
    end

    subgraph MCP["MCP Integration Layer"]
        MCPServers["12+ MCP Servers<br/>Email, Calendar, Banking,<br/>Web, File System, etc."]
    end

    Core -->|Commands & Skills| Sales
    Core -->|Commands & Skills| Invest
    Core -->|Commands & Skills| Home
    Sales -->|Read/Write| SQLite
    Invest -->|Read/Write| SQLite
    Home -->|Read/Write| SQLite
    SQLite -->|Query Results| Core
    Core -->|API Calls| MCPServers
    MCPServers -->|Data| SQLite
```

---

## Diagram 2: Agent Architecture

```mermaid
graph TB
    subgraph Agents["Agent Tier Architecture"]
        Daily["Daily Briefing<br/>FAST / HAIKU<br/>⚡ Low-latency"]
        Account["Account Intelligence<br/>MID / SONNET<br/>📊 Data-heavy"]
        Email["Email Drafter<br/>BEST / OPUS<br/>✍️ Complex reasoning"]
        Meeting["Meeting Prep<br/>MID / SONNET<br/>📅 Context synthesis"]
        Research["Research Analyst<br/>MID / SONNET<br/>🔍 Web + synthesis"]
        HomeManager["Home Lab Manager<br/>MID / SONNET<br/>🏠 Config + scripting"]
    end

    subgraph DataSources["Data Sources"]
        Cal["Calendar"]
        Mail["Email"]
        Remind["Reminders"]
        Port["Portfolio"]
        Fin["Finance"]
        DB["Database"]
        Contacts["Contacts"]
        Notes["Notes"]
        Web["Web Search"]
        Perplexity["Perplexity API"]
        FS["File System"]
        Config["Config"]
    end

    Daily --> Cal
    Daily --> Mail
    Daily --> Remind
    Daily --> Port
    Daily --> Fin

    Account --> DB
    Account --> Mail
    Account --> Cal
    Account --> Contacts

    Email --> Notes
    Email --> Mail
    Email --> Contacts

    Meeting --> Cal
    Meeting --> Contacts
    Meeting --> Mail
    Meeting --> DB

    Research --> Web
    Research --> Perplexity
    Research --> FS

    HomeManager --> Config
    HomeManager --> FS
    HomeManager --> DB
```

---

## Diagram 3: Hook Pipeline

```mermaid
graph LR
    Start["SessionStart"] -->|Initialize| PreTool["PreToolUse<br/>RTK Token Rewrite<br/>60-90% savings"]
    
    PreTool -->|Rewritten Command| ToolExec["Tool Execution<br/>API calls<br/>MCP servers"]
    
    ToolExec -->|Results| PostTool["PostToolUse<br/>Data processing<br/>DB writes"]
    
    PostTool -->|Finalize| Stop["Stop<br/>Auto-email digest<br/>Backup sync"]

    TaskComp["TaskCompleted"] -->|Trigger| Notify["macOS Notification<br/>User alert"]
    
    UserPrompt["UserPromptSubmit"] -->|Enrich| PromptExec["Prompt Enrichment<br/>Context loading<br/>Memory injection"]
    
    PromptExec -->|Enhanced| PreTool

    Notify -->|Visual Feedback| User["User"]
    Stop -->|Complete| User
```

---

## Diagram 4: Data Flow Architecture

```mermaid
graph TB
    subgraph External["External Data Sources"]
        Gmail["Gmail"]
        GCal["Google Calendar"]
        Apple["Apple Ecosystem<br/>Mail, Calendar, Contacts,<br/>Messages, Notes"]
        Plaid["Plaid<br/>Banking Data"]
        Web["Web Scraping<br/>Playwright"]
        GitHub["GitHub API"]
        Search["Web Search<br/>Perplexity"]
    end

    subgraph MCPLayer["MCP Integration Layer<br/>12+ Servers"]
        Gmail -->|gmail_search| MCPGmail["Gmail MCP"]
        GCal -->|calendar_list| MCPCal["Google Calendar MCP"]
        Apple -->|read_messages| MCPApple["Apple MCP<br/>Mail, Calendar,<br/>Contacts, Notes, Messages"]
        Plaid -->|sync_transactions| MCPPlaid["Plaid MCP"]
        Web -->|browser_navigate| MCPPlay["Playwright MCP"]
        GitHub -->|search_code| MCPGit["GitHub MCP"]
        Search -->|perplexity_ask| MCPPerp["Perplexity MCP"]
    end

    subgraph DBLayer["SQLite Database<br/>40+ tables<br/>9 domains"]
        DBEmail["Email Domain<br/>4 tables"]
        DBCal["Calendar Domain<br/>5 tables"]
        DBDeal["Deal Domain<br/>8 tables"]
        DBPort["Portfolio Domain<br/>6 tables"]
        DBRes["Research Domain<br/>4 tables"]
        DBOther["Other Domains<br/>13+ tables"]
    end

    subgraph SkillsAgents["Skills & Agents<br/>14 skills, 6 agents"]
        SkillOne["Skills"]
        AgentOne["Agents"]
    end

    subgraph Commands["Slash Commands<br/>35 commands"]
        CmdDaily["/morning, /eod"]
        CmdSales["/account, /pipeline, /contact"]
        CmdJob["/jobs, /applyfor"]
        CmdEmail["/draft, /followup"]
        CmdOther["/prepare, /research, etc."]
    end

    subgraph Output["User Interface"]
        Terminal["Claude Code Terminal<br/>Output & Interaction"]
    end

    MCPGmail -->|transactions, threads| DBEmail
    MCPCal -->|events| DBCal
    MCPApple -->|messages, events, notes| DBEmail
    MCPPlaid -->|account data, transactions| DBPort
    MCPPlay -->|page content| DBRes
    MCPGit -->|code search| DBRes
    MCPPerp -->|research results| DBRes

    DBEmail -->|data| SkillsAgents
    DBCal -->|data| SkillsAgents
    DBDeal -->|data| SkillsAgents
    DBPort -->|data| SkillsAgents
    DBRes -->|data| SkillsAgents

    SkillsAgents -->|execute| Commands
    Commands -->|present| Terminal
    Terminal -->|user interaction| Output
```

---

## Diagram 5: Scheduled Automation

```mermaid
graph TB
    subgraph Cron["Cron Jobs<br/>System scheduler"]
        C1["Weekly Receipt<br/>Processing<br/>Every Monday 6am"]
        C2["Weekly Emailer<br/>Digest<br/>Every Friday 5pm"]
        C3["Weekly Backup<br/>Data sync<br/>Every Sunday 2am"]
        C4["Monthly Fund<br/>Analysis<br/>1st of month 9am"]
        C5["Daily Health<br/>Check<br/>Every day 7am"]
    end

    subgraph Launch["LaunchAgents<br/>macOS native scheduler"]
        L1["Weekly Status<br/>Report<br/>Generate + deliver"]
        L2["Token Audit<br/>RTK Analytics<br/>Track savings"]
        L3["Trending<br/>Research<br/>Auto-categorize"]
        L4["Trending Email<br/>Thread analysis<br/>Important threads"]
    end

    subgraph Actions["Execution Actions"]
        Action1["Insert into expense DB<br/>Tag & categorize"]
        Action2["Query accounts + deals<br/>Send email summary"]
        Action3["SQLite backup<br/>Sync remote"]
        Action4["Pull fund data<br/>Compare performance<br/>Generate report"]
        Action5["Verify all systems<br/>Alert on failures"]
        Action6["Generate markdown<br/>save to research/"]
        Action7["Query email threads<br/>Surface top 5"]
        Action8["Parse tokens<br/>Display savings %"]
    end

    subgraph Output["Output Destinations"]
        DB1["Database"]
        Email["Email delivery"]
        Backup["Cloud backup"]
        Report["Research files"]
        Dashboard["Dashboard view"]
    end

    C1 --> Action1
    C2 --> Action2
    C3 --> Action3
    C4 --> Action4
    C5 --> Action5

    L1 --> Action6
    L2 --> Action8
    L3 --> Action6
    L4 --> Action7

    Action1 --> DB1
    Action2 --> Email
    Action3 --> Backup
    Action4 --> Report
    Action5 --> Dashboard
    Action6 --> Report
    Action7 --> Email
    Action8 --> Dashboard
```

---

## Architecture Notes

### System Pillars
- **Claude Code Harness**: Core automation runtime with agents, skills, commands, hooks, plugins, and scripts
- **Sales Intelligence**: Account tracking, deal management, follow-ups, and meeting preparation
- **Investment & Financial Tracking**: Portfolio monitoring, fund analysis, spending analytics, and banking integration
- **Household Automation**: Meal planning, IoT control, network management, and task scheduling

### Data Organization
- **SQLite Database**: 40+ tables across 9 domains (Email, Calendar, Deals, Portfolio, Research, and others)
- **MCP Servers**: 12+ integration points for cloud services and local system access

### Agent Tier Strategy
- **HAIKU (Fast)**: Simple, low-latency briefings with strict context limits
- **SONNET (Mid)**: Standard complexity tasks with balanced cost/capability
- **OPUS (Best)**: Complex reasoning tasks requiring maximum accuracy (email drafting)

### Automation Scope
- **Cron jobs**: 5 recurring system tasks running on fixed schedules
- **LaunchAgents**: 4 native macOS scheduled processes for background work
- **Hook-driven events**: Real-time processing on prompt submission, tool execution, and session lifecycle

