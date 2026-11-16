# AT Protocol Plan 3: Ships

Part of the [AT Protocol adoption plan](../ATPROTO.md). This document covers ships. It builds on [Planets](1-planets.md) and [People](2-people.md) and is to be implemented after people can travel on shuttles.

## 1. What a Ship Is

A ship is a first-class entity with a life of its own, not a possession scoped to one account. It is born at a planet, keeps one identity for its whole life, changes owners and operators, gets upgraded, carries many people, and accumulates a log.

- **Position belongs to ships.** A person aboard a ship is wherever the ship is; `person.location` points at the ship instead of a trip. A person not aboard any ship is on the planet where they last disembarked.
- **Travel plans move ships**, not people. Everyone aboard moves with the ship.
- **Shuttles can become ordinary ships owned by a planet**, restricted to that planet's warp ring neighbors and the caps in `TRAVEL.md` §8.

## 2. Proposed Model: A Chain of Signed Events

A ship is not an atproto account and does not live in any one repo. It is a chain of signed events, each stored by the planet where it happened and each pointing at the one before it. The planet a ship is docked at always holds the head of its log.

| Collection                         | Repo                      | Key | Contents                                                                                      |
| ---------------------------------- | ------------------------- | --- | --------------------------------------------------------------------------------------------- |
| `com.federatedplanets.ship.launch` | Planet where it was built | TID | The ship's birth: `name`, `class`, first owner. Its AT-URI and CID are the `ship_id` for life |
| `com.federatedplanets.ship.event`  | Planet where it happened  | TID | One log entry: `ship`, `prev` (previous event), `type`, payload, signatures                   |

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
- **People point at ships.** A person's `person.location` names the ship they are on. It is their consent and a lookup shortcut; the boarding event on the planet is the authority. A person can be aboard only one ship at a time.
- **Everything is public.** A ship's full history, including who was aboard, is readable by anyone.

## 3. Alternative: A Ship as Its Own Atproto Account

Giving each ship a DID and repo would keep its log in one place that survives any planet. But an account is controlled by whoever holds its credentials, so every ship would need a custodian (a "port of registry" planet running a multi-account PDS), ownership transfer would mean handing over account control, and the log could still be rewritten by that controller. A ship account could also be added later as an optional mirror of the chain.

## 4. Surviving Planets That Disappear

In the proposed model a ship has no PDS of its own, so it is exactly as durable as the planets holding its events (see [Planets §5](1-planets.md)). Voyages are held by two planets; every other event by one. If the planet a ship is docked at disappears, nobody is left to confirm its latest state or sign its departure; if the building planet disappears, the launch record can no longer be fetched.

Because every event is signed, copies are as good as originals. Proposed mitigations:

- **The owner carries the full log.** Every event is also written to the current owner's repo, and a new owner copies the whole chain on transfer. Planets become witnesses, not the only store.
- **Controllers keep what they sign**, spreading each voyage across several neighbors.
- **Longest valid chain wins.** An old owner cannot present a shortened log to undo a sale, because the new owner holds the longer one carrying the old owner's signature.
- **Rescue rule.** A ship docked at a vanished planet may depart with approval from that planet's former neighbors.

## 5. Open Questions

1. **Event chain or ship accounts?** The proposed chain needs no extra hosting; accounts are simpler to reason about but need custodians.
2. **Reach.** Whether ships may fly to non-neighbors, and whether they have a concurrency cap like shuttles.
3. **Who must sign local events.** Whether transfers and upgrades need elected controllers or only the local planet's signature.
4. **Where ships come from.** Who may build one, and at what cost (ties into [Goods and Economy](4-economy.md)).

---

_Document Version: 0.7.0_

_Status: Exploration, deferred_
