# Event files

Three fixed-size files tell the client about tournaments. `b/g/TM0EventList`
marks which directory rooms are event rooms; `b/g/TM0EventDataList` and
`b/g/TM0EventMemberList` describe one event and its members. The client
reads all three through the profile file service ([service.md](service.md));
the event card service ([services.md](../irc/services.md#event-card-service))
decides what a member receives, these files only display. A server that
hosts no event serves all three with a zero count.

## Access

### TM0EventList

`b/g/TM0EventList` is shared directory data of 0x2C08 bytes, not specific to
a room. The client reads it while browsing the room directory and joins its
records to zone and room names.

### TM0EventDataList

`b/g/TM0EventDataList` is 0x1308 bytes, owned by the event room. The client
reads it when it enters the event room, naming the room as the file owner.
A server resolves the read by that owner and never falls back to another
room's data when the owner is unknown. The owner packs the 41-bit raw POL
identity in the low bits with directory and volume above it; the room is
selected by the low 41 bits (unverified).

### TM0EventMemberList

`b/g/TM0EventMemberList` is 0x2808 bytes, owned by the event room and read
on entry in the same way as `TM0EventDataList`.

## Layout

### TM0EventList

```c
struct EventListRecord {                       // stride 0x160; offsets of scalar, flags, start and end (unverified)
    /* 0x00 */ uint32_t scalar;                // value shown in the room view, purpose unknown (unverified); never 0xFFFFFFFF
    /* 0x04 */ uint8_t  flags[4];              // copied to the room view, purpose unknown (unverified)
    /* 0x08 */ int32_t  start;                 // listing start, copied to the room view, purpose unknown (unverified)
    /* 0x0C */ int32_t  end;                   // listing end, copied to the room view, purpose unknown (unverified)
    /* 0x10 */ char     key_region[32];        // zone name the event belongs to, NUL-terminated
    /* 0x30 */ char     key_event[32];         // room name of the event, NUL-terminated
    /* 0x50 */ char     description[0x110];    // event description, NUL-terminated
};

struct EventList {                             // b/g/TM0EventList, 0x2C08 bytes
    /* 0x00 */ uint8_t  pad_00[4];
    /* 0x04 */ uint32_t count;                 // record count, at most 32
    /* 0x08 */ struct EventListRecord records[32];
};
```

### TM0EventDataList

```c
struct EventPrizeRow {                         // stride 0x38
    /* 0x00 */ uint8_t  flags;                 // bit 0 cards present, bit 1 money present, bit 2 text present
    /* 0x01 */ uint8_t  pad_01;
    /* 0x02 */ uint16_t cards[3];              // card models 0..249, 0xFFFF when empty
    /* 0x08 */ uint32_t money;                 // money shown for the row
    /* 0x0C */ uint8_t  pad_0C[4];
    /* 0x10 */ char     description[40];       // row text, NUL-terminated
};

struct EventDataList {                         // b/g/TM0EventDataList, 0x1308 bytes; offsets and domains (unverified)
    /* 0x0000 */ uint8_t  pad_0000[0x20];
    /* 0x0020 */ uint32_t start[5];            // start date components; the component order is not established
    /* 0x0034 */ uint8_t  pad_0034[4];
    /* 0x0038 */ uint32_t end[5];              // end date components
    /* 0x004C */ uint8_t  pad_004C[8];
    /* 0x0054 */ uint32_t class_count;         // number of classes: 0, 2, 3 or 4
    /* 0x0058 */ uint8_t  pad_0058[8];
    /* 0x0060 */ char     class_name_0[32];    // class name, NUL-terminated
    /* 0x0080 */ uint32_t steps_per_class;     // board selector, see Rules
    /* 0x0084 */ uint8_t  pad_0084[4];
    /* 0x0088 */ char     class_name_1[32];    // class name, NUL-terminated
    /* 0x00A8 */ uint8_t  pad_00A8[8];         // zero
    /* 0x00B0 */ char     class_name_2[32];    // class name, NUL-terminated
    /* 0x00D0 */ uint8_t  pad_00D0[8];         // zero
    /* 0x00D8 */ char     class_name_3[32];    // class name, NUL-terminated
    /* 0x00F8 */ uint8_t  pad_00F8[0x1008];
    /* 0x1100 */ struct EventPrizeRow prizes[9];   // prize display rows
    /* 0x12F8 */ uint8_t  pad_12F8[4];
    /* 0x12FC */ uint32_t mission_mask;        // missions displayed, one bit each in the low byte, at most three set
    /* 0x1300 */ uint8_t  mission_target[8];   // target per mission bit; a displayed mission needs a positive target
};
```

The class names are four 32-byte fields at stride 0x28; the board selector
sits in the eight bytes after the first name and the matching bytes after
the other three names are zero.

### TM0EventMemberList

```c
struct EventMemberRecord {                     // stride 0x28; offsets of steps, missions and eligibility (unverified)
    /* 0x00 */ uint64_t identity;              // member identity, the account NN
    /* 0x08 */ char     name[16];              // display name, NUL-terminated, at most 15 bytes of text
    /* 0x18 */ uint8_t  pad_18[4];
    /* 0x1C */ uint32_t steps;                 // steps reached; at most 0x7FFFFFFF because the client sorts it as signed
    /* 0x20 */ uint32_t missions;              // completed mission bits
    /* 0x24 */ int32_t  eligibility;           // negative means excluded
};

struct EventMemberList {                       // b/g/TM0EventMemberList, 0x2808 bytes
    /* 0x00 */ uint8_t  pad_00[4];
    /* 0x04 */ uint32_t count;                 // member count, at most 256
    /* 0x08 */ struct EventMemberRecord records[256];
};
```

## Rules

### TM0EventList

The client associates a record with a directory entry by exact,
case-sensitive string compare, not by an identifier: the selected zone name
from the ZL record against `key_region`, and the RL room name up to its `*`
terminator against `key_event`. On a match it shows an event view for the
room with the record's scalar, flags, times and description. Two events may
share `key_event` under different regions.

Because the directory fields are shorter than the 32-byte keys, a served
`key_region` is 1..26 bytes and a served `key_event` is 1..30 bytes,
containing neither NUL nor `*`. The client uses 0xFFFFFFFF as its own "no
match" value for the scalar, so a served scalar is never 0xFFFFFFFF.

### TM0EventDataList

The file is display data: schedule, class names, board selector, prize rows
and mission targets. Nothing in it authorises a grant; the event card
service decides what a member receives ([event-cards.md](../flows/event-cards.md)).

The accepted `steps_per_class` range depends on `class_count`: 0 gives 1, 2
gives 4..8, 3 gives 1..5 and 4 gives 1..4.

The client opens the event card collection screen only when the member's
steps give rank 1..3, a mission is complete, the class count is 3, or the
class count is 4 and `(steps - 1) / steps_per_class` (integer division) is
positive. That is a screen-entry condition, not a reward rule.

### TM0EventMemberList

The client finds its own record by comparing `identity` with its account
NN, not with its character id. Identities are nonzero and unique within the
file.
