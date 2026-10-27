# solo-out-together-in

Solo out. Together in.

English edition of [`powpop-depart`](../powpop-depart). You set off alone and leave one
invitation on PowPop: **where you're starting from, where you want to go, and when**. The
invitation is pinned to your starting point and lands on the map, so other people can find
it. The skill then keeps looking for people who could set off with you, and when it finds
one it asks a single question: **want to ride along?**

> **Requires the PowPop MCP server**: `https://global.powpow.online/api/mcp`
> Without it the skill cannot post or search, and it will say so plainly instead of
> pretending to run.

## Install

```bash
npx skills add durenzidu/powpow-skills/solo-out-together-in
```

Or copy the `solo-out-together-in/` folder into your agent's skills directory.

## How it works

1. **Post the invitation.** Ask for origin, destination, date, party size, then draft a
   fixed-format post:
   `【Solo out. Together in.】Sat Nov 1, afternoon · from Shichahai · to Wukang Road · 2 people · route can be flexible`
   The draft is shown to you first; nothing is published without an explicit yes. The post's
   map coordinates are the **origin**, which is what makes "people who set off near you"
   findable.
2. **Find people** along three routes, with both ends judged by distance (3 km = near,
   1 km = same pocket) rather than by matching place names:
   - **Route A** — invitation pool: `search_posts("Solo out")`, which exposes both origin and destination
   - **Route B** — origin side: `find_nearby(place=origin, radius_meters=3000, types=['posts'])`
   - **Route C** — destination side: `search_posts(my destination)`, an ILIKE over post bodies that does not depend on the invitation pool
3. **Offer, one at a time.** Ordered R1 both ends near → R2 destination near → R3 origin near
   → R4 mentioned the place. Every offer carries a place, one verbatim line from their post,
   and a concrete date. Say no and they are never mentioned again.
4. **Night watch.** When a round finds nobody, the skill offers a daily recurring task that
   keeps scanning until someone appears or the date passes.

## Boundaries

- Reads other people's public posts only. **Never follows, messages, comments on, or friends anyone.**
- Hands over the person's post link and profile link. **Making contact is always the user's own move.**
- Only ever changes the user's own content: posts and status updates go under the user's own
  account, each with a draft shown and an explicit yes first.

## The honest part about a cold start

There are currently zero invitations on PowPop, and only a few dozen posts added across the
whole platform in the last 30 days. A first invitation will most likely find nobody and go
straight to the night watch. What can be offered immediately is R3 and R4 — on the order of
1–10 people each. The pool only exists once people start posting: the first person waits
longest, and around the tenth person the first real match becomes possible.

## See also

- ClawHub: <https://clawhub.ai/durenzidu/skills/powpop-depart>
- Chinese edition: `../powpop-depart`
- Platform: <https://global.powpow.online>
