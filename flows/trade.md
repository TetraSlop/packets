# Trade

A trade is a two-player card exchange with no money. The players agree
through peer-relayed controls, one of them registers the pair with the
trade service ([services.md](../irc/services.md#trade-service)) on
[A0](../irc/G/A0.md), both upload offers, each selects cards from the
other's offer, both confirm, and both acknowledge the exchange, all on
[A2](../irc/G/A2.md). Card tuples and the inventory formulas are in
[catalogue.md](../client/catalogue.md). The flow starts with an invitation
between two room members and ends with each side's Quit.

## Sequence

1. Inviter to invitee, relayed: [21/00 Tr](../irc/G/21.md#2100-tr), the
   invitation with the inviter's identity. The server relays it without
   answering it, and only between a pair allowed by the
   [peer relay guard](../irc/envelope.md#relay-guard).
2. Invitee to inviter, relayed: [22/00 TrAns](../irc/G/22.md#2200-trans)
   with `Ans=1` and the invitee's identity to accept, or `Ans=-1` alone to
   refuse. Either peer may instead send
   [23/00 TrCan](../irc/G/23.md#2300-trcan), answered by the other peer
   with [2A/00 TrCanAck](../irc/G/2A.md#2a00-trcanack), or
   [25/00 Tr](../irc/G/25.md#2500-tr) to reject. The server records the
   invitation and the positive answer; the accepted inviter is side 0 and
   the invitee side 1.
3. Side 0 to side 1, relayed: [A0/00 TrID](../irc/G/A0.md#a000-trid-and-trent)
   with the trade service identity.
4. Side 0 to the trade service:
   [A0/00 TrEnt](../irc/G/A0.md#a000-trid-and-trent) with both identities.
   The server waits for this real `TrEnt` (the client discards two queued
   messages while it is still polling for 21, 22 and 25), then sends both
   players [A1/00 TrEntAns](../irc/G/A1.md#a100-trentans) and
   [A2/18 Init](../irc/G/A2.md#a218-init), whose `ID` is the session of
   every later A2 message. Both clients poll for A1.
5. Each side to the trade service: its offer as
   [A2/1A Card](../irc/G/A2.md#a21a-card) pages of at most 15 cards; a
   30-card offer is two 15-card pages sent together. The server checks
   that every card is an owned copy and relays the complete offer to the
   other side in the same format. A new offer clears the other side's
   selection and both confirmations. Returning the offered cards sends
   `E=0`.
6. Each side to the trade service: its selection over the other side's
   offer as [A2/1B Data](../irc/G/A2.md#a21b-data) with its state `M`. The
   server forwards it to the other side. A changed selection drops the
   other side's ready state back to 1. A side cannot send `M=2` while an
   offer upload is incomplete.
7. When both sides are in state 2, server to both:
   [A2/1C Data](../irc/G/A2.md#a21c-data-and-ans) `M=3`. Each client
   answers with `Ans` only when its own state and the peer state are both
   2. When both answers have arrived the server sends both `Data` `M=4`
   once, which reconciles the inventories in the client.
8. Either side to the trade service, at any point:
   [A2/10 Erase](../irc/G/A2.md#a210-erase) and
   [A2/22 Decks](../irc/G/A2.md#a222-decks). Before `M=4` they apply to the
   side's own inventory excluding its offered cards; after `M=4` to the
   reconciled inventory, so overflow can be resolved by discarding incoming
   or existing cards.
9. Each side to the trade service: [A2/1D Get](../irc/G/A2.md#a21d-get)
   once its reconciled inventory fits 1000 cards. When both Gets have
   arrived the server commits both inventories and answers each side with
   its own [A2/1E Quit](../irc/G/A2.md#a21e-quit). Money is unchanged.

## Rules

### Service identity

The client finds the trade service in the room's PTL: it takes the last
table record whose metadata begins with `D`
([directory-files.md](../file/directory-files.md), metadata in
[L.md](../irc/L.md#table-metadata)). `D` records are not playable tables.
All A1 and A2 replies must come from this identity, never from a player.

Server choice: raw POL service 0x400, game identity
`0x400 XOR 0xB62DC8D0E4BCBB90`, channel `#LOCALTRADE1`, metadata `D`
followed by 48 `A` bytes.

### Sides and authentication

The accepted inviter is side 0 and the invitee side 1. When both players
invite each other at once, the lower unsigned game identity becomes side
0. The client does not authenticate the sender of A1 or A2: `TrEntAns`
adopts whatever sender it sees and `Init` replaces the current service and
session and resets offers, so a server must not relay A1 or A2 between
players, and must relay only the controls of steps 1 to 3 between a
validated pair.

Server choice: the pair is admitted only after the server has seen the 21
invitation and a positive 22 answer between exactly these two players;
admission consumes every outstanding invitation involving either player.

### Offer pages

Server choice: a relayed page is shrunk below 15 records when its body
would exceed 447 bytes (a maximum-width 30-card offer relays as 14, 14, 2);
an exact resend of a page already buffered is ignored, and a changed page
replaces the buffered suffix from its `S`.

### Confirmation

`Data` `M=4` must be sent once; a duplicate answer or a late answer after
an offer change must not trigger it again.

### Completion

Server choice:

- A server replaces both committed inventories atomically.
- A repeated Get after commit replays that side's Quit.
- A Decks edit after commit still applies to the committed inventory while
  the account is idle.

## Exit

Before entry, a cancellation (`TrCan`, acknowledged with `TrCanAck`), a
rejection (25/00 `Tr`) or a refusal (`TrAns` with `Ans=-1`) ends the
invitation; nothing was registered, so nothing is undone.

After entry, a client leaves with [A2/19 Quit](../irc/G/A2.md#a219-quit);
the server then sends it to both. On timeout or error the server sends
[A2/27 Dead](../irc/G/A2.md#a227-dead) to both. Neither undoes an `M=4`
reconciliation already displayed, so after either message the clients must
reload their committed profile. The server discards both offers and
selections and leaves both committed inventories as they were before the
trade, unless both Gets had already arrived.
