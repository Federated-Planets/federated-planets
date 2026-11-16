# AT Protocol Plan 1: Planets

Part of the [AT Protocol adoption plan](../ATPROTO.md). This document covers planets only: their identity, their records, the warp ring, and verifiable space ports. It introduces no traveler identity; `ship_id` stays a free-form string until [People](2-people.md).

This document can be implemented and shipped on its own.

## 1. Planet Identity

- **Planet** = an atproto account whose handle is the planet's domain. The DID is the durable identifier; the domain remains the location.
- **Coordinates stay hashed from the domain** (unchanged MD5 algorithm from `README.md`). If a planet changes its handle it moves in space but keeps its DID, history, and neighbors' links. A domain change becomes "planet relocated" instead of "planet destroyed".
- **Verification:** resolve handle to DID, resolve the DID document, confirm it lists `at://<domain>`, then read the planet record from the PDS named in the document.

## 2. Lexicons

| Collection                             | Repo                   | Key    | Contents                                                                                            |
| -------------------------------------- | ---------------------- | ------ | --------------------------------------------------------------------------------------------------- |
| `com.federatedplanets.planet`          | Planet                 | `self` | `name`, `description`, `landingSite`, `spacePort`, `travelSafetyLevel`, `spacePortKey` (public key) |
| `com.federatedplanets.warpLink`        | Planet                 | TID    | `subject` (neighbor DID), `domain`, optional `name`. One record per neighbor                        |
| `com.federatedplanets.travel.trip`     | Origin and destination | TID    | An approved travel plan: route, times, `ship_id`, controller signatures                             |
| `com.federatedplanets.travel.declined` | Declining planet       | TID    | Declined travel requests with the stated reason (`TRAVEL.md` §7)                                    |

One record per warp link (like a follow) rather than an array in the planet record means mutual-neighbor checks for shuttle limits (`TRAVEL.md` §8) become a single lookup, and optional indexers can answer "who links to me?".

## 3. Verifiable Space Ports

- Election pool, seed, PBFT phases, timing, and shuttle limits are unchanged. Consensus stays on direct HTTP between space ports.
- Each space port keeps its own Ed25519 key and publishes the public half as `spacePortKey` in its planet record.
- Controllers identify themselves by DID and verify each other's signatures during `prepare` and `commit`. Any party can verify the $2f+1$ signatures on a finished plan, closing the auditability gap in `TRAVEL.md` §5.
- Approved trips are written by both origin and destination to their own repos; declined requests by the declining planet. A planet cannot forge a trip (controller signatures) and cannot quietly drop one (the other planet holds a copy).
- Signed records should carry the signers' public keys, so a copy stays checkable if a signing planet later disappears.

## 4. Backward Compatibility

`<link rel="space-manifest">` and `manifest.json` remain valid. The manifest gains an optional `did` field. Atproto-aware planets prefer repo records; a planet with no atproto account still appears on the map and can be linked to, but cannot act as a verified traffic controller once signature verification is enforced.

## 5. Where Planet Data Lives, and What Survives

A planet's records live only on the PDS hosting its account. What happens when the planet's domain is lost depends on the setup:

| Planet's setup                                        | Where the repo is    | If the domain is lost                                                                                              |
| ----------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Hosted account with the domain as handle, `did:plc`   | The hosting provider | Records stay readable by DID. The handle stops verifying, so in game terms the planet has lost its location.       |
| Self-hosted PDS on the planet's own domain, `did:plc` | The planet's server  | Records are unreachable. Recoverable only if the owner kept a repo backup and recovery key and moves to a new PDS. |
| Self-hosted PDS, `did:web`                            | The planet's server  | Gone for good, since the identity itself is the domain.                                                            |

The most sovereign setups are the most fragile. In every case the records are also gone if the owner deletes the account or the host shuts down. Trips are protected against this by being stored on two planets; anything stored on one planet only is exactly as durable as that planet.

### Archive depth

The cap of 10 most recent arrivals and departures from `TRAVEL.md` §4 stays as it is in this document: planets keep only those trip records in their repos. Nothing here depends on older history. The cap is to be removed or made time-based once history starts to matter, first in [People](2-people.md), where a person's location is proven by looking up a past trip.

## 6. Implementation Fit (Reference Planet)

- **Reads:** plain `fetch` to PDS XRPC endpoints, cached in the existing `traffic_controllers` table.
- **Writes:** the space port holds credentials for the planet's own account (OAuth session or app password) in the Durable Object. No other account's permission is needed in this document.
- **Self-hosted PDS:** single-user PDS implementations for Cloudflare Workers and Durable Objects exist. Combined with `did:web`, a planet could be domain, website, space port, and data host in one deployment. This is the most on-ethos configuration and should be an option, not a requirement.

## 7. Phases

### Phase 0: Spike

- Read latency and failure behavior when resolving a ring of neighbors directly from their PDSs.
- Writing records to the planet's own repo from a Worker.

### Phase 1: Identity binding (spec only, no behavior change)

- Add the optional `did` field to the manifest and document handle-based verification in `README.md`.
- Publish the `planet` and `warpLink` lexicons.
- Reference planet reads neighbor data from repos when available, falling back to `manifest.json`.

### Phase 2: Verifiable space ports

- Publish `spacePortKey` in the planet record.
- Controllers verify signatures during `prepare` / `commit`; update `TRAVEL.md` to require it.
- Planets write approved trips and declined requests to their repos.

### Phase 3: Discovery (optional, never required)

- Map and explorer tools that consume a firehose or backlink index to show the whole federation. Conveniences any party may run; the protocol must keep working without them.

### Phase 4: Fully sovereign planets

- Document and template the `did:web` plus self-hosted PDS on Workers configuration in `create-planet`.

## 8. Open Questions

1. **Coordinates from the domain or the DID?** Domain keeps the founding rule and makes relocation possible. DID would make position permanent but break "your domain name is your destiny".

---

_Document Version: 0.7.0_

_Status: Exploration_
