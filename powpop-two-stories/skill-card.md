## Description:

After the user publishes a post at a place, the skill finds what other people left at the same place — any date, with no time filter at all — and shows those words verbatim next to the user's own. Location is decided by distance (within 500 m counts as the same place), not by whether the place name was typed identically. Duplicate content that the platform has scattered across several locations is collapsed to its earliest occurrence. It then asks one question: do you want to like any of these? Liking is the only action the skill ever takes on someone else's content, and it happens only after an explicit yes.

This skill is ready for commercial/non-commercial use.

## Publisher:

[durenzidu](https://clawhub.ai/user/durenzidu)

### License/Terms of Use:

MIT-0

## Use Case:

A person who has just written something at a place — a view, a moment, a paragraph — wants to know whether anyone else left something at that same place. The agent locates the place, searches within 500 m, verifies each candidate, shows the other posts as quoted text with dates and distances, and then offers to like them.

The originating community is PowPop (泡泡), a location-based social app. The skill requires the PowPop MCP server and does nothing without it.

### Deployment Geography for Use:

Global

## Known Risks and Mitigations:

Risk: The skill publishes content to the user's own PowPop account.

Mitigation: Posting is always two steps — the draft is shown first and `publish_post` runs only after an explicit yes. Drafts expire after one hour. Posts always carry the place the user gave; the skill never invents one.

Risk: Liking other people's posts is an action on someone else's content.

Mitigation: It happens only after the skill has asked, listed exactly which posts, and received an explicit answer. "Whatever you think" is treated as no answer. Only the posts already shown may be liked, never anything beyond them.

Risk: The skill could present a post as being at the user's location when it is not, because place names on the platform are inconsistent.

Mitigation: Distance is the deciding check (500 m), and a place name that clearly belongs elsewhere is discarded rather than described. The skill never claims the user is the first person there — when nothing is found it says it cannot see anything, not that nobody came.

Risk: The same text has been seeded across many locations, so quoting each copy would misrepresent one sentence as several people.

Mitigation: Content is normalised and de-duplicated, keeping the earliest copy only. The skill also never names authors and never shows like counts, because neither is reliable here.

Risk: The skill asks the user to add a remote MCP server to their agent configuration.

Mitigation: The address is provided for the user to add themselves. The skill writes that configuration only when the user explicitly asks it to, and only after showing the full diff and getting consent. It writes no local state at all and reads no other user file.

## Reference(s):

- [PowPop website](https://global.powpow.online)
- [PowPop MCP server](https://global.powpow.online/api/mcp)

## Skill Output:

**Output Type(s):** [Text, Markdown, Guidance, API calls]

**Output Format:** [Conversational Markdown: the user's own post quoted, followed by numbered quotations from other posts at the same place with dates and distances, then one question about liking]

**Output Parameters:** [1D]

**Other Properties Related to Output:** [May publish one post on the user's own account after approval; may like the posts it displayed after an explicit yes; writes no local files]

## Skill Version(s):

1.0.0

## Ethical Considerations:

The skill works only from public posts and quotes them verbatim without commentary. It deliberately refuses to attribute authorship, display like counts, or assert that a place was empty — all three are things the platform cannot answer reliably. Its empty state says "I cannot see anything here", never "you are the first", because the cause of an empty result cannot be distinguished. Liking is presented as a small, disclosed gesture with a stated side effect (liking can extend an expired post by 24 hours), never as a default.
