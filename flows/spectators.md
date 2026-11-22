# Spectators

A client in the observer role can watch a table it is not seated at. It
joins the table's channel, reads the table's settings and roster from the
table service, then asks for a snapshot with
[43/25 Watch](../irc/G/43.md#4325-watch-and-watchinfo). The server answers
with WatchInfo, during a turn also
[43/26 WatchCard](../irc/G/43.md#4326-watchcard) and the current TurnData,
and from then on copies the public gameplay stream to the observer. The
observer never takes part in the turn acknowledgement barrier and sends no
gameplay actions. The flow starts from the table's Observe action in the
room and ends at the first ResultData.

## Sequence

1. Client to the table service:
   [2D/00 CheckJoinTable](../irc/G/2D.md#2d00-checkjointable), after the
   client has resolved the table's advertised identity from the room's
   table records. The client then changes to the table's channel. The
   server answers [2E/00 CheckJoinTable_Ans](../irc/G/2E.md#2e00-checkjointable_ans);
   the successful path never reads it (see Rules).
2. Client to the table service: [01/00 Req](../irc/G/01.md#0100-req) once
   the channel is up. The server answers
   [02/00 PLAYACK](../irc/G/02.md#0200-playack) with `EN=1`, the current
   settings and the roster, without reserving a seat; a repeated Req
   repeats that read-only reply. The watch path does not use `GameEN`.
3. Client to the table service:
   [43/25 Data Watch](../irc/G/43.md#4325-watch-and-watchinfo) with
   `Watch=0`. The server answers WatchInfo with argument 0 during card
   selection or 1 during a turn. During a turn it follows with
   [43/26 WatchCard](../irc/G/43.md#4326-watchcard) (the played cards) and
   then the exact [43/09 TurnData](../irc/G/43.md#4309-turndata) last sent
   to the participants, including its random bytes. The snapshot is taken
   at a clean boundary: a cell with a pending combat marker has no
   representation in WatchInfo.
4. Server to the observer: copies of the public stream under Rules, in the
   order the participants received them, until the first ResultData.

## Rules

### Channel probe

The client packs the `2E` reply as `Join | (Mem << 1)` and reads it only
on its recovery path, where a nonzero value sends `ExitCh`
([tables.md](tables.md#exit)). The meaning of the two integers is not
established.

Server choice: the reply is `@CheckJoinTable_Ans=/Join=1/Mem=1`.

### Snapshot

Server choice:

- Only `Watch=0` is accepted.
- A seated participant cannot watch its own table.
- A repeated request from an admitted observer gets no second snapshot.
- A request during combat is answered at the next selection or turn
  boundary.
- A round in its result phase cannot be watched.

### Relayed stream

After the snapshot the observer receives copies of
[43/08 StartData](../irc/G/43.md#4308-startdata),
[43/09 TurnData](../irc/G/43.md#4309-turndata),
[43/0A PutCard](../irc/G/43.md#430a-putdata-and-putcard),
[43/0C BattleData](../irc/G/43.md#430c-battledata),
[43/0D ResultData](../irc/G/43.md#430d-resultdata),
[43/23 GetAway](../irc/G/43.md#4323-getaway),
[43/24 Change](../irc/G/43.md#4324-change),
Dead ([43/27](../irc/G/43.md#4327-break)) and
[43/28 DataError](../irc/G/43.md#4328-dataerror). VsGameInit, card
selection, TCList and every account-specific message are not relayed.

### Pacing

The client has no presentation-complete acknowledgement for an observer,
and its end-of-turn cleanup can discard a PutCard or BattleData that
arrives while the previous turn is still being drawn. The bootstrap also
waits for the user to dismiss a dialog of unbounded duration.

Server choice: the observer stream is paced by the server (about 8 seconds
after the snapshot, 3 seconds after each TurnData, PutCard and BattleData),
one packet in flight at a time, with a bounded backlog.

### Event tables

An observer of a playable event table goes through the same Req, PLAYACK
and Watch sequence after its event room membership and table channel are
bound to the authenticated account. The private EventCard and the
participants' selected hands and TCList are not relayed. An observer
receiving EventResult updates no personal profile state and sends no
acknowledgement.

## Exit

The stream ends at the first ResultData; the result screen, prizes and
rematch are not presented to an observer. An observer holds no seat and no
stake, so a departure or disconnect releases only its observer admission
(a player holds at most one seat or observer admission,
[L.md](../irc/L.md)).

Server choice: an observer whose backlog overflows, whose turn sequence no
longer follows the skip rule from [departures.md](departures.md#rules), or
whose round or match is replaced is disconnected without affecting the
participants.
