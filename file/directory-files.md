# Directory files

The lobby directory is three files a server publishes and the client reads
through the profile file service ([service.md](service.md)): the zone list
`b/g/ZL`, one room list `b/g/RL%03d` per zone and the player and table list
`b/g/PTL` of one room. Each file has a record count at offset 0x40 followed
by fixed-stride records. The PTL also carries the room sequence number that
the room service ([services.md](../irc/services.md#room-service)) continues
with its delta stream in [L.md](../irc/L.md#dd-dn-dc-delta-header).

Multi-byte integers are little-endian. "Alphabetic hex" means each hex digit
is encoded as ASCII `A` plus the digit (`A`..`P`), so `AAC` is 2 and `PDC` is
0xF32. Boolean and small enumerated fields use the same encoding.

## Access

### ZL

`b/g/ZL` is shared directory data of 0x848 bytes. The client reads it with a
file read when it shows the zone list and again every time it returns to
that list, so a server fills the player and room counts at read time.

### RL

`b/g/RL%03d`, with the zone index as three decimal digits, is shared
directory data of up to 0xC848 bytes. Selecting a zone opens a second game
IRC connection to the zone's host and then reads that zone's file. The client
re-reads it every time it returns to the room list, so a server fills the
member and occupied-table counts at read time.

### PTL

`b/g/PTL` is 0xC050 bytes and is read once on room entry. The request names
the room service as file owner, and a server answers with that room's
snapshot. After that the client applies the room's `GTM0L` deltas, whose `DC`
value continues the sequence in the file, and resyncs with `DR` when it
believes it missed one ([L.md](../irc/L.md#dd-dn-dc-delta-header)).

## Layout

### ZL

The record count is at 0x40 and up to 32 records of 0x40 bytes follow from
0x48.

```c
struct ZoneRecord {
    /* 0x00 */ uint32_t player_count;   // player count shown for the zone
    /* 0x04 */ uint32_t room_count;     // room count shown for the zone
    /* 0x08 */ uint8_t  pad_08[4];
    /* 0x0C */ char     language[2];    // language code, `EN`
    /* 0x0E */ char     restriction[3]; // restriction filter values; `AAA` accepts every player
    /* 0x11 */ char     name[27];       // NUL-terminated zone name
    /* 0x2C */ char     host[16];       // NUL-terminated IRC host of the zone's game connection
    /* 0x3C */ uint8_t  index;          // zone index, the `%03d` of the zone's RL file
    /* 0x3D */ uint8_t  pad_3D[3];
};
```

### RL

The record count is at 0x40 and up to 256 records of 0xC8 bytes follow from
0x48.

```c
struct RoomRecord {
    /* 0x00 */ uint64_t identity;        // room identity, the file owner of the room's PTL
    /* 0x08 */ uint8_t  pad_08[8];
    /* 0x10 */ uint32_t member_count;    // current member count
    /* 0x14 */ uint8_t  pad_14[4];
    /* 0x18 */ uint32_t occupied_tables; // occupied table count
    /* 0x1C */ uint8_t  pad_1C[8];
    /* 0x24 */ uint16_t one_24;          // always 1, purpose unknown (unverified)
    /* 0x26 */ uint8_t  pad_26;
    /* 0x27 */ uint8_t  one_27;          // always 1, purpose unknown (unverified)
    /* 0x28 */ char     rule_kind;       // `A` fixed room rules; `B` custom rules, shown as "Not Set"
    /* 0x29 */ char     computer;        // `A` offers the Computer action on an empty table; 0 and `B` disable it
    /* 0x2A */ char     name[31];        // NUL-terminated room name as published, including a trailing `*`
    /* 0x49 */ char     rules[7];        // fixed rules, alphabetic hex, see below
    /* 0x50 */ uint8_t  pad_50[0x1A];
    /* 0x6A */ char     table_count[3];  // number of playable tables, alphabetic hex
    /* 0x6D */ char     capacity[3];     // room member capacity, alphabetic hex (16 in the known configuration)
    /* 0x70 */ uint8_t  chat;            // nonzero shows "Chat", zero shows "Off"
    /* 0x71 */ uint8_t  pad_71[0x47];
    /* 0xB8 */ char     channel[16];     // NUL-terminated room IRC channel, for example `#LOCALROOM01`
};
```

The seven rule bytes are four alphabetic-hex nibbles of the wager `bm`, then
`A` + (`du` | `st` << 1 | `cb` << 2 | `ca` << 3), `A` + `tl`, `A` + `gs`.

### PTL

```c
struct PlayerTableList {
    /* 0x0000 */ uint8_t  pad_0000[0x40];
    /* 0x0040 */ uint32_t sequence;     // current room sequence number
    /* 0x0044 */ uint32_t player_count; // player record count
    /* 0x0048 */ uint32_t table_count;  // table record count
    /* 0x004C */ uint8_t  pad_004C[4];
    /* 0x0050 */ struct PlayerRecord players[256]; // 0x58 each
    /* 0x5850 */ struct TableRecord  tables[256];  // 0x68 each
};
```

The player record is the one `DE` and `PD` carry, laid out in
[L.md](../irc/L.md#de-enter). A table record:

```c
struct TableRecord {
    /* 0x00 */ uint64_t identity;     // table identity, the target of the table's messages
    /* 0x08 */ uint8_t  pad_08[4];
    /* 0x0C */ uint32_t discovery;    // 0 for an ordinary table; an event match service record needs a value below 8
    /* 0x10 */ uint8_t  pad_10[4];
    /* 0x14 */ uint8_t  state;        // table state, see tables.md
    /* 0x15 */ uint8_t  pad_15[3];
    /* 0x18 */ char     channel[16];  // NUL-terminated table IRC channel
    /* 0x28 */ char     metadata[49]; // table metadata; first byte `A` or `B` playable, `D` trade service, `C` skipped
    /* 0x59 */ uint8_t  pad_59[15];
};
```

The table states are in [tables.md](../flows/tables.md#rules) and the
metadata layout in [L.md](../irc/L.md#table-metadata).

An event room's PTL advertises its two event services as table records of
the same 0x68 bytes, with a metadata string of `E` or `F` followed by 48 `A`
bytes:

```c
struct EventServiceRecord {                    // PTL table record, 0x68 bytes
    /* 0x00 */ uint64_t identity;              // identity of the event service
    /* 0x08 */ uint8_t  pad_08[4];
    /* 0x0C */ uint32_t zero_0C;               // zero; the `E` record needs a value below 8
    /* 0x10 */ uint8_t  pad_10[4];
    /* 0x14 */ uint8_t  state;                 // table state
    /* 0x15 */ uint8_t  pad_15[3];
    /* 0x18 */ char     channel[16];           // IRC channel, NUL-terminated, at most 15 bytes
    /* 0x28 */ char     metadata[49];          // `E` or `F` followed by 48 `A` bytes
    /* 0x59 */ uint8_t  pad_59[15];
};
```

## Rules

### ZL

A server terminates the zone name within 27 bytes and the host within 16.
An event's zone name is matched against the event list by exact string
compare, see [event-files.md](event-files.md#tm0eventlist-1).

### RL

The name is published as `Local Room*` and the client renders it without the
asterisk (meaning of `*` unverified). The `rule_kind` and `computer` bytes
are independent. `rule_kind` becomes the room context of every table in the
room: `A` fixed rules, `B` custom rules, see
[tables.md](../flows/tables.md). A room whose `computer` byte is zero shows
an enabled human Join beside a greyed Computer action, and the client never
sends `GameENC` there. The table count and the capacity are room-level
values; neither is a per-table seat limit.

### PTL

The client copies player records without validating their strings, so a
record that a `DE` or `PD` would reject can still arrive through the PTL; a
server validates before publishing.

The file has room for 256 table records; the client keeps 128 of them. The
client selects the last `D` record as the trade service
([services.md](../irc/services.md#trade-service)) and excludes `D` records
from the playable tables, so one room can publish 127 playable tables plus
the trade service. Playable tables in one room each have their own identity
and channel and share the room's roster and sequence.

The client finds a table by comparing a requested channel with each record's
channel byte for byte: `TABLE000001` is found, `table000001` is not. A server
keeps the advertised spelling in every later reference to the channel.

`E` is the event match service
([services.md](../irc/services.md#event-match-service),
[event-matches.md](../flows/event-matches.md)) and `F` the event card
service ([services.md](../irc/services.md#event-card-service),
[event-cards.md](../flows/event-cards.md)); the room that carries them is in
[event-rooms.md](../flows/event-rooms.md).
