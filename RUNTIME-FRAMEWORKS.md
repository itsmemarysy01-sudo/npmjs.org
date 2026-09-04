# Runtime Frameworks

What actually executes this platform, why each piece was chosen over
its alternatives, and what changes if you swap one out. Maps onto the
Runtime Layers from the architecture manual:

```
Layer 0  Telegram Cloud          — fixed, not a choice
Layer 1  Bot API                 — chosen over MTProto/TDLib (see below)
Layer 2  Webhook Transport        — chosen over long polling (see below)
Layer 3  grammY Runtime           — chosen over Telegraf/node-telegram-bot-api/python-telegram-bot
Layer 4  Middleware Pipeline       — grammY Composer
Layer 5+ Everything above         — this project's own code (src/lib, src/modules)
Deploy   Cloudflare Workers        — chosen over Deno Deploy/Fly.io/Railway/traditional VPS
Storage  Cloudflare KV             — chosen over Durable Objects/D1/external DB
```

---

## Layer 1 — Bot API vs. MTProto/TDLib

| | Bot API | TDLib / MTProto |
|---|---|---|
| Credentials | `BOT_TOKEN` only | `api_id` + `api_hash` + phone number |
| Identity | Bot account | Full user account |
| Hosting | Any HTTP-capable runtime, including edge/serverless | Needs a persistent process + local DB directory |
| Capability ceiling | Everything a bot can do (messages, media, moderation, forums, mini apps) | Everything a *user* can do (broader, but a different trust/legal surface) |
| Fit for this project | ✅ chosen | Would require abandoning edge deployment |

