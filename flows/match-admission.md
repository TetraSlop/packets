# Match admission

The sequence from table selection up to the game initializer, for a human
table and for the Computer menu, between the client and the table service.
It starts when the client's local gates in [tables.md](tables.md#rules)
have enabled Join or Computer, and ends with
[43/01 VsGameInit](../irc/G/43.md#4301-vsgameinit) or
[43/02 ComGameInit](../irc/G/43.md#4302-comgameinit). The room service
([services.md](../irc/services.md#room-service)) carries the `TD` delta and
the room `PD` that the handshake depends on.

## Sequence

Human table, in wire order:

1. Client to the table service: [41/01 GameEN](../irc/G/41.md#4101-gameen)
   with the identity fields. The server requires `NN` and `CN` to match
   the authenticated account, checks that a seat is free, reserves it and
   answers [41/02 GameEA](../irc/G/41.md#4102-gameea) with `/EN=1` to the
   claiming owner or `/EN=2` to a joiner. A negative `EN` rejects, see
   Rules. A joiner subtracts the advertised wager locally on `/EN=2`
   ([wagers.md](wagers.md)).
2. Owner only, client to the table service:
   [14/00 Apply](../irc/G/14.md#1400-tet-and-tab) from the settings editor
   ([tables.md](tables.md#sequence)). No reply. The owner subtracts its
   chosen wager locally just before sending it.
3. The client JOINs the table channel and sends a room `PD` with status 3
   and substatus 1 ([L.md](../irc/L.md#information-text)). This is the
   guest's readiness signal; the server sends that guest nothing from step
   4 or step 6 before it.
4. Server to the members: the `TD` delta
   ([L.md](../irc/L.md#dd-dn-dc-delta-header)), then
   [41/04 GameML](../irc/G/41.md#4104-gameml) and
   [1A/00 GameOwner](../irc/G/1A.md#1a00-gameowner).
5. Client to the table service, any time:
   [41/03 GameM](../irc/G/41.md#4103-gamem). The server answers
   [41/04 GameML](../irc/G/41.md#4104-gameml) with `/ID=`.
6. Owner to the table service:
   [41/0A GameReady](../irc/G/41.md#410a-gameready). The server sends
   [41/07](../irc/G/41.md#4107-start-offer) to the other humans that are
   ready.
7. Each human to the table service: [01/00 Req](../irc/G/01.md#0100-req).
   The server answers [02/00 PLAYACK](../irc/G/02.md#0200-playack) with
   `/EN=1`, the current `Tet` and `Tab` and the roster. From here the rules
   are fixed; a later Apply is not accepted.
8. Each human to the table service:
   [41/08 GameOK](../irc/G/41.md#4108-gameok). Once every seat has sent it
   the server sends [41/09](../irc/G/41.md#4109-start-confirmation) to all,
   debits the stake ([wagers.md](wagers.md#rules)) and sends
   [43/01 VsGameInit](../irc/G/43.md#4301-vsgameinit), whose `/M`
   overwrites each client's local balance.

Computer menu, in wire order:

1. Client to the table service:
   [41/12 GameENC](../irc/G/41.md#4112-gameenc). The server answers
   [41/13 GameECA](../irc/G/41.md#4113-gameeca) with `/EN=1`. The client
   then PARTs the room channel; the PART only ends room chat, and the
   server keeps the room roster entry, the pending admission and the
   account reservation across it
   ([L.md](../irc/L.md#pc-remove)).
2. Client to the table service:
   [2D/00 CheckJoinTable](../irc/G/2D.md#2d00-checkjointable). The server
   answers [2E/00 CheckJoinTable_Ans](../irc/G/2E.md#2e00-checkjointable_ans);
   the client does not wait for it.
3. Client to the table service: [01/00 Req](../irc/G/01.md#0100-req), sent
   before the table JOIN completes. The server answers
   [02/00 PLAYACK](../irc/G/02.md#0200-playack) with `/EN=1`, which the
   client polls once and ignores, then
   [43/02 ComGameInit](../irc/G/43.md#4302-comgameinit) with `ComList`,
   which the Computer game waits for.
4. The client JOINs the table channel and sends its room `PD`.
5. Client to the table service: [43/06 ComGame](../irc/G/43.md#4306-comgame)
   with argument 0 (save the menu choices), 2 or 3 (start against one or two
   opponents). No reply.
6. Client to the table service:
   [43/07 CardSelect](../irc/G/43.md#4307-cardselect). The server answers
   with the selection status, then `StartData`
   ([43.md](../irc/G/43.md#sequence)).

There is no `GameOK` and no `VsGameInit` on the Computer route.

## Rules

### Readiness barrier

The client discards any queued 41/04 and 41/07 when it receives `GameEA`.
A server therefore sends a guest the roster and a pending start offer only
after that guest's ready room `PD` (status 3, substatus 1). A `GameReady`
relayed while a guest is still joining strands that guest without an
invitation.

The owner's Start action is enabled when at least two humans are ready and
every roster member, including a server-published computer opponent,
appears in the room's player list with status 3 and substatus 1.

### Rejections

The negative `EN` values the client maps to its own text are listed under
[41/02 GameEA](../irc/G/41.md#4102-gameea),
[41/13 GameECA](../irc/G/41.md#4113-gameeca) and
[02/00 PLAYACK](../irc/G/02.md#0200-playack). The admission messages carry
no proof of the client's local eligibility gates, so a server that wants
the same rules checks its own committed inventory and balance on `GameEN`
and `Req`.

Server choice: use the mapped `GameEA`/`PLAYACK` failure codes for a lost
seat race or a failed eligibility check; no condition-specific codes are
known.

### Computer route rules

The `gs` and `du` values the Computer menu sends conflict with a fixed
room's advertised rules; the conflict and the server choice for it are
under [43/06](../irc/G/43.md#4306-comgame).

### Viewer notices

[41/11 Viewer](../irc/G/41.md#4111-viewer) `IN` and `OUT` presence notices
arrive at any time and get no reply.

## Exit

| Client | Server | Meaning |
| --- | --- | --- |
| [41/05 GameQT](../irc/G/41.md#4105-gameqt) | [41/06 GameQA](../irc/G/41.md#4106-gameqa) `/QT=1` | Member quits the table; the server frees the seat |
| [41/05 GameQT](../irc/G/41.md#4105-gameqt) `/ID=` (owner, naming another member) | [41/06 GameQA](../irc/G/41.md#4106-gameqa) `/QT=2` to the owner, [41/0C GameKick](../irc/G/41.md#410c-gamekick) to all | Kick (server choice) |
| [41/0D GameExit](../irc/G/41.md#410d-gameexit) | [41/0E](../irc/G/41.md#410e-gameea-exit) `/Exit=1` | Member exits the table UI |
| [41/0F GameNG](../irc/G/41.md#410f-gameng) | [41/10 GameWa](../irc/G/41.md#4110-gamewa) to all | Start declined; the server cancels the pending start |
| [20/00 ExitCh](../irc/G/20.md#2000-exitch) | none, or `GameWa` to the others | Channel cleanup from recovery, see [tables.md](tables.md#exit) |

On a disconnect the server frees the seat and the roster entry as for a
quit. Any stake the server has already debited is refunded
([wagers.md](wagers.md#exit)); the client's own local debit is overwritten
by the next balance it receives.

Server choice: on a quit or exit from a configured multi-human lobby, keep
the table, publish the shrunken `GameML` and a `GameOwner` with `/QN=` for
host election; otherwise end the table.
