# convex-agent-skillz

A curated collection of Claude Code skills for Convex backend development. These skills give Claude deep expertise in Convex patterns, components, authentication, and AI agent development.

## What Are Claude Code Skills?

Skills are knowledge bases that Claude Code loads automatically based on context. When you're working in a Convex project and ask Claude to help with a query, mutation, or component — it already knows the best practices, patterns, and gotchas.

## What's Included

### Skills (6)

| Skill                 | Description                                                                                       | Triggers On                                                        |
| --------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **convex**            | Core backend development — queries, mutations, actions, schemas, scheduling, file storage, search | `ctx.db`, `useQuery`, `useMutation`, schemas, crons, file storage  |
| **convex-agent**      | AI agents with persistent threads, tool calling, and streaming                                    | Chat interfaces, AI assistants, RAG systems, multi-agent workflows |
| **convex-components** | Component patterns — Rate Limiter, Aggregate, Workpool, Workflow, and more                        | Any `@convex-dev/*` component usage                                |
| **convex-auth**       | Universal authentication patterns (provider-agnostic)                                             | Auth checks, `getUserIdentity()`, user storage                     |
| **convex-clerk**      | Clerk-specific integration                                                                        | Clerk setup, webhooks, JWT configuration                           |
| **convex-workos**     | WorkOS AuthKit integration                                                                        | WorkOS setup, auto-provisioning, SSO                               |

### Custom Command (1)

| Command            | Description                                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------------------- |
| `/create-features` | Converts implementation plans into structured feature documentation with dependencies and acceptance criteria |

### Custom Agent (1)

| Agent             | Description                                                                                                        |
| ----------------- | ------------------------------------------------------------------------------------------------------------------ |
| **convex-expert** | Expert backend developer that auto-activates when working in `convex/` directory. Has access to all Convex skills. |

## Convex Components Covered

