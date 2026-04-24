# Cue

**An AI agent that turns life moments into coordinated orders across Swiggy Food, Instamart, and Dineout.**

Users describe an occasion in natural language — a sister's birthday, a friend down with the flu, in-laws visiting Saturday. Cue plans the timeline, fills the right carts across the right Swiggy services, and executes in a single flow.

This is a design document for a project being submitted to the Swiggy Builders Club.

---

## The problem

Humans don't think in app categories. They think in moments.

"My sister's birthday is Saturday" is one thought in a user's head, but on Swiggy today it decomposes into at least three separate shopping sessions: flowers and cake on Instamart, a nice dinner via Food, maybe a Dineout booking if she's local. Each session requires rebuilding context — who it's for, the budget, the timing, the vibe. Nothing carries across.

The MCP platform is the first time an agent can hold the *whole moment* as a single unit of intent and resolve it into coordinated action across all three services. That's the opportunity Cue exists to capture.

## What Cue does — three archetypes for v1

I'm deliberately scoping v1 to three occasion types. They cover the emotional range of what people actually ask for, they each exercise a different mix of the three MCPs, and together they prove the "one brain, three servers" thesis.

**Birthday (remote).** User prompt: *"My sister's birthday is Saturday, she's in Bangalore, I'm in Pune, ₹2000 budget, she loves chocolate and Thai food, want it to be a surprise."* Cue proposes a timeline — morning flowers, afternoon cake, evening Thai dinner — and executes across Instamart and Food, all delivered to her address. Uses Instamart + Food, optionally Dineout if the user is local and wants a booking instead.

**Sick friend / care package.** User prompt: *"My friend Arjun has the flu, wanted to send something."* Cue assembles a care package — hot khichdi or soup from Food, fruits and electrolytes and tissues from Instamart — and coordinates delivery to his address in a single window. Uses Food + Instamart.

**Hosting at home.** User prompt: *"Having six people over Saturday evening for drinks."* Cue handles the logistics — snacks, ice, mixers, napkins on Instamart for same-day delivery, and optionally a Dineout booking as a "Plan B venue" if the user wants to move it out. Uses Instamart + Dineout.

Festivals (Diwali, Rakhi, Karva Chauth) are an obvious and India-specific extension, but they're seasonal and would hurt year-round demo-ability. I'm holding them for v2.

## User flow — walking through the birthday archetype

1. **Intake.** User opens Cue and types or speaks the occasion in one message. No forms.
2. **Clarify.** Cue asks at most two follow-up questions where the answer genuinely changes the plan — typically around timing ("morning flowers or evening?") and recipient address if not already known.
3. **Plan generation.** Cue returns a single "plan card" — a visual timeline showing each item, its delivery window, the service it's coming from, and the running total against budget. This is the core UI moment of the product.
4. **Tweak loop.** User can swap any item ("different cake," "no flowers"), drag sliders to rebalance the budget, or change the delivery window. Cue re-plans.
5. **Approval.** One tap confirms the whole plan. Cue creates carts and places orders across the relevant MCP servers in parallel.
6. **Live tracking.** User sees all three (or two) orders in one tracking view. If anything fails mid-flight — a restaurant closes, an Instamart item goes out of stock — Cue surfaces it immediately with a proposed fix rather than silently dropping it.

## Architecture

```
┌─────────────────┐       ┌──────────────────┐       ┌──────────────────────┐
│                 │       │                  │       │  Swiggy MCP Servers  │
│  React frontend │ ◄───► │ FastAPI backend  │ ◄───► │  • Food              │
│  (chat + plan   │       │ (agent orchestr- │       │  • Instamart         │
│   card UI)      │       │  ator)           │       │  • Dineout           │
└─────────────────┘       └──────────────────┘       └──────────────────────┘
                                   │
                                   ▼
                          ┌──────────────────┐
                          │  Claude API      │
                          │  (agent brain)   │
                          └──────────────────┘
```

**Frontend** — React + TypeScript. A conversational surface plus the plan card component.

