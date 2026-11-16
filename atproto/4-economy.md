# AT Protocol Plan 4: Goods and Economy

Part of the [AT Protocol adoption plan](../ATPROTO.md). This document is a placeholder for goods, trade, and currency. Nothing here is designed yet, and nothing in [Planets](1-planets.md) or [People](2-people.md) should assume its design.

## 1. Starting Points

These follow from decisions already made in the earlier documents, and are assumptions to revisit, not a design:

- **Only signed-in people can own or trade.** Anonymous visitors have no identity to own with.
- **Planets are the record keepers.** Holdings on a planet would be recorded by that planet, alongside its traffic log.
- **Trading** would be a transfer between two parties on the same planet, recorded by that planet.
- **Shipping** would be cargo added to a trip manifest, moved by the same consensus that approves the trip: debited at the origin, credited at the destination, so goods cannot be in two places.
- **Capacity** would follow the vessel: shuttle limits in `TRAVEL.md` §8 make warp ring links economically meaningful, and [Ships](3-ships.md) would lift those limits.
- **Everything is public.** Private cargo conflicts with atproto as it stands (see the overview's Fog of War question).

## 2. Open Questions

1. **Goods.** What is tradeable and who creates it. Planets issuing their own goods is the natural sovereign answer.
2. **Provenance.** Whether a good's identity includes its issuing planet, and how other planets detect a planet inflating stock it did not issue.
3. **Currency.** One shared currency, per-planet currencies, or barter only; and who is allowed to create it.
4. **Durability.** Holdings recorded on one planet are lost if that planet disappears (see [Planets §5](1-planets.md)); decide whether owners carry signed proof of what they hold.
5. **Ordering.** Whether this comes before or after ships.

---

_Document Version: 0.7.0_

_Status: Placeholder_
