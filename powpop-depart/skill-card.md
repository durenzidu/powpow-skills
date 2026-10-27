## Description:

Posts a fixed-format "setting off" invitation on PowPop, then keeps looking for people who could set off with the user — people whose destination is near theirs, people who started from somewhere near them, and people who have written about that destination. Candidates are offered one at a time. When nobody shows up, it offers a nightly watch.

This skill is ready for commercial/non-commercial use.

## Publisher:

[durenzidu](https://clawhub.ai/user/durenzidu)

### License/Terms of Use:

MIT-0

## Use Case:

A user who wants to travel somewhere and would rather not go alone asks an agent to post an invitation and look for companions. The agent asks for the starting point, the destination, the date, and the party size; shows the drafted post for approval; publishes only after the user explicitly agrees; then searches for candidates and presents them one at a time.

The originating community is PowPop (泡泡), a location-based social app. The skill requires the PowPop MCP server and does nothing without it.

### Deployment Geography for Use:

Global

## Known Risks and Mitigations:

Risk: The skill publishes content to the user's own PowPop account.

Mitigation: Posting is always two steps — a draft is shown to the user first, and `publish_post` runs only after an explicit yes. Drafts expire after one hour.

Risk: Reading other people's public posts and profiles could be mistaken for permission to contact them.

Mitigation: The skill does not follow, message, comment on, or friend anyone. It never acts on another person's account or content. It only hands the user that person's post link and profile link; any contact is made by the user, in their own app, on their own initiative.

Risk: A suggested candidate may be misrepresented as wanting to travel together when the only evidence is that they mentioned the place.

Mitigation: Each match tier has fixed wording that states exactly what is known. A "mentioned the place" candidate may never be described as wanting to go there.

Risk: Place-name geocoding can be wrong, producing a candidate near the wrong location.

Mitigation: A geocoding confidence below 0.8 is treated as a hard failure. The matched name is shown to the user for correction rather than silently retried.

Risk: A nightly recurring task reads and writes local state.

Mitigation: The only local file is `state/orders.json` (falling back to `~/.workbuddy/state/powpop-depart/` when the install directory is read-only). The watch task is created only after the user agrees, and the user is told its schedule and task id.

Risk: The skill asks the user to add a remote MCP server to their agent configuration.

Mitigation: The address is provided for the user to add themselves. The skill writes that configuration only when the user explicitly asks it to, and only after showing the full diff and getting consent. No other configuration or user file is read or modified.

## Reference(s):

- [PowPop Depart on ClawHub](https://clawhub.ai/durenzidu/skills/powpop-depart)
- [PowPop website](https://global.powpow.online)
- [PowPop MCP server](https://global.powpow.online/api/mcp)

## Skill Output:

**Output Type(s):** [Text, Markdown, Guidance, API calls]

**Output Format:** [Conversational Markdown with post links, profile links, and a suggested itinerary]

**Output Parameters:** [1D]

**Other Properties Related to Output:** [Creates one post on the user's own account after approval; writes a local order ledger; may create one recurring reminder task]

## Skill Version(s):

1.1.1

## Ethical Considerations:

The skill works from public posts only and does not contact anyone on the user's behalf. It is designed to state honestly that the pool of invitations is currently empty, and it is explicitly forbidden from implying that companions are plentiful or that someone has agreed to come along. Users should treat candidate suggestions as leads to verify themselves, not as confirmed companions.