**Backend** — FastAPI. Holds the MCP credentials, handles auth flows, and orchestrates across the three MCP servers. The frontend never talks to MCP directly — this is important for both security posture and for keeping tool-call logic server-side where it can be reasoned about and rate-limited.

**Agent brain** — Claude via the Anthropic API. Given the occasion prompt plus the available MCP tools, Claude produces the plan, decides which tools to call in what order, and handles the tweak loop. The backend mediates tool execution and returns results.

**State** — a lightweight session store (Redis or Postgres) for in-flight plans, cart IDs, and order tracking handles.

## Hard problems and how I'm planning to handle them

These are the four places where this product either works or falls apart. Calling them out explicitly so the thinking is visible.

**Address handling.** The recipient often isn't the user — a birthday gift goes to the sister, a care package goes to the sick friend. The first thing I'll verify on API access is that the MCP tools accept an arbitrary delivery address per order (Swiggy's consumer app already supports gift delivery, so the primitive should exist). UX-side, Cue will ask for the recipient's address once per occasion and reuse it across all orders in that plan.

**Timing choreography.** Instamart is instant, Food can be scheduled, Dineout is a fixed reservation slot. Coordinating "flowers at 11am, cake at 5pm, dinner at 8pm" across three delivery models is the interesting orchestration problem. For v1 I'll take a pragmatic approach: the user picks the primary time window (say, "evening"), and Cue serializes the other items backward from that anchor using each service's known delivery characteristics. Full independent scheduling is a v2 problem.

**Budget decomposition.** If the user says ₹2000, how does it split across three carts? I'll start with occasion-type priors (birthday ≈ 30% flowers/card, 35% cake, 35% meal; care package ≈ 60% food, 40% groceries; hosting ≈ 100% Instamart unless Dineout is chosen). The plan card will expose sliders so the user can rebalance before approving. Learning personalized priors is a v2 problem.

**Failure handling.** A Dineout slot gets taken, an Instamart item is out of stock in the recipient's pincode, a restaurant closes. Cue needs to fail gracefully with a clear user-facing message and a proposed Plan B — never silently drop an item from a plan the user already approved. I'll implement this as an explicit state in the order-execution pipeline: any tool-call failure pauses the plan, surfaces the issue, and asks for a one-tap resolution.

## Explicitly out of scope for v1

- Festival packages (Diwali, Rakhi, etc.) — seasonal, held for v2
- Corporate gifting at scale — different auth model, different product
- WhatsApp as a distribution channel — huge opportunity but adds Meta approval complexity that doesn't belong in v1
- Recurring occasions ("remind me for Mom's birthday next year") — needs a notification layer and a persistent user model
- Group/shared planning — valuable but makes the data model much more complex

## Roadmap

- **Weeks 1–2:** Intake flow + plan generation with mocked product data. Get the plan card UI right — that's the demo's money shot.
- **Weeks 3–4:** Real MCP integration for the birthday archetype end-to-end. One archetype working for real beats three working on mocks.
- **Weeks 5–6:** Extend to sick-friend and hosting archetypes. Live tracking across services.
- **Weeks 7–8:** Polish, failure-handling edge cases, a clean demo video for the Builders Club submission.

## Compliance notes

Cue is designed to stay inside the Builders Club guidelines:

- Swiggy is clearly the merchant on every order — no aggregation layer, no brand hiding. The plan card names which Swiggy service each item comes from.
- No price, availability, or delivery-time rewriting — whatever the MCP returns is what the user sees.
- No ranking manipulation, no scraping, no competitive benchmarking.
- User data from MCP stays governed by Swiggy's platform terms and is not used for any purpose beyond fulfilling the user's own occasion.

---

**Contact** — happy to walk through any of this on a call. Cue is being built by a solo developer as a Swiggy Builders Club submission.
Name: Shreyas Chitransh
Email: shreyaschit15@gmail.com
Linkedin: https://www.linkedin.com/in/shreyaschitransh/
Github: https://github.com/Shreychit
