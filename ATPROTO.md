# AT Protocol Exploration: Identity and Data Storage

This document evaluates moving Federated Planets onto the [AT Protocol](https://atproto.com) (atproto) for identity and data storage, and proposes a phased plan.

**Verdict:** feasible and a good fit for _identity_ and _public game state_; a poor fit for _consensus messaging_ and _private data_. The recommendation is a **hybrid, additive adoption** rather than a full move: atproto becomes the identity and record layer, while space ports keep talking to each other directly over HTTP.

## 1. Where the Game Stands Today

| Concern             | Today                                                                            | Gap                                                                            |
| ------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Planet identity     | The domain itself; `<link rel="space-manifest">` on the homepage                 | Works, but identity cannot survive a domain change                             |
| Planet data         | `manifest.json` plus warp ring links in HTML/config                              | No way to ask "who links to me?" without crawling                              |
| Space port identity | Ed25519 key pair generated and kept inside the planet's Durable Object           | Public key is never published, so signatures on travel plans cannot be audited |
| Traveler identity   | `ship_id`, a free-form string                                                    | No traveler identity at all: anyone can claim any ship                         |
| Travel ledger       | Each space port's own SQLite (Durable Object), last 10 records shown             | Travelers hold no portable proof of their journeys                             |
| Consensus           | Direct HTTP between space ports (`initiate` / `prepare` / `commit` / `register`) | Fine as is                                                                     |

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
- **Shuttle** = a planet-funded vessel running between warp ring neighbors. Shuttles are the first and, initially, the only way to travel. They carry passengers and cargo.
- **Anonymous visitors** have no identity. They may ride shuttles as passengers but cannot own or trade anything. No account is needed to explore.
- **Position.** A person is on a planet or in transit on a shuttle. Their position comes from the last signed trip that carried them.
- **Ship** (later) = a first-class entity with a life of its own. Introduced only after shuttle travel and trade work; see §3.5.
- **Verification** of a planet: resolve handle to DID, resolve DID document, confirm it lists `at://<domain>`, read the planet record from the PDS named in the document.

### 3.2 Lexicons

NSIDs live under the project's domain (`com.federatedplanets.*`) and are published as `com.atproto.lexicon.schema` records with a `_lexicon` DNS TXT record.

| Collection                             | Repo                      | Key    | Contents                                                                                            |
| -------------------------------------- | ------------------------- | ------ | --------------------------------------------------------------------------------------------------- |
| `com.federatedplanets.planet`          | Planet                    | `self` | `name`, `description`, `landingSite`, `spacePort`, `travelSafetyLevel`, `spacePortKey` (public key) |
| `com.federatedplanets.warpLink`        | Planet                    | TID    | `subject` (neighbor DID), `domain`, optional `name`. One record per neighbor                        |
| `com.federatedplanets.travel.trip`     | Origin and destination    | TID    | An approved shuttle trip: route, times, passenger and cargo manifest, controller signatures         |
| `com.federatedplanets.travel.declined` | Declining planet          | TID    | Declined travel requests with the stated reason (`TRAVEL.md` §7)                                    |
| `com.federatedplanets.trade.transfer`  | Planet where it happened  | TID    | Goods changing hands between two people on the same planet                                          |
| `com.federatedplanets.person.location` | Person                    | `self` | Pointer to the trip that last moved this person; a lookup shortcut, not the authority               |
| `com.federatedplanets.ship.launch`     | Planet where it was built | TID    | _Later._ The ship's birth. Its AT-URI and CID are the `ship_id` for life                            |
| `com.federatedplanets.ship.event`      | Planet where it happened  | TID    | _Later._ One entry in a ship's log: `ship`, `prev` (previous event), `type`, payload, signatures    |

### 3.3 What changes in the travel protocol

- Election pool, seed, PBFT phases, timing, and shuttle limits are unchanged.
- Controllers identify themselves by DID and sign with the key published in their planet record. Any party can now actually verify the $2f+1$ signatures, closing the auditability gap in `TRAVEL.md` §5.
- A travel plan is a **shuttle trip** with a manifest: the DIDs of signed-in passengers, a head count of anonymous passengers, and any cargo with its owner.
- Controllers check that each signed-in passenger and each cargo item is at the origin, by reading the origin planet's records.
- Finished, signed results are stored by **planets in their own repos**: an approved trip is written by both origin and destination. In-flight consensus state stays in the space port's own storage and is never published.
- A planet cannot forge a trip (controller signatures) and cannot quietly drop one (the other planet holds a copy).

### 3.4 Shuttles: travel and trade (first release)

Everything in the first release runs on shuttles, so no ship model is needed yet. A shuttle is just capacity on a route that a planet funds; the trip record is the only thing that persists.

**Travel**

|                     | Anonymous visitor    | Signed-in person                                               |
| ------------------- | -------------------- | -------------------------------------------------------------- |
| Identity            | None                 | atproto DID                                                    |
| Ride shuttles       | Yes, as a head count | Yes, named in the manifest                                     |
| Record of the trip  | None                 | Named in the planets' `travel.trip`; `person.location` updated |
| Persistent location | No                   | Yes                                                            |
| Own and trade goods | No                   | Yes                                                            |

**Trade**

- **Goods sit on planets.** Each planet keeps the record of who holds what there, the same way it keeps its traffic log. Goods are owned by signed-in people (or by the planet itself).
- **Trading** is a transfer between two people who are on the same planet, recorded by that planet.
- **Shipping** is cargo on a shuttle trip. The same consensus that approves the trip moves the goods: debited at the origin, credited at the destination, signed by the elected controllers, so goods cannot be in two places.
- **Cargo-only trips** are allowed: a person can send goods to someone on a neighboring planet without travelling.
- **Capacity** is bounded by the shuttle limits in `TRAVEL.md` §8, which makes warp ring links economically meaningful: mutual neighbors get twice the trade capacity.

What goods actually are, where they come from, and what stops a planet inflating its own stock is game design that still needs to be written (open question 6).

### 3.5 Ships (later)

Ships come after shuttle travel and trade are working. When they arrive, people aboard a ship take their position from the ship, and `person.location` points at the ship instead of a trip. The intended model:

A ship is not an atproto account and does not live in any one repo. It is a **chain of signed events**, each stored by the planet where it happened and each pointing at the one before it. The chain travels with the ship: the planet a ship is docked at always holds the head of its log.

| Event              | Where it is recorded       | Authorized by                               |
| ------------------ | -------------------------- | ------------------------------------------- |
| Launch             | Planet that built the ship | The planet and the first owner              |
| Voyage             | Origin and destination     | Owner or operator, plus elected controllers |
| Boarding / leaving | Planet where the ship is   | The person, plus owner or operator          |
| Operator change    | Planet where the ship is   | Owner                                       |
| Ownership transfer | Planet where the ship is   | Old owner and new owner                     |
| Upgrade            | Planet performing it       | Owner, plus the planet                      |
| Other transactions | Planet where it happened   | Per transaction type                        |

Consequences:

- **Identity outlives ownership.** Selling a ship appends a transfer event; the `ship_id`, history, and upgrades stay with the ship.
- **Roles are state, not scope.** Owner, operators, and passengers are derived by replaying the log. Owners and operators may file travel plans; passengers may not.
- **People point at ships.** A person's `person.location` record names the ship they are on. It is the person's consent and a lookup shortcut; the boarding event on the planet is the authority. A person can be aboard only one ship at a time.
- **Shuttles can then become ordinary ships owned by a planet**, restricted to that planet's warp ring neighbors and the caps in `TRAVEL.md` §8. Anonymous visitors ride as a head count and leave no record; signed-in people riding a shuttle board it like any other ship.
- **Everything is public.** A ship's full history, including who was aboard, is readable by anyone.

**Alternative considered: a ship as its own atproto account.** Giving each ship a DID and repo would make its log self-contained and portable. But an account is controlled by whoever holds its credentials, so every ship would need a custodian (a "port of registry" planet running a multi-account data server), ownership transfer would mean handing over account control, and the log could still be rewritten by that controller. The event chain gets the same "life of its own" with no extra hosting and with transfers validated by the game itself. A ship account could be added later as an optional mirror of the chain.

**Known weakness:** if a planet disappears, the events it stored go with it. The chain stays verifiable from the surviving links, but history from that planet is lost unless the owner (or anyone) kept a copy. Owners mirroring their ship's log into their own repo is the suggested mitigation.

### 3.6 Backward compatibility

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

### Phase 3: People on shuttles

- "Sign in with your handle" at the space port. Anonymous visitors keep working as head-count passengers.
- Publish the `travel.trip` and `person.location` lexicons; travel plans gain a passenger manifest.
- Planets write approved trips and declined requests to their repos.
- Controllers validate signed-in passengers' locations from the origin planet's records.
- Update `TRAVEL.md` (§4 and §8) to describe shuttles as the travel vehicle and the two kinds of passenger.

### Phase 4: Trade on shuttles

- Design goods (open question 6), then holdings on planets and same-planet transfers.
- Cargo in the trip manifest, including cargo-only trips, moved by the same consensus as the trip.

### Phase 5: Ships

- Launch, voyages, boarding, operators; then ownership transfer and upgrades (§3.5).
- People aboard a ship take their position from it; cargo can travel by ship.

### Phase 6: Discovery (optional, never required)

- Map and explorer tools that consume a firehose or backlink index to show the whole federation.
- These are conveniences any party may run; the protocol must keep working without them.

### Phase 7: Fully sovereign planets

- Document and template the `did:web` plus self-hosted PDS on Workers configuration in `create-planet`.

## 6. Open Questions

1. **Additive or replacement?** This plan keeps `manifest.json` as a first-class path. Requiring atproto would simplify the spec but raise the bar for joining and add third-party dependencies.
2. **Coordinates from the domain or the DID?** Domain keeps the founding rule and makes relocation possible. DID would make position permanent but break "your domain name is your destiny".
3. **Ships (deferred).** Decided: shuttles first, carrying passengers and cargo; ships are first-class entities introduced later (§3.5). Open for later: event chains on planets (proposed) or ships as their own atproto accounts; whether ships may fly to non-neighbors or have a concurrency cap; whether transfers and upgrades need elected controllers.
4. **Fog of War.** Accept that travel records are public and keep secrets off-repo, or wait for atproto private data before building cargo features.
5. **`did:plc` acceptance.** Allow it for reach (nearly all existing accounts) while documenting the dependency, or require `did:web` for planets.
6. **Goods.** What is tradeable, who creates it (planets issuing their own goods is the natural sovereign answer), whether a good's identity includes its issuing planet, and how other planets detect a planet that inflates stock it did not issue.

---

_Document Version: 0.5.0_

_Status: Exploration_
