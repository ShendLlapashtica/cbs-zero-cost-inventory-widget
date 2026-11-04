# CBS — a zero-marginal-cost inventory widget that runs in the visitor's browser

**CBS (Client Browser Search)** embeds live motorcycle inventory from an
upstream marketplace index into a dealer's website. A dealer pastes one line of
HTML and gets a grid of current listings.

The interesting part is not the grid. It is that serving it costs the operator
**nothing per request** — no function invocation, no database command, no proxy
bandwidth — and that it keeps working unattended for months without anyone
touching it.

This is a write-up of the architecture and, more usefully, of the eight things
that turned out to be hard. Figures come from a production deployment serving a
dealer network; exact counts are rounded and the upstream is not named.

---

## 1. The cost problem, and why the usual answer is wrong

The obvious design is a thin API in front of the upstream index:

```
visitor's browser  →  our API  →  upstream index
```

Every listing view is then a function invocation, a cache lookup, and
upstream bandwidth on the operator's bill. On a usage-billed platform this is
the single most expensive shape you can choose, because you pay for the whole
wall-clock time an instance is held — and most of that time is spent *waiting
on the upstream*. A slower upstream is a bigger invoice.

We measured a conventional build of this at roughly **two to three invocations
per page view** once you count the listing grid, the detail page and the
count-for-pagination call. For a dealer network with real traffic that is the
dominant line item, and it grows linearly with their success.

CBS removes the middle box entirely:

```
visitor's browser  →  upstream index
       ↑
   one static file, served once from a CDN
```

The widget is a static JavaScript file. Every request after it is issued *by
the visitor's own browser, from the visitor's own IP address*. The operator's
marginal cost per listing view is not "small". It is **zero** — there is no
server in the path to bill for.

This works only because the upstream answers browsers with a permissive CORS
header. That single fact is what makes the whole design possible, and §6 is
about the fact that it can be taken away.

### What you give up

Be honest about this up front, because it is not a small list. With no server
in the path there is:

- **no shared cache between visitors** — every visitor pays their own latency;
- **no "sold" or "removed" marker** — that needs stored state to diff against;
- **no price-drop history**, for the same reason;
- **no new-arrivals feed** — you cannot notice what changed without remembering
  what was there;
- **no metering, no quota, no domain lock, no key enforcement.** There is
  nothing to enforce them with. A key parameter is accepted and ignored so that
  existing integrations keep working, but it authorises nothing.

If you need any of those, you need the server, and you should price it
honestly rather than pretend this design covers it.

---

## 2. The threat model is the visitor, not the bill

This is the part most people get wrong, and it inverts the usual instinct.

When your server calls an upstream, rate limiting protects **your** budget and
**your** IP reputation. When the *visitor's browser* calls the upstream, the
address being judged is **theirs**. A visitor who gets throttled does not see
an error message they can report — they see an empty page, decide the site is
broken, and leave. You never hear about it.

So every constraint in CBS exists to make an ordinary visitor look like an
ordinary visitor:

| Constraint | Value | Why |
|---|---|---|
| Requests per minute | 30 | A real page uses one to three. The cap is never reached by a human. |
| Concurrent in flight | 3 | A burst is what gets an address flagged, not the total. |
| Minimum gap between starts | 150ms → 500ms → 1.2s | Widens on its own as the minute fills. |
| Per tab, lifetime | 1,500 | Memory only, so a reload starts clean. |
| Per browser, per day | 5,000 | The tab cap is per tab, and a loop can open tabs. |
| Over any cap | 1 request / 10s | It **crawls**. It never refuses. |

Two of these deserve explaining.

**Pacing beats counting.** The first version enforced only a per-minute
ceiling. All of the allowance could leave inside three milliseconds, which is
precisely the signature that gets an address cut off — the count was legal and
the shape was not. Admission is now serialised, so two callers in the same tick
cannot both skip the gap.

**Over the cap, crawl — never refuse.** A refusal renders an empty grid, which
is indistinguishable from "this dealer has no motorcycles". Degrading to one
request every ten seconds keeps the page honest and the operator's phone quiet.

