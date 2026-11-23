# Multi-household, travel, and shared-lot findings

## Purpose

The next scale beyond two players controlling Sims in one household is to
place players on separate lots, then in separate households, and finally to
meet at a shared community lot. These scenarios stress different assumptions:
lot visibility, active-household rules, travel, third-client boot cost and
social-state symmetry.

The archived records include partial designs, one-run scenario work and known
failures. They do not establish that every multi-household or three-client
gate passed.

## What changes with separate lots

A lot boundary affects simulation detail and routing, but does not itself
transfer authority. A player can claim a Sim on a different lot while the
server remains the single simulator. The server's active lot or selected
household may still affect native simulation priorities, so a remote player's
household is not automatically equivalent to the active one.

| Concern | Same lot | Separate lots | Separate households |
|---|---|---|---|
| Actor ownership | Claim arbitration | Same authority rules, different location | Same authority rules, different household context |
| Simulation detail | Often locally active | May receive lower native detail priority | Household services may still be keyed to active selection |
| Routing | In-lot path | Cross-lot or travel route | Home lot and travel transition add state |
| Menu/state tests | Local object and Sim state | More stale objects and lot-specific tests | Household funds, active selection and home lot matter |
| Join/rejoin | Snapshot may suffice | Interest and lot context matter | May require a fresh saved-world bootstrap |

## Community-lot meeting design

A staged community-lot scenario separates three questions:

1. Can two players' Sims appear together after controlled placement?
2. Can they travel by normal gameplay and arrive with coherent server/client
state?
3. Can a third client join, participate in pairwise socials, and observe
relationship state consistently?

A teleport-only smoke test can establish that multiple Sims are visible
together, but it does not validate normal travel, route handoff, or three-way
social behavior. Each stage needs an explicit verdict and the same scenario
should test all relevant client views.

## Findings and caveats

- Separate-lot scenario tooling was created around walking, separated
positions and late join, but the early run record includes an invalid/rejoined
client pass and an open inventory/menu discrepancy. Do not treat it as a clean
acceptance result.
- Testing a Sim from another household exposed that local client game code
could still run interaction behavior for a remote household's Sims in some
cases. Autonomy and other local code paths must be checked independently from
server claim ownership.
- Active-household-derived game behavior can give the server's selected
household special treatment. Bills, opportunities and similar systems cannot
be assumed to target every remote household correctly without a dedicated
test.
- Route failures after a reset can reflect invalid starting positions or stale
post-interaction state rather than a general travel defect; record server and
client traces before assigning cause.
- Vehicle behavior was an acknowledged gap in the historical multi-household
work.

## Test matrix

| Scenario | Primary assertion | Secondary assertion |
|---|---|---|
| Two players on separate lots | Both actors remain server-authoritative | Positions and menu-visible state agree on both clients |
| Late join to a split-lot world | Joiner receives correct world and actor mapping | No interaction history is silently omitted |
| Separate households | Claims remain independent of household membership | Household funds, home-lot and active-household reads are coherent |
| Community-lot arrival | All expected Sims share the same world location | Pairwise social options and relationship state agree |
| Third client | New peer receives stable identity and presentation | Boot/resource constraints do not invalidate the run |

## Failure modes and limits

- A remote household can be present without receiving equal native simulation
attention.
- Client-only interactions can run even when the server owns all player
commands.
- A late join can use an old save or wrong active household if bootstrap
inputs are ambiguous.
- A route test can pass with forced placement while ordinary travel remains
broken.
- Social interaction between two participants does not imply a coherent
third-party view.

Related pages: [authority](decision-authority.md), [two-player
gameplay](milestone-two-player-gameplay.md), [menu honesty](menu-honesty.md),
[persistence](decision-persistence.md), [open problems](open-problems.md).


## Household state model

Household membership, player ownership, active selection and lot simulation scope are related but are not the same relation. A household may have several claimed Sims; a player can move a claim without changing the server's local active Sim; a lot may be simulated at full rate because a claimed occupant is present. Avoid using "active household" as a synonym for ?owned by a player.?

| Relation | Example question |
|---|---|
| Membership | Which Sims belong to this household? |
| Ownership | Which player may issue commands for this Sim? |
| Selectability | Should this Sim appear as player-controlled in the server model? |
| Active household | Which household's game services receive active treatment? |
| Home lot | Where does this household's durable home state live? |
| Lot scope | Which lots receive frequent simulation updates? |

The kernel's current authority model tracks claims, household membership, home-lot mapping, player owners and a never-selectable set separately. The local active actor is preserved during a remote claim swap. Lot simulation scope can run active lots every tick and throttle lots with no claimed occupant; this is a resource policy, not proof that all household services are simulated equally.

## Travel and identity boundaries

Travel may rebuild world objects or reuse a saved world. A public Sim identity must map to the correct process-local object in the new world epoch. A claim should not accidentally carry to a different Sim that reused a local slot. Travel tests should verify the actor before departure, after arrival, after snapshot and after client rejoin.

## Social meeting across households

A shared park test should check both participants and every observer. The actor and target may see different local menu choices; a third client can have a different view again. Pairwise relationship values should be read from each relevant copy after the social ends, and transient posture or mood effects should be distinguished from durable relationship changes. A successful teleport to a community lot is only the setup stage, not evidence that normal travel or three-client sessions work.

## How to verify/test

- Create two households with one claimed Sim each and verify independent claims.
- Change a remote claim and assert the server's local active actor is unchanged.
- Check household membership, funds and home-lot identity on server and clients.
- Move between lots using ordinary travel and preserve object mapping.
- Repeat with late join and a cold save bootstrap.
- Test bills/opportunities only after identifying whether they are active-household keyed.
- Use a third observer for social state rather than relying on actor/target views alone.

Related pages: [authority](decision-authority.md), [social-session research](social-session-research.md), [persistence](decision-persistence.md), [status](../status.md).


## Separating household symptoms from lot symptoms

A household can be active in the multiplayer ownership model while its lot receives reduced native simulation detail. These are separate axes. If a Sim's opportunity or bill does not advance, first determine whether the subsystem is keyed to the active household, the home lot, the Sim's claim, or simply the current world time. Then compare a household on the server's selected lot with a remote household under the same test.

A useful diagnostic record includes household identity, member Sims, home-lot identity, claimed actor set, server's locally active actor, observed lot simulation scope, and the time interval. This record helps explain why "claim succeeded" does not guarantee every household service is being simulated at the same rate.

## Player role and household membership

A player role (for example, visitor or owner) is not interchangeable with household membership. A client may be permitted to see or interact with actors outside its household while still having no authority to issue commands for them. If a role grants service spawning or editing, that permission must be checked independently from the Sim claim.

| Policy | Scope | Example enforcement point |
|---|---|---|
| Sim claim | One actor | Command authority gate |
| Household ownership | Household-level operations | Economy or household service |
| Lot scope | Simulation budget | World/lot scheduler |
| Client role | Feature permission | Role gate before kernel command |
| Presentation interest | Event recipient | Stream filter |

## How to verify/test

Run paired scenarios where the only changed variable is whether the household is active on the server's selected lot. Record service ticks and durable household state. Then repeat with a claimed Sim on that lot, with the player role changed, and with a late join. This separates service simulation from claim and presentation policy.
