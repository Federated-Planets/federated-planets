# AT Protocol Exploration: Identity and Data Storage

This document evaluates moving Federated Planets onto the [AT Protocol](https://atproto.com) (atproto) for identity and data storage, and proposes a phased plan.

**Verdict:** feasible and a good fit for _identity_ and _public game state_; a poor fit for _consensus messaging_ and _private data_. The recommendation is a **hybrid, additive adoption** rather than a full move: atproto becomes the identity and record layer, while space ports keep talking to each other directly over HTTP.

## 1. Where the Game Stands Today

| Concern             | Today                                                                                         | Gap                                                                               |
| ------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Planet identity     | The domain itself; `<link rel="space-manifest">` on the homepage                               | Works, but identity cannot survive a domain change                                |
| Planet data         | `manifest.json` plus warp ring links in HTML/config                                           | No way to ask "who links to me?" without crawling                                 |
| Space port identity | Ed25519 key pair generated and kept inside the planet's Durable Object                        | Public key is never published, so signatures on travel plans cannot be audited    |
| Traveler identity   | `ship_id`, a free-form string                                                                 | No traveler identity at all: anyone can claim any ship                            |
| Travel ledger       | Each space port's own SQLite (Durable Object), last 10 records shown                          | Travelers hold no portable proof of their journeys                                |
| Consensus           | Direct HTTP between space ports (`initiate` / `prepare` / `commit` / `register`)              | Fine as is                                                                        |

The largest gaps are traveler identity and verifiable signatures. Both are things atproto is good at.

## 2. Ethos Fit

### What matches

- **"Your domain name is your destiny."** Atproto handles _are_ domain names, verified bidirectionally (DNS `_atproto` TXT record or `/.well-known/atproto-did` on the domain, and the DID document pointing back). Proving a planet's domain and proving an atproto handle are the same act.
- **One planet per domain.** An account has exactly one primary handle, so one account maps to one planet.
- **Sovereign hosting.** Data lives in a repository the owner controls and can migrate between hosts. A planet can self-host its PDS, including on its own domain.
- **No central game server.** Every record the game needs can be read straight from the owner's PDS (`com.atproto.repo.getRecord` / `listRecords`). No relay, AppView, or game-run indexer is required for the core protocol.
- **Static planets get easier.** A static site cannot run a space port today and must redeploy to change its warp ring. With records in a repo, a static planet edits its warp ring without touching the site.

### Where it strains

- **Hidden centralization.** Most accounts use `did:plc` (resolved through a single directory) and are hosted on Bluesky's PDS. A planet built that way depends on two third parties that a plain website does not. Mitigations: support `did:web` (fully domain-rooted), never require a relay or AppView, and keep `manifest.json` valid so a planet can exist with no atproto account at all.
- **Everything is public.** Atproto repositories have no private records. This conflicts directly with the "Fog of War" goal in `TRAVEL.md` §6. Cargo and armaments must stay off-repo (space port to space port), or be published only as hash commitments.
- **Repos are mutable.** An owner can delete or rewrite their own records. A repo is a publication medium, not a ledger. Travel records must therefore be self-certifying (carrying controller signatures), and a planet's own state remains the authority for "is this traveler here?".
- **Not a message bus.** PBFT rounds through the firehose would require a relay (centralizing) and add latency for no benefit. Consensus stays on direct HTTP.
- **Signing keys are not yours to use.** The atproto signing key (secp256k1 / P-256) is held by the PDS and signs repo commits only. Space ports keep their own Ed25519 key and _publish_ the public half in the repo.

## 3. Proposed Design

### 3.1 Identity

- **Planet** = an atproto account whose handle is the planet's domain. The DID is the durable identifier; the domain remains the location.
- **Coordinates stay hashed from the domain** (unchanged MD5 algorithm). If a planet changes its handle it moves in space but keeps its DID, history, and neighbors' links. This turns a domain change from "planet destroyed" into "planet relocated".
- **Person** = any atproto account, signed in with atproto OAuth at a space port. A person does not need to own a planet or a ship.
- **Ship** = a first-class entity with a life of its own, not a possession scoped to one account. It is born at a planet, keeps one identity for its whole life, changes owners and operators, gets upgraded, carries many people, and accumulates a log. Its `ship_id` is the permanent reference (AT-URI plus CID) to its launch record. See §3.4.
- **Position belongs to ships.** A person aboard a ship is wherever the ship is. A person not aboard any ship is on the planet where they last disembarked.
- **Shuttle** = a ship whose owner is a planet.
- **Anonymous visitors** have no identity and may ride shuttles only. No account is needed to explore.
- **Verification** of a planet: resolve handle to DID, resolve DID document, confirm it lists `at://<domain>`, read the planet record from the PDS named in the document.

### 3.2 Lexicons

NSIDs live under the project's domain (`com.federatedplanets.*`) and are published as `com.atproto.lexicon.schema` records with a `_lexicon` DNS TXT record.

| Collection                               | Repo                       | Key    | Contents                                                                                           |
| ---------------------------------------- | -------------------------- | ------ | -------------------------------------------------------------------------------------------------- |
| `com.federatedplanets.planet`            | Planet                     | `self` | `name`, `description`, `landingSite`, `spacePort`, `travelSafetyLevel`, `spacePortKey` (public key) |
| `com.federatedplanets.warpLink`          | Planet                     | TID    | `subject` (neighbor DID), `domain`, optional `name`. One record per neighbor                        |
| `com.federatedplanets.ship.launch`       | Planet where it was built  | TID    | The ship's birth: `name`, `class`, first owner. Its AT-URI and CID are the `ship_id` for life        |
| `com.federatedplanets.ship.event`        | Planet where it happened   | TID    | One entry in the ship's log: `ship`, `prev` (previous event), `type`, payload, signatures           |
| `com.federatedplanets.travel.declined`   | Declining planet           | TID    | Declined travel requests with the stated reason (`TRAVEL.md` §7)                                   |
| `com.federatedplanets.person.aboard`     | Person                     | `self` | Which ship the person is aboard and in what role; their consent to the matching boarding event     |

One record per warp link (like a follow) rather than an array in the planet record means mutual-neighbor checks for shuttle limits (`TRAVEL.md` §8) become a single lookup, and optional indexers can answer "who links to me?".

### 3.3 What changes in the travel protocol

- Election pool, seed, PBFT phases, timing, and shuttle limits are unchanged.
- Controllers identify themselves by DID and sign with the key published in their planet record. Any party can now actually verify the $2f+1$ signatures, closing the auditability gap in `TRAVEL.md` §5.
- A travel plan moves a **ship**, not a person. It must be filed by the ship's current owner or an operator, signed in at the origin space port. Everyone aboard moves with it; nobody's location is checked separately.
- Controllers check that the ship is at the origin by reading the head of its log from the origin planet.
- Finished, signed results are stored by **planets in their own repos**: an approved voyage is written by both origin and destination. In-flight consensus state stays in the space port's own storage and is never published.
- A planet cannot forge an event (controller signatures) and cannot quietly drop one (the next event names its predecessor, and for voyages the other planet holds a copy).

### 3.4 Ships

A ship is not an atproto account and does not live in any one repo. It is a **chain of signed events**, each stored by the planet where it happened and each pointing at the one before it. The chain travels with the ship: the planet a ship is docked at always holds the head of its log.

| Event               | Where it is recorded        | Authorized by                               |
| ------------------- | --------------------------- | ------------------------------------------- |
| Launch              | Planet that built the ship  | The planet and the first owner              |
| Voyage              | Origin and destination      | Owner or operator, plus elected controllers |
| Boarding / leaving  | Planet where the ship is    | The person, plus owner or operator          |
| Operator change     | Planet where the ship is    | Owner                                       |
| Ownership transfer  | Planet where the ship is    | Old owner and new owner                     |
| Upgrade             | Planet performing it        | Owner, plus the planet                      |
| Other transactions  | Planet where it happened    | Per transaction type                        |

Consequences:

- **Identity outlives ownership.** Selling a ship appends a transfer event; the `ship_id`, history, and upgrades stay with the ship.
- **Roles are state, not scope.** Owner, operators, and passengers are derived by replaying the log. Owners and operators may file travel plans; passengers may not.
- **People point at ships.** A person's `person.aboard` record names the ship they are on. It is the person's consent and a lookup shortcut; the boarding event on the planet is the authority. A person can be aboard only one ship at a time.
- **Shuttles are ordinary ships owned by a planet**, restricted to that planet's warp ring neighbors and the caps in `TRAVEL.md` §8. Anonymous visitors ride as a head count and leave no record; signed-in people riding a shuttle board it like any other ship.
- **Everything is public.** A ship's full history, including who was aboard, is readable by anyone.

**Alternative considered: a ship as its own atproto account.** Giving each ship a DID and repo would make its log self-contained and portable. But an account is controlled by whoever holds its credentials, so every ship would need a custodian (a "port of registry" planet running a multi-account data server), ownership transfer would mean handing over account control, and the log could still be rewritten by that controller. The event chain gets the same "life of its own" with no extra hosting and with transfers validated by the game itself. A ship account could be added later as an optional mirror of the chain.

**Known weakness:** if a planet disappears, the events it stored go with it. The chain stays verifiable from the surviving links, but history from that planet is lost unless the owner (or anyone) kept a copy. Owners mirroring their ship's log into their own repo is the suggested mitigation.

### 3.5 Backward compatibility

`<link rel="space-manifest">` and `manifest.json` remain valid. The manifest gains an optional `did` field. Atproto-aware planets prefer repo records; a planet with no atproto account still appears on the map and can be linked to, but cannot act as a verified traffic controller once signature verification is enforced.

## 4. Implementation Fit (Reference Planet)

The reference planet is Astro SSR on Cloudflare Workers with a SQLite-backed Durable Object. Nothing in this plan changes that.

- **Reads:** plain `fetch` to PDS XRPC endpoints, cached in the existing `traffic_controllers` table.
- **Writes:** the space port holds an OAuth session (or app password) for the planet's own account and stores tokens in the Durable Object.
- **Durable Object stays authoritative** for in-flight consensus and current traffic; the repo is where finished, signed results are published.
- **Self-hosted PDS:** single-user PDS implementations for Cloudflare Workers and Durable Objects exist. Combined with `did:web`, a planet could be domain, website, space port, and data host in one deployment, with no third party involved. This is the most on-ethos configuration and should be an option, not a requirement.

## 5. Plan

### Phase 0: Decisions and spikes

- Confirm the open questions in §6.
- Spike: atproto OAuth (confidential client with DPoP) running on Workers, with tokens in Durable Object SQLite. Library support on Workers is the main technical unknown.
- Spike: read latency and failure behavior when resolving a ring of neighbors directly from their PDSs.

### Phase 1: Identity binding (spec only, no behavior change)

- Add the optional `did` field to the manifest and document handle-based verification in `README.md`.
- Publish the `planet` and `warpLink` lexicons.
- Reference planet reads neighbor data from repos when available, falling back to `manifest.json`.

### Phase 2: Verifiable space ports

- Publish `spacePortKey` in the planet record.
- Controllers verify signatures during `prepare` / `commit`; update `TRAVEL.md` to require it.

### Phase 3: People and ships

- "Sign in with your handle" at the space port.
- Publish the `ship.launch`, `ship.event`, and `person.aboard` lexicons. Planets launch shuttles; signed-in people commission ships at a planet.
- Travel plans move ships; `ship_id` becomes the launch reference. Update `TRAVEL.md` (§4 and §8) accordingly.
- Boarding, leaving, and operator changes. Ship position checked from the log head at the origin.

### Phase 3b: Ship lifecycle

- Ownership transfer, upgrades, and other transactions as further event types.

### Phase 4: Discovery (optional, never required)

- Map and explorer tools that consume a firehose or backlink index to show the whole federation.
- These are conveniences any party may run; the protocol must keep working without them.

### Phase 5: Fully sovereign planets

- Document and template the `did:web` plus self-hosted PDS on Workers configuration in `create-planet`.

## 6. Open Questions

1. **Additive or replacement?** This plan keeps `manifest.json` as a first-class path. Requiring atproto would simplify the spec but raise the bar for joining and add third-party dependencies.
2. **Coordinates from the domain or the DID?** Domain keeps the founding rule and makes relocation possible. DID would make position permanent but break "your domain name is your destiny".
3. **Ships.** Decided: ships are first-class and people take their position from the ship they are aboard (§3.4). Still open: whether ships are event chains on planets (proposed) or their own atproto accounts; exactly what shuttles may carry (passengers, cargo, or both); whether private ships may fly to non-neighbors or have a concurrency cap; and whether transfers and upgrades need elected controllers or only the local planet's signature.
4. **Fog of War.** Accept that travel records are public and keep secrets off-repo, or wait for atproto private data before building cargo features.
5. **`did:plc` acceptance.** Allow it for reach (nearly all existing accounts) while documenting the dependency, or require `did:web` for planets.

---

_Document Version: 0.4.0_

_Status: Exploration_
