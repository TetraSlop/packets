# TetraPackets

A protocol reference for PlayOnline Tetra Master: what the PC client (TM
1.6.11.0) sends, what it expects back, and what a compatible server has to
do. Recovered from the client's code and from captured traffic. The
reference server is [tmserver](https://github.com/sruon/tmserver); where the
original behaviour is unknown, its choice is marked `Server choice:`.

This is an unofficial project. It is not affiliated with or endorsed by
Square Enix.

## Pages

The client talks to two endpoints. Message pages are named after what they
carry: the family letter for the command-codec families, the primary opcode
for family `G`. Flow pages describe the sequences and rules that span
opcodes and never restate a layout.

### IRC game connection, TCP 51241

Every message is the trailing argument of an IRC NOTICE or PRIVMSG, prefixed
`GTM0` and a family letter: `G` named sections, `L` lobby, `P` profile,
`R` ranking, `A` auction.

| Family | Page | Covers |
| --- | --- | --- |
| | [session.md](irc/session.md) | Ports, challenge, RSA key wrap, SqBlowfish OFB, NICK/USER, relay verbs |
| | [envelope.md](irc/envelope.md) | `GTM0` header, named sections, size limits, command codec, peer-relay guard |
| | [services.md](irc/services.md) | The services, their nicknames, which opcodes each speaks |
| `A` | [A.md](irc/A.md) | Auction: command codec, exhibit, bid, search, collection |
| `L` | [L.md](irc/L.md) | Rooms: entry, presence, roster deltas, player and table records |
| `P` | [P.md](irc/P.md) | Character profile: CR, PG, GR |
| `R` | [R.md](irc/R.md) | Ranking: category requests, generated file names, cleanup |

Family `G`, one page per primary opcode, one section per subtype:

| Opcode | Page | Name |
| --- | --- | --- |
| `01` | [01.md](irc/G/01.md) | Req |
| `02` | [02.md](irc/G/02.md) | PLAYACK |
| `12` | [12.md](irc/G/12.md) | Rule |
| `13` | [13.md](irc/G/13.md) | Rule reply |
| `14` | [14.md](irc/G/14.md) | Apply |
| `1A` | [1A.md](irc/G/1A.md) | GameOwner |
| `20` | [20.md](irc/G/20.md) | ExitCh |
| `21` | [21.md](irc/G/21.md) | Trade invitation |
| `22` | [22.md](irc/G/22.md) | Trade invitation answer |
| `23` | [23.md](irc/G/23.md) | Trade cancellation |
| `24` | [24.md](irc/G/24.md) | Save |
| `25` | [25.md](irc/G/25.md) | Trade rejection |
| `2A` | [2A.md](irc/G/2A.md) | Trade cancellation acknowledgement |
| `2D` | [2D.md](irc/G/2D.md) | CheckJoinTable |
| `2E` | [2E.md](irc/G/2E.md) | CheckJoinTable_Ans |
| `41` | [41.md](irc/G/41.md) | Table admission |
| `42` | [42.md](irc/G/42.md) | Chat |
| `43` | [43.md](irc/G/43.md) | Game messages |
| `70` | [70.md](irc/G/70.md) | Prize Center request |
| `71` | [71.md](irc/G/71.md) | Prize Center entry |
| `80` | [80.md](irc/G/80.md) | Init |
| `81` | [81.md](irc/G/81.md) | InitAns |
| `84` | [84.md](irc/G/84.md) | Opt |
| `85` | [85.md](irc/G/85.md) | OptAns |
| `90` | [90.md](irc/G/90.md) | Event cards request |
| `91` | [91.md](irc/G/91.md) | Event cards reply |
| `98` | [98.md](irc/G/98.md) | Event match request |
| `99` | [99.md](irc/G/99.md) | Event match reply |
| `A0` | [A0.md](irc/G/A0.md) | Trade entry |
| `A1` | [A1.md](irc/G/A1.md) | Trade entry answer |
| `A2` | [A2.md](irc/G/A2.md) | Trade exchange |
| `B0` | [B0.md](irc/G/B0.md) | Shop request |
| `B1` | [B1.md](irc/G/B1.md) | Shop entry |
| `B2` | [B2.md](irc/G/B2.md) | Inventory editor |
| `D0` | [D0.md](irc/G/D0.md) | Event entry |
| `D1` | [D1.md](irc/G/D1.md) | Event time |
| `D2` | [D2.md](irc/G/D2.md) | Match offer |
| `D3` | [D3.md](irc/G/D3.md) | Match offer answer |
| `D7` | [D7.md](irc/G/D7.md) | Announcements |
| `E1` | [E1.md](irc/G/E1.md) | TeachDV |
| `E2` | [E2.md](irc/G/E2.md) | TeachDVAns |

### Flows

| Page | Covers |
| --- | --- |
| [startup.md](flows/startup.md) | Login, profile reads, TeachDV, Init and guild choice, title |
| [tables.md](flows/tables.md) | Table states, eligibility, passwords, rule query and editor, CheckJoinTable recovery |
| [match-admission.md](flows/match-admission.md) | Human start handshake, readiness, rejections, Computer menu |
| [departures.md](flows/departures.md) | Triggers, gs rules, turn advance and ranking with departed seats |
| [results-and-prizes.md](flows/results-and-prizes.md) | Statistics, prize resolution, generated cards, result edits, rematch |
| [wagers.md](flows/wagers.md) | Debit points, doubling, payout formula |
| [spectators.md](flows/spectators.md) | Entry, snapshot order, relayed stream |
| [shop.md](flows/shop.md) | Entry, buy, get, edits, quit |
| [prize-center.md](flows/prize-center.md) | Points, awards, pack purchase |
| [trade.md](flows/trade.md) | Invitation, entry, offers, selection, confirmation, completion, abort |
| [event-rooms.md](flows/event-rooms.md) | Directory entries, entry and denial, mode 8, time notices |
| [event-matches.md](flows/event-matches.md) | E service admission, decks, proposals, rounds |
| [event-cards.md](flows/event-cards.md) | F service claims |

### Profile file service, TCP 51220

A binary open/request/response protocol that reads and writes named objects.

| Page | Covers |
| --- | --- |
| [service.md](file/service.md) | Framing, operations, file names, checksums, errors |
| [save-file.md](file/save-file.md) | `TM0DataFile` layout and value bounds |
| [directory-files.md](file/directory-files.md) | `ZL`, `RL`, `PTL` records |
| [event-files.md](file/event-files.md) | Event list, data list and member list layouts |
| [ranking-files.md](file/ranking-files.md) | Result record and `TM0RkData` |
| [auction-files.md](file/auction-files.md) | Listing, actor and bid records, delivery markers |

### Client data and logic

Not on the wire, but what the messages refer to.

| Page | Covers |
| --- | --- |
| [catalogue.md](client/catalogue.md) | Card and pack data, generation, guild filters |
| [rules.md](client/rules.md) | Card bytes, battle, growth, random, points, sale value |
| [board.md](client/board.md) | Cell numbering, field codes, the board generator |
| [computer-player.md](client/computer-player.md) | Computer opponent table, pools and hands, move and target choice |
| [ratings-and-statistics.md](client/ratings-and-statistics.md) | Profile statistics, rating ring, VS. Rating |
| [economy_catalog.json](client/economy_catalog.json) | `CardPrm`, `CardPri` and `PackPrm` records decoded to JSON, with the hash of each decompressed file |

## Conventions

- Addresses such as `0x100C0F16` are RVAs in TM 1.6.11.0 relative to image
  base `0x10000000`.
- Opcodes are written family/subtype in hex, `43/0A`. Wire tokens are shown
  as sent.
- Records are written as C structs. Every struct is byte-packed, `/* 0xNN */`
  prefixes give the offset where the record has fixed offsets, and multi-byte
  integers are little-endian unless the member comment says otherwise. For a
  text message the struct is the decoded form: a decimal token is an integer
  member, a hex string is a byte array, a repeated section is an array sized
  to its documented maximum.
- `(unverified)` marks a statement carried over without a routine address or
  capture behind it.
- `Server choice:` introduces a decision a server has to make where the
  original service's behaviour is unknown. It is not a protocol fact.

See [STYLE.md](STYLE.md) for how pages are written.

## Licence

Licensed under [CC BY 4.0](LICENSE).
