# SitePM Platform Spec

> **Goal:** "I hand over an agent to a customer. Here is your website product manager."

## Architecture

```
VPS (DigitalOcean London)
├── Clawdbot              (agent runtime — multi-agent, Telegram, sessions, memory)
├── SitePM Web App        (Next.js self-hosted — dashboard + API routes)
│   ├── /app              (customer dashboard)
│   ├── /api/agents       (provisioning API)
│   └── /api/auth         (authentication)
└── Caddy                 (reverse proxy, auto-SSL, routing)
    ├── sitepm.com → Next.js :3000
    └── api.sitepm.com → Next.js :3000/api (optional subdomain)
```

All on one box. No external dependencies except Anthropic API and Telegram Bot API.

---

## Core Concepts

### Agent
A customer-facing "website product manager" powered by Clawdbot. Each agent has:
- A Clawdbot agent ID (e.g., `customer-mark`)
- A dedicated workspace with persona files (AGENTS.md, SOUL.md, TOOLS.md, USER.md)
- A dedicated Telegram bot (own BotFather token, own name/avatar)
- Isolated sessions, memory, and tool access
- Sandboxed execution (Docker, no network by default)

### Customer
A person who owns one or more agents. Has:
- Dashboard login (email + password or magic link)
- Billing status
- List of their agents

### Site
A website the agent manages. Has:
- URL
- Tech stack info (auto-detected or declared)
- Associated agent

---

## Data Model

### `customers` table
| Field | Type | Notes |
|-------|------|-------|
| id | uuid | Primary key |
| email | string | Unique, login identifier |
| name | string | Display name |
| plan | enum | `trial`, `starter`, `pro` |
| created_at | timestamp | |
| stripe_customer_id | string | Nullable, for billing |

### `agents` table
| Field | Type | Notes |
|-------|------|-------|
| id | uuid | Primary key |
| customer_id | uuid | FK → customers |
| clawdbot_agent_id | string | e.g., `customer-mark` |
| name | string | Display name for the agent |
| telegram_bot_token | string | Encrypted |
| telegram_bot_username | string | e.g., `@MarkSitePMBot` |
| site_url | string | The website being managed |
| status | enum | `provisioning`, `active`, `paused`, `deleted` |
| created_at | timestamp | |

### `agent_config` table (or JSON in workspace)
| Field | Type | Notes |
|-------|------|-------|
| agent_id | uuid | FK → agents |
| soul_md | text | Generated SOUL.md content |
| agents_md | text | Generated AGENTS.md content |
| tools_md | text | Generated TOOLS.md content |
| user_md | text | Customer info for the agent |
| custom_instructions | text | Customer's additional instructions |

---

## Provisioning Pipeline

### What happens when a customer creates an agent:

```
1. Customer fills form: name, site URL, Telegram bot token
                              │
2. API validates inputs       │
   - Check bot token works (Telegram getMe)
   - Check site URL is reachable
                              │
3. Generate agent config      │
   - clawdbot_agent_id = slugify(customer-name)
   - Create workspace dir: ~/.clawdbot/agents/{agent_id}/
   - Generate SOUL.md (website PM persona, tailored to their site)
   - Generate AGENTS.md (site context, goals, tech stack)
   - Generate TOOLS.md (site-specific tools/APIs)
   - Generate USER.md (customer info)
                              │
4. Patch clawdbot.json        │
   - Add agent to agents.list[]
   - Add Telegram account to channels.telegram.accounts
   - Add binding: telegram account → agent
   - Set sandbox config (mode: all, scope: agent)
   - Set tool policy (restricted set for customer agents)
                              │
5. Restart gateway            │
   - gateway.restart() or SIGUSR1
   - Verify agent is live (health check)
                              │
6. Return success             │
   - Agent status → "active"
   - Customer gets Telegram bot link
   - Dashboard shows agent as live
```

### Agent Template (what gets generated)

**SOUL.md:**
```markdown
# SOUL.md — {agent_name}

You are {agent_name}, a website product manager for {site_url}.

## Your Role
- You manage and improve {customer_name}'s website
- You understand web development, SEO, content strategy, and analytics
- You proactively suggest improvements and flag issues
- You execute changes when asked (with approval)

## Your Style
- Professional but approachable
- Clear, actionable recommendations
- Always explain the "why" behind suggestions
- Prioritise impact — what moves the needle most?

## Boundaries
- Always get approval before making changes to production
- Flag anything you're unsure about
- Escalate security concerns immediately
```

**AGENTS.md:**
```markdown
# AGENTS.md — {agent_name} Workspace

## Site
- **URL:** {site_url}
- **Owner:** {customer_name}
- **Tech Stack:** {detected_or_declared}
- **Repo:** {if_connected}

## Goals
- Keep the site performing well (speed, SEO, uptime)
- Suggest and implement content improvements
- Monitor competitors and industry trends
- Report on key metrics

## Memory
- Daily: memory/YYYY-MM-DD.md
```

---

## API Endpoints

### Authentication
```
POST   /api/auth/register    — Create account
POST   /api/auth/login       — Login (returns JWT)
POST   /api/auth/logout      — Logout
GET    /api/auth/me          — Current user
```

### Agents
```
GET    /api/agents            — List customer's agents
POST   /api/agents            — Create new agent (triggers provisioning)
GET    /api/agents/:id        — Get agent details
PATCH  /api/agents/:id        — Update agent config
DELETE /api/agents/:id        — Delete agent (deprovision)
POST   /api/agents/:id/pause  — Pause agent
POST   /api/agents/:id/resume — Resume agent
```