---

## 3. The upstream's block signal was invisible

A circuit breaker is table stakes: four consecutive failures, stop for 30
seconds, double up to ten minutes, then let exactly **one** probe through. A
`429` trips it immediately and `Retry-After` is obeyed, because that response is
the upstream naming your address and arguing with it is how an address gets
banned permanently.

The breaker was useless for months, and the reason is worth knowing.

The upstream does not block with a status code. It answers with a **redirect to
an interstitial page**, and that page returns `404`. Our failure detector
ignored `404` deliberately — a missing listing is not an outage — so a fully
blocked browser kept knocking at full speed, having classified every block as
"nothing to see here".

The fix is not status-based at all:

> **If the final URL of a response differs from the URL requested, that is a
> block — whatever status it wears.**

`403` was promoted to a hard stop alongside `429` at the same time. The lesson
generalises: when you depend on an upstream you do not control, **discover its
failure signature empirically.** Do not assume it is in the status code.

Breaker state persists per tab in `sessionStorage`, clamped to ten minutes, so
a reload does not spend four fresh requests rediscovering a door that is still
shut. It is deliberately *not* persisted longer than that — a sticky dead page
would be worse than the problem it solves.

---

## 4. Zero has to be enforced by a build, not by discipline

This is the single most valuable thing in the project, and it is twelve lines of
build script.

"No server in the path" is an architectural claim that any future contributor
can quietly break — usually while trying to help during an outage. So the build
**asserts** that the compiled bundle contains none of: the operator's own
domain, its API paths, its auth header name, or the global installed by its
paid client library. If any string appears, the build fails.

That assert was not written defensively. It was written after a real leak.

The widget imports three display helpers (price formatting, photo URL building,
localisation) from the operator's *paid* API client. That module's last line is
a top-level `window.PaidApi = api`. A top-level side effect defeats
tree-shaking, so the bundler kept the entire module — and **every dealer page
running the free widget was silently installing a working, authenticated client
for the paid API into global scope**, where any script on that page could have
spent the operator's money.

The build now strips that one line from the copy it compiles, leaves the
original untouched (other integrations load it and that global is its
documented entry point), and then asserts the result is clean.

Related: there is **no fallback path**, and that is deliberate. An earlier
version could fail over to the operator's paid API behind a constant. It was
**deleted, not disabled** — "off behind a flag" is one edit away from not being
zero, and that edit gets made at 2am during an outage by someone being helpful.
A route that cannot be called is the only kind that is certainly free.

### Guarding a copied function

The widget has to build upstream query URLs, and a browser cannot import the
server's request builder. So it holds a **copy**.

Copies drift, silently, and the failure is a wrong result rather than an error.
The build therefore extracts the server's real builder **as source text**,
exercises both over six representative query shapes, and **fails on any
difference**. It reads the text rather than importing the module because
importing it would pull in the whole HTTP stack.

Five commits rewrote engine and chassis-code handling in the server builder
during one afternoon. The drift check reported all six shapes identical, and
that is the only reason anyone could be confident the widget had not quietly
started asking the wrong questions.

---

## 5. "It's not showing the latest" — the upstream had no usable date sort

A dealer reported that the grid was serving stale inventory and implied he was
being sold old data. He was right, and the cause was not caching — there is no
cache anywhere in this design.

The upstream publishes its sortable fields. That list contains year, price,
mileage, two lease-price variants, and a creation date. It **rejects**
`CreatedDateDesc`, `ModifiedDateAsc` and `YearAsc` outright, with
`"... is not valid sort name"`. And neither the creation-date sort nor the
modification-date sort we had defaulted to actually puts recent listings on page
one: over a single 20-row page, listing ids spanned the entire history of the
marketplace. A grid of "latest" opened with a ten-year-old bike.

Worse: **no row in the response carries a date field at all.** Date was not
merely mis-sorted, it was unavailable to sort on afterwards.

