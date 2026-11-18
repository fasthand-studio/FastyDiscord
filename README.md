
# GitHub Activity Discord Bot

A private GitHub activity tracker that connects GitHub repositories to Discord. Built with TypeScript, Vercel Serverless Functions, Supabase PostgreSQL, and a GitHub App.

It tracks development activity across authorized public and private repositories, sends updates to Discord in real time, and generates weekly development reports. The entire system runs serverlessly, without a VPS or a permanently running bot process.

**Stack:** TypeScript · Vercel · Supabase · GitHub App · Discord

## Features

| Feature | Description |
|---|---|
| Repository Tracking | Monitor authorized repositories across GitHub installations and organizations. |
| Automatic Discovery | Detect newly added repositories and synchronize repository access. |
| Activity Notifications | Receive updates for commits, pushes, pull requests, releases, and issues. |
| Activity History | Store development events in PostgreSQL for historical reporting. |
| Slash Commands | Access repository and contributor statistics directly from Discord. |
| Weekly Reports | Generate activity summaries, contribution heatmaps, and code statistics. |
| AI Summaries | Generate repository-level and weekly summaries using configurable AI providers. |
| Event Deduplication | Prevent duplicate notifications using GitHub delivery IDs and event identifiers. |

## Discord Commands

The bot uses Discord HTTP interactions rather than a continuously running Gateway connection.

| Command | Purpose |
|---|---|
| `/stats` | View development activity statistics. |
| `/contributors` | View contributor activity. |
| `/repositories` | Browse tracked repositories. |
| `/weekly` | View the weekly development report. |
| `/repo` | View activity for a specific repository. |

## Weekly Reports

The weekly report is generated automatically and delivered to a dedicated Discord channel.

Each report includes overall activity totals, repository-level summaries, top contributors, and code change statistics.

It also includes a GitHub-style contribution heatmap and an optional AI-generated summary covering what happened across repositories during the reporting period.

The default schedule is **Friday at 1:05 PM Bangladesh time (07:05 UTC)**.

For exact-minute scheduling, the project also supports Supabase `pg_cron` and `pg_net`, avoiding the execution-window limitations of Vercel Hobby Cron.

## Architecture

| Layer | Technology | Responsibility |
|---|---|---|
| Application | TypeScript | Application logic and event processing |
| Runtime | Vercel Functions | Serverless API endpoints |
| Database | Supabase PostgreSQL | Activity records, repository data, and reports |
| GitHub | GitHub App | Authentication, webhooks, and repository discovery |
| Discord | Webhooks & Interactions | Notifications and slash commands |
| Scheduling | Vercel Cron / Supabase pg_cron | Weekly report execution |
| AI | Configurable providers | Activity summaries |

### API Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/github/webhook` | Receive and process GitHub events |
| POST | `/api/github/sync` | Synchronize accessible repositories |
| POST | `/api/discord/interactions` | Handle Discord slash commands |
| GET / POST | `/api/cron/weekly` | Generate and publish weekly reports |
| GET | `/api/health` | Health check |

GitHub webhook requests are verified using `X-Hub-Signature-256`. Discord interactions use Ed25519 signature verification.

## Project Structure

```text
api/
├── github/
│   ├── webhook.ts
│   └── sync.ts
├── discord/
│   └── interactions.ts
├── cron/
│   └── weekly.ts
└── health.ts

lib/
├── ai.ts
├── dedup.ts
├── discord.ts
├── env.ts
├── format.ts
├── github.ts
├── github-events.ts
├── heatmap.ts
├── stats.ts
└── supabase.ts

scripts/
└── register-discord-commands.ts

supabase/
└── migrations/
    └── 0001_init.sql
```

## Getting Started

### Requirements

Node.js 20+, a Supabase project, a GitHub App with repository access, a Discord application, and a Vercel project.

An AI API key is optional and only required for AI-generated summaries.

### 1. Install

```bash
npm install
```

### 2. Database

Create a Supabase project and execute:

`supabase/migrations/0001_init.sql`

Configure the following credentials:

```env
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
```

The service role key is used exclusively for server-side database operations. Never expose it to the browser.

### 3. GitHub App

Create a GitHub App and configure the webhook endpoint:

```text
https://<your-vercel-domain>/api/github/webhook
```

Enable read-only access to Contents, Issues, Metadata, and Pull Requests.

Subscribe to Installation, Installation repositories, Push, Pull request, Release, Issues, Repository, and Meta/Ping events.

Configure the application credentials:

```env
GITHUB_APP_ID=
GITHUB_APP_PRIVATE_KEY=
GITHUB_WEBHOOK_SECRET=
GITHUB_INSTALLATION_IDS=
```

`GITHUB_INSTALLATION_IDS` is optional and can be used to restrict processing to specific installations.

### 4. Discord

