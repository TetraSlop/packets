# Departures during a match

A seated player can leave an active round by sending
[43/05 Quit](../irc/G/43.md#4305-quit) or
[43/27 Break](../irc/G/43.md#4327-break), or by losing the connection. The
remaining clients learn of it through
[43/23 GetAway](../irc/G/43.md#4323-getaway) and, under rule `gs=0`,
[43/24 Change](../irc/G/43.md#4324-change). The table rule `gs` (see
[rules.md](../client/rules.md)) decides what happens to the departed seat's
hand, its board cards, the turn order and the result. This page covers the
ordinary two- and three-player table on the table service; leaving before
the start is in [match-admission.md](match-admission.md#exit).

## Sequence

1. Client to the table service: [43/05 Quit](../irc/G/43.md#4305-quit),
   argument the client's current screen code, answered to the quitter only with
   the same argument; or [43/27 Break](../irc/G/43.md#4327-break), no
   reply; or the socket is lost. The server records the departure once per
   seat and applies the `gs` rule under Rules to the seat's hand, board
   cards and turn order.
2. Server to every remaining live participant:
   [43/23 GetAway](../irc/G/43.md#4323-getaway) with the departed seat,
   once.
3. Under `gs=0`, server to every remaining live participant:
   [43/24 Change](../irc/G/43.md#4324-change) naming the computer opponent
   that takes over the seat. It must come after GetAway for the same seat.
4. The round continues. A skipped seat's turn is announced with the pass
   form of [43/0A PutCard](../irc/G/43.md#430a-putdata-and-putcard); the
   sequence and the active seat both advance for every skipped seat.
5. Server to every remaining participant, and to a connected voluntary
   quitter: [43/0D ResultData](../irc/G/43.md#430d-resultdata) with the
   actual final sequence as argument, computed with the departed-seat
   ranking under Rules.
6. Each remaining client: [43/0E Ready](../irc/G/43.md#430e-ready) with the
   advanced sequence. The server answers
   [43/1A TCList](../irc/G/43.md#431a-tclist) with the rebuilt hands.
7. Each remaining client: [43/11 Continue](../irc/G/43.md#4311-continue).
   The server's reply is under Exit.

## Rules

### Departure rules

| gs | Name | Departed hand | Departed board cards | Turn order | End of round |
|---|---|---|---|---|---|
| 0 | Substitute | kept, played by the computer opponent named in Change | unchanged | unchanged | ordinary |
| 1 | Invalid | cleared | base owner becomes 3, current owner kept | seat skipped | when one participant remains the game ends with no ResultData |
| 2 | Loss | cleared | unchanged | seat skipped | ordinary, or at the end of the current turn when one participant remains |

Under `gs=1` the base owner 3 only changes how those cells are drawn: the
cards still belong to their original owner for every inventory purpose.

A server applies a departure once per seat: a later Quit, Break or socket
loss for a seat that has already departed changes nothing.

### Turn advance with skipped seats

When a seat is skipped, both the turn sequence and the active seat advance
for every skipped seat, so a three-player round with one `gs=2` departure
still reaches sequence 15. The round is complete when the sequence reaches
`5 * player_count`, or under `gs=2` when only one participant remains. A
state where every seat is skipped does not terminate in the client; a
server must not produce it.

`Ready` carries the advanced sequence as argument: 10 for a complete
two-player round, 15 for three players, and the actual boundary for a round
ended early.

### Result ranking with departed seats

A departed seat's score becomes 0 but its preset class from GetAway
survives. For each surviving seat let `equal` be the number of surviving
seats with the same score (including itself) and `higher` the number with a
strictly higher score:

| gs | class | lower |
|---|---|---|
| 1 | `equal + 2*higher` | `2 - equal - higher` |
| 2 | `2 - player_count + equal + 2*higher` | `player_count - equal - higher` |

The perfect multiplier and the finishing order are preserved. Positions 1
and 2 are written back to their own records and not swapped, so a
three-player departure is not the same as giving the departed seat the
two-player loss class 3.

### Result hands

The hands in TCList are rebuilt by appending each seat's board cards in
ascending board index to its remaining compacted hand. A `gs=1` departed
seat rebuilds zero cards; a `gs=2` departed seat rebuilds only the cards it
played. TCList `H` carries those counts, not 5 for every seat.

### The departed account's cards

What the client does with its own cards on a voluntary exit differs by
rule:

- `gs=1`: it refunds the stake into its local balance, compacts the
  collection, then restores every remaining hand card and every board card
  whose base owner is its own seat, regardless of current owner. Captured
  foreign cards are not restored. The local hand count becomes 0.
- `gs=0`: it compacts the collection only.
- `gs=2`: it compacts the collection only and waits for ResultData.

The client's forced cleanup path in an ordinary game refunds the stake and
restores the same remaining-hand plus own-board-card set for every `gs`.

An empty hand on screen does not mean the cards are lost, and a server
keeps the departed account's five selected cards.

### When a departure applies

Server choice:

- A server treats a dropped socket as a Quit at the same point in the
  round.
- A server applies a departure only in the turn and battle phases. In any
  other phase, and in any round with wagers, card prizes or ratings, the
  match ends and refunds instead ([wagers.md](wagers.md#exit)).
- A departed account keeps its reservation until the round's result is
  stored or the table is reset. A new session cannot bind the account while
  its substitute is still playing.

## Exit

A client that quits under `gs=2` waits for ResultData before it updates its
local departure result. A server must still deliver the terminal ResultData
to a connected quitter, although that client sends no further turn or
result acknowledgements. A dead socket gets nothing.

Server choice: a round that had a departure ends at Continue with the stop
reply (argument 2, `Ans=3|3`) regardless of the vote; no rematch is offered
without the missing participant. After a departure the client's Continue
dialog may preselect leave (unverified).
