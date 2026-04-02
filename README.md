# Jimmy's Cigars — Website

The website for Jimmy's Cigars, Melbourne FL. Single-file HTML/CSS/JS, no framework dependencies.

## What's built

- Hero section with Melbourne FL identity
- About section
- Daily News spotlight (Córdoba & Morales, Z's blend — Connecticut, Habano, Maduro Toro)
- Brands showcase: Perdomo, My Father, Crux, Roma Craft Tobac, Ramon Bueso, Daily News
- The Lounge section
- Contact section with Google Maps embed and Formspree contact form (`xdaplyjz`)

---

## Roadmap

### Phase 1 — Square Inventory Integration

Pull live inventory from the Square Catalog API so the brands grid reflects what's actually in the case.

**Plan:**
- Set up a lightweight backend (Cloudflare Worker or Netlify Function) to proxy Square API calls — keeps the API key server-side
- Fetch from `GET /v2/catalog/list` filtered by category (cigars)
- Map Square catalog items to brand cards: name, description, price, in-stock status
- Add a "In Stock" / "Ask Us" badge to each brand card
- Show individual vitolas/sizes per brand when available
- Cache responses (5–15 min) to avoid hammering the API on every page load

**Square API docs:** https://developer.squareup.com/docs/catalog-api/what-it-does

---

### Phase 2 — AI Cigar Assistant

An interactive chat widget on the page that knows the current inventory and can help customers find the right smoke or suggest alternatives for things we don't carry.

**Plan:**
- Add a floating chat button (bottom-right) that opens a slide-up chat panel
- Feed the assistant a system prompt that includes:
  - Jimmy's personality (the Cheers of cigar lounges, straight talk, no pressure)
  - Current inventory pulled from Square (updated each session)
  - Brand knowledge for the big six and common alternatives
- Use the Claude API (`claude-sonnet-4-6`) via a Cloudflare Worker or Netlify Function to keep the API key off the client
- Conversation flows to support:
  - "What do you have that's mild?" → recommends from live inventory
  - "Do you carry [Brand X]?" → honest yes/no, suggests closest alternative if not
  - "I usually smoke [Brand Y], what else would I like?" → similarity matching from brand profiles
  - "What's the Daily News?" → tells the origin story, explains the three wrappers

**Stretch goals:**
- Persist chat history in `sessionStorage` so it survives page refreshes
- "Add to wishlist" button on AI recommendations (local storage, shareable link)
- Staff mode: same assistant but with inventory management prompts

---

### Phase 3 — Polish & Hosting

- Move to a proper domain (jimmyscigars.com or similar)
- Host on Cloudflare Pages (free, fast, git-deploy)
- Add Google Analytics or Plausible for basic traffic tracking
- Meta tags / Open Graph for social sharing
- Mobile nav hamburger menu
