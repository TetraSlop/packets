# Event rooms

An event room is an ordinary room directory entry whose zone and room
names match an `EventList` record
([event-files.md](../file/event-files.md#tm0eventlist)). Entering it adds
an event admission step after the normal `DE` to the room service
([services.md](../irc/services.md#room-service)), and the room identity
also carries the event time, announcement and matchmaking control
messages: [D0](../irc/G/D0.md), [D1](../irc/G/D1.md),
[D2](../irc/G/D2.md), [D3](../irc/G/D3.md) and [D7](../irc/G/D7.md). The
flow starts when the client selects the event room and ends when it leaves
the room or is denied.

## Sequence

1. Room service to client: the event zone as a normal `ZL` record and the
   event room as a normal `RL` record
   ([directory-files.md](../file/directory-files.md)), with the room's
   `PTL` advertising the event match service and the event card service as
   table records in the
   [event service form](../file/directory-files.md#layout).
   A server publishes them under the Directory entries rule.
2. Client to the profile file service: the room-owned
   [`TM0EventDataList`](../file/event-files.md#tm0eventdatalist) read.
   Client to the room identity:
   [D1/04 EventTimeReq](../irc/G/D1.md#d104-eventtimereq). Client to the
   room service: the room `DE` ([L.md](../irc/L.md#de-enter)). The client
   sends all three without waiting, so the server answers the file read and
   `D1/04` (with [D1/05](../irc/G/D1.md#d105-eventtimereqcheck)) before it
   answers `DE`.
3. Room service to client: `DS`, then the roster `PD` and `DD` updates,
   then last, from the room identity,
   [D0/00 EventEn](../irc/G/D0.md#d000-eventen). The client consumes `D0`
   only after a positive `DE` result. `DE` alone establishes room presence;
   event membership is whatever the server records when it answers
   `Ans=1`.
4. Room identity to every member, unsolicited:
   [D1/03](../irc/G/D1.md#d103-eventtime) (schedule refresh),
   [D1/01](../irc/G/D1.md#d101-eventtime-start) (start),
   [D1/02](../irc/G/D1.md#d102-eventtime-end) (end),
   [D1/08](../irc/G/D1.md#d108-eventtime-close) (close) and
   [D7/00 ETelop](../irc/G/D7.md#d700-etelop) (scrolling text).
5. The member continues with [event-matches.md](event-matches.md) or
   [event-cards.md](event-cards.md).

## Rules

### Directory entries

The event room's `RL` name is `key_event` followed by `*`; its rules kind
is `B` (the client shows the Not Set rule panel for it) and its computer
byte is zero. Member capacity is 16. The default zone and room pair is
reserved for the ordinary lobby.

The room's `PTL` carries no ordinary playable table by default. A playable
event table is a separate `A` record with a positive table number; the
event match service record never enters the client's playable-table list.

Server choice: the room channel is `#EV` followed by eight base36 digits of
the 41-bit raw room identity, and any event identity or channel that
collides with an account or another route is rejected.

### Admission

Server choice: `D0` answers `-32870` to a new member after the event end,
to an excluded member and when 256 members are recorded; the server
registers the member on `Ans=1` and keeps that record across reconnects.

### Non-playing entry

When the event match service answers
[99/00 EInitEnt](../irc/G/99.md#9900-einitent) with `Ans=2` the client
enters a non-playing mode. It skips [B2/13 EInit](../irc/G/B2.md#b213-init)
and the owned-card selection, so it works with zero owned cards; an `EInit`
sent anyway is left unconsumed. It cannot reserve a table and refuses any
[D2](../irc/G/D2.md) offer. The table Observe action still works. The
client does not treat this mode as a computer opponent mode.

Server choice: `Ans=2` is a non-playing admission that needs no playable
table and makes no reservation.

### Event time

The client stores the [D1/05](../irc/G/D1.md#d105-eventtimereqcheck)
answer and derives a countdown from it. The room becomes eligible for
matchmaking proposals when `D1/05` reports `Start=-1` with an `End` other
than -1; `D1/01` sets that flag, `D1/02` or `D1/08` ends it and `D1/03`
only refreshes the countdown (unverified).

### Other traffic on the room identity

[84/00 Opt](../irc/G/84.md#8400-opt) option saves and
[42/01 chat entry](../irc/G/42.md#4201-chat-entry) addressed to the event
room identity behave as on an ordinary room.

## Exit

On `D0` `Ans=-32870` the client leaves the room by itself: it sends the
room `PC` for the selected room identity with its authenticated nickname
([L.md](../irc/L.md#pc-remove)), then shows the rejection notice. A server
therefore sees an ordinary `PC` after a denied admission and clears the
transient presence on that `PC` or on disconnect. Nothing in the denial
path restores the previously occupied room.

Moving to another room removes the event room presence; durable membership
and progress survive. `D1/02` and `D1/08` end matchmaking eligibility for
every member.