What *is* ordered is the **advertisement id**, which the upstream assigns in
sequence. Confirmed from the opposite direction: the one ascending date sort it
does accept returns the lowest ids in the system, while the newest listings sit
at the top of the id range.

So the default now fetches one 200-row page, orders it by id descending, and
returns the requested slice:

```js
// Still ONE request. The visitor's browser pays ~21KB gzipped instead of ~3KB,
// and the operator pays nothing either way — the only trade this design is
// always allowed to make.
const latest = !params.sort;
const rows = latest
  ? page.slice().sort((a, b) => Number(b.Id) - Number(a.Id)).slice(0, count)
  : page;
```

A caller that asked for an explicit order still gets exactly that order. This
replaced only the default that nobody had chosen.

**The lesson:** when a client says the data is stale, check the *ordering*
before you check the cache. And verify a sort does what its name says, against
real responses, rather than trusting that passing a plausible field name worked.

---

## 6. Name resolution, in three directions

Dealers type `"Ninja 400"`. The upstream wants its own internal model-group
code. In between sits a surprising amount of wrongness.

**Matching needs three directions, not two.** The original matcher tested
whether the request *starts with* a known group name, then fell back to whether
a group name *contains* the request. Both assume the request is the shorter
string. Neither fires when the request is **longer and carries the group name
inside it** — which is the shape of every manufacturer marketing prefix
("The New …", "All New …", "2nd Gen …"). The index lists a plain group, the
dealer asks for the generation the manufacturer advertises, nothing matches, and
the search silently widens to the entire brand — tens of thousands of bikes
where the page asked for one model.

Adding "request contains a group name, longest wins" fixed it. Note what was
**not** done: stripping a list of known marketing prefixes would have fixed
today's names and missed whatever the marketing department invents next.

**The upstream's own English names contain typos.** One high-volume model is
published with a misspelling. The server had carried a correction table for
years; the widget read the raw field instead. So that model resolved to nothing,
widened to the whole brand — and printed the misspelling on the dealer's
cards. Both paths now go through the same correction table.

**Merged brands resolve to dead values.** Three manufacturers in this market
were merged or renamed, and the index files them under a value naming *both*
brands. The plain name a dealer would obviously type resolved to a legacy value
holding **57 listings** in one case and **zero** in another, while the value
holding roughly five thousand listings went unused. An explicit alias table now
maps every plausible name onto the value that actually holds the inventory, and
it is applied *after* the general name loop so it overwrites the broken mapping
rather than losing to it on a first-wins basis.

### The worst bug in the project

Filter construction dropped an unresolved manufacturer. So a brand name that
resolved to nothing left the search with **no brand filter at all** and returned
the entire marketplace — roughly 120,000 listings — to a page that had asked
for one brand.

It had been there a long time, and nobody reported it, because:

> A grid full of the wrong motorcycles looks exactly like a working grid.

It does not throw. It does not log. It renders beautifully. A visitor just
scrolls inventory the dealer never meant to show, and nobody can tell.

An unresolved *model* still falls back to the brand deliberately — that is a
narrower answer than requested, and defensible. An unresolved *brand* has no
narrower answer available, so it now returns zero results and says so.

**The lesson:** audit every place a filter can be dropped. A silently widened
query is more dangerous than an exception, because an exception gets fixed.

---

## 7. The upstream can close the door, and the right response is to wait

The one dependency that cannot be engineered around: the upstream chooses
whether to answer browsers at all. It has withdrawn that permission before, for
roughly four weeks, by returning `403` to any request carrying an `Origin`
header. When that happens the widget reports itself unavailable and retries
itself at 20s, 45s, 90s, 3min, then 5min — bounded, only while the tab is
visible, stopping on success or if the element leaves the page. While the
breaker is open those retries cost no request at all.

Verified in a headless browser with the upstream blocked at the network layer:
listings reappeared 20 seconds after unblocking, three requests total, nobody
touching the page.

The reflex to resist, and it will be proposed in good faith the first time a
grid goes empty:

> **Do not answer a closure by routing through your own servers.**

