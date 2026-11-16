# AT Protocol Plan 2: People

Part of the [AT Protocol adoption plan](../ATPROTO.md). This document covers people: signing in with an atproto identity and traveling on planet-sponsored shuttles. It builds on [Planets](1-planets.md) and needs nothing from ships or the economy.

## 1. Identity

- **Person** = any atproto account, signed in with atproto OAuth at a space port. A person does not need to own a planet.
- **Anonymous visitors** have no identity. They may ride shuttles as passengers but have no persistent location or history. No account is needed to explore.
- **Shuttle** = a planet-funded vessel running between warp ring neighbors. Shuttles are the only way to travel until [Ships](3-ships.md). They carry people only.
- **Position.** A signed-in person is on a planet or in transit on a shuttle. Their position comes from the last signed trip that carried them.

|                     | Anonymous visitor    | Signed-in person                                               |
| ------------------- | -------------------- | -------------------------------------------------------------- |
| Identity            | None                 | atproto DID                                                    |
| Ride shuttles       | Yes, as a head count | Yes, named in the manifest                                     |
| Record of the trip  | None                 | Named in the planets' `travel.trip`; `person.location` updated |
| Persistent location | No                   | Yes                                                            |
| Travel history      | No                   | Yes, readable from the planets visited                         |

## 2. Lexicons

| Collection                             | Repo                   | Key    | Contents                                                                              |
| -------------------------------------- | ---------------------- | ------ | ------------------------------------------------------------------------------------- |
| `com.federatedplanets.travel.trip`     | Origin and destination | TID    | Extended from [Planets](1-planets.md) with a passenger manifest                       |
| `com.federatedplanets.person.location` | Person                 | `self` | Pointer to the trip that last moved this person; a lookup shortcut, not the authority |

## 3. What Changes in the Travel Protocol

- A travel plan is a **shuttle trip** with a manifest: the DIDs of signed-in passengers and a head count of anonymous passengers. The free-form `ship_id` is retired.
- A signed-in passenger consents by being signed in at the origin space port when the plan is filed.
- Controllers check that each signed-in passenger is at the origin, by reading the origin planet's records. The planets' records are the authority; a person who deletes their own `person.location` only loses the shortcut.
- Routes and capacity follow the shuttle limits in `TRAVEL.md` §8.
- A person's first appearance needs a rule, since they have no prior trip (open question 1).

## 4. Implementation Fit (Reference Planet)

- **Sign-in:** atproto OAuth as a confidential client with DPoP, sessions stored in Durable Object SQLite. Library support on Workers is the main technical unknown.
- **Progressive enhancement:** the space port keeps working for visitors who never sign in.
- **Writes to a person's repo** are limited to their own `person.location`, made through their session while they are present.

## 5. Phases

### Phase 0: Spike

- Atproto OAuth on Workers with tokens in Durable Object SQLite.

### Phase 1: Sign-in

- "Sign in with your handle" at the space port; show who is signed in.

### Phase 2: Named passengers

- Publish the `person.location` lexicon and the extended `travel.trip`.
- Travel plans gain a passenger manifest; anonymous visitors keep working as a head count.
- Controllers validate signed-in passengers' locations.
- Update `TRAVEL.md` (§4 and §8) to describe shuttles as the travel vehicle and the two kinds of passenger.

## 6. Open Questions

1. **Where does a person start?** Options: the planet where they first sign in, or a home position hashed from their own handle.
2. **Must a person be at the origin?** A person signing in at a planet's website from elsewhere in the universe: is that allowed for viewing only, or is their location simply wherever their last trip left them?
3. **Stranded people.** What happens to a person whose current planet disappears.

---

_Document Version: 0.7.0_

_Status: Exploration_
