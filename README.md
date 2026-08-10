# Style & Glitz AI Chatbot

A customer-facing AI shopping assistant built in n8n for [Style & Glitz](https://styleandglitz.netlify.app), a luxury e-commerce platform for fine watches and moissanite jewelry. Answers product questions, store policy questions, and — for logged-in customers — their own real order history. The guiding rule throughout: **the bot never invents a price, policy, or order detail.** Everything it states is either grounded in a live tool call or explicitly deferred to a human.

Paired with an embedded [`@n8n/chat`](https://www.npmjs.com/package/@n8n/chat) widget on the frontend, themed to match the site's own design system.

## Why this is more than "connect a chatbot to a database"

The interesting engineering problem here wasn't the chatbot itself — it was making sure an AI agent, reachable by anonymous website visitors, could safely look up **real customer PII** (order history, saved addresses) without a privileged key or a filter condition that could be wrong.

## Architecture

```
Chat Trigger (public webhook)
        │
        ▼
   AI Agent ("Style & Glitz Assistant")
   ├── Gemini Model (primary LLM)
   ├── Claude Model (automatic fallback if Gemini fails)
   ├── Conversation Memory (5-message window)
   ├── Product Catalog Lookup (anon key — public data)
   ├── Order History Lookup   (customer's own real session token)
   ├── Customer Profile Lookup (customer's own real session token)
   └── Order Items Lookup      (customer's own real session token)
        │
        ▼
   Response Check (is the reply actually non-empty?)
   ├── yes → reply sent to customer
   └── no, or Agent hard-errored → Fallback Message
                                    "Sorry, I'm currently unavailable.
                                     Please reach out to us directly..."
```

## The security problem, and how it actually got solved

The product catalog is public data — no issue using a shared, RLS-scoped anon key for that lookup.

Order history and customer profile data are not public. The first version of those tools used the same shared anon-key credential, with a `customer_id = <value>` filter built into the tool call. It *looked* secure. It wasn't — real Row Level Security in Postgres blocks the anonymous role from these tables entirely (`auth.uid()` evaluates to null for it), so the filter never actually got a chance to matter. The tool just silently returned nothing.

**The real fix:** the lookup tools now call Supabase's REST API directly using the *customer's own real, live access token* as the bearer credential — passed through from the frontend's actual Supabase Auth session, not typed or guessed by the AI. This means Row Level Security evaluates `auth.uid()` as the real logged-in customer, and enforces ownership exactly the same way it does for the website itself. No privileged key sitting around. No filter condition that has to be written correctly forever and never audited again.

## Two-layer failure handling — tested against real failures, not simulated ones

**Model-level:** the Agent has `needsFallback` enabled with Claude wired in as a second language model. If Gemini fails — quota, billing, outage — n8n retries automatically on Claude, invisibly, within the same turn.

**Total-failure-level:** testing surfaced a real gap — the Agent's own fallback logic can swallow a *total* failure (both models down) and return what looks like a successful item, just with an `error` field and no actual `output`. That would have silently bypassed the intended error handling. The `Response Check` node exists specifically to catch this: it checks that `$json.output` is genuinely non-empty, and routes to the same fallback message if not.

Both paths have fired for real in production — not a simulated test. A customer's message came in while both models were genuinely unavailable, and they received the graceful fallback message instead of a broken chat.

## Why product lookups and order lookups are architecturally different tools

It would have been simpler to give the agent one generic "query Supabase" tool. Deliberately didn't:

- **Product Catalog Lookup** — public data, shared anon-key credential, `returnAll: true`, no filter. Safe by nature of what it exposes.
- **Order History / Customer Profile / Order Items Lookup** — PII, so each one authenticates as the *specific customer asking*, not a shared credential. Order Items Lookup additionally relies on Postgres RLS — not the model — to guarantee a supplied order ID actually belongs to the caller, even if the model were tricked into requesting someone else's order ID.

## Why product/policy lookups happen before the model ever answers

The system prompt explicitly instructs the agent to ground every price, stock, material, certification, and policy claim in what its tools return — not memory, not a plausible guess. A dynamic line in the prompt (`{{ $json.metadata.customerId ? ... : ... }}`) tells the model, on every single turn, whether the current visitor is actually logged in — so it never attempts an order lookup for an anonymous visitor, and never has to be told twice mid-conversation.

## Known limitation

There's currently no separate rate-limiting layer beyond the model-level quota itself — a logged-in customer could in theory send requests fast enough to hit the same limits the fallback system is designed to catch gracefully. A production version at real scale would add per-user request throttling before the model call, not just handle the failure after it happens.

## Setup

1. Import `style-glitz-ai-chatbot.json` into your n8n instance.
2. Reconnect your own Gemini, Anthropic, and Supabase credentials — the ones in this file are placeholders.
3. Replace `YOUR_PROJECT_REF` in the three HTTP Request tool nodes with your actual Supabase project reference, and `YOUR_SUPABASE_ANON_KEY` with your project's real anon key (safe to use here specifically *because* RLS is what actually enforces access — see the security section above).
4. On your frontend, pass the logged-in customer's `customerId` and Supabase `accessToken` into the chat widget's `metadata` — see [`@n8n/chat`](https://www.npmjs.com/package/@n8n/chat)'s `metadata` option. Never let the AI or the customer's own message set these values themselves.
5. Webhook IDs regenerate automatically on import — expected behavior.

## Stack

n8n · Google Gemini API · Anthropic Claude API · Supabase (Postgres, Auth, Row Level Security) · `@n8n/chat`
