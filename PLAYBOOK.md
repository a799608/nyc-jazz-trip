# Trip-Tool Playbook

How the NYC trip tool (Liverpool at Yankee Stadium + jazz, Jul 29–31 2026) was built, and how to
rebuild the same thing for any future trip in a few hours — including what went wrong and how to
not repeat it. Sister project: the Tucson home-finder (github.com/a799608/tucson-home-finder)
shares the same architecture.

## Architecture (identical for every trip)

- **One public GitHub Pages repo** (`gh repo create a799608/<trip-name> --public --source . --push`,
  then `gh api -X POST repos/.../pages -f "source[branch]=main" -f "source[path]=/"`).
  Push = deploy, live in ~60–90 s. Poll with curl until the new content string appears — the
  first 2–3 polls always return the stale build.
- **index.html** — self-contained Leaflet map (CDN), venue/event pins with popups, filter panel.
  Data is NEVER baked into the page: it `fetch()`es sibling JSON files, so data refreshes are
  data-only pushes.
- **route.html** — leg-by-leg "getting there" guide for a transit-nervous user: one card per leg,
  each with its own small Leaflet map, numbered plain-English steps, and an
  **"Open in Google Maps" deep link** (`google.com/maps/dir/?api=1&origin=...&destination=...&travelmode=...`)
  so the phone does live navigation per leg. Header carries the day's suggested clock
  (leave X, park Y, check in Z, train at W).
- **Data files**: `<topic>.json` per research stream (fixture/logistics, events, lodging).
  Produced by parallel background agents writing into the session scratchpad; validated in the
  main thread (row counts, field coverage, spot checks) before copying into the repo and pushing.
- **Mobile**: `@media (max-width:700px)` + a JS bottom tab bar — panels become bottom sheets,
  one open at a time. (On Google Apps Script pages the equivalent killer detail is the missing
  wrapper viewport meta — not applicable to Pages, where the meta is in our own head.)
- **og:image**: 1200×630 Playwright screenshot of the finished page, og/twitter meta tags,
  verify the jpg serves HTTP 200. Re-shoot when the page's look changes materially.
- **User interaction**: eliminate/interest buttons write to `localStorage` (no backend needed for
  a short trip tool). "✗ Exclude" removes; counters get a reset link.
- **Zero spend**: free sources only (venue sites, official transit pages, Google Hotels pages,
  Nominatim geocoding at 1 req/s with a cache file). No Apify, no paid APIs — hard rule.

## Process pattern

1. **Fire parallel research agents immediately** (one per stream) while the main thread scaffolds
   the repo/page. Streams for a trip: (a) anchor event — verify date/time/venue from 2–3 sources;
   (b) activity calendars — ONLY from the venues' own sites/ticketing, never "best of" listicles;
   (c) lodging; (d) transit/park-and-ride specifics.
2. **Watch the disk, not the notifications.** Agent completion notices can be missed (a merge once
   sat unnoticed for 90 minutes). Run a background watcher loop that checks for the output file /
   field coverage every 30–45 s and proceed the moment data lands.
3. **Stream, don't batch, anything time-critical.** For the same-day hotel hunt the winning shape
   was: agent flushes the JSON after EVERY item → background loop commits+pushes each change →
   the live page fills in as confirmations land → user books from partial results. Also give the
   agent an early-stop ("stop at 15 confirmed") so it never checks the long tail.
4. **Gate filters on data existence.** A "must have X" filter deployed before the X field exists
   blanks the whole page. `state.needX && dataHasX() && item.x !== true`.
5. **Sequence pushes honestly**: never ship a UI claim (a filter label) ahead of the data that
   makes it true.

## THE HOTEL AVAILABILITY PROTOCOL (the expensive lesson)

What went wrong on 7/29: availability was "confirmed" from Google Hotels price chips; the site's
Book buttons pointed at Booking.com deep links; the user hit "no availability" everywhere, and the
first hand-check on a hotel's own engine (Edison/SynXis) showed SOLD OUT while Google resellers
still flashed rates. On a big-event night, cached reseller rates are fiction.