### Agent Management
```
GET    /api/agents/:id/status   — Agent health + session info
GET    /api/agents/:id/history  — Recent chat history
POST   /api/agents/:id/message  — Send a message to the agent
PATCH  /api/agents/:id/config   — Update workspace files (SOUL.md etc.)
```

### Webhooks (internal)
```
POST   /api/webhooks/stripe    — Billing events
POST   /api/webhooks/telegram  — (if using webhook mode per-bot)
```

---

## Tech Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Runtime** | Clawdbot | Already runs 17 agents, proven, all the features we need |
| **Web Framework** | Next.js 15 (App Router) | Self-hosted, API routes = provisioning API, SSR dashboard |
| **Database** | SQLite (via better-sqlite3 or Drizzle) | Single file, zero ops, on the VPS, enough for hundreds of customers |
| **Auth** | NextAuth.js / custom JWT | Simple email+password to start |
| **Styling** | Tailwind CSS | Fast to build, consistent |
| **Reverse Proxy** | Caddy | Auto-SSL, simple config, production-ready |
| **Process Manager** | systemd | Both Clawdbot and Next.js as services |

### Why SQLite?
- One VPS, one file, zero database ops
- Handles thousands of concurrent reads
- WAL mode for concurrent read/write
- Backup = copy one file
- Upgrade path to Postgres if/when needed (Drizzle makes this trivial)

---

## Security

### Customer Agent Isolation
```json5
{
  // Each customer agent gets:
  sandbox: {
    mode: "all",
    scope: "agent",
    workspaceAccess: "rw",
    docker: {
      network: "none"  // no network access by default
    }
  },
  tools: {
    allow: ["read", "write", "edit", "exec", "web_search", "web_fetch", "memory_search", "memory_get"],
    deny: ["browser", "canvas", "nodes", "cron", "gateway", "message", "sessions_send", "sessions_spawn"]
  }
}
```

### What customers CAN'T do:
- Access other agents' workspaces
- Modify Clawdbot config
- Access the host filesystem
- Make network calls from sandbox (unless explicitly enabled)
- Use admin tools (gateway, cron, nodes, browser)

### What customers CAN do:
- Chat with their agent via Telegram
- Read/write files in their workspace
- Run sandboxed commands
- Search the web (for research)
- Manage their agent config via dashboard

---

## Dashboard Pages

```
/                     — Landing page (marketing)
/login                — Login
/register             — Sign up
/dashboard            — Agent list, quick stats
/dashboard/agents/new — Create new agent (onboarding wizard)
/dashboard/agents/:id — Agent detail (status, config, chat history)
/dashboard/agents/:id/settings — Edit SOUL.md, instructions, site URL
/dashboard/billing    — Plan, usage, invoices
```

---

## Onboarding Flow (Customer Experience)

1. **Sign up** — Email + password
2. **Create agent** — Name, site URL
3. **Connect Telegram** — We generate a BotFather link with instructions, OR we create the bot programmatically via BotFather API (stretch goal)
4. **Paste bot token** — We validate it, show the bot name
5. **Customise** (optional) — Adjust persona, add instructions
6. **Go live** — Agent provisioned, customer gets Telegram link
7. **Chat** — Customer DMs their bot, agent responds

---

## Scaling Considerations

### Current: 1 VPS
- 17 agents on 2vCPU/4GB
- Opus 4.6 with 200K context window
- Comfortable headroom for ~50 customer agents (estimate)

### Growth Path
- **Vertical:** Upgrade VPS (4vCPU/8GB handles ~100+ agents)
- **Horizontal:** Second Clawdbot instance on second VPS, load-balanced by customer
- **API costs:** Per-customer billing based on token usage (tracked by Clawdbot)
- **Database:** SQLite → Postgres migration if/when multi-VPS

### Cost Model (per customer agent)
- Anthropic API: usage-based (~$0.50–5/day depending on activity)
- VPS share: ~$2–5/month per agent (amortised)
- Telegram: free
- **Target price point:** £29–99/month depending on plan

---

## MVP Scope (v0.1)

### Must Have
- [ ] Customer sign-up (email + password)
- [ ] Create agent (name, site URL, Telegram bot token)
- [ ] Provisioning pipeline (generate workspace, patch config, restart)
- [ ] Agent dashboard (status, basic chat history)
- [ ] Deprovisioning (delete agent cleanly)

### Nice to Have (v0.2)
- [ ] Agent config editing (SOUL.md, instructions) from dashboard
- [ ] Usage tracking / cost display
- [ ] Stripe billing integration
- [ ] Agent health monitoring + alerts
- [ ] Automated BotFather bot creation

### Future (v1.0+)
- [ ] Custom domain per agent
- [ ] GitHub repo connection (agent can push code)
- [ ] Multi-channel (WhatsApp, Slack, web widget)
- [ ] Team access (multiple users per customer)
- [ ] Agent marketplace (pre-built personas)

---

## Open Questions

1. **Domain:** What domain for the platform? (sitepm.com? markpm.com? something else?)
2. **BotFather flow:** Do we walk customers through creating their own bot, or do we automate it?
3. **Billing model:** Flat monthly fee, usage-based, or hybrid?
4. **Agent capabilities:** What tools should customer agents have access to by default? (web_search? code execution? file editing?)
5. **Watt as template:** Use Watt's setup for Mark as the reference implementation?

---

## Next Steps

1. **Will to confirm** spec direction + answer open questions
2. **Forge** to build the provisioning pipeline (the engine)
3. **Forge** to scaffold the Next.js app (dashboard shell)
4. **Watt** continues as first customer agent (Mark) — validates the model
5. **Ship MVP** — first external customer live