This project only ever needs bot-level capabilities (Community,
Content, Broadcast, Support, Scheduler, Automation, Analytics, Audit
are all things a bot account can do), so Bot API is the correct and
simpler choice. TDLib would only become necessary if a future
requirement needed user-account behavior (e.g. joining chats as a
real user, reading full chat history predating the bot's membership).

## Layer 2 — Webhook vs. long polling

| | Webhook (chosen) | Long polling |
|---|---|---|
| Runtime shape | Stateless HTTP handler, invoked per update | Long-lived process continuously calling `getUpdates` |
| Fits serverless/edge | ✅ yes — this is *why* Workers/Deno Deploy work at all | ❌ no — needs an always-on process |
| Latency | Telegram pushes immediately | Small polling-interval delay |
| Requires public HTTPS | Yes | No |

Webhook is the only viable transport for a platform targeting
card-free edge hosting, which is the project's stated hard constraint.

## Layer 3 — grammY vs. other bot frameworks

| | grammY (chosen) | Telegraf | node-telegram-bot-api | python-telegram-bot |
|---|---|---|---|---|
| Language | TypeScript/JavaScript | JavaScript | JavaScript | Python |
| Edge/serverless adapters | First-class (Cloudflare Workers, Deno Deploy, Vercel, AWS Lambda, etc.) | Possible but less first-class | Primarily long-polling oriented | Needs a Python runtime (not available on Cloudflare Workers) |
| Middleware model | Composer-based, matches Express/Koa mental model | Similar middleware model | Callback-based, older API surface | Handler/dispatcher based |
| Plugin ecosystem | Sessions, conversations, rate limiting, i18n, etc. | Mature, large ecosystem | Minimal | Very mature (older project) |
| Type safety | Strong (built TypeScript-first) | Partial | Minimal | N/A (Python) |

grammY was chosen specifically for its Cloudflare Workers adapter
(`webhookCallback(bot, "cloudflare-mod")`, used in `src/index.js`) and
its Composer middleware model, which is what let this project's Layer
8 modules (`src/modules/*.js`) each be built as an independent,
mountable `Composer` — see `src/bot.js` for how they're wired together.

**If you outgrow grammY:** the Layer 8 module logic (business rules,
KV reads/writes, audit/metrics calls) is framework-agnostic in spirit
— only the outermost handler registration (`composer.command(...)`,
`composer.on(...)`) is grammY-specific. Porting to Telegraf would mean
rewriting the registration calls per module, not the business logic
inside them.

## Deployment platform — Cloudflare Workers vs. alternatives

| Platform | Free tier, no card at signup | Cron Triggers | KV/persistent storage | Fit here |
|---|---|---|---|---|
| **Cloudflare Workers** | Yes | Yes (native) | Yes (Workers KV) | ✅ chosen — everything this project needs, natively |
| Deno Deploy | Yes | Via external cron ping or Deno KV + Deploy's own cron (check current docs) | Deno KV | Viable alternative — see swap notes below |
| Fly.io / Railway / Render | No — card required | Varies | Varies | Ruled out by the card-free requirement |
| Traditional VPS | Depends on provider | Yes (crontab) | Anything you install | Works, but reintroduces the "persistent process to babysit" problem this architecture avoids |

Verify current free-tier terms before committing — they change. This
table reflects the state described in this project's README as of
initial setup; recheck each platform's pricing page directly.

### Swapping to Deno Deploy

1. Replace `src/index.js`'s Workers `fetch`/`scheduled` exports with a
   Deno entrypoint using `Deno.serve` and grammY's
   `webhookCallback(bot, "std/http")` adapter.
2. Replace the `PLATFORM_KV` Workers KV binding with Deno KV
   (`Deno.openKv()`) — update `src/lib/state.js`'s `createStore()` to
   wrap Deno KV's `get`/`set`/`list` instead of the Workers KV API.
   Every module downstream (`ctx.store.get/put/listValues/...`) is
   unaffected since they only depend on `state.js`'s interface.
3. Replace the Cron Trigger with whatever Deno Deploy's current
   scheduled-execution mechanism is (check `deno.com/deploy/docs` —
   this has changed across Deno Deploy versions).
4. `src/lib/security.js`, `src/lib/audit.js`, `src/lib/observability.js`,
   and every file in `src/modules/` require **no changes** — they only
   touch `ctx.store`/`ctx.api`/`ctx.audit`/`ctx.metrics`, not the
   underlying platform.

This is the intended payoff of the layering: only the Layer 0–2
transport/hosting code (`src/index.js`) and the Layer 9 storage
adapter (`state.js`'s `createStore`) are platform-specific. Layers 8
and 10–11 (business modules, security, observability) are portable.

## Storage — Workers KV vs. alternatives

| | Workers KV (chosen) | Durable Objects | D1 (SQL) | External DB |
|---|---|---|---|---|
| Consistency | Eventually consistent, no compare-and-swap | Strongly consistent, single-writer per object | Strongly consistent (SQLite semantics) | Depends |
| Query model | Key-value + prefix listing | Custom (you write the logic) | Full SQL | Full SQL/whatever |
| Cost/complexity | Simplest to provision and reason about | More setup, per-object routing | More setup, schema migrations | Most setup, separate hosting |
| Fit for this project's scale | ✅ chosen — community/small-team bot traffic | Overkill unless you need strict per-ticket/per-job locking | Reasonable if you want relational queries over tickets/audit later | Only if integrating with existing external systems |

The known limitation is documented inline in `src/lib/state.js`:
`nextId()`'s auto-increment can race under high concurrent writes on
the *same* sequence (e.g. many simultaneous new support tickets in the
same second). If that becomes a real bottleneck, migrate just the
affected ID-generation calls to a Durable Object rather than replacing
KV wholesale — the rest of Platform State doesn't need the stronger
guarantee.

---

## Summary: what's fixed vs. what's swappable

**Fixed by Telegram itself:** Layer 0 (Telegram Cloud), the Bot API
surface.

**This project's considered choices, each swappable independently:**
Bot API over TDLib (Layer 1) · Webhook over long polling (Layer 2) ·
grammY over other bot frameworks (Layer 3) · Cloudflare Workers over
Deno Deploy/VPS (hosting) · Workers KV over Durable Objects/D1
(storage).

The module boundaries in `src/lib/` and `src/modules/` exist
specifically so that swapping any one of these doesn't cascade into a
rewrite of the business logic in Layer 8.