**Truth hierarchy for "is there a room" (same-day or near-term):**
1. **The hotel's OWN booking engine** — SynXis (`be.synxis.com/?Hotel=NNN&arrive=...`),
   TravelClick, marriott.com, wyndhamhotels.com, riu.com, etc. Find it via the hotel site's
   BOOK NOW link (grep the DOM for synxis|travelclick|reservations). This is the ONLY source that
   reflects live inventory and exposes ROOM TYPES (bed count!).
2. **Google Hotels entity page with dates** (`google.com/travel/search?q=<hotel>&checkin=YYYY-MM-DD&checkout=YYYY-MM-DD`)
   — good for "roughly what's open at what price," and the right target for rate links, but its
   reseller chips (Vio/Billabook/etc.) can lag reality by hours.
3. **Booking.com deep links** — bot-walled to scripts AND runs its own allotment pool; on a hot
   night it shows empty while the hotel sells elsewhere. Never make it the primary button.

**Rules going forward:**
- Bake every room requirement (TWO BEDS, etc.) into the FIRST availability pass — bed config is
  invisible in price chips and only appears on direct engines.
- The action button must link to the SAME source where availability was verified.
- Availability decays in minutes on event nights: check it LAST, stream it, and book fast.
  Static research (venues, transit, neighborhoods) doesn't decay: do it FIRST.
- Ideal: run the lodging search DAYS ahead, not day-of. Same-day is a knife fight.
- If the user books off-tool (they often will), rip the lodging layer out rather than maintain it.

## Transit / logistics lessons

- **Park-and-ride overnight rules are the load-bearing fact** — many commuter lots ban overnight.
  Verify on the OFFICIAL operator/station page (njtransit.com station pages state it plainly:
  Secaucus Junction = "PARKING ALLOWED 24/7", ~$6/day prepaid via LAZ/OnAirParking, 10 min to Penn).
  Journal Square: overnight explicitly banned. Don't trust third-party rate pages (a Harrison NY
  vs Harrison NJ mixup nearly shipped).
- **Verify the transit line actually goes where you assume.** (Harrison PATH → WTC only, not 33rd
  St — an "obvious" one-seat ride that wasn't.)
- **Last-train times matter** if the car is at a park-and-ride: NJT's last Penn→Secaucus weeknight
  train ≈ 1:20 AM; with a Manhattan hotel it's irrelevant. NJ Transit DepartureVision
  (njtransit.com/dv-to) = live track + time.
- **Venue bag policy drives the luggage plan**: Yankee Stadium bans backpacks outright → bags must
  reach the hotel first; check-in time (4 PM) + front-desk bag holding sets the day's clock.
- **Prepay everything in one box on the route page**: parking (LAZ/OnAir), train (NJ Transit app
  MyTix, buy all legs upfront), subway (OMNY = tap phone/credit card, NOTHING to buy — say so),
  event/venue tickets. Include live-schedule links (mta.info/schedules, DepartureVision).

## Event-research rules

- Real listings only, from each venue's own calendar/ticketing (SmallsLIVE, ticketing.jazz.org,
  venue Squarespace calendars...). If a calendar isn't published for the date, say so per venue —
  never fill with plausible acts.
- Classify venues on verified physical facts (dinner-at-table vs drinks-only, floor level, seat
  count, sightlines) — the user's "no tiny basements with poor visibility" filter needed
  venue_type/food/room_note fields sourced from venue FAQ/press pages.
- Flag post-event feasibility explicitly (late_ok: can you make the set after the game ends?).

## Repos

- This trip: github.com/a799608/nyc-jazz-trip (index.html, route.html, nyc_jazz.json,
  nyc_fixture.json; hotel layer removed 7/29 after off-tool booking — see git history for the
  full lodging implementation if ever needed again).
- Tucson home-finder (longer-lived sibling, richer filters + feedback loop):
  github.com/a799608/tucson-home-finder.