Create a Discord application through the [Developer Portal](https://discord.com/developers/applications).

Set the Interactions Endpoint URL to:

```text
https://<your-vercel-domain>/api/discord/interactions
```

Create two channel webhooks, one for activity updates and another for weekly reports.

Configure:

```env
DISCORD_PUBLIC_KEY=
DISCORD_APPLICATION_ID=
DISCORD_BOT_TOKEN=
DISCORD_UPDATES_WEBHOOK_URL=
DISCORD_WEEKLY_WEBHOOK_URL=
```

Register slash commands:

```bash
cp .env.example .env.local
npx tsx scripts/register-discord-commands.ts
```

Set `DISCORD_GUILD_ID` for faster guild-specific command registration during development.

### 5. AI Configuration

The weekly reporting system supports three configurable providers.

| Provider | Configuration |
|---|---|
| OpenAI-compatible | `AI_PROVIDER=openai-compatible` |
| Anthropic Messages | `AI_PROVIDER=anthropic-messages` |
| Cloudflare Workers AI | `AI_PROVIDER=cloudflare` |

Required configuration:

```env
AI_PROVIDER=
AI_API_KEY=
AI_MODEL=
AI_BASE_URL=
CLOUDFLARE_ACCOUNT_ID=
```

`AI_BASE_URL` is optional. Cloudflare additionally requires `CLOUDFLARE_ACCOUNT_ID` and a Workers AI-enabled API token.

The summarizer receives structured activity data and is instructed not to invent repository changes. If AI is unavailable, weekly reports can still be published using a fallback message.

### 6. Deployment

Import the repository into Vercel, configure the environment variables from `.env.example`, and deploy.

Set a strong `CRON_SECRET` to protect scheduled and manual maintenance endpoints.

The included `vercel.json` configures weekly execution at `5 7 * * 5` (Friday, 07:05 UTC) and a maximum function duration of 60 seconds.

**Initial repository synchronization**

After deployment, populate the repository catalog:

```bash
curl -X POST "https://<your-vercel-domain>/api/github/sync" \
  -H "Authorization: Bearer $CRON_SECRET"
```

Subsequent GitHub installation and repository events refresh the catalog automatically.

## Database & Environment

The database schema is maintained through Supabase migrations. The initial migration is located at `supabase/migrations/0001_init.sql`.

For the full environment configuration, refer to `.env.example`.

| Variable | Required | Purpose |
|---|---|---|
| `SUPABASE_URL` | Yes | Database project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Yes | Server-side database access |
| `GITHUB_APP_ID` | Yes | GitHub App authentication |
| `GITHUB_APP_PRIVATE_KEY` | Yes | GitHub App private key |
| `GITHUB_WEBHOOK_SECRET` | Yes | Webhook signature verification |
| `GITHUB_INSTALLATION_IDS` | No | Installation restrictions |
| `DISCORD_PUBLIC_KEY` | Yes | Interaction verification |
| `DISCORD_APPLICATION_ID` | Yes | Discord application |
| `DISCORD_BOT_TOKEN` | Yes | Command registration |
| `DISCORD_UPDATES_WEBHOOK_URL` | Yes | Activity notifications |
| `DISCORD_WEEKLY_WEBHOOK_URL` | Yes | Weekly reports |
| `DATABASE_URL` | No | Direct PostgreSQL connection |
| `AI_PROVIDER` | No | AI provider selection |
| `AI_API_KEY` | AI enabled | Provider credentials |
| `AI_BASE_URL` | No | Custom AI API endpoint |
| `AI_MODEL` | No | Selected AI model |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare only | Workers AI account |
| `CRON_SECRET` | Recommended | Endpoint protection |

`DATABASE_URL` enables direct SQL aggregation through the Supabase connection pooler (port 6543). When unavailable, the reporting system falls back to PostgREST.

## Reliability & Security

GitHub delivery IDs are stored in `webhook_deliveries` to prevent duplicate processing. Selected event types also use unique `external_id` constraints.

Webhook signatures are verified before processing. Maintenance endpoints use Bearer authentication through `CRON_SECRET` and are denied in production when the secret is missing.

Repository synchronization can repair catalog inconsistencies, although historical event backfilling requires a separate import process.

Keep all credentials outside the repository. Never commit `.env` files, private keys, GitHub tokens, Discord webhook URLs, or AI credentials. If exposed, rotate them immediately.

See `SECURITY.md` for additional security information.

## Development

Run TypeScript checks:

```bash
npm run typecheck
```

Start the local Vercel development environment:

```bash
npm install -g vercel
vercel dev
```

For webhook testing, expose the local server through a secure tunnel and configure the GitHub App and Discord interaction endpoints to use that URL.

## Scheduling Reference

The optional Supabase scheduling migration is:

`supabase/migrations/0003_weekly_report_pg_cron.sql`

It uses `pg_cron` and `pg_net` to trigger the weekly report at an exact minute.

The job name is `fastydiscord-weekly-report`, scheduled at `5 7 * * 5`.

Store `CRON_SECRET` in Supabase Vault under `fastydiscord_cron_secret` before enabling the migration.

The report handler is idempotent on `weekly_reports.week_start`, so Vercel Cron can remain as a fallback without producing duplicate reports.

## Operational Notes

GitHub App permissions should remain read-only. Activity updates and weekly reports should use separate Discord channels.

Vercel function logs are the primary place to investigate webhook failures, database errors, and AI provider issues. For higher activity volumes, SQL aggregation views or materialized summaries can be introduced in `lib/stats.ts`.

---

**GitHub Activity Discord Bot**  
TypeScript · Vercel · Supabase · GitHub · Discord