These skills provide deep expertise on [Convex Components](https://www.convex.dev/components) — modular, drop-in building blocks for your backend. Each component documented here includes installation, configuration, usage patterns, and best practices.

### AI

| Component | Package             | Description                                                                                                                                                                                                |
| --------- | ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Agent** | `@convex-dev/agent` | Build AI agents with persistent threads, tool calling, real-time streaming, and message history. Supports multi-agent workflows, RAG, and durable agent processes. _(Has dedicated skill: `convex-agent`)_ |

### Durable Functions

| Component          | Package                      | Description                                                                                                 |
| ------------------ | ---------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Workflow**       | `@convex-dev/workflow`       | Durable, long-running code flows with retries, delays, and guaranteed completion. Survives server restarts. |
| **Workpool**       | `@convex-dev/workpool`       | Queue async operations with configurable parallelism limits and priority scheduling.                        |
| **Action Retrier** | `@convex-dev/action-retrier` | Retry failed actions with exponential backoff and idempotency guarantees.                                   |

### Backend Utilities

| Component           | Package                       | Description                                                                                          |
| ------------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------- |
| **Rate Limiter**    | `@convex-dev/rate-limiter`    | Application-layer rate limiting with token bucket and fixed window algorithms. Transactionally safe. |
| **Aggregate**       | `@convex-dev/aggregate`       | Efficient O(log n) COUNT, SUM, MIN, MAX operations using denormalized data structures.               |
| **Sharded Counter** | `@convex-dev/sharded-counter` | High-throughput counting that avoids write contention through sharding.                              |
| **Presence**        | `@convex-dev/presence`        | Real-time user presence tracking (online status, typing indicators, cursors).                        |
| **Action Cache**    | `@convex-dev/action-cache`    | Cache expensive action results (LLM calls, API requests) with TTL support.                           |
| **Migrations**      | `@convex-dev/migrations`      | Run long-running data migrations with progress tracking and resumability.                            |

### Integrations

| Component            | Package                        | Description                                                              |
| -------------------- | ------------------------------ | ------------------------------------------------------------------------ |
| **Resend**           | `@convex-dev/resend`           | Transactional email with queuing, retry logic, and webhook handling.     |
| **Stripe**           | `@convex-dev/stripe`           | Payments, subscriptions, and billing with webhook sync to your database. |
| **ProseMirror Sync** | `@convex-dev/prosemirror-sync` | Real-time collaborative text editing for Tiptap and BlockNote editors.   |

> **Note:** The Convex ecosystem has many more components. These are the ones with detailed skill documentation in this repo. See [convex.dev/components](https://www.convex.dev/components) for the full catalog.

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/claude-code) installed
- A Convex project (or create one with `npm create convex`)

### Setup

1. **Clone or download this repository**

2. **Copy the `.claude` folder to your project root**

   ```bash
   cp -r convex-agent-skillz/.claude /path/to/your/project/
   ```

3. **Configure settings** (optional)

   Edit `.claude/settings.local.json` to customize permissions:

   ```json
   {
     "permissions": {
       "allow": [
         "WebFetch(domain:docs.convex.dev)",
         "WebFetch(domain:www.convex.dev)",
         "WebFetch(domain:stack.convex.dev)",
         "WebSearch"
       ]
     },
     "enableAllProjectMcpServers": true,
     "enabledMcpjsonServers": ["convex"]
   }
   ```

4. **Verify skills are loaded**

   Start Claude Code in your project and ask:

   ```
   What Convex skills do you have available?
   ```

## Usage

### Automatic Skill Activation

Skills activate automatically based on what you're working on:

- **Working in `convex/` directory** — The `convex-expert` agent activates with full Convex knowledge
- **Asking about queries or mutations** — `convex` skill provides patterns
- **Implementing AI chat** — `convex-agent` skill guides you through the Agent component
- **Setting up Clerk auth** — Both `convex-auth` and `convex-clerk` skills activate

### Using the `/create-features` Command

When you have an implementation plan (from Claude or your own), use this command to break it into trackable features:

1. Share or create an implementation plan with Claude
2. Run `/create-features`
3. Claude creates a `docs/features/` folder with:
   - `00-overview.md` — Feature matrix with status tracking
   - `01-feature-name.md`, `02-feature-name.md`, etc. — Individual feature specs

### Example Interactions

**Ask about Convex patterns:**

```
How do I implement a paginated query with cursor-based pagination?
```

**Get help with components:**

```
I need to rate limit user actions to 10 per minute
```

**Set up authentication:**

```
Help me configure Clerk authentication for my Convex app
```

**Build an AI agent:**

```
I want to create a support chatbot with persistent message history
```

## Skill Details

### convex (Core)

The foundation skill covering:

- **Functions** — Queries, mutations, actions, internal functions, HTTP actions
- **Database** — Schema design, indexes, efficient queries
- **Validation** — Argument validators, custom types
- **Scheduling** — Crons, scheduled functions, workflow patterns
- **File Storage** — Upload, storage, serving patterns
- **Search** — Full-text search, vector search, RAG
- **Next.js** — App Router integration, SSR, Server Actions

Reference docs: `FUNCTIONS.md`, `DATABASE.md`, `VALIDATION.md`, `SEARCH.md`, `AUTH.md`, `SCHEDULING.md`, `FILE_STORAGE.md`, `HTTP_ACTIONS.md`, `ERROR_HANDLING.md`, `ADVANCED.md`, `RUNTIMES.md`, `NEXTJS.md`

### convex-agent

Build AI-powered features:

- **Threads** — Persistent conversation history
- **Streaming** — Delta streaming, HTTP streaming, text smoothing
- **Tools** — Define tools with Convex context access
- **Workflows** — Durable multi-step agent processes
- **RAG** — Context injection and retrieval patterns

Reference docs: `STREAMING.md`, `TOOLS.md`, `CONTEXT.md`, `THREADS.md`, `MESSAGES.md`, `WORKFLOWS.md`, `HUMAN-AGENTS.md`, `FILES.md`, `RATE-LIMITING.md`, `USAGE-TRACKING.md`, `DEBUGGING.md`

### convex-components

Patterns for the Convex component ecosystem:

| Component           | Use Case                                     |
| ------------------- | -------------------------------------------- |
| **Workflow**        | Long-running, durable processes with retries |
| **Workpool**        | Queue actions with parallelism limits        |
| **Rate Limiter**    | Application-layer rate limiting              |
| **Aggregate**       | Efficient COUNT, SUM, MAX operations         |
| **Sharded Counter** | High-throughput counting                     |
| **Presence**        | Real-time user presence                      |
| **Action Cache**    | Cache expensive action results               |
| **Migrations**      | Stateful online data migrations              |
| **Stripe**          | Payments and subscriptions                   |
| **Resend**          | Transactional email                          |

### convex-auth (Universal)

Provider-agnostic authentication patterns:

- `ctx.auth.getUserIdentity()` usage
- Race condition prevention
- Database user storage patterns
- Custom auth wrappers with convex-helpers
- HTTP actions with auth

### convex-clerk

Clerk-specific setup:

- `auth.config.ts` configuration
- JWT template requirements (must be named "convex")
- `ConvexProviderWithClerk` setup
- Webhook implementation for user sync
- Environment variables for React and Next.js

### convex-workos

WorkOS AuthKit integration:

- Dual JWT provider configuration
- Auto-provisioning for development
- CORS setup for React apps
- `ConvexProviderWithAuthKit` usage

## Customization

### Adding Reference Docs

Each skill has a `references/` folder with topic-specific guides:

```
.claude/skills/convex-skill/
├── SKILL.md              # Main skill file (required)
└── references/
    ├── FUNCTIONS.md
    ├── DATABASE.md
    └── YOUR_NEW_TOPIC.md  # Add your own
```

### Creating Custom Agents

Copy the agent template and modify:

```yaml
# .claude/agents/my-agent.md
---
name: my-agent
description: "Description that tells Claude when to use this agent"
tools: Read, Edit, Write, Bash, Grep, Glob
model: inherit
skills: convex, convex-auth # Skills this agent should use
color: blue
---
Your agent instructions go here...
```

### Adding New Skills

Create a new folder in `.claude/skills/`:

```
.claude/skills/my-new-skill/
├── SKILL.md
└── references/
    └── TOPIC.md
```

The `SKILL.md` needs YAML frontmatter:

```yaml
---
name: my-skill
description: "When to use this skill and what triggers it."
---
# Skill content with patterns and examples...
```

## Contributing

Contributions are welcome! Here's how you can help:

### Adding or Improving Skills

1. Fork this repository
2. Create a branch for your changes
3. Add or modify skill content in `.claude/skills/`
4. Follow the existing file structure and formatting conventions
5. Test your changes by using the skills in a real Convex project
6. Submit a pull request with a clear description of your changes

### File Structure Conventions

- **SKILL.md** — Main entry point with overview and common patterns
- **references/\*.md** — Detailed topic-specific guides
- Use code examples that are complete and runnable
- Include both "Do" and "Don't" patterns where helpful
- Keep content actionable — Claude uses this to help developers write code

### What Makes a Good Skill Contribution

- Fills a gap in current Convex coverage
- Based on real-world usage patterns
- Includes working code examples
- Explains the "why" not just the "how"
- Tested against current Convex version

## Project Structure

```
.claude/
├── agents/
│   └── convex-expert.md        # Expert agent definition
├── commands/
│   └── create-features.md      # Feature breakdown command
├── skills/
│   ├── convex-skill/           # Core Convex patterns
│   ├── convex-agent-skill/     # AI agent component
│   ├── convex-components-skill/# Component ecosystem
│   ├── convex-auth-skill/      # Universal auth patterns
│   ├── convex-clerk-skill/     # Clerk integration
│   └── convex-workos-skill/    # WorkOS integration
└── settings.local.json         # Permissions configuration
```
