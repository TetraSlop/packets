# Event matches

The event match service ([services.md](../irc/services.md#event-match-service))
admits a room member to an event match, lets the client pick five owned
cards to borrow, proposes a table through the room identity, and then runs
a round on that table with event-specific start and result messages. Owned
cards, ratings, money and ordinary statistics are not changed by an event
round. The service is advertised in the event room's `PTL`
([event-rooms.md](event-rooms.md#rules)). The flow starts once the client
is present in the event room and ends when the round's result screen
closes.

## Sequence

1. Client to the event match service:
   [98/00 EInitReq](../irc/G/98.md#9800-einitreq), after the client is
   present in the event room. The server looks the member up by `NN` and
   answers [99/00 EInitEnt](../irc/G/99.md#9900-einitent): `Ans=1`
   continues below, `Ans=2` is the non-playing entry of
   [event-rooms.md](event-rooms.md#rules), the negative values deny. A
   client with fewer than five owned cards stops at its own shortage
   dialog.
2. Event match service to client:
   [B2/13 EInit](../irc/G/B2.md#b213-init) with argument 0. `N` is the
   session the client repeats as the argument of every later `B2` message.
3. Client to the event match service:
   [B2/07 CardSelect](../irc/G/B2.md#b207-cardselect) with five owned
   cards, argument the session. The server checks that the five are owned
   with correct multiplicity, confirms with `Select` and closes the session
   with [EQuit](../irc/G/B2.md#b217-quit).
   [B2/22 Decks](../irc/G/B2.md#b222-decks) edits are accepted during the
   session with no reply. Card data is described in
   [catalogue.md](../client/catalogue.md#card-values-on-the-wire).
4. When the room is eligible ([event-rooms.md](event-rooms.md#rules)) and
   enough selected players are waiting, the server publishes the table
   state through the room `DD` and sends each proposed participant
   [D2/00 MuchMake](../irc/G/D2.md#d200-muchmake) from the room identity.
   Each client answers [D3/00 MuchMakeAns](../irc/G/D3.md#d300-muchmakeans).
   The server sends [D2/01 Start=1](../irc/G/D2.md#d201-muchmake-start)
   only after every participant answered `Ans=1`; `Start=-1` cancels an
   offer that was already accepted.
5. Client to the table: [41/01 GameEN](../irc/G/41.md#4101-gameen). The
   table must already be published as an `A` record. The server answers
   every participant [41/02 GameEA](../irc/G/41.md#4102-gameea) `EN=1`;
   there is no owner and no settings editor.
6. The client JOINs the table channel and sends
   [01/00 Req](../irc/G/01.md#0100-req). The server answers
   [02/00 PLAYACK](../irc/G/02.md#0200-playack), whose `Tet` and `Tab`
   must agree with the published `A` record. No `GameReady` is sent on this
   path.
7. Client to the table: [41/08 GameOK](../irc/G/41.md#4108-gameok). After
   the last one the server sends
   [41/09](../irc/G/41.md#4109-start-confirmation) to every participant and
   starts the round.
8. Table identity to every participant:
   [43/01 EventGameInit](../irc/G/43.md#4301-vsgameinit), then a private
   [43/07 EventCard](../irc/G/43.md#4307-cardselect) to each seat with
   that seat's five borrowed cards, then the ordinary
   [43/08 StartData](../irc/G/43.md#4308-startdata) and
   [43/09 TurnData](../irc/G/43.md#4309-turndata). Moves use the ordinary
   TurnData, [43/0A PutData](../irc/G/43.md#430a-putdata-and-putcard) and
   [43/0B BattleSelect](../irc/G/43.md#430b-battleselect).
9. Server to every participant:
   [43/0D EventResult](../irc/G/43.md#430d-resultdata) with the final
   sequence as argument. The client sends
   [43/0E Ready](../irc/G/43.md#430e-ready) twice; the server answers the
   first valid `Ready` with [43/11 Continue](../irc/G/43.md#4311-continue),
   argument 2 and `Ans=3` for every seat, which exits the result screen.

## Rules

### Selection session

Server choice: a selection session expires after 120 seconds.

### Proposal

Server choice: a declined offer is retried after 2 seconds; the offer and
the table admission share a 120-second deadline from the oldest entry.

### Event round

Rounds differ from ordinary ones in these ways:

| Aspect | Event round |
| --- | --- |
| board | fixed empty board (all cells 0) unless the server configures one; CBC tiles are not used |
| hands | the five borrowed cards; points recomputed with the reduced-card formula |
| `tl` | turn limit 0..5 (server choice: default 2) |
| `gs` | departure rule 0..2 (server choice: default 1) |
| `in` | observation flag 1, or 0 when observers are admitted |
| end | `EventResult` replaces the result, growth, prize and inventory messages |

### Observers

Observers of an event table use the ordinary watch path of
[spectators.md](spectators.md): [2D CheckJoinTable](../irc/G/2D.md)
answered with `Join=n/Mem=0`, `Req` answered with the first seat's
`PLAYACK`, then [43/25 Watch](../irc/G/43.md#4325-watch-and-watchinfo).
They receive the public StartData, TurnData, PutCard, BattleData, GetAway,
Change and EventResult messages and never EventCard, selected hands or
TCList.

## Exit

A cancel during card selection is the ordinary
[B2/17 Quit](../irc/G/B2.md#b217-quit) with the session, answered with the
same `EQuit`; the server drops the selection.

A cancelled table admission is answered with
[41/10 GameWa](../irc/G/41.md#4110-gamewa) `Error=-1`.
[41/05 GameQT](../irc/G/41.md#4105-gameqt) is answered with
[41/06 GameQA](../irc/G/41.md#4106-gameqa) `QT=1`,
[41/0D GameExit](../irc/G/41.md#410d-gameexit) with
[41/0E](../irc/G/41.md#410e-gameea-exit) `Exit=1`, and
[41/0F GameNG](../irc/G/41.md#410f-gameng) has no reply.

During the round, [43/05 Quit](../irc/G/43.md#4305-quit) is answered with
the same argument and ends the round; [43/27 Break](../irc/G/43.md#4327-break)
ends it with no reply. The server releases the table and the table chat
group; a new `98` admission is needed for another round.
