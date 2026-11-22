# P Character profile

The `P` family reads and edits the character profile, a 0xE8-byte record
that is separate from the save file and is never a profile file service
transaction. Every command is addressed to the
[registration service](services.md#registration-service), the only service
that speaks this family. The client sends `CR` and `PG` during startup
([startup.md](../flows/startup.md)) and `GR` from the profile editor.

## Framing

A `GTM0P` payload is a sequence of commands in the command codec of
[envelope.md](envelope.md#command-codec): each command is a two-letter
token in angle brackets, BEL (0x07), the payload, BEL. Fields inside a
payload are separated by ACK (0x06). Commands follow each other without
separators and the whole payload, including `GTM0P`, is at most 469 bytes.

The first command of every request carries a locator that names the
character:

```c
struct Locator {
    uint64_t identity;  // character identity, written `0x` plus 16 hex digits
    uint32_t key;       // character key, decimal
};
```

A reply carries no locator. The client accepts a reply from any sender and
does not compare it with the request, so a server answers from the
registration service's nickname and only to the requesting client.

## CR Registration

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | registration service | `CS`, or `NS` with a signed error, below | `struct CR` |

```c
struct CR {
    struct Locator locator;  // character to register
    char AN[16];             // player name
    uint32_t CI_locale;      // language setting: 0 selects ISO-2022-JP text, any other value ISO-8859-1
    char CI_summary[];       // summary text built from a client resource string, for example `Card Level 0`
};
```

```
C2S GTM0P<CR>\x070x0000000000000001\x061\x07<AN>\x07LocalPlayer\x07<CI>\x071\x06Card Level 0\x07
S2C GTM0P<CS>\x07\x07
S2C GTM0P<NS>\x07<signed error>\x07
```

`CS` has an empty payload and means the character is registered; the client
continues startup. `NS` carries a signed decimal error and is a failure.
The summary text is not typed by the player.

## PG Read

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | registration service | `PO` with the record, or `PM` with a signed error, below | `struct PG`, `struct CharacterProfile` |

```c
struct PG {
    struct Locator locator;  // character to read
};
```

```
C2S GTM0P<PG>\x070x0000000000000001\x061\x07
S2C GTM0P<PO>\x07<field 0>\x06<field 1>\x06...\x06<field 70>\x07
S2C GTM0P<PM>\x07<signed error>\x07
```

`PO` carries 71 ACK-separated fields that the client decodes into the record
below. Identities are hex u64, floats decimal text, everything else decimal
integers.

```c
struct CharacterProfile {
    /* 0x00 */ char name[16];               // PO field 2, player name, terminated, so at most 15 bytes
    /* 0x10 */ uint64_t identity;           // PO field 0, character identity
    /* 0x18 */ uint32_t key;                // PO field 1, character key
    /* 0x1C */ uint8_t purpose;             // PO field 3, purpose, 0..5, see the table below
    /* 0x1D */ uint8_t f4;                  // PO field 4, purpose unknown (unverified)
    /* 0x1E */ uint8_t f5;                  // PO field 5, purpose unknown (unverified)
    /* 0x1F */ uint8_t f6;                  // PO field 6, purpose unknown (unverified)
    /* 0x20 */ uint8_t pad_20[9];
    /* 0x29 */ uint8_t f7;                  // PO field 7, purpose unknown (unverified)
    /* 0x2A */ uint8_t FO;                  // PO field 8, Player Profile visibility, master gate for the three statistics
    /* 0x2B */ uint8_t pad_2B[5];           // bytes 0x2C and 0x2E are zeroed by the client, not read from the wire
    /* 0x30 */ char average_rank[64];       // PO field 9, Average Rank display text
    /* 0x70 */ uint64_t f10;                // PO field 10, purpose unknown (unverified)
    /* 0x78 */ uint64_t f11;                // PO field 11, purpose unknown (unverified)
    /* 0x80 */ float f12;                   // PO field 12, purpose unknown (unverified)
    /* 0x84 */ float f13;                   // PO field 13, purpose unknown (unverified)
    /* 0x88 */ float f14;                   // PO field 14, purpose unknown (unverified)
    /* 0x8C */ float f15;                   // PO field 15, purpose unknown (unverified)
    /* 0x90 */ uint32_t card_level;         // PO field 16, Card Level displayed integer
    /* 0x94 */ uint32_t f17;                // PO field 17, purpose unknown (unverified)
    /* 0x98 */ uint32_t f18;                // PO field 18, purpose unknown (unverified)
    /* 0x9C */ uint32_t f19;                // PO field 19, purpose unknown (unverified)
    /* 0xA0 */ uint32_t f20;                // PO field 20, purpose unknown (unverified)
    /* 0xA4 */ uint32_t f21;                // PO field 21, purpose unknown (unverified)
    /* 0xA8 */ uint32_t f22;                // PO field 22, purpose unknown (unverified)
    /* 0xAC */ uint32_t f23;                // PO field 23, purpose unknown (unverified)
    /* 0xB0 */ uint16_t title;              // PO field 24, Title selection, read as signed
    /* 0xB2 */ uint16_t f25;                // PO field 25, purpose unknown (unverified)
    /* 0xB4 */ uint16_t f26;                // PO field 26, purpose unknown (unverified)
    /* 0xB6 */ uint16_t f27;                // PO field 27, purpose unknown (unverified)
    /* 0xB8 */ uint16_t f28;                // PO field 28, purpose unknown (unverified)
    /* 0xBA */ uint16_t f29;                // PO field 29, purpose unknown (unverified)
    /* 0xBC */ uint16_t f30;                // PO field 30, purpose unknown (unverified)
    /* 0xBE */ uint16_t f31;                // PO field 31, purpose unknown (unverified)
    /* 0xC0 */ uint8_t f32;                 // PO field 32, purpose unknown (unverified)
    /* 0xC1 */ uint8_t f33;                 // PO field 33, purpose unknown (unverified)
    /* 0xC2 */ uint8_t f34;                 // PO field 34, purpose unknown (unverified)
    /* 0xC3 */ uint8_t f35;                 // PO field 35, purpose unknown (unverified)
    /* 0xC4 */ uint8_t f36;                 // PO field 36, purpose unknown (unverified)
    /* 0xC5 */ uint8_t f37;                 // PO field 37, purpose unknown (unverified)
    /* 0xC6 */ uint8_t f38;                 // PO field 38, purpose unknown (unverified)
    /* 0xC7 */ uint8_t f39;                 // PO field 39, purpose unknown (unverified)
    /* 0xC8 */ uint8_t VO;                  // PO field 40, Average Rank visibility
    /* 0xC9 */ uint8_t f41;                 // PO field 41, purpose unknown (unverified)
    /* 0xCA */ uint8_t f42;                 // PO field 42, purpose unknown (unverified)
    /* 0xCB */ uint8_t f43;                 // PO field 43, purpose unknown (unverified)
    /* 0xCC */ uint8_t f44;                 // PO field 44, purpose unknown (unverified)
    /* 0xCD */ uint8_t f45;                 // PO field 45, purpose unknown (unverified)
    /* 0xCE */ uint8_t f46;                 // PO field 46, purpose unknown (unverified)
    /* 0xCF */ uint8_t NO6;                 // PO field 47, Card Level visibility
    /* 0xD0 */ uint8_t f48;                 // PO field 48, purpose unknown (unverified)
    /* 0xD1 */ uint8_t f49;                 // PO field 49, purpose unknown (unverified)
    /* 0xD2 */ uint8_t f50;                 // PO field 50, purpose unknown (unverified)
    /* 0xD3 */ uint8_t f51;                 // PO field 51, purpose unknown (unverified)
    /* 0xD4 */ uint8_t f52;                 // PO field 52, purpose unknown (unverified)
    /* 0xD5 */ uint8_t f53;                 // PO field 53, purpose unknown (unverified)
    /* 0xD6 */ uint8_t f54;                 // PO field 54, purpose unknown (unverified)
    /* 0xD7 */ uint8_t NO14;                // PO field 55, Title visibility
    /* 0xD8 */ uint8_t f56;                 // PO field 56, purpose unknown (unverified)
    /* 0xD9 */ uint8_t f57;                 // PO field 57, purpose unknown (unverified)
    /* 0xDA */ uint8_t f58;                 // PO field 58, purpose unknown (unverified)
    /* 0xDB */ uint8_t f59;                 // PO field 59, purpose unknown (unverified)
    /* 0xDC */ uint8_t f60;                 // PO field 60, purpose unknown (unverified)
    /* 0xDD */ uint8_t f61;                 // PO field 61, purpose unknown (unverified)
    /* 0xDE */ uint8_t f62;                 // PO field 62, purpose unknown (unverified)
    /* 0xDF */ uint8_t f63;                 // PO field 63, purpose unknown (unverified)
    /* 0xE0 */ uint8_t f64;                 // PO field 64, purpose unknown (unverified)
    /* 0xE1 */ uint8_t f65;                 // PO field 65, purpose unknown (unverified)
    /* 0xE2 */ uint8_t f66;                 // PO field 66, purpose unknown (unverified)
    /* 0xE3 */ uint8_t f67;                 // PO field 67, purpose unknown (unverified)
    /* 0xE4 */ uint8_t f68;                 // PO field 68, purpose unknown (unverified)
    /* 0xE5 */ uint8_t f69;                 // PO field 69, purpose unknown (unverified)
    /* 0xE6 */ uint8_t f70;                 // PO field 70, purpose unknown (unverified)
    /* 0xE7 */ uint8_t pad_E7[1];
};
```

The `f` members have no recovered meaning. The client decodes them and the
editor never sends them back, so a server stores and returns them
unchanged.

The client parses each float field with `atof` from a 128-byte buffer with
no length check. A server emits at most 127 characters per float field, a
finite float32 value, and only the characters digits, `+`, `-`, `E`, `e`
and `.`.

| Purpose | Meaning |
|---|---|
| 0 | None in particular |
| 1 | Let's play! |
| 2 | I need advice! |
| 3 | Let's talk! |
| 4 | Experts wanted! |
| 5 | Let's trade! |

Visibility bytes: 1 is private. The editor writes exactly 1 or 3, and on
load treats every value other than 1 as public and normalizes it to 3.
`FO=1` hides Card Level, Title and Average Rank together; name and purpose
still render.

The client completes the read on the first `PO` or `PM` it receives and
copies the record once.

Server choice: an unknown character is answered `<PM>\x07-8706\x07`; a
default record for an account that has never edited its profile presents the
first 15 bytes of the save name, purpose 0 and all visibility bytes 3.

## GR Update

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | registration service | `GK`, or `GG` followed by `PE` with a signed result, below | `struct GR` |

```c
struct GR {
    struct Locator locator;  // character to update
    uint8_t PP;              // purpose, 0..5 (PO field 3)
    struct {
        uint8_t selector;    // 6 Card Level visibility (PO field 47); 14 Title visibility (PO field 55)
        uint8_t flag;        // visibility, 1 private or 3 public
    } NO[2];
    uint8_t VO;              // Average Rank visibility (PO field 40)
    uint8_t FO;              // Player Profile visibility (PO field 8)
};
```

```
C2S GTM0P<GR>\x070x0000000000000001\x061\x07<PP>\x071\x07<NO>\x076\x063\x07<NO>\x0714\x061\x07<VO>\x073\x07<FO>\x073\x07
S2C GTM0P<GK>\x07\x07
S2C GTM0P<GG>\x07\x07<PE>\x07<signed error>\x07
```

The editor always sends the commands in this order: `PP`, `NO` with
selector 6, `NO` with selector 14, `VO`, `FO`. `GR` may also carry `AN` and
`CI` with the same payloads as `CR`.

The two `NO` selectors are distinct updates and both must survive. The
client writes the selectors as `6` and `14`; a server that accepts leading
zeros must still treat `06` as selector 6. The editor only ever writes
visibility 1 or 3.

A new `GR` replaces any outstanding one; the client has no busy rejection.
`GK` has an empty payload and means success. `GG` is followed by `PE`
carrying a signed decimal result; the editor finishes on a non-negative
result and shows a dialog on a negative one.

Server choice: a `GR` for a character the account does not own is answered
`<GG>\x07\x07<PE>\x07-8706\x07`; an unknown tag or a duplicate tag is
rejected. Partial updates preserve every other field.