That converts a free outage into a permanent bill, and nobody removes it
afterwards. A closure is an outage that costs nothing. Wait for the door, or
change how the browser knocks — never move the knocking onto infrastructure you
pay for.

Monitoring it is free too: the operator's own site uses the same browser path,
so an existing alert on that traffic fires if the door shuts. No extra job, no
extra cost.

---

## 8. Small things that mattered more than expected

**Lazy, not eager.** Three grids on one page used to fire three searches the
instant the page loaded, in the same millisecond, for two grids the visitor may
never scroll to. Each now waits until it is near the viewport *and* the tab is
visible. Two grids become one request until the visitor scrolls — and the
requests spread themselves out, which also helps §2.

*If you benchmark this in a headless browser, size the window tall. A grid
below the fold correctly does nothing, and that looks like a failure.*

**Automation gets markup and an empty container.** Crawlers and headless agents
receive the page structure and no listings: no request, no retry timer, nothing
to index. Development hosts are exempt so it stays testable. This is one
function, which means the decision to allow indexing is a one-line change rather
than an argument.

**Ask the CDN for the size you will display.** Card thumbnails are requested at
400×300 — 31% fewer bytes per image, measured. The image CDN *upscales*, so
requesting 1600×1000 for a 240px card costs 168KB instead of 35KB and looks
identical.

**Cache your own static file properly.** The widget shipped with
`Cache-Control: public, max-age=0, must-revalidate`, so a browser was not
permitted to keep it even between two page views on the same site. Every page
load re-asked the CDN. Mostly `304`s and a few hundred bytes — but a round trip
per page view on the one asset whose entire purpose is costing nothing. Now
`max-age=3600, stale-while-revalidate=86400`: repeat visitors are free, and a
stale copy keeps serving for a day while a fresh one is fetched behind it, so
the widget never blocks on the operator's infrastructure even mid-deploy. The
price of that trade is honest and worth stating: a change reaches visitors
within the hour rather than instantly.

**One line beats two.** The integration asked a dealer to paste a `<script>`
*and* a `<div>`, and to put the `<div>` in the right place. Carrying the same
configuration attributes on the script tag removes the second step entirely —
the script inserts its own container immediately after itself, so the grid
appears exactly where the line was pasted:

```html
<script src="https://cdn.example.com/cbs.js" data-cbs-items
        data-manufacturer="Kawasaki" data-count="12"></script>
```

`document.currentScript` must be read at module evaluation time; it is `null`
by the time `DOMContentLoaded` fires. The inserted container is then left for
the normal auto-render pass to fill, so the one-line and two-line forms share a
single rendering path and both stay lazy, paced, breakered and self-healing.
Two rendering paths would have been two things to keep in step forever.

---

## What to take from this

1. **Moving egress to the client is not a trick, it is a different cost class.**
   Zero marginal cost per request, in exchange for a real and specific list of
   capabilities you cannot have (§1). Decide against that list, not against the
   headline.
2. **When the client makes the requests, the client's reputation is what you
   are protecting.** Pace rather than count; degrade rather than refuse.
3. **Enforce your invariants in the build.** "We will remember not to add a
   server call" is not an invariant. A failing build is.
4. **Discover your upstream's failure signature. Do not assume it.** Ours was a
   redirect to a page that 404s, and our detector ignored 404 on purpose.
5. **A silently widened query is worse than an exception**, because a grid of
   the wrong inventory looks exactly like a working grid and nobody reports it.
6. **Verify that a sort does what its name says**, against real responses.
   Passing a plausible field name is not the same as sorting.
7. **An outage that costs nothing is an acceptable outage.** Do not fix it by
   building something you have to pay for forever.

---

*Published as a write-up of production work. The upstream marketplace and the
dealer network are not named, and no proprietary source is included — the code
excerpts are illustrative.*

*Related work by the same author: [Encar API](https://encar-api.com) — a
commercial data API serving live Korean used-car listings as JSON, where the
per-request cost decides the margin the same way it does here.*
