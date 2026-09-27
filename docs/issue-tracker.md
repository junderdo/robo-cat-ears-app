# Issue tracker

Issues for this repo are cards on the **Robo Cat Ears** Trello board.

The board is shared with the other Robo Cat Ears projects, so every card for this repo carries the `mobile-app` label:

- When creating a card for work in this repo, add the `mobile-app` label.
- When looking for this repo's work, filter the board by the `mobile-app` label.

## Wayfinding operations

How a wayfinding map and its tickets live on the board.

### Labels

Every map and ticket card carries `mobile-app` plus one wayfinding label:

| Label | Card |
| --- | --- |
| `wayfinder:map` | the map |
| `wayfinder:grilling` | a decision reached by conversation |
| `wayfinder:prototype` | a decision reached by reacting to a rough artifact |
| `wayfinder:research` | a fact-finding investigation |
| `wayfinder:task` | manual work that unblocks a decision |

### The map

- One card, labelled `wayfinder:map`, in **Todo** while the map is live.
- Its description is a one-line summary of the destination and a link to the map body in `docs/maps/`. The body lives in the repo because it outgrows Trello's card description limit.
- A **Tickets** checklist on the map holds one item per child ticket: the ticket card's URL.

### Tickets

- **Parent**: each ticket card has the map card attached (`trello card:attach --url <map card URL>`), and is added to the map's **Tickets** checklist.
- **Blocking**: the description's first line is `Blocked by: <card URL>, <card URL>`, or `Blocked by: none`. A ticket is unblocked when every card it names is in **Done**.
- **Frontier**: cards labelled `mobile-app` and a `wayfinder:` ticket label, in **Todo**, unassigned, and unblocked.
- **Claim**: assign the card to the dev driving the map and move it to **In Progress**, before any work.
- **Resolve**: write the outcome into the repo (below), post a resolution comment with a one-line gist and the doc's path, move the card to **Done**, tick its item on the map's **Tickets** checklist, and add the ticket to the map doc's Decisions so far.
- **Out of scope**: archive the card and add a line to the map doc's Out of scope section.

### Outcomes live in the repo

Card descriptions and comments hold gists and pointers; the full outcome is a doc under `docs/`, following that folder's `README.md`:

- A decision (grilling or prototype ticket) → an ADR in `docs/adrs/`. Link any prototype from the ADR.
- A research ticket's findings → `docs/research/`.
- A task ticket's resulting facts (URLs, where credentials live) → the resolution comment, or a `docs/agents/` doc when later work in the repo depends on them.
