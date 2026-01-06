---
layout: post
title: Building with agentic AI, a Defence of Britain story
date: 2026-01-01 00:00:00 +0000
summary: "Building Defence of Britain: a map-first web app (from file-new-project to a functioning app)"
minutes: 15
---

>TL;DR - Try the app here: [https://defence.tobyreid.co.uk](https://defence.tobyreid.co.uk). If the app is slow on first load, it's because hosting is FREE!

Back in Dec/Jan 2020 I built the first version of **Defence of Britain**: a hobby app that read the publicly available [Pillbox Study Group KMZ](http://www.pillbox-study-group.org.uk/links/downloads/) and put it on a map. It worked, but I was never truly happy with it - and at the time I didn't know of any other hobby projects doing the same thing.

Fast‑forward a few years: other people created similar (and better) applications, and my original project quietly gathered dust. Here's an excellent one from [Matt Aldred](https://edob.mattaldred.com/).

With the recent rise of agentic AI tooling (Claude Code, Cursor, GitHub Copilot), it felt like the right moment to revisit the idea - not just to fix what bothered me about the old version, but to see what it looks like to build _with_ an agentic assistant while using my own experience to avoid dead‑ends.

I started from a single SFC (Single File Component) in an otherwise blank GitHub repo. This is the story of building the app again.

Crucially, given my day job and how time‑poor I am, I had to keep this tightly timeboxed into a small number of focused sessions over the holiday period. That constraint ended up shaping the implementation style: fewer big rewrites, more "get one thing working end‑to‑end, then refine".

## What were the goals?

The first goal was simple: make it easy to explore UK WWII defence locations on an interactive map, make locations shareable, and make it _feel_ like an app.

The second goal was to keep it _cheap_. I didn't want to pay for expensive hosting, or incur charges for containers sitting idle in the background.

The third and final goal was to deliberately work _with_ agentic AI (GitHub Copilot) rather than against it - letting it move fast on implementation, while using my own experience to spot wrong turns early and swerve away from dead‑ends.

## Context matters

One thing that made GitHub Copilot genuinely useful (rather than just fast) was giving it the right context up front.

Instead of treating the frontend, backend, and infrastructure as separate worlds, I created a single VS Code workspace that included:

- the frontend repo
- the backend repo
- the Terraform repo

Then I added two kinds of "guidance" that an agent can actually use:

- GitHub Copilot instruction files to set expectations and guardrails (what good looks like, what to avoid, how we structure code)
- extensive `README.md` files that explain intent - constraints, non-goals, how to run things locally, and why the architecture looks the way it does

That meant Copilot spent less time guessing and more time executing - and when it suggested something questionable, it was usually obvious why it didn't fit the stated intent.

The other practical reality is token length: Copilot only has a finite context window, and it's surprisingly easy to blow it by pasting logs, huge files, or repeating the same background in every prompt. Being deliberate about context helped - keeping instruction files short, keeping READMEs focused on intent (not walls of prose), and asking the agent to summarize before moving on so the "working set" stayed relevant.

## The Copilot development feedback loop (practically)

Once the repos and guidance were in place, the main thing that made Copilot feel "agentic" was the feedback loop: give it _exactly_ the right slice of context, ask for a concrete change, let it execute, then review and iterate.

In practice, the most reliable ways I found to steer it were:

- **Add specific files to the conversation context**. If you're asking about a particular behaviour, include the relevant files rather than describing them. In VS Code that often means opening the file(s) and attaching them to the chat, or explicitly referencing them in the prompt (e.g. "update `useSelection.ts` and `MapPage.vue` to…").
- **Highlight the exact lines you care about**. If there's a bug in a handler or a weird edge case in a composable, selecting the relevant block and asking "fix this" (or "refactor this") produces much better results than asking at a distance.
- **Constrain the task to an executable unit of work**. I had the best results when the request was something like: "add a new endpoint", "make this state deep-linkable", or "change the proxy to buffer upstream responses" - rather than a broad "improve the architecture".

With Copilot in **Agent mode**, the loop gets tighter again because it can do the mechanical steps for you:

- open and inspect files across the workspace
- search for usage and refactor consistently
- run commands (tests, builds, linters) to validate changes
- apply edits directly, then show diffs for review

That doesn't remove the need for judgment - it just moves your time from typing to reviewing. The trick is to treat the agent like a fast junior engineer: be very specific about what "done" looks like, and keep the working set small enough that you can meaningfully review what it changed.

One more practical workflow detail: I enabled **auto-approval for certain non-destructive actions** (things like read-only file inspection/search, or other operations that don't modify code). It helps maintain flow, but it comes with an obvious trade-off: if the agent is allowed to execute tools automatically, you should stay alert.

This matters even more if you connect the agent to external tooling via **MCP servers** (Model Context Protocol). MCP is powerful, but it effectively extends the agent's reach. Only connect to MCP servers you trust, understand what capabilities they expose, and be cautious about granting broad permissions - especially anything that can write files, run shell commands, or access secrets.

That "cheap + timeboxed" combination influenced the shape of the architecture: a static-ish frontend with serverless endpoints is perfect for hobby projects where you want to pay little (or nothing) when nobody's using it.

The finished system is split into two parts:

- **UI**: Nuxt 4 + Vue 3 + TypeScript + Vuetify, rendering a Google Map and a details drawer.
- **API**: Azure Functions (isolated worker, .NET 8) that reads a KMZ dataset, clusters points for map performance, and serves embedded icon/assets.

From the start, I wanted the UI to feel like a native map app: fast pan/zoom, meaningful clusters, reliable selection, deep links, and mobile-friendly controls.

One more constraint that mattered for this dataset: some POI details come through as HTML. That’s useful for formatting, but it meant the UI needed to treat the content as hostile, and **sanitize before rendering**.

## Shipping it: CI/CD and infrastructure

Even though most of the work described here is product-facing, getting it reliably deployed was part of the project from the start.

### GitHub Actions deployment

The application is deployed via GitHub Actions. The workflow builds the UI and publishes it to the hosting platform (Azure Static Web Apps), and deploys the API to Azure Functions.

The main value of having this automated was speed and safety: I could make changes in small timeboxed bursts and still have confidence that a clean build and deployment was only a push away.

### Infrastructure as code (Terraform)

Separately (in another repository), I keep the infrastructure definition in Terraform and deploy it via its own pipeline. That pipeline provisions and updates the Azure resources the app needs (and keeps the "cheap to run" goal honest).

That infra repo also contains secrets, but not in plaintext: I use `git-crypt` so encrypted `.tfvars` can live alongside the Terraform code. I wrote up the approach in [this post]({% post_url 2025-11-29-securing-terraform-secrets-with-git-crypt-and-github-actions %}).

Splitting "app deploy" and "infra deploy" kept responsibilities clear:

- the app pipeline ships code
- the infra pipeline manages the underlying Azure resources

I'm particularly happy with the way this turned out:

![](/uploads/2026/01/01/Screenshot 2026-01-06 085211.png)

## Night 1: get the map working, then make it usable

The first commits were classic "get to something you can see" work:

- Nuxt/Vuetify scaffold, the initial map page, and early CI wiring.
- A quick return to "**basic map working again**" - the app could render a Google Map and place markers.

Once the map existed, UX improvements landed rapidly:

- A **layout shell** and the first **navigation drawer**.
- A **details side drawer** so tapping a marker immediately gave context.
- Early performance tightening (including a deliberate **O(n²) → O(n)** improvement when dealing with marker lists). This was a real game changer over the original application and led to far less janky behaviour.
- **Cluster behaviour**: dense areas became clusters, and clicking a cluster zoomed in rather than opening a meaningless "cluster details" view.

The "native-ish" feel started immediately too:

- **Locate / Follow** modes (one-shot location vs tracking/centering while moving).
- **Heading/compass** work, including early fixes for noisy orientation.
- A global loading indicator so data fetches weren't mysterious.

On the backend, the API repo started with the Azure Functions isolated worker host plus the core services:

- Parsing the KMZ dataset.
- Clustering markers to keep the map fast.
- Serving icons/assets out of the KMZ so markers could use the correct imagery.

## Night 2: turning a map into a product: structure and deep links

Once the basics worked, the next wave was about _stability and predictability_.

I made selection and navigation feel consistent:

- Marker selection opened the details drawer.
- Selection became **deep-linkable** (URLs could represent "what you're looking at").
- The drawer behaviour was refined (closing rules, widths, and small SSR quirks).

This was also the moment where the codebase needed to stay maintainable.

Instead of letting the main page grow into a mega-component, I did a major refactor: **move behaviour into composables**. Each "area of concern" became its own composable:

- Loading locations for the viewport
- Selection state + URL sync
- Marker click behaviour
- Map event wiring (bounds changes, debouncing, drag interactions)
- User tracking + follow logic

That refactor made subsequent features much easier to add without fear.

Once selection was stable and URL-addressable, a few high-value "product" features became obvious:

- making locations **shareable** (so you can send someone a link and it opens the same place)
- adding "open in Google Maps" actions
- ensuring the details drawer didn't fight you when you pan/zoom around

The About page landed around here too, establishing:

- attribution
- data credit
- disclaimer

Under the hood, I treated composables as the seam between "map SDK world" and "Vue world". That made it much easier to reason about problems like: "what closes the drawer?", "what disables Follow?", and "what state belongs in the URL?".

## Night 3: search and "map feel" polish

With map browsing and selection stable, the focus shifted to _finding_ places and making the map feel good under your fingers.

On the UI side:

- **POI search** work expanded significantly.
- Gesture and interaction polish landed: improved drag behaviour, handling fast "fling" interactions, and smoothing jank.
- Selection UX improved again: **pin history**, and allowing deliberate de-selection.

This was also where I leaned into the idea of it feeling like an app:

- smoother gesture handling
- less jank when dragging
- better behaviour when you "fling" the map around

On the API side, search became a first-class feature:

- A POI search endpoint was added.
- Search quality improved with **term splitting**.
- The dataset was updated and a **summary endpoint** was introduced so the UI could show "how many places are in this build" (and which KMZ it's based on).

To support those UX flows, the API surface became clearer:

- A bounding-box endpoint for the map
- A lightweight search endpoint for autocomplete
- A details endpoint for a single place
- A safe way to serve embedded KMZ assets (icons/images)

This phase is where everything started feeling like a real app rather than a prototype.

## Night 4: what GitHub Copilot and I built together: PWA + BFF + production hardening

This is the most collaborative chapter.

### 1) Hiding the backend: introducing a BFF/proxy

I wanted the UI to call its own origin and avoid baking the backend domain into client config.

So we added a lightweight **BFF proxy** in the Nuxt server layer:

- The browser calls **`/api/...`** on the UI origin.
- Nitro forwards the request to the real Functions API base URL (kept server-side).

This keeps the client surface area clean and avoids exposing the "real" API URL in the browser.

This was a good example of the "work with the agent" approach: it's easy for an assistant to suggest simply wiring `NUXT_PUBLIC_*` variables and calling the API directly, but the long-term cleanliness comes from drawing the boundary properly.

### 2) A real production issue: streaming responses

In local dev, streaming proxied responses can seem fine.

In Azure Static Web Apps / serverless environments, we discovered an important nuance: a streamed response could be serialized incorrectly.

We fixed this by changing the proxy to **buffer upstream responses** (handling JSON/text/binary correctly) instead of streaming.

It was a reminder that "works on my machine" is especially deceptive when the runtime changes under you (local Node dev server vs serverless production).

### 3) PWA install: debugging reality and designing the UX

PWA install sounds simple ("add manifest, get install prompt"), but real browsers have a lot of rules.

We did three things:

- Added **dev-only diagnostics** so we could see what was happening (manifest status, service worker state, install events).
- Confirmed that "default install prompt not firing" was often a **platform/criteria** issue, not a missing button.
- Built an install UX that behaves well across scenarios:
  - shows install affordances when a prompt is actually available
  - doesn't nag forever (a **24-hour snooze** rather than a permanent opt-out)
  - hides install UI when already installed
  - provides a best-effort "Open" action after install (while acknowledging browsers don't allow reliably auto-launching the newly-installed PWA)

Another "agentic AI" lesson: it's very easy to get stuck in a loop of "it should work" assumptions with PWA install prompts. Instrumentation and a fallback UX beat guesswork.

## Features shipped (high-level)

By today, the app includes:

- **Fast map browsing** with clustered markers
- **Details drawer** for rich POI information
- **Deep-linking** so a URL can represent selection/state
- **Next/previous navigation** through markers
- **POI search** (plus relevance improvements server-side)
- **Locate / Follow** with heading/orientation support
- **Share + open in Google Maps**
- **About page** with attribution, disclaimer, and dataset summary
- **Optional telemetry** via Application Insights
- **BFF proxy** so the backend domain isn't exposed client-side
- **PWA support** with an install UX that respects real browser behaviour

## Dev lessons learned

- Map apps live or die by **interaction quality** (clustering and pan/zoom polish mattered as much as "having data").
- Refactoring early into composables prevented the UI from collapsing under its own weight.
- Timeboxing helped: shipping an end-to-end slice each session beat "big rewrite" plans.
- "PWA install" is not a single feature; it's a mix of criteria, platform quirks, and UX design.
- Instrumentation beats guesswork: a tiny bit of diagnostics saved hours of blind iteration.
- When you introduce proxying/BFF layers, test them in the real deployment runtime - **streaming vs buffering** can matter.
- If you render third-party HTML, treat it as hostile by default and sanitize aggressively.

## Closing thoughts on AI

One of the biggest takeaways from this experience is that leveraging agentic AI to build apps can lead to a very goal-shaped codebase. Most of what gets produced is tightly coupled to the original problem, and there isn’t a large chunk you can easily lift out into a reusable library.

That said, I think this is as much the fault of the operator (me) as it is of the agent. If I’d guided the AI with reusability in mind from the start, the project code would likely look different and be more applicable to future endeavours.

Still, one genuinely reusable piece did fall out of it: a Nuxt 4-compatible Application Insights module that works both in the browser and server-side in Nitro. You can take a look at that here: [https://www.npmjs.com/package/nuxt-otel-appinsights](https://www.npmjs.com/package/nuxt-otel-appinsights)

If you want to see where this ended up, the app is live at [https://defence.tobyreid.co.uk](https://defence.tobyreid.co.uk).
