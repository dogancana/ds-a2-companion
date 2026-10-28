# Aion 2 Discord Companion — Plan

Status: **D1–D27 locked.** Channel types are deferred until after test runs. The minor cleanup list in section 10 is still open. Implementation has not started.

## 0. Decisions log

| # | Topic | Decision |
| --- | --- | --- |
| D1 | Repository | Monorepo, **public** (free GitHub Actions minutes). |
| D2 | Events | Fetch **everything** the source offers. Admins choose which events each subscription posts. The event catalogue is data-driven, not hard-coded in the UI. |
| D3 | Message style | **Configurable per subscription**: board (edited in place), reminders (new posts), or both. |
| D4 | Role mentions | The UI fetches the guild's roles through the bot (`GET /guilds/{id}/roles`) so the admin can pick which role(s) to ping. |
| D5 | Region / server | **Part of each subscription's configuration**, not global. |
| D6 | Data source | **Grimvane API v1**, behind a `TimerSource` interface. Tasks and the UI never depend on Grimvane types or URLs. See section 4. |
| D7 | Database | **MongoDB Atlas M0** (free tier). Network access `0.0.0.0/0`, because GitHub Actions and Vercel have no fixed IPs. |
| D8 | Scheduler | **GitHub Actions** `schedule`, every 15 minutes, offset from the top of the hour. Accepted because the shortest field-boss windows are on the order of 3 hours, so a delayed or dropped run still leaves useful timer data. |
| D9 | Posting | **Bot only.** One bot token posts and edits messages, and the same token lists channels and roles for the UI. Webhooks are not used. See section 3.3. |
| D10 | Language | **English only** for v1. UI copy and posted messages are English. `locale` is stored as `en` so a later language does not need a schema change. |
| D11 | Web hosting | **Vercel Hobby.** The Next.js app and its server routes run there. GitHub Pages cannot (section 7.1). The worker stays on GitHub Actions (D8); Vercel Cron is not used. |
| D12 | Event/boss IDs | **Our own IDs**, not Grimvane's. `sources/grimvane.py` maps Grimvane's event/boss identifiers to internal ones before anything reaches a task, the database, or the UI. A later source adapter maps to the same internal IDs. |
| D13 | Board content | The board is **a rolling "next 5 upcoming events" list**, using Discord timestamps, refreshed every run. This is why the 15-minute cron (D8) is fine: the board never needs to be instantly fresh, only correct as of the last run. Reminders (separate ping messages, D3) stay available per subscription but are best-effort with no minimum lead-time restriction — short-window events (hourly Shugo Festival, the 10-minute rift portal) may still get a late or skipped reminder; the board is the reliable display for those. |
| D14 | Catalogue delivery | The **worker writes `event_catalogue` and a new `server_catalogue` to Mongo** on each run. The web app only reads Mongo for these; it never holds a Grimvane key and never calls Grimvane. The UI has nothing to show until the worker's first run has populated both collections. |
| D15 | Schema ownership | **One JSON Schema file** (in a shared location, e.g. `schema/`) is the source of truth for every Mongo document shape. Mongo's `$jsonSchema` validator enforces it server-side; both the worker's Pydantic models and the web app's TypeScript types are checked against (or generated from) it, so the two languages can't silently drift apart. Conditional fields (e.g. `server_id`, required only when `field_boss_ids` is non-empty) are expressed as `if`/`then` in the schema, not folded into a flat `required` list. |
| D16 | Board scope | **One board message per subscription.** Two board-mode subscriptions are **not allowed to target the same channel** — the UI blocks creating a second one, rather than merging or silently duplicating messages in a channel. |
| D17 | Broken subscriptions | Add a **status field** (e.g. `status: "active" \| "needs_reconnection"`, `status_reason`, `status_since`) to each subscription. On a `403`/`404` when posting, the worker sets `needs_reconnection`, stops retrying that subscription until it is explicitly reactivated, and the web app surfaces it so the admin can fix access and re-enable it. **Detection is lazy and per-subscription** (D22): there is no separate per-guild membership check; a subscription only flips to `needs_reconnection` the next time *it* tries to post and fails. A whole guild losing the bot takes up to one cron interval to be fully reflected, spread across its subscriptions. |
| D18 | Board edit cadence | The board message is **edited every run**, even when the next-5-events list is unchanged, because the embed carries a "last checked `<t:UNIX:R>`" footer. That footer is the health signal admins use to see the bot is alive — a skipped edit would look identical to a stuck bot. (This supersedes the "skip if unchanged" option that was considered.) |
| D19 | Log redaction | Discord IDs, guild names, and other identifiers are **never logged in raw form** on this public repository. Where a log line needs to correlate the same ID across lines (e.g. "subscription X failed twice"), it logs `HMAC-SHA256(id, LOG_HASH_SALT)` instead of the ID, using a dedicated secret (`LOG_HASH_SALT`, section 8) that is never derivable from the logs themselves. Logs otherwise carry counts and error types, not identities. |
| D20 | Mentionable roles only | The UI only lists roles where Discord's `mentionable` flag is already `true`. The bot never requests the `Mention Everyone` permission. An admin who wants to ping a non-mentionable role must first make it mentionable in Discord (or the UI links to that setting) — the bot does not get a broader permission just to work around it. |
| D21 | Grimvane request throttling | `sources/grimvane.py` applies a **token-bucket limiter scoped to the API key** (below Grimvane's 60 req/min), retries on `429` using `Retry-After`, and **stores the last ETag per `(region, server)`**, sending it as `If-None-Match` on the next call — a `304` means "unchanged, reuse the previous result" and doesn't count against is as a wasted call. This is in addition to the existing in-run dedupe (one request per distinct `(region, server)` pair per worker run, noted in section 4.3). |
| D22 | Guild-kick detection | See D17 — detection is lazy and per-subscription, not an eager per-guild check. |
| D23 | Workflow concurrency | The GitHub Actions workflow sets `concurrency: { group: timers, cancel-in-progress: false }`, so if a run is delayed past the next scheduled trigger, the new run **queues** behind it instead of running in parallel or canceling it mid-post. |
| D24 | Subscription edit rights | **Any guild member with `MANAGE_GUILD` can edit any subscription in that guild** — rights follow the live Discord permission, not a stored "creator" field. Consistent with how Discord admin teams typically share server management. |
| D25 | Board size and deletion | **Assume "next 5 events" fits** Discord's embed limits. If a real board overflows, shrink the list then. If editing the board returns `404` (message deleted), the worker **posts a new board** and stores the new `board_message_id`. It does not mark the subscription broken for a deleted message. |
| D26 | Duplicate boards | An occasional second board (crash after post, before `board_message_id` is saved) is **fixed by hand**. No dedupe scan in v1. If it happens often, a later task can read the bot's own recent messages in the channel and collapse duplicates. |
| D27 | Pinning | **Boards are pinned.** The bot requests `Manage Messages`. After posting or recreating a board (D25), the worker pins that message. A failed pin does not fail the subscription. |

## 1. Goal

A Discord bot for Aion 2 communities. Phase 1 scope: **post game timer information to selected Discord channels**.

Two projects in one repository:

| Project | Tech | Purpose |
| --- | --- | --- |
| `web/` — Configuration UI | Next.js (App Router, TypeScript, Tailwind) | Discord server admins log in, add the bot to their server, choose channels and timer options. |
| `worker/` — Task runner | Python 3.12 | Runs scheduled tasks on GitHub Actions. Phase 1: fetch timers through `TimerSource` (Grimvane adapter) and post them to Discord. |

Both projects share one database (see section 5): the web app writes configuration and the worker reads it.

## 2. Repository layout (proposal)

```
ds-a2-companion/
├── docs/PLAN.md
├── web/                      # Next.js configuration UI
├── worker/                   # Python task runner
│   ├── pyproject.toml
│   └── src/a2_worker/
│       ├── main.py           # CLI entrypoint: `python -m a2_worker run <task>`
│       ├── tasks/            # one module per task (timers.py, ...)
│       ├── sources/
│       │   ├── timer_source.py   # protocol + internal models only
│       │   └── grimvane.py       # the only module that knows the Grimvane API
│       ├── discord_client.py # thin wrapper over Discord REST API
│       └── db.py             # typed Mongo repository methods
└── .github/workflows/
    ├── worker-timers.yml     # cron schedule
    └── ci.yml                # lint/test both projects
```

Monorepo (D1): the web app and the worker share one document shape.

## 3. Discord integration

### 3.1 Bot application

- One Discord application with a **bot user**, created in the Discord Developer Portal.
- The worker posts through the **Discord REST API** using the bot token (`POST /channels/{id}/messages`, `PATCH /channels/{id}/messages/{msg_id}`). It does **not** need a gateway/WebSocket connection, so it can run as a short-lived cron job.
- Required bot permissions: `View Channel`, `Send Messages`, `Embed Links`, `Manage Messages` (D27, so the board can be pinned). `Mention Everyone` is not requested (D20). Which channel types a subscription may target is deferred until after test runs.

### 3.2 Admin login and bot install (web)

- Admins log in with **Discord OAuth2** (`identify` + `guilds` scopes) via Auth.js (NextAuth).
- The UI lists the servers where the user has `MANAGE_GUILD` permission.
- An "Add bot" button uses the OAuth2 `bot` scope install link with the required permissions.
- Once the bot is in the server, the web backend uses the bot token to list text channels (`GET /guilds/{id}/channels`) and roles (`GET /guilds/{id}/roles`), so the admin picks both from dropdowns.
- The role picker only lists roles that are already "mentionable" in Discord (D20); non-mentionable roles don't appear, rather than the bot requesting `Mention Everyone` to ping them anyway.

### 3.3 Bot vs. webhooks, in detail

**Decision (D9): bot only.** The worker posts and edits with the bot token. The web app uses the same token to list channels and roles. No webhook URLs are stored.

Both can post and later edit a message. The comparison below is the reasoning for D9. The difference is in *what each one is allowed to do*, and that has direct consequences for D4 (role picker) and for channel discovery.

**How each works**

- **Bot.** One Discord application with a bot user and one bot token, shared across every server the bot joins. The worker calls `POST /channels/{channel_id}/messages` and `PATCH /channels/{channel_id}/messages/{message_id}` with `Authorization: Bot <token>`. The bot token also authorizes read endpoints like `GET /guilds/{id}/channels` and `GET /guilds/{id}/roles`, which is how the web app builds the channel and role dropdowns.
- **Webhook.** A per-channel endpoint created inside Discord (Server Settings → Integrations → Webhooks → New Webhook), bound to exactly one channel. Discord gives the admin a URL containing a webhook ID and a secret token. Posting is `POST /webhooks/{id}/{token}`, and the message it just sent can be edited with `PATCH /webhooks/{id}/{token}/messages/{message_id}`. A webhook token only authorizes actions on that one webhook — it cannot list a guild's channels or roles, and cannot see anything outside its own channel.

**Setup flow**

| | Bot | Webhook |
| --- | --- | --- |
| One-time setup | Admin clicks one OAuth "Add to Server" link, approves the requested permissions | Admin manually creates a webhook in Discord, per channel, and pastes its URL into our UI |
| Per additional channel | Pick from a dropdown already populated by the bot | Admin repeats the manual webhook creation in Discord for each new channel |
| What the admin must trust | A persistent member in their server with the permissions it was granted | A URL that behaves like a password for posting in that one channel |

**Channel and role discovery (D4)**

This is the deciding constraint: **a webhook alone cannot populate the channel or role dropdowns**, because a webhook token has no guild-wide read access. A webhook-only design would require the admin to manually copy-paste a channel's webhook URL *and* manually type a role ID for mentions, with no validation that the role exists or is mentionable — which does not meet D4. Satisfying D4 with webhooks still means running a bot for discovery, so "webhook-only" and "bot, with discovery" are the real alternatives, not "webhook vs. bot" as two equally capable options.

**Permission and trust surface**

- A bot's permissions are visible on the OAuth consent screen and apply everywhere the bot can see, even if we only request `View Channel`, `Send Messages`, and `Embed Links`. Some admins are cautious about adding a bot at all, regardless of how narrow the request is.
- A webhook has no permission bits of its own: it can only post and edit messages in the one channel it is bound to, nothing else. It reads nothing, sees no members, and can be revoked by deleting it in Discord with no OAuth flow involved. This is a smaller, easier-to-reason-about trust surface per channel.
- A webhook message can impersonate a different name and avatar per request (the `username` / `avatar_url` fields on `POST /webhooks/{id}/{token}`). Discord does mark it as a webhook message, but it is worth keeping our own branding fixed rather than letting it vary, since a changeable display name is more easily mistaken for a real member.

**Mentions and rate limits**

- Both can mention a role, using `allowed_mentions: {parse: ["roles"], roles: [...]}`. `@everyone` / `@here` still need the channel's `Mention Everyone` permission; that check does not apply to webhooks the same way it does to bot/member messages, since a webhook is not a guild member with permissions.
- Webhook execute/edit/delete share one rate-limit bucket per webhook (roughly 5 requests per 2 seconds). Bot messages are rate-limited per channel, with a separate global bot rate limit. At our cadence (one post or edit per channel per cron run), neither limit is a practical constraint.

**Resilience to revocation**

- If an admin deletes the bot's access (kicks it, or removes a permission), calls return `403`.
- If an admin deletes a webhook in Discord, calls return `404` and the webhook cannot be recovered — a new one has to be created and re-pasted.
- Either way, the worker must detect the failure, mark that subscription as needing reconnection, and stop retrying it until the admin fixes it, rather than failing the whole run.

**Growth path**

- A bot is a prerequisite for anything beyond posting: slash commands, buttons, or reacting to messages all require the Interactions system, which only exists for applications with a bot/command registration.
- A webhook can never host a slash command. If webhook-only is chosen now, adding interactive features later means introducing a bot anyway, in addition to the existing webhooks.

**Precedent:** the field-boss alert feature on the Grimvane site itself (seen while researching the data source) uses the webhook-URL-paste model, with up to five alerts per account and no bot involved. That is a legitimate design for a simple, single-purpose poster, but that feature also has no channel/role picker — the admin pastes a URL and picks from options the *site* already knows (its own event list and server list), not Discord's.

**Three realistic options**

1. **Bot only** (meets D4 as specified). One install click gives channel and role dropdowns for every subscription, in every server the bot is in. This is the only option that delivers the UI described in sections 3.2 and 7 without changes.
2. **Webhook only, with D4 relaxed.** Admin pastes a webhook URL per channel and types a role ID (or a `<@&role_id>` mention) by hand, since the UI can't fetch roles. Lower trust footprint per channel, but more manual setup per channel and no path to slash commands later.
3. **Hybrid: bot for discovery, webhook for posting.** The bot is installed only to read channels/roles for the dropdowns; posting uses a webhook created automatically through the bot's `Manage Webhooks` permission (`POST /channels/{id}/webhooks`) instead of the bot posting directly. This keeps the UX of option 1 while posting through a scoped-to-one-channel token instead of the general-purpose bot token. It is more moving parts (bot token and one webhook token per subscription) for a security benefit that may not matter much, since the bot's own posting permissions are already minimal.

D9 is option 1. Options 2 and 3 stay here as rejected alternatives.

## 4. Timer data source

**Decision (D6): Grimvane is the only source in v1, and it stays behind `TimerSource`.**

`tasks/` and the web app import internal models (`Event`, `Occurrence`) from `sources/timer_source.py`. They do not import Grimvane response types, URLs, or the API key. `sources/grimvane.py` is the only module that calls `https://aion2.grimvane.app`. A later source is a new adapter that implements the same protocol; tasks do not change.

```python
class TimerSource(Protocol):
    def list_events(self) -> list[Event]: ...
    def upcoming(self, region: str, server_id: str | None) -> list[Occurrence]: ...
```

The worker constructs `GrimvaneSource` in `main.py` and passes it into the task. Tests pass a fake `TimerSource`.

### 4.1 questlog.gg (not used)

questlog.gg has **no documented public API**. Its Nuxt front-end calls an internal tRPC API that currently answers plain HTTP requests (checked during planning):

- `GET https://questlog.gg/aion-2/api/trpc/fieldBoss.list?input={"language":"en"}` returns the field boss catalogue (id, name, map, level, drops, icon).
- `GET https://questlog.gg/aion-2/api/trpc/fieldBoss.timers?input={"server":<id>,"language":"en"}` returns per-server respawn timers. The server IDs still need to be discovered; an invalid ID returns `"Not a global server"`.
- Recurring events (Spacetime Rift every 3h, Shugo Festival hourly, daily/weekly resets, sieges, Nahma, etc.) appear to be **computed client-side from fixed schedules**. We would reimplement these schedules in Python (`sources/schedules.py`).

Risks:
- This API is unofficial and can change or be blocked (Cloudflare) without notice.
- It may conflict with questlog's terms of service.

questlog is not implemented. It remains a possible later adapter for richer boss info (drops, icons), and only as another `TimerSource`.

### 4.2 Grimvane API

`https://aion2.grimvane.app/api/v1` is a **documented, versioned** Aion 2 timers API ("Aion 2 Timers API v1"). It has an OpenAPI 3.1 spec at `/openapi.json`, which is public.

- **Routes:** `/servers` (regions with time zones, plus the game's server IDs, names and factions), `/next` (upcoming events), `/timers`, `/timers/{id}`, `/field-bosses` (live spawns per server from player reports, with stable boss IDs), `/calendar.ics`, and `/changes` / `/changes.atom` (API changelog).
- **Coverage:** Spacetime Rift, Shugo Festival, Dimensional Invasion, Abyss events, sieges (Artifact Siege, Executors, Nahma), daily/weekly resets, season dates, and field bosses. Every region uses its own server clock including daylight saving, and `region` is a parameter (`na-east`, `na-west`, `eu`, `sa`, `asia`).
- **Auth:** a free developer key, obtained by signing in with Discord and sent as `Authorization: Bearer`.
- **Limits:** 60 requests/minute per key. Replies are cached for 60 s (field bosses for 15 s). Replies include an ETag, so `If-None-Match` returns `304` when nothing changed.
- **Stability promise:** v1 only adds fields and routes, never breaking changes. Every reply carries an `API-Changes` header so a client can detect updates.
- **Testing:** `region=demo` with `server=1|2` returns a simulated field boss board for bot development.
- **Data quality:** field bosses are crowd-sourced from player reports, the same as questlog (whose data comes from desktop app users reading the world map).

### 4.3 Comparison and recommendation

| | questlog.gg (internal tRPC) | Grimvane API v1 |
| --- | --- | --- |
| Documented / supported for bots | No | **Yes** (OpenAPI, changelog, demo servers) |
| Breaking-change risk | High: internal routes can change with any deploy | Low: additive-only v1 promise |
| Recurring events (rift, festival, sieges, resets) | We must reimplement the schedules ourselves | **Provided**, per region, DST-aware |
| Field bosses per server | Yes (desktop-app crowd data) | Yes (player reports) |
| Item data (drops, icons) | **Yes, rich** | Limited |
| Auth | None | Free key |
| ToS risk | Unclear | None (intended use) |

One worker run requests each distinct `(region, server)` pair once and reuses the result for every subscription on that pair. On top of that in-run dedupe, D21 adds a per-key token-bucket limiter (with `429`/`Retry-After` backoff) and reuses Grimvane's own `ETag` across runs via `If-None-Match`, so the worker never relies solely on "usage is currently low" to stay under the 60 requests/minute key limit.

## 5. Storage

GitHub Secrets alone **won't work** for per-server configuration: secrets are write-only from the API, the web app can't read them back, and they don't scale to many Discord servers. Secrets will only hold credentials (bot token, DB URL).

Free database options. All three have permanent free tiers. The data is tiny (one row per guild/subscription), so storage limits don't matter.

| | MongoDB Atlas M0 | Neon (Postgres) | Supabase (Postgres) |
| --- | --- | --- | --- |
| Free storage | 512 MB | 0.5 GB | 500 MB |
| Idle behaviour | Always on | Scales to zero; first query after idle takes about 0.5–1 s | **Project pauses after 7 days idle** (our cron prevents this) |
| Network access | **IP allowlist required.** GitHub Actions and Vercel use changing IPs, so we must allow `0.0.0.0/0` (then only the password protects it) | Open by default (TLS + password) | Open by default (TLS + password) |
| Python client | `pymongo` (excellent) | `psycopg` | `psycopg` or `supabase-py` |
| Next.js client | Official driver; works on Vercel | `@neondatabase/serverless` + Drizzle; built for Vercel | `supabase-js` or Drizzle |
| Schema | Schemaless; validation in app code (Pydantic / Zod) | SQL migrations (Drizzle) | SQL migrations; dashboard table editor |
| Extras | — | Database branching | Auth, storage, admin UI (unused here) |

**Decision (D7): MongoDB Atlas M0.** The free tier has no expiry and 512 MB, which is more than this data needs. Subscriptions are nested on the guild document. Atlas requires an IP allowlist; GitHub Actions and Vercel have no fixed addresses, so the allowlist is `0.0.0.0/0` and the connection string is the credential. Atlas's shared/free-tier terms have changed before (plan availability, retirement of specific tiers) — re-confirm current terms at Phase 2 kickoff, since D7 depends on the tier staying free and persistent.

Both projects access Mongo only through typed repository methods. Filters are not written at call sites.

### 5.1 Document shape

```
guilds
  _id: guild_id (string)
  name, installed_by_user_id, created_at
  subscriptions: [
    {
      _id,
      channel_id,
      region,                 # D5: na-east | na-west | eu | sa | asia
      server_id,              # required when field bosses are selected
      board_enabled,          # D3
      reminders_enabled,
      events: [],             # event ids from the catalogue (D2)
      field_boss_ids: [],     # optional subset; empty means all bosses
      lead_minutes: [],       # reminders, e.g. [10, 5]
      mention_role_ids: [],   # D4
      locale, enabled,
      board_message_id,       # Discord message edited in place
      board_updated_at,
      created_at, updated_at
    }
  ]

event_catalogue
  _id: event_id
  name, category, updated_at   # worker refreshes this from TimerSource.list_events()

server_catalogue               # D14: populates the region/server picker (D5) without the web app calling Grimvane
  _id: region                  # na-east | na-west | eu | sa | asia
  timezone
  servers: [{ server_id, name, faction }]
  updated_at                    # worker refreshes this alongside event_catalogue, each run

sent_reminders
  subscription_id, event_key, event_start_at, sent_at
  unique index on (subscription_id, event_key, event_start_at)
```

## 6. Worker on GitHub Actions

**Decision (D8): GitHub Actions is the scheduler.** Vercel Cron on the free plan cannot run more than once a day (section 6.1b), so it is not used. A 15-minute cadence is enough: the shortest field-boss windows are on the order of 3 hours, and Discord timestamps keep a posted board correct between runs. The cron is `7,22,37,52 * * * *` so runs miss the top of the hour, which is when GitHub delays schedules most.

### 6.1 GitHub Actions schedule constraints

From the GitHub docs (`schedule` event):

- Shortest interval is **once every 5 minutes**. `* * * * *` does not run every minute.
- Runs are **best-effort**. During high load (especially the start of each hour) they are delayed, and queued runs can be **dropped**. Scheduling away from minute 0 reduces the chance of delay.
- A scheduled workflow runs only from the **latest commit on the default branch**.
- On a **public** repository, scheduled workflows are **disabled after 60 days** with no repository activity. Re-enable in the Actions UI, or by committing a change to the cron.
- Minutes are free on a public repository (D1), so run count is not a cost problem.
- Each run is a fresh machine: checkout and dependency install add tens of seconds before the task starts.

### 6.1b Vercel Cron is not a substitute on the free plan

Vercel Cron calls an HTTP route on a schedule. The route can be TypeScript or Python; the language does not change the schedule limits ([usage and pricing](https://vercel.com/docs/cron-jobs/usage-and-pricing)):

| | GitHub Actions (public repo) | Vercel Hobby | Vercel Pro |
| --- | --- | --- | --- |
| Minimum interval | 5 minutes | **Once per day** | Once per minute |
| Timing | Best-effort; can slip or be dropped | Any time within the scheduled hour | Within the scheduled minute |
| Cost | Free minutes | Free; a more frequent expression **fails the deploy** | Paid plan |
| Runtime | Full runner, any language | Function, max 300 s | Function, up to 800 s |

Moving the worker into the Next.js app and triggering it with Vercel Cron does **not** get around the GitHub limits. On Hobby it is stricter: one run per day, and that run can land anywhere in the hour.

What each kind of data needs:

- **Fixed schedules** (rift, festival, resets, sieges): a board that uses Discord timestamps stays correct after a single post. A daily refresh is enough.
- **Live field bosses**: times change when players report a kill. A daily job leaves those stale for up to a day. A run every 5–15 minutes is the useful range.

If the worker should live in the web app anyway, a free external scheduler (for example cron-job.org) can `GET` a protected route (`Authorization: Bearer <CRON_SECRET>`) every few minutes. That route does the same work as the Python task. The route must reject requests that lack the secret, because the URL is public.

### 6.2 Design consequences

- **Use Discord timestamps** (`<t:UNIX:R>` / `<t:UNIX:t>`). Discord renders countdowns and local times on the client, so a message posted late still shows correct times in each user's own timezone.
- **Prefer a "timer board" mode:** one message per channel is **edited in place** each run with upcoming events. A delayed cron doesn't matter because the countdowns are client-side.
- **Reminder mode** ("Rift in 10 min" plus a role ping) is best-effort. Each run sends reminders for any event whose `start - lead` falls between the last run and now (plus a grace window), deduplicated through `sent_reminders`. Reminders that arrive too late (after the event starts) are skipped.
- A weekly keep-alive commit or workflow avoids the 60-day auto-disable on a public repository.

### 6.3 Workflow sketch

```yaml
on:
  schedule: [{ cron: "7,22,37,52 * * * *" }]
  workflow_dispatch: {}
concurrency:
  group: timers          # D23: queue a delayed run instead of overlapping it
  cancel-in-progress: false
jobs:
  timers:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv run --project worker python -m a2_worker run timers
        env:
          DISCORD_BOT_TOKEN: ${{ secrets.DISCORD_BOT_TOKEN }}
          MONGODB_URI: ${{ secrets.MONGODB_URI }}
          GRIMVANE_API_KEY: ${{ secrets.GRIMVANE_API_KEY }}
```

### 6.4 Python stack

`uv`, `httpx`, `pydantic` (internal models and settings), `pymongo`, `pytest`, and `ruff`. Logs are structured JSON to stdout.

## 7. Web app

- Hosted on **Vercel Hobby** (D11).
- Next.js 15 App Router, Tailwind, and Auth.js with the Discord provider. Mongo access uses the official driver, through typed repository methods in `web/`.
- Pages:
  - `/` — landing page and "Login with Discord".
  - `/servers` — the admin's manageable servers, with bot install status.
  - `/servers/[guildId]` — the list of timer subscriptions.
  - `/servers/[guildId]/subscriptions/new` and `/[id]` — channel picker, region/server, mode, events, lead times, and role ping.
  - A "Send test message" button that posts immediately through the bot.
- Every write is checked server-side: the user must have `MANAGE_GUILD` on the guild (re-verified through the Discord API, not trusted from the client).
- Language is English only (D10). No i18n framework in v1.

### 7.1 Hosting: GitHub Pages cannot run this app

GitHub Pages publishes static files (HTML, CSS, JavaScript). It does not run a server, API routes, or background code. Published site size is at most 1 GB, bandwidth is a soft 100 GB per month, and GitHub's terms say Pages is not for running a commercial SaaS. Those limits are not the blocker. The blocker is that the configuration UI, as specified, has to keep secrets on a server:

| Need | Why a static host fails |
| --- | --- |
| Discord login | The OAuth client secret is exchanged for a token on the server. Putting it in the browser bundle makes it public. |
| Channel and role dropdowns (D4, D9) | Listing them uses the bot token. That token cannot be shipped to the browser. |
| Subscription writes | `MONGODB_URI` cannot be shipped to the browser. `MANAGE_GUILD` is re-checked on the server, not trusted from the client. |
| "Send test message" | Posting uses the bot token. |

A Next.js static export (`output: 'export'`) drops Route Handlers and Server Actions, which are exactly the pieces that hold those secrets. GitHub Pages can host a static landing page. It cannot host the admin UI in sections 3.2 and 7.

**Decision (D11): Vercel Hobby.** It runs the Next.js pages and the server routes that hold the Discord and Mongo secrets, on the free tier. The worker is not moved there. Scheduling stays on GitHub Actions (D8).

## 8. Secrets and configuration

| Name | Used by | Where |
| --- | --- | --- |
| `DISCORD_BOT_TOKEN` | worker, web | GitHub Secrets, Vercel env |
| `DISCORD_CLIENT_ID` / `DISCORD_CLIENT_SECRET` | web (OAuth) | Vercel env |
| `AUTH_SECRET` | web | Vercel env |
| `MONGODB_URI` | worker, web | GitHub Secrets, Vercel env |
| `GRIMVANE_API_KEY` | worker only | GitHub Secrets. Not exposed to the browser or the UI. |
| `LOG_HASH_SALT` | worker, web | GitHub Secrets, Vercel env. Used only to hash Discord IDs before logging (D19); never logged itself. |

## 9. Phases

### Phase 0 — Foundations
- [x] Agree on the open questions that block implementation (section 10) — the "minor, low-risk cleanup" list there is tracked separately and isn't a Phase 0 blocker
- [ ] Create the Discord application and bot; set up the DB project
- [ ] Repository scaffold: `web/`, `worker/`, CI workflow

### Phase 1 — Worker MVP (hard-coded config)
- [ ] **Spike (OQ1):** get a free Grimvane key; save real `/servers`, `/next`, `/timers`, `/field-bosses` responses (incl. `region=demo`) as fixtures; read Grimvane's terms. Do this before the two tasks below.
- [ ] `sources/timer_source.py`: protocol and internal `Event` / `Occurrence` models, using internal IDs (D12)
- [ ] `sources/grimvane.py`: the only Grimvane adapter (`/servers`, `/next` or `/timers`, `/field-bosses`)
- [ ] `discord_client.py`: post and edit messages with embeds and Discord timestamps
- [ ] `timers` task in board mode for a single channel from env vars, depending only on `TimerSource`. **No persistence yet:** each run posts a new message rather than editing one in place, since there is nowhere to store `board_message_id` before Phase 2's DB lands. D18's edit-in-place behavior (and the "last checked" footer) starts in Phase 2.
- [ ] GitHub Actions cron workflow (`7,22,37,52 * * * *`)

### Phase 2 — DB-backed config
- [ ] Atlas cluster, `0.0.0.0/0` allowlist, collections and the `sent_reminders` unique index
- [ ] Typed repositories in the worker; worker reads subscriptions and writes `board_message_id`
- [ ] Worker refreshes `event_catalogue` from `TimerSource.list_events()`
- [ ] Reminder mode with dedupe on `sent_reminders`

### Phase 3 — Configuration UI
- [ ] Discord OAuth login, server list, bot install
- [ ] Subscription CRUD, channel picker, test message
- [ ] Deploy to Vercel

### Later ideas
- Slash commands (`/timers`, `/boss`) — needs an Interactions endpoint (could be a Vercel route; no gateway needed)
- Additional tasks: patch notes / news posts, maintenance alerts

## 10. Open questions

D1–D11 are locked: repository (D1), events (D2), message style (D3), roles (D4), region (D5), Grimvane behind `TimerSource` (D6), MongoDB Atlas M0 (D7), GitHub Actions every 15 minutes (D8), bot only (D9), English only (D10), Vercel Hobby (D11).

An architecture review before implementation raised the items below. None are implemented yet; each needs a decision before the code it affects is written.

### Blocks worker code (Phase 1) — resolved

- ~~OQ1~~ **Resolved → spike task.** Added as the first Phase 1 task (section 9): get a free Grimvane key and save real `/servers`, `/next`, `/timers`, `/field-bosses` responses (including `region=demo`) as fixtures before writing `sources/grimvane.py` or finalizing `TimerSource`. Also read Grimvane's terms.
- ~~OQ2~~ **Resolved → D13.** The board is a rolling "next 5 events" list, not a point-in-time snapshot, so a 15-minute refresh is always correct. Reminders stay best-effort with no minimum lead.
- ~~OQ3~~ **Resolved → D12.** Internal event/boss IDs; `sources/grimvane.py` maps Grimvane's IDs to them.

### Should resolve before Phase 2–3

- ~~OQ4~~ **Resolved → D14.** Worker writes `event_catalogue` + `server_catalogue`; web app reads Mongo only.
- ~~OQ5~~ **Resolved → D15.** One JSON Schema is the source of truth, enforced by Mongo and checked by both languages.
- ~~OQ6~~ **Resolved → D16** for scope (one board per subscription, no two board subscriptions sharing a channel). Still open: does "next 5 events" always fit Discord's 6,000-character / 25-field embed cap once `field_boss_ids` expands to many bosses? What happens if the board message is deleted? Should an unchanged board skip the edit call? (Carried into OQ6b below.)
- ~~OQ7~~ **Resolved → D17.** Status field added; kicking the bot is handled per-subscription via that status, not a separate guild-wide flag — still need to decide (OQ7b) whether losing the *guild* entirely (bot kicked from the server, not just one channel) should flip every subscription in that guild to `needs_reconnection` in one pass, or be detected lazily per subscription on its next post attempt.

- ~~OQ6b~~ **Resolved → D18.** Board is edited every run regardless of content change, to keep a live "last checked" timestamp as a health signal.
- ~~OQ6c~~ **Resolved → D25.** Assume five events fit; recreate the board on `404`.
- ~~OQ7b~~ **Resolved → D17/D22.** Lazy, per-subscription detection; no eager per-guild check.
- ~~OQ8~~ **Resolved → D23.** `concurrency: { group: timers, cancel-in-progress: false }` added to the workflow.
- ~~OQ8b~~ **Resolved → D26.** Duplicate boards are fixed by hand; a self-scan is a later idea only.
- ~~OQ9~~ **Resolved → D19.** Hash Discord IDs with a dedicated secret (`LOG_HASH_SALT`) before logging; never log raw IDs or names.
- ~~OQ10~~ **Resolved → D20.** Only mentionable roles are selectable; the bot never requests `Mention Everyone`.
- ~~OQ11~~ **Resolved → D21.** Per-key token-bucket limiter with `429` backoff, plus ETag/`If-None-Match` reuse across runs, on top of the existing in-run dedupe.
- ~~OQ12~~ **Resolved → D24.** Any member with `MANAGE_GUILD` can edit any subscription in that guild.

- ~~OQ10b~~ **Resolved → D27** for pinning (boards are pinned; bot requests `Manage Messages`). Channel types stay **deferred** until after a few test runs; the channel picker must not lock a type filter before that.

### Minor, low-risk cleanup

- Pick one keep-alive mechanism for the 60-day auto-disable (scheduled commit vs. manual re-enable) and confirm it counts as activity.
- Separate Atlas database users for the worker and the web app, since the network allowlist is open to everyone.
- Note Discord's bot-verification requirement (ToS + privacy policy page) once the bot approaches 100 servers — not a v1 blocker.
- Add a one-line "done" criterion to each phase in section 9.
- Stale wording: section 2 still says "(proposal)"; section 5 still says "DB URL"; section 6.1b still offers cron-job.org as a live option; section 4.3's heading still says "recommendation" after D6 was decided; the Next.js version should be set at scaffold time rather than fixed to 15.
