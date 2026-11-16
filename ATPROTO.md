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
- **Ship** = a vessel, separate from the people aboard it. A ship is a record in its owner's repo and is identified by that record's AT-URI, which becomes the `ship_id`. Ships are not accounts and never sign in; authority over a ship comes from its owner's DID.
- **Shuttle** = a ship owned and funded by a planet (the record lives in the planet's repo).
- **Anonymous visitors** have no identity and may ride shuttles only (see §3.4). No account is needed to explore.
- **Verification** of a planet: resolve handle to DID, resolve DID document, confirm it lists `at://<domain>`, read the planet record from the PDS named in the document.

### 3.2 Lexicons

NSIDs live under the project's domain (`com.federatedplanets.*`) and are published as `com.atproto.lexicon.schema` records with a `_lexicon` DNS TXT record.

| Collection                               | Repo                       | Key    | Contents                                                                                           |
| ---------------------------------------- | -------------------------- | ------ | -------------------------------------------------------------------------------------------------- |
| `com.federatedplanets.planet`            | Planet                     | `self` | `name`, `description`, `landingSite`, `spacePort`, `travelSafetyLevel`, `spacePortKey` (public key) |
| `com.federatedplanets.warpLink`          | Planet                     | TID    | `subject` (neighbor DID), `domain`, optional `name`. One record per neighbor                        |
| `com.federatedplanets.ship`              | Ship owner (person/planet) | TID    | `name`, `class` (`ship` or `shuttle`), optional description. Its AT-URI is the `ship_id`           |
| `com.federatedplanets.travel.voyage`     | Ship owner ("ship's log")  | TID    | The approved travel plan for the ship, with all controller signatures                              |
| `com.federatedplanets.travel.journey`    | Person ("passport")        | TID    | The same signed plan, recording that this person was aboard                                        |
| `com.federatedplanets.travel.logEntry`   | Origin and destination     | TID    | Same signed plan, forming the public Mission Archive; also declined requests with their reason     |

One record per warp link (like a follow) rather than an array in the planet record means mutual-neighbor checks for shuttle limits (`TRAVEL.md` §8) become a single lookup, and optional indexers can answer "who links to me?".

### 3.3 What changes in the travel protocol

- Election pool, seed, PBFT phases, timing, and shuttle limits are unchanged.
- Controllers identify themselves by DID and sign with the key published in their planet record. Any party can now actually verify the $2f+1$ signatures, closing the auditability gap in `TRAVEL.md` §5.
- A travel plan now names a **ship** plus a **manifest**: the DIDs of signed-in people aboard, a count of anonymous passengers (shuttles only), and cargo if any.
- Two locations are tracked independently: the ship's (from its last signed voyage) and each person's (from their last signed journey). Controllers check that the ship is at the origin and that every listed person is at the origin too.
- A plan must be authorized by the ship's owner: the signed-in owner for a private ship, the origin planet for a shuttle. Each listed passenger consents by being signed in at the origin space port.
- On approval, the origin space port writes the signed plan to its own repo, to the ship's log, and to each signed-in passenger's passport (via their OAuth session).
- Locations are cross-checked with the origin's own state. Someone who deletes their own records only harms themselves.

### 3.4 People and ships

People and ships are separate. Who you are decides what you may board; what you board decides what the trip can do.

**People**

| | Anonymous visitor | Signed-in person |
| --- | --- | --- |
| Identity | None | atproto DID |
| May board | Shuttles only | Shuttles and private ships |
| Record of the trip | None; only counted in the shuttle's manifest | `travel.journey` in their passport |
| Persistence | None | Portable history and verified current location |

**Ships**

| | Planet-sponsored shuttle | Private ship |
| --- | --- | --- |
| Owner and accountable party | The sponsoring planet | A signed-in person |
| Record | `ship` in the planet's repo | `ship` in the owner's repo |
| Routes | Warp ring neighbors only, within the shuttle limits of `TRAVEL.md` §8 | Set by the travel protocol, not by shuttle capacity |
| Capabilities | Restricted (see open question 3) | Full: everything that needs a durable, owned vessel |

Shuttles keep casual visiting frictionless while every vessel still has an accountable owner, and the existing shuttle caps bound how much anonymous traffic any planet can generate. A signed-in person riding a shuttle still gets a passport entry, so their location stays consistent whichever kind of vessel they use.

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
- Publish the `ship` lexicon; `ship_id` becomes a ship record's AT-URI. Planets register their shuttles; signed-in people can register ships.
- Extend the travel plan with a passenger and cargo manifest; update `TRAVEL.md` (§4 and §8) accordingly.
- Write `travel.voyage`, `travel.journey`, and `travel.logEntry` records.
- Validate ship and person locations from signed records.

### Phase 4: Discovery (optional, never required)

- Map and explorer tools that consume a firehose or backlink index to show the whole federation.
- These are conveniences any party may run; the protocol must keep working without them.

### Phase 5: Fully sovereign planets

- Document and template the `did:web` plus self-hosted PDS on Workers configuration in `create-planet`.

## 6. Open Questions

1. **Additive or replacement?** This plan keeps `manifest.json` as a first-class path. Requiring atproto would simplify the spec but raise the bar for joining and add third-party dependencies.
2. **Coordinates from the domain or the DID?** Domain keeps the founding rule and makes relocation possible. DID would make position permanent but break "your domain name is your destiny".
3. **Shuttle and ship limits.** Decided: people and ships are separate; anonymous visitors ride shuttles only; signed-in people may use either (§3.4). Still open: exactly what shuttles may carry (passengers, cargo, or both), whether private ships may fly to non-neighbors or have a concurrency cap, and how ship ownership is transferred given that repo records are mutable.
4. **Fog of War.** Accept that travel records are public and keep secrets off-repo, or wait for atproto private data before building cargo features.
5. **`did:plc` acceptance.** Allow it for reach (nearly all existing accounts) while documenting the dependency, or require `did:web` for planets.

---

_Document Version: 0.3.0_

_Status: Exploration_
