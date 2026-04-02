# Jimmy's Cigars — Website

The website for Jimmy's Cigars, Melbourne FL. Single-file HTML/CSS/JS, no framework dependencies.

## What's built

- Hero section with Melbourne FL identity
- About section
- Daily News spotlight (Córdoba & Morales, Z's blend — Connecticut, Habano, Maduro Toro)
- Brands showcase: Perdomo, My Father, Crux, Roma Craft Tobac, Ramon Bueso, Daily News
- The Lounge section
- Contact section with Google Maps embed and Formspree contact form (`xdaplyjz`)
- `static/image.jpg` — shop photo

---

## Roadmap

---

### Phase 1 — Backend API (Foundation for Everything Else)

Before Square or AI can work, we need a small backend that lives between the browser and external services. This keeps all API keys server-side and gives us one place to add caching, auth, and provider swapping later.

**Stack:** Cloudflare Workers (free tier, deploys in seconds, lives at the edge)

**Structure:**
```
/worker
  index.js          ← router — all requests come in here
  square.js         ← Square API calls
  ai.js             ← model-agnostic AI handler (see Phase 3)
  cache.js          ← simple KV-backed cache helper
  system-prompt.js  ← Jimmy's personality + inventory injection
```

**Endpoints the worker exposes to the frontend:**
```
GET  /api/inventory        ← returns formatted cigar inventory from Square
POST /api/chat             ← sends messages to the configured AI provider
```

**Config (environment variables in Cloudflare dashboard — never in code):**
```
SQUARE_ACCESS_TOKEN
SQUARE_LOCATION_ID
AI_PROVIDER          ← "claude" | "openai" | "gemini" (swap anytime)
AI_API_KEY
```

---

### Phase 2 — Square Inventory Integration

Pull live inventory from Square so the brands grid reflects what's actually in the case.

**How it works:**
1. Frontend calls `GET /api/inventory`
2. Worker checks KV cache (15-min TTL)
3. On cache miss, calls Square `GET /v2/catalog/list?types=ITEM`
4. Filters to cigar items, shapes response, writes to cache, returns JSON
5. Frontend renders brand cards from the response instead of the hardcoded JS array

**Square response → brand card mapping:**
```
catalog item name        → brand-name
item description         → brand-desc
variations (sizes)       → vitola tags
variation price          → price display
inventory count > 0      → "In Stock" badge
inventory count = 0      → "Ask Us" badge
custom attribute: origin → brand-origin
custom attribute: strength → strength meter (1–5)
custom attribute: profile  → flavor tags
```

**What changes on the frontend:**
- `renderBrands()` fetches `/api/inventory` first, falls back to the hardcoded array if the call fails (so the page never breaks)
- Brand cards get an in-stock/out-of-stock indicator
- Vitola chips become real data (e.g., Toro, Robusto, Churchill per brand)

**Square setup needed:**
- Enable Square Catalog API in the developer dashboard
- Add custom attributes for `origin`, `strength`, `profile` to cigar catalog items (or we can map by category/tag naming convention)
- Create a `Cigars` category so we can filter cleanly

**Square API ref:** https://developer.squareup.com/docs/catalog-api/what-it-does

---

### Phase 3 — Model-Agnostic AI Assistant

A chat widget on the page. The backend handles all AI calls behind a provider abstraction — swapping from Claude to OpenAI to Gemini is a one-line config change.

#### 3a — Provider abstraction layer

All providers implement the same internal interface in `ai.js`:

```js
// ai.js
const providers = {
  claude: async ({ messages, system }) => {
    // POST to api.anthropic.com/v1/messages
    // returns { role: "assistant", content: string }
  },
  openai: async ({ messages, system }) => {
    // POST to api.openai.com/v1/chat/completions
    // returns { role: "assistant", content: string }
  },
  gemini: async ({ messages, system }) => {
    // POST to generativelanguage.googleapis.com/...
    // returns { role: "assistant", content: string }
  }
};

export async function chat({ messages, system, env }) {
  const provider = providers[env.AI_PROVIDER];
  if (!provider) throw new Error(`Unknown provider: ${env.AI_PROVIDER}`);
  return provider({ messages, system, env });
}
```

The rest of the worker never touches provider-specific code. To switch models, update `AI_PROVIDER` and `AI_API_KEY` in the Cloudflare dashboard — no deploy needed.

#### 3b — System prompt

Built fresh on each request in `system-prompt.js`, combining static personality with live inventory:

```
You are the AI assistant for Jimmy's Cigars in Melbourne, FL —
the Cheers of cigar lounges on the Space Coast.

Your job is to help customers find the right smoke or a good
alternative if we don't carry what they're looking for.
Be direct and friendly. No pressure, no upselling.
If you don't know something, say so.

Current inventory:
{inventory injected as JSON from Square cache}

Brand knowledge:
{static brand profiles for the big six + common alternatives}
```

#### 3c — Chat widget (frontend)

A floating button in the bottom-right corner opens a slide-up panel:

```
[💬]  →  opens  →  ┌─────────────────────┐
                    │  Ask Jimmy's        │
                    │─────────────────────│
                    │  [message bubbles]  │
                    │                     │
                    │  [input] [Send]     │
                    └─────────────────────┘
```

- Conversation history held in `sessionStorage` (survives refresh, cleared when tab closes)
- Each user message posts to `POST /api/chat` with the full history
- Streams the response if the provider supports it (shows typing indicator otherwise)
- On any API failure, falls back to: *"Something went wrong — come ask us in person, we don't bite."*

**Conversation flows to support:**
| Customer asks | Assistant does |
|---|---|
| "What do you have that's mild?" | Recommends from in-stock inventory, strength ≤ 2 |
| "Do you carry Arturo Fuente?" | Honest no, suggests closest alternative we do carry |
| "I like Padróns, what else would I like?" | Matches on profile (full, Nicaraguan, cocoa/pepper) |
| "What's the Daily News?" | Tells Z's story, describes the three wrappers |
| "How much is a Perdomo Toro?" | Pulls price from Square inventory data |

**Stretch goals:**
- Staff mode toggle (password-gated) with inventory management prompts
- "Save this recommendation" → adds to a local wishlist, shareable URL
- Weekly email digest of chat questions to surface what customers are asking for

---

### Phase 4 — Polish & Hosting

- Custom domain (add to Cloudflare, point DNS)
- Deploy frontend to Cloudflare Pages (git-push-to-deploy, free)
- Worker auto-deploys alongside via `wrangler.toml`
- Meta tags + Open Graph image for social sharing
- Mobile nav hamburger menu
- Google Analytics or Plausible (privacy-friendly) for traffic
