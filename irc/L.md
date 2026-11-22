# L Rooms

A room is a chat channel plus a roster kept by the
[room service](services.md#room-service). The client enters a room by
sending a `GTM0L` presence record to the room service, joins the room's IRC
channel for chat, and from then on receives sequenced roster and table
deltas. The room and table records the client reads from files before
entry are in [directory-files.md](../file/directory-files.md). Table
selection and admission are in [tables.md](../flows/tables.md) and
[match-admission.md](../flows/match-admission.md); event rooms use the same
commands, see [event-rooms.md](../flows/event-rooms.md).

## Framing

The payload after `GTM0L` is one or more commands, each `<XX>` BEL (0x07)
payload BEL, where `XX` is a two-letter token from the client's command
dictionary ([envelope.md](envelope.md#command-codec)).
The whole payload is at most 469 bytes and may not contain NUL, CR or LF.
Fields inside a payload are separated by 0x06 where the command defines
fields. Identities are written `0x` followed by sixteen hex digits.

| Token | Direction | Meaning |
| --- | --- | --- |
| `DE` | client | Enter the room with a player record |
| `DS` | server | Entry accepted; payload is the room identity |
| `DF` | server | Entry refused; payload `-1` when the room is full |
| `PD` | both | Presence update with a player record |
| `PC` | both | Remove a player by identity |
| `DD`, `DN`, `DC` | server | Delta header: `DD`=0, `DN`=1, `DC`=sequence |
| `TD` | server | Table record update |
| `DR` | client | Resync request with the client's last sequence |
| `DO` | server | Resync impossible, re-read the PTL |

## DE Enter

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | room service | [DS](#ds-entry-accepted) or [DF](#df-entry-refused), then a [PD](#pd-presence) delta to every member | `struct PlayerRecord` |

`DE` and `PD` carry eleven 0x06-separated fields, numbered 0..10, that both
sides map onto a 0x58-byte record. The same record appears in the PTL
player array.

```c
struct PlayerRecord {
    /* 0x00 */ uint64_t NN;        // field 0, `0x` hex: nickname identity, the account's POL identity
    /* 0x08 */ uint64_t room;      // field 9, `0x` hex: room identity the record belongs to
    /* 0x10 */ uint8_t  pad_10[8];
    /* 0x18 */ uint32_t character; // field 1, decimal: character identity
    /* 0x1C */ uint32_t field_7;   // field 7, decimal: purpose unknown (unverified)
    /* 0x20 */ uint32_t field_8;   // field 8, decimal: purpose unknown (unverified)
    /* 0x24 */ uint16_t field_3;   // field 3, decimal: purpose unknown (unverified)
    /* 0x26 */ uint8_t  field_2;   // field 2, decimal: purpose unknown (unverified); the client sends 2
    /* 0x27 */ uint8_t  field_5;   // field 5, decimal: purpose unknown (unverified)
    /* 0x28 */ char     CN[16];    // field 4: character name, raw bytes
    /* 0x38 */ char     info[32];  // field 10: information text, see below
};
```

Field 6 is a decimal that the client does not store (purpose unknown,
unverified).

```
C2S GTM0L<DE>\x070x0000000000000001\x061\x062\x060\x06Alice\x060\x060\x060\x060\x060xB62DC8D0E4BCBB90\x06AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x07
```

Field 4 is the character name, not the handle name: it is the same name the
client sends as `CN` in `GameEN`. A server should authenticate field 4
against the account's full canonical name.

The client copies both string fields into the record with a capacity check
(16 bytes for the name, 32 for the information text). A field with no NUL
inside the capacity makes the client report a diagnostic, but the room
update still applies and the sequence still advances; it is not an
admission rejection.

The client serializes the record by scanning for NUL without stopping at
either field boundary. A 16-byte name therefore goes out with whatever
follows it (observed: `ABCDEFGHIJKLMNOPMetadata`), and 32 information bytes
go out with trailing bytes after them. The safe widths are 15 name bytes and
31 information bytes. A server should reject longer fields rather than
republish them.

A server answers the sender with `DS` carrying the room identity, then
broadcasts a delta with the new `PD` record to every member including the
sender. A room with 16 members answers `DF` `-1` instead.

A `DE` into a different room moves the player: the old room removes the
member and broadcasts `PC`. A server should refuse the move while the player
holds a seat or observer admission at a table, and should keep the old
membership when the destination is full or invalid.

### Information text

The client reads fixed offsets of field 10 rather than parsing it as a
string:

```c
struct InformationText {
    /* 0x00 */ char    name[15];    // display name, space padded
    /* 0x0F */ uint8_t name_length; // `A` + display name length, constrained to 1..15
    /* 0x10 */ uint8_t pad_10[7];
    /* 0x17 */ uint8_t status;      // `A` + status
    /* 0x18 */ uint8_t substatus;   // `A` + substatus
    /* 0x19 */ uint8_t pad_19[7];
};
```

Status 3 with substatus 1 (`DB` at offsets 23..24) is the "seated and ready"
state. The owner's Start action requires every roster member, including a
computer opponent the server has published, to appear in the room's player
list with that status. The client reads nothing past offset 27; a server may
leave the remaining bytes as `A`.

## DS Entry accepted

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | room service to the entering client | none | room identity |

```
S2C GTM0L<DS>\x070xB62DC8D0E4BCBB90\x07
```

The payload is the room identity. In an event room the client consumes the
event entry answer only after a positive `DS`, see
[event-rooms.md](../flows/event-rooms.md#sequence).

## DF Entry refused

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | room service to the entering client | none | status, `-1` when the room is full |

```
S2C GTM0L<DF>\x07-1\x07
```

The payload is `-1` when the room is full (16 members).

## PD Presence

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | room service | a delta to every member | `struct PlayerRecord`, as for [DE](#de-enter) |
| S2C | room service to every member, inside a delta | none | `struct PlayerRecord`, as for [DE](#de-enter) |

```
C2S GTM0L<PD>\x070x0000000000000001\x061\x062\x060\x06Alice\x060\x060\x060\x060\x060xB62DC8D0E4BCBB90\x06AAAAAAAAAAAAAAADBAAAAAAADBAAAAA\x07
S2C GTM0L<DD>\x070\x07<DN>\x071\x07<DC>\x07101\x07<PD>\x070x0000000000000001\x061\x062\x060\x06Alice\x060\x060\x060\x060\x060xB62DC8D0E4BCBB90\x06AAAAAAAAAAAAAAADBAAAAAAADBAAAAA\x07
```

A member sends `PD` with its full `struct PlayerRecord`; the server
rebroadcasts it inside a delta. A `PD` from a non-member is an error, and a
member cannot change its identity (field 0) with a `PD`. A guest signals
its readiness at a table with a `PD` whose information text carries status
3, substatus 1, see
[match-admission.md](../flows/match-admission.md#rules).

## PC Remove

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | room service | a delta to the remaining members | member identity |
| S2C | room service to the remaining members, inside a delta | none | member identity |

```
C2S GTM0L<PC>\x070x0000000000000001\x07
S2C GTM0L<DD>\x070\x07<DN>\x071\x07<DC>\x07102\x07<PC>\x070x0000000000000001\x07
```

The payload is the member's own `0x` identity. A `PC`, an IRC disconnect, or
a successful `DE` into another room removes the member and broadcasts a `PC`
delta to the remaining members.

Leaving the room's IRC channel is separate from leaving the roster. The
captured Computer startup is `GameENC`, `GameECA`, PART room channel,
`CheckJoinTable`, `Req`, JOIN table channel, room `PD`. The PART only ends
room chat: a server must keep the room roster entry, the pending table
admission and the account reservation across it, and must treat only a
`PC`, a disconnect or a successful `DE` elsewhere as leaving the room.

## DD DN DC Delta header

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | room service to every member | none | delta header, then one command |

```
S2C GTM0L<DD>\x070\x07<DN>\x071\x07<DC>\x07101\x07<PD>\x07...\x07
```

Every roster or table change is one delta: `DD` 0, `DN` 1, `DC` with the
next sequence number, then the `PD`, `PC` or `TD` command. The sequence
starts from the value published in the PTL and increases by one per delta.
All tables in a room share the roster and this one sequence.

## TD Table update

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | room service to every member inside a delta | none | `struct TableDelta` |

```c
struct TableDelta {
    char     channel[16];  // field 0: table IRC channel
    uint32_t zero_1;       // field 1: 0
    uint64_t identity;     // field 2: table identity, `0x` hex
    uint32_t state;        // field 3: table state, see tables.md
    uint32_t zero_4;       // field 4: 0
    uint32_t zero_5;       // field 5: 0
    char     metadata[49]; // field 6: table metadata, see below
};
```

```
S2C GTM0L<DD>\x070\x07<DN>\x071\x07<DC>\x07103\x07<TD>\x07#LOCALTABLE1\x060\x060x1122334455667788\x061\x060\x060\x06AAAAAC********AAAAAAAAAAAAAAABAADCACBBEAAABEAAAAK\x07
```

`TD` carries seven 0x06-separated fields. The table states are in
[tables.md](../flows/tables.md#rules). A server sends a `TD` after
accepting an Apply, before the roster and owner notices.

A room may publish several playable tables. Each is a separate PTL record
and `TD` target with its own identity and channel; metadata positions 1..3
carry the table number minus one. The limit is the client's 128-slot table
collection: 127 playable tables plus the trade record. A player holds at
most one table seat or observer admission at a time.

Three-player versus uses the same `A` metadata and states. The metadata
count at positions 4..5 is the current occupancy, not a seat limit; the
client's join gate allows a new player while occupancy is below 3. No RL or
PTL field advertises the seat count: the client takes it from
`VsGameInit /N`.

Server choice: a two-seat table must enforce its own cap, because the client
still offers Join at occupancy 2.

### Table metadata

All numbers are alphabetic hex (`A` + digit per hex digit). The 49 bytes:

```c
struct TableMetadata {
    /* 0x00 */ char type;      // `A` ordinary, may offer Computer; `B` human-only; `D` trade service; `C` skipped
    /* 0x01 */ char number[3]; // table number minus one
    /* 0x04 */ char count[2];  // current occupancy
    /* 0x06 */ char pw[8];     // password, padded with `*`; `********` when no password
    /* 0x0E */ char owner[16]; // owner nickname identity
    /* 0x1E */ char bm[4];     // wager
    /* 0x22 */ char flags;     // `A` + (`du` | `st`<<1 | `cb`<<2 | `ca`<<3)
    /* 0x23 */ char tl;        // `A` + time limit 0..5
    /* 0x24 */ char gs;        // `A` + departure rule 0..2
    /* 0x25 */ char in_pa;     // `A` + `in` + (`pa`<<2); `in` 0..2, `pa` 1 when a password is set
    /* 0x26 */ char co;        // `A` + comment index 0..8
    /* 0x27 */ char lu[5];     // upper collection-point bound
    /* 0x2C */ char ll[5];     // lower collection-point bound
};
```

```
AAAAAC********AAAAAAAAAAAAAAABAADCACBBEAAABEAAAAK
```

That example is table 1, occupancy 2, no password, owner 1, `bm`=50
(`AADC`), flags `A` (all four zero), `tl`=2, `gs`=1, `in`=1 with `pa`=0,
`co`=4, `lu`=20, `ll`=10.

The client reads the low nibble before the high nibble of each byte. There
is no field for the rank bounds `au`/`al`: the client substitutes 300 and
100. A password longer than eight bytes, or containing NUL, 0x06, 0x07, CR,
LF or a non-trailing `*`, cannot be represented. The `D` trade service
record is not playable. Type `A` and `B` decode to the same rules and
options; the client keeps the type to decide whether to offer Computer.
What the client does with the decoded record is in
[tables.md](../flows/tables.md).

## DR Resync

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | room service | the retained deltas above the sequence, or [DO](#do-resync-impossible) | last delta sequence |

```
C2S GTM0L<DR>\x07101\x07
```

A client that thinks it missed a delta sends `DR` with its last sequence. A
server replays every retained delta above that sequence.

## DO Resync impossible

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | room service to the requesting client | none | empty |

```
S2C GTM0L<DO>\x07\x07
```

Sent when the requested sequence is ahead of the server or older than its
retained history. The client must re-read the PTL.

## Sequence

Entry:

```
C2S GTM0L<DE>\x070x0000000000000001\x061\x062\x060\x06Alice\x060\x060\x060\x060\x060xB62DC8D0E4BCBB90\x06AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x07
S2C GTM0L<DS>\x070xB62DC8D0E4BCBB90\x07
S2C GTM0L<DD>\x070\x07<DN>\x071\x07<DC>\x07101\x07<PD>\x070x0000000000000001\x061\x06...\x07
C2S JOIN #room-channel
```

Departure:

```
C2S GTM0L<PC>\x070x0000000000000001\x07
S2C GTM0L<DD>\x070\x07<DN>\x071\x07<DC>\x07102\x07<PC>\x070x0000000000000001\x07
```

Resync after a missed delta:

```
C2S GTM0L<DR>\x07101\x07
S2C GTM0L<DD>\x070\x07<DN>\x071\x07<DC>\x07102\x07<PC>\x070x0000000000000001\x07
S2C GTM0L<DD>\x070\x07<DN>\x071\x07<DC>\x07103\x07<TD>\x07#LOCALTABLE1\x060\x06...\x07
```
