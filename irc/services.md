# Services

A service is an IRC nickname on the game connection described in
[session.md](session.md). The client learns a service's 64-bit game identity
from the directory files ([directory-files.md](../file/directory-files.md)),
from a reply such as [E2/00 TeachDVAns](G/E2.md#e200-teachdvans), or from
its own configuration, and addresses the service by the scrambled nickname
of that identity (the `U` plus eight base36 digits form in
[session.md](session.md#nick)). A message is the trailing argument of a
`NOTICE` or `PRIVMSG` to that nickname (see [envelope.md](envelope.md)),
and the reply arrives from the same nickname; a server routes by nickname
and answers from the nickname the client addressed. The tokens below
(`Shm`, `E`, `F`, `D`) are the names the directory files and replies use
for a service; outside packet examples the pages use the plain names of the
headings. Server choice: a service's game identity is its raw POL identity
XOR 0xB62DC8D0E4BCBB90, and its nickname is the scrambled form of the raw
identity.

## Room service

A room is a `RoomRecord` in the room list file `b/g/RL%03d`
([directory-files.md](../file/directory-files.md)); the record's `identity`
is the room service's game identity and its `channel` the room's chat
channel. The client reads the room list from the profile file service
before it enters a room.

The room service speaks family `L` ([L.md](L.md)): `DE`, `PD`, `PC` and
`DR` from the client, `DS`, `DF`, the `DD`/`DN`/`DC` delta header, `TD`
and `DO` from the service. It also receives
[42/01 Chat entry](G/42.md#4201-chat-entry) after room entry and
[84/00 Opt](G/84.md#8400-opt).

Each table the room advertises is a `TableRecord` in the room's `b/g/PTL`
with its own identity and channel, repeated in `TD` deltas. The client
addresses table messages to the table's identity: [41](G/41.md),
[01](G/01.md), [02](G/02.md), [12](G/12.md), [13](G/13.md), [14](G/14.md),
[1A](G/1A.md), [20](G/20.md), [24](G/24.md), [2D](G/2D.md), [2E](G/2E.md),
[42](G/42.md) and [43](G/43.md). The sequences are in
[tables.md](../flows/tables.md) and
[match-admission.md](../flows/match-admission.md).

An event room is an ordinary room whose identity also carries
[D0](G/D0.md), [D1](G/D1.md), [D2](G/D2.md), [D3](G/D3.md) and
[D7](G/D7.md); see [event-rooms.md](../flows/event-rooms.md).

## Shop service

The client learns the shop service from the `Shm` value of
[E2/00 TeachDVAns](G/E2.md#e200-teachdvans), `Shm=<16 hex>|<dir>|<vol>`,
during startup ([startup.md](../flows/startup.md)). The TeachDV request
itself goes to a directory service whose nickname the client takes from its
configuration (unverified).

The shop service speaks [80/00 Init](G/80.md#8000-init) answered by
[81/00 InitAns](G/81.md#8100-initans), [B0/00 ShReq](G/B0.md#b000-shreq)
answered by [B1/00 ShEnter](G/B1.md#b100-shenter), and the
[B2](G/B2.md) session that follows ([shop.md](../flows/shop.md)). The
Prize Center runs on the same service: [70/00 CvReq](G/70.md#7000-cvreq)
answered by [71/00 CvEnter](G/71.md#7100-cventer) and a B2 session opened
with `ExcInit` ([prize-center.md](../flows/prize-center.md)). The auction
collection also runs on it: [B0/01 ShReq](G/B0.md#b001-shreq) answered by
a B2 session with no B1 ([A.md](A.md)). The three sessions are numbered
independently, so the service routes a B2 message by which session is open
for the account.

Server choice: the shop service is raw POL service 0x300.

## Trade service

The client finds the trade service in the room's `b/g/PTL`: it takes the
last `TableRecord` whose metadata begins with `D`
([directory-files.md](../file/directory-files.md)). The record's identity
is the trade service's game identity. `D` records are not playable tables.
The inviter tells the invitee the identity with
[A0/00 TrID](G/A0.md#a000-trid-and-trent) through the peer relay.

The trade service speaks [A0/00 TrEnt](G/A0.md#a000-trid-and-trent),
answered to both players by [A1/00 TrEntAns](G/A1.md#a100-trentans) and
[A2/18 Init](G/A2.md#a218-init), and every later [A2](G/A2.md) message of
the session ([trade.md](../flows/trade.md)). The client does not
authenticate the sender of A1 or A2, so a server sends them only from this
nickname and never relays them between players.

Server choice: the trade service is raw POL service 0x400, with channel
`#LOCALTRADE1` and metadata `D` followed by 48 `A` bytes.

## Auction service

How the client learns the auction service's nickname is not recovered
(unverified); the `AC` value of TeachDVAns gives only the directory and
volume of the auction files.

The auction service speaks family `A` ([A.md](A.md)): `ER` answered by
`ES` or `EF`, `BR` answered by `BS` or `BF`, `SI` answered by `SS` or
`SF`, and `BH` answered by `HS` or `HF`. Search results are files read
through the profile file service
([auction-files.md](../file/auction-files.md)). Settled money and cards
are collected on the shop service.

## Ranking service

How the client learns the ranking service's nickname is not recovered
(unverified); the `RK` value of TeachDVAns gives only the directory and
volume of the ranking files. The request names a separate owner identity
for the files, so a server answers from the nickname the client addressed,
not from the file owner.

The ranking service speaks family `R` ([R.md](R.md)): `RR` with `PI`,
answered by `RF` with `LN` or by `RG`. The result file is read through the
profile file service ([ranking-files.md](../file/ranking-files.md)).

## Event match service

An event room's `b/g/PTL` advertises the event match service as a
`TableRecord` whose metadata is `E` followed by 48 `A` bytes
([directory-files.md](../file/directory-files.md)); the record's identity
is the service's game identity and its channel the service's channel. The
client finds it only while its `discovery` member is below 8.

The event match service speaks [98/00 EInitReq](G/98.md#9800-einitreq)
answered by [99/00 EInitEnt](G/99.md#9900-einitent), then
[B2/13 EInit](G/B2.md#b213-init),
[B2/07 CardSelect](G/B2.md#b207-cardselect),
[B2/22 Decks](G/B2.md#b222-decks) and [B2/17 Quit](G/B2.md#b217-quit)
([event-matches.md](../flows/event-matches.md)). The match itself runs on
an event table of the room.

## Event card service

An event room's `b/g/PTL` advertises the event card service as a
`TableRecord` whose metadata is `F` followed by 48 `A` bytes, in the same
form as the event match service.

The event card service speaks [90/00 CardReq](G/90.md#9000-cardreq)
answered by [91/00 CardEnt](G/91.md#9100-cardent), then
[B2/13 EcmInit](G/B2.md#b213-init), [B2/10 Erase](G/B2.md#b210-erase),
[B2/22 Decks](G/B2.md#b222-decks), [B2/15 Get](G/B2.md#b215-get) and
[B2/17 Quit](G/B2.md#b217-quit) ([event-cards.md](../flows/event-cards.md)).

## Registration service

The client takes the registration service's nickname from its own
configuration (unverified); no directory file or reply carries it.

The registration service speaks family `P` ([P.md](P.md)): `CR` answered by
`CS` or `NS`, `PG` answered by `PO` or `PM`, and `GR` answered by `GK` or
by `GG` with `PE`. `CR` and `PG` are sent during startup
([startup.md](../flows/startup.md)).

## Profile file service

The profile file service is not an IRC nickname. It is a separate TCP
service on pp000.pol.com port 51220 that carries one request and one
response per connection, encrypted with the cipher of the client's game
IRC session ([service.md](../file/service.md)). It serves the handle,
character and status records and every file the client reads or deletes:
the save file, the directory files, and the ranking, auction and event
files. A file request names an owner identity: the account for `U/` files and
the room service for the room's `b/g/PTL`.
