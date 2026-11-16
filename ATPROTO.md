# AT Protocol Adoption: Overview

This document evaluates moving Federated Planets onto the [AT Protocol](https://atproto.com) (atproto) for identity and data storage. The plan itself is split into four documents, to be implemented in order:

| #   | Document                                  | Scope                                                              | Status            |
| --- | ----------------------------------------- | ------------------------------------------------------------------ | ----------------- |
| 1   | [Planets](atproto/1-planets.md)           | Planet identity, planet records, warp ring, verifiable space ports | First             |
| 2   | [People](atproto/2-people.md)             | Signing in with an atproto identity and traveling on shuttles      | Second            |
| 3   | [Ships](atproto/3-ships.md)               | Ships as first-class entities                                      | Later             |
| 4   | [Goods and Economy](atproto/4-economy.md) | Goods, trade, and currency                                         | Later, undesigned |

Each document depends only on the ones before it.

**Verdict:** feasible and a good fit for _identity_ and _public game state_; a poor fit for _consensus messaging_ and _private data_. The recommendation is a **hybrid, additive adoption** rather than a full move: atproto becomes the identity and record layer, while space ports keep talking to each other directly over HTTP.

## 1. Where the Game Stands Today

| Concern             | Today                                                                            | Gap                                                                            | Addressed in |
| ------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | ------------ |
| Planet identity     | The domain itself; `<link rel="space-manifest">` on the homepage                 | Works, but identity cannot survive a domain change                             | Planets      |
| Planet data         | `manifest.json` plus warp ring links in HTML/config                              | No way to ask "who links to me?" without crawling                              | Planets      |
| Space port identity | Ed25519 key pair generated and kept inside the planet's Durable Object           | Public key is never published, so signatures on travel plans cannot be audited | Planets      |
| Travel ledger       | Each space port's own SQLite (Durable Object), last 10 records shown             | Not readable in a standard way; no portable proof of journeys                  | Planets      |
| Traveler identity   | `ship_id`, a free-form string                                                    | No traveler identity at all: anyone can claim any ship                         | People       |
| Consensus           | Direct HTTP between space ports (`initiate` / `prepare` / `commit` / `register`) | Fine as is                                                                     | Unchanged    |

## 2. Ethos Fit

### What matches

- **"Your domain name is your destiny."** Atproto handles _are_ domain names, verified bidirectionally (DNS `_atproto` TXT record or `/.well-known/atproto-did` on the domain, and the DID document pointing back). Proving a planet's domain and proving an atproto handle are the same act.
- **One planet per domain.** An account has exactly one primary handle, so one account maps to one planet.
- **Sovereign hosting.** Data lives in a repository the owner controls and can migrate between hosts. A planet can self-host its PDS, including on its own domain.
- **No central game server.** Every record the game needs can be read straight from the owner's PDS (`com.atproto.repo.getRecord` / `listRecords`). No relay, AppView, or game-run indexer is required for the core protocol.
- **Static planets get easier.** A static site cannot run a space port today and must redeploy to change its warp ring. With records in a repo, a static planet edits its warp ring without touching the site.

### Where it strains

- **Hidden centralization.** Most accounts use `did:plc` (resolved through a single directory) and are hosted on Bluesky's PDS. A planet built that way depends on two third parties that a plain website does not. Mitigations: support `did:web` (fully domain-rooted), never require a relay or AppView, and keep `manifest.json` valid so a planet can exist with no atproto account at all.
- **No network-wide storage.** A repo lives on exactly one PDS, the one hosting that account. Nobody else is obliged to keep a copy, and relays are not archives. Anything that must outlive its host needs replication designed into the game (see [Planets §5](atproto/1-planets.md) and [Ships §4](atproto/3-ships.md)).
- **Everything is public.** Atproto repositories have no private records. This conflicts directly with the "Fog of War" goal in `TRAVEL.md` §6. Secrets must stay off-repo (space port to space port), or be published only as hash commitments.
- **Repos are mutable.** An owner can delete or rewrite their own records. A repo is a publication medium, not a ledger. Game records must therefore be self-certifying (carrying controller signatures).
- **Not a message bus.** PBFT rounds through the firehose would require a relay (centralizing) and add latency for no benefit. Consensus stays on direct HTTP.
- **Signing keys are not yours to use.** The atproto signing key (secp256k1 / P-256) is held by the PDS and signs repo commits only. Space ports keep their own Ed25519 key and _publish_ the public half in the repo.

## 3. Shared Conventions

- **Lexicons.** NSIDs live under the project's domain (`com.federatedplanets.*`) and are published as `com.atproto.lexicon.schema` records with a `_lexicon` DNS TXT record. Each document lists the collections it introduces.
- **Planets are the record keepers.** Finished, signed results are stored by planets in their own repos. In-flight consensus state stays in the space port's own storage and is never published.
- **Reference stack is unchanged.** Astro SSR on Cloudflare Workers with a SQLite-backed Durable Object. Reads are plain `fetch` calls to PDS XRPC endpoints; the Durable Object stays authoritative for in-flight state.

## 4. Cross-Cutting Open Questions

1. **Additive or replacement?** The plan keeps `manifest.json` as a first-class path. Requiring atproto would simplify the spec but raise the bar for joining and add third-party dependencies.
2. **`did:plc` acceptance.** Allow it for reach (nearly all existing accounts) while documenting the dependency, or require `did:web` for planets.
3. **Fog of War.** Accept that game records are public and keep secrets off-repo, or wait for atproto private data.

Questions specific to one area are listed in that area's document.

---

_Document Version: 0.7.0_

_Status: Exploration_
