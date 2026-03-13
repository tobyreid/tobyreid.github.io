---
title: "Rebuilding YourSound: revisiting a 2004 dissertation-era web radio project"
date: 2026-03-08
summary: "A look back at YourSound, a 2004 final-year university project that combined Classic ASP, Access, shoutcASP, SHOUTcast and Winamp control, and how I am recreating it with Nuxt, Nitro and AzuraCast."
tags:
  - classic-asp
  - nuxt
  - nitro
  - web-radio
  - software-archaeology
  - azuracast
minutes: 15
---

>TL;DR - Try the app here: [https://yoursound.tobyreid.co.uk](https://yoursound.tobyreid.co.uk).  

I have been rebuilding YourSound, a web radio site I originally completed in 2004 as a final-year university dissertation project. Looking back through the source has been a reminder that it was far more than a student brochure site. It was a working Classic ASP application with authentication, comments, artist bios, scheduled shows, newsletters, track requests, queue and play statistics, and a surprisingly direct relationship with a shoutcASP and Shoutcast radio backend. This post is partly software archaeology and partly a record of how I have been recreating it in Nuxt and Nitro without losing the character of the original.

## The 22-Year Itch: Why Revisit YourSound?

There's a specific kind of bravery required to open a folder of code you haven’t touched since the early 2000s.

In 2003/4, web development looked very different. A lot of us were building dynamic sites with Classic ASP, Dreamweaver-generated recordset code, Access databases, table layouts, Flash banners, and hand-rolled login flows. Streaming radio also felt far more improvised than it does now. You were often stitching together separate pieces of software, persuading them to talk to each other, and hoping the entire thing would keep running on a modest Windows machine.

YourSound was of that world - but I was not trying to make a nostalgic art project. I was trying to build something ambitious: a radio website that did not just describe a station, but actively participated in running it.

That is the part that still stands out.

This was not a static marketing site with a Winamp link thrown on the side. It had editorial content, user accounts, community features, scheduling, requests, and operational links into the actual radio system. The code shows a project that was very obviously of its era, but also more capable than I half-remembered.

## A Snapshot of 2004: Duct Tape and Winamp

Opening the original source code felt like walking into an old bedroom. It used a single index.asp?page=... front controller (groundbreaking at the time, I’m sure) and a layout built on the structural integrity of HTML tables and Flash banners.

Under the hood, it was doing a lot more than I remembered:

Public site functionality included:

- A news-driven homepage with paged posts and per-item comment counts.
- A full comments system for news posts, including add, edit and delete behaviour.
- Artist bios with images, outbound artist links, and direct links into the request system for that artist.
- Show schedules for this week and next week, with detail pages for individual shows.
- A help page explaining how to listen, what players to use, and how to get connected.
- A right-rail featured artist panel.
- A left-rail "today's shows" panel and multiple stream links for 128k, 64k and 32k listening.
- A track history page driven from live Shoutcast metadata.
- A request queue page and a statistics view for most requested and most played tracks.

Member functionality included:

- User registration.
- Email verification via a hash-based confirmation link.
- Login and logout.
- Profile editing.
- Request submission, with explicit anti-abuse messaging when requests were denied.
- Comment posting on news items.
- Newsletter opt-in.

Admin functionality included:

- Adding, previewing, editing and deleting news posts.
- Adding, previewing, editing, deleting and featuring artist bios.
- Adding, previewing, editing and deleting show entries.
- Composing, previewing, sending, editing and deleting newsletters.
- Access to the shoutcASP backend.
- Direct Winamp transport controls from the site itself.

Even the data model was split in a way that makes sense in hindsight. Site-owned content lived across several Access `.mdb` files for users, news, bios, shows and newsletters. Radio request data lived in the separate shoutcASP database. In other words, the application already had a primitive boundary between "website content" and "radio operations", even if the implementation was very much local-machine and Windows-host driven.

## The Part That Still Blows My Mind

The most "mad scientist" part of the original project was the radio integration. I hadn't just built a website about a radio station; I'd built a website that was the remote control for the station.

As an avid Winamp user in my university days, and after spending plenty of time listening to SHOUTcast streams in halls, I started digging into how the stack worked and discovered shoutcASP. That led me into COM and, eventually, to instantiating a `shoutcASP.Winamp` object from the site itself. The website was literally reaching into the server's OS to tell Winamp to play, pause, or skip. If you clicked a link in the admin panel, a physical instance of Winamp on a desktop somewhere would react.



First, the request catalogue and queue were not fake. The site queried the shoutcASP Access database directly, listing artists, albums and tracks that were marked as requestable. Users could browse alphabetically, drill into artists, and submit requests that were then reflected in the request queue.

Second, the statistics page was reading real operational counters. It surfaced the most requested and most played tracks directly from the backend tables, using request and play counters from the radio-side data.

Third, the history page called the Shoutcast admin XML endpoint directly. It parsed the live XML response to show recently played tracks, along with listener metrics pulled from the same source.

And fourth, the site instantiated a COM object called `shoutcASP.Winamp` and used it to send transport commands to Winamp, including core playback controls and playlist actions. The admin panel exposed some of those controls as web links.

That means the original YourSound site was not merely adjacent to the radio station. It was, in places, part of the operating surface of the station.

From a 2026 perspective, that is exactly the kind of thing you would not reproduce literally. It is tightly coupled, server-local, Windows-specific, and bound to assumptions about COM, Winamp, and a shoutcASP installation on the same host. But as a 2004 dissertation project, it was a serious integration exercise, and I think that deserves to be recognised rather than airbrushed away.

## What the code says about the era

There are plenty of very 2004 fingerprints all over the source.

- The top banner was a Flash movie.
- The now-playing strip used a marquee.
- Authentication depended on cookies in a way no one should copy now.
- The login flow compared usernames and passwords directly from the database.
- Verification and newsletters were sent with server-side mail code.
- Several pages have the unmistakable shape of Dreamweaver-generated data access code.
- The entire application was routed through query strings and server-side includes.

None of that is embarrassing to me. It is just honest. This was a final-year university project built with the tools, patterns, and constraints that were common then. If anything, it makes the ambition of the project clearer. Within those constraints, YourSound tried to be a complete product: content site, community layer, and radio-control surface.

## Recreating the site without flattening its identity

The current rewrite is intentionally not a lift-and-shift.

I am rebuilding YourSound as a Nuxt application with a Nitro backend, targeting Azure Static Web Apps, and using AzuraCast as the modern external radio backend. That means the website and the radio system are now separate systems connected over HTTP APIs rather than living together in one Windows box full of local assumptions.

The methodology for recreating it has been fairly disciplined.

First, I treated the original site as the historical source of truth for behaviour, layout, and tone. When modernising an old project, it is very easy to erase what made it distinctive. I wanted to keep the recognisable left-nav, centre-content, right-rail structure, the teal/orange/grey palette, and the general feel of an early-2000s web radio site, while still making it workable on modern devices.

Second, I used the old ASP source and recovered Access data as two separate kinds of reference. The source files tell you what the site did. The `.mdb` files tell you what content it actually held. Between them, it is possible to reconstruct not just the outer chrome of the project but the information architecture underneath it.

Third, I drew a harder boundary around the radio backend than the original ever had. In the rewrite, the browser calls Nitro endpoints owned by the site. Nitro then talks to AzuraCast and maps the responses into stable site-specific payloads. That preserves the useful separation the old system only hinted at, while avoiding another generation of direct frontend coupling to radio-admin internals.

And fourth, I have been selective about what gets recreated literally versus conceptually. The visual identity should survive. The request flow, queue, now playing, history, shows, bios and news should survive. But direct COM control of Winamp should not. Plaintext-password-era auth should not. Access databases should not. The goal is to preserve the shape and spirit of YourSound, not every idiomatic technical step that was normal in 2004.

## What has already been recreated

The current Nuxt and Nitro application already restores a surprising amount of the original public-facing site.

- The three-column layout and vintage visual shell are back.
- Public pages exist for news, bios, shows, help, history, requests and queue views.
- The radio-facing side of the site has modern endpoints for now playing, history, stream metadata, request browsing and queue data.
- Legacy route shapes like `index.asp?page=...` are redirected into the modern routes.
- Recovered legacy content from the old databases has been merged into the rewrite so the site is not just visually nostalgic, but historically grounded.

That has been satisfying, because it shows the original project was structured enough for a lot of its intent to be recovered.

## Where future development is still needed

There is still quite a lot to do before the rewrite reaches feature parity with the old site, let alone surpasses it.

The main gaps are:

- Secure session-based authentication backed by a real database rather than placeholders.
- Proper profile editing and user account management.
- A real comments system, including moderation decisions.
- Admin CRUD for news, bios and shows in the modern app.
- A replacement for the old newsletter tooling.
- Full request statistics parity, especially the most requested and most played views.
- Final production integration against a live AzuraCast installation.
- Decisions on how much historic user-generated data, especially comments, should actually be migrated.

There is also a more subtle challenge: how faithful should the rewrite be?

Some parts of the old project deserve direct homage. Others deserve reinterpretation. The trick is knowing which is which. A Flash banner becomes modern artwork or animation. A marquee becomes an accessible now-playing ticker. A shoutcASP Access database becomes a stable Nitro API over AzuraCast. But the overall experience should still feel unmistakably like YourSound, not a generic modern starter template that happens to mention radio.

## Why bother?

Looking back at YourSound, I think what pleases me most is not that it was polished by the standards of 2004, or even that it earned a 1st. It is that it was trying to do something concrete and end-to-end.

It combined content, user interaction, scheduling, live system integration, and operations into one product. It was opinionated. It had a point of view. And even where the implementation now looks period-specific, the ambition behind it still feels legitimate.

Rebuilding it now is not about pretending the old code was timeless. It was not. It is about recognising that the project had a real product shape, a real technical challenge, and enough architectural substance that it can still be meaningfully reborn two decades later.

It’s a reminder that even when our tools are "primitive," the drive to build something end-to-end and interactive is what makes being a developer fun.
