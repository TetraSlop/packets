# Profile file service

The profile file service is a separate TCP service on pp000.pol.com port
51220 ([services.md](../irc/services.md#profile-file-service)). Below it is
a plain TCP connection that carries exactly one request and one response;
above it are the handle, character and status records described here and
every file the client reads or deletes: the save file
([save-file.md](save-file.md)), the directory files
([directory-files.md](directory-files.md)), and the ranking, auction and
event files. Requests are encrypted with the cipher of the client's game
IRC session ([session.md](../irc/session.md)), so a server must match a
request to the session whose key decrypts it.

## Open

The client sends 40 plaintext bytes.

```c
struct ProfileOpen {
    /* 0x00 */ uint8_t zero;            // 0
    /* 0x01 */ uint8_t plain;           // 0 encrypted mode, nonzero plaintext mode
    /* 0x02 */ uint8_t pad_02[2];       // zero
    /* 0x04 */ uint16_t one;            // 1
    /* 0x06 */ uint8_t pad_06[34];      // zero
};
```

The server answers 24 bytes.

```c
struct ProfileOpenAns {
    /* 0x00 */ uint8_t marker;          // 0x81
    /* 0x01 */ uint8_t error;           // 0 for success
    /* 0x02 */ uint8_t pad_02[2];
    /* 0x04 */ uint32_t size;           // object size, 0 here
    /* 0x08 */ uint8_t pad_08[12];
    /* 0x14 */ uint32_t nonce;          // nonce the request digest is computed over
};
```

The open packet does not identify the account.

## Request

The request is a 40-byte header followed by the object.

```c
struct ProfileRequest {
    /* 0x00 */ uint8_t marker;          // 2
    /* 0x01 */ uint8_t command;         // operation, see Operations
    /* 0x02 */ uint8_t subcommand;      // operation, see Operations
    /* 0x03 */ uint8_t pad_03[1];
    /* 0x04 */ uint32_t size;           // object size
    /* 0x08 */ uint8_t pad_08[16];
    /* 0x18 */ uint8_t digest[16];      // MD5(login ID (8 chars) + password + nonce as LE32)
};
```

In encrypted mode the header and object are one OFB keystream from the IV,
starting at offset 0 of the header. The response (24-byte header plus object) is a
fresh keystream from the IV. Object fields are little-endian.

The cipher context belongs to the game IRC session that was open when the
PlayOnline Viewer seeded it. Selecting Play closes and reopens game IRC with
a new key, but later profile requests still use the original context. A
server must therefore keep earlier nonce-authenticated contexts per
account, not just the live IRC key.

Server choice: a server tries each candidate context for the account and
accepts the one whose decrypted header reproduces the 16-byte digest; on
failure in encrypted mode it closes without a reply, since no response key
can be chosen.

## Response

```c
struct ProfileResponse {
    /* 0x00 */ uint8_t marker;          // 0x81
    /* 0x01 */ uint8_t error;           // error code, see below
    /* 0x02 */ uint8_t pad_02[2];
    /* 0x04 */ uint32_t size;           // object size
    /* 0x08 */ uint8_t pad_08[12];
    /* 0x14 */ uint32_t zero;           // nonce slot, zero
};
```

The object follows immediately. The client reads the response and then
closes; a server should keep the socket open until the client does.

| Error | Meaning |
|---|---|
| 0 | success |
| 1 | unsupported operation (unverified) |
| 0x7E | file not found |
| 0xE4 | authentication failure, plaintext mode only (unverified) |

## Object checksum

Checked objects end with a 4-byte little-endian checksum of the preceding
bytes: the sum of little-endian 32-bit words, with a partial trailing word
packed into the high end.

```
checksum(01)             = 0x01000000
checksum(01 02 03)       = 0x03020100
checksum(01 02 03 04 05) = 0x09030201
```

Requests for 4/5, 4/6, 4/7, 3/0, 3/2 and 3/3 carry the checksum; responses
marked "checked" below carry it too.

## Operations

| cmd/sub | Request object | Response object |
|---|---|---|
| 0/9 | empty | 8-byte count, then 0x88-byte handle records |
| 1/3 | empty | 8-byte count, then 0x68-byte character records |
| 2/3 | empty | 8-byte zero count (empty list) |
| 4/6 | 0x18 checked (20 zero bytes) | 0x7C status record, checked |
| 4/5 | 0x28 checked; bytes 0x10..0x17 are the new first 8 bytes of the status record | 0x20 checked (16 + 12 bytes) |
| 4/7 | 0x40 checked, all zero | empty |
| 3/0 | 0x1A0 checked file read | file slice, checked |
| 3/2 | 0x198 checked file delete | empty |
| 3/3 | 0x1A0 checked list | 8-byte count, then a checked trailer |

Handle record (0/9):

```c
struct HandleRecord {
    /* 0x00 */ uint8_t pad_00[8];
    /* 0x08 */ uint64_t pol_id;         // raw POL identity
    /* 0x10 */ char name[16];           // handle name
    /* 0x20 */ uint8_t pad_20[104];
};
```

Character record (1/3):

```c
struct CharacterRecord {
    /* 0x00 */ uint8_t pad_00[4];
    /* 0x04 */ uint8_t attached;        // 1 = attached to handle 0, slot 0
    /* 0x05 */ uint8_t pad_05[3];
    /* 0x08 */ uint16_t content_class;  // 2 for Tetra Master
    /* 0x0A */ uint8_t pad_0A[2];
    /* 0x0C */ uint32_t player_id;      // character (player) id
    /* 0x10 */ uint64_t pol_id;         // raw POL identity
    /* 0x18 */ char name[16];           // character name
    /* 0x28 */ uint8_t pad_28[64];
};
```

Status record (4/6):

```c
struct StatusRecord {
    /* 0x00 */ uint8_t handle;          // handle index
    /* 0x01 */ uint8_t pad_01[2];
    /* 0x03 */ uint8_t one;             // 1
    /* 0x04 */ uint16_t content_class;  // 1 in the Viewer, 2 once the game has set it through 4/5
    /* 0x06 */ uint8_t online;          // online status
    /* 0x07 */ uint8_t pad_07[117];
};
```

The 4/5 request replaces bytes 0..7 of the status record.

## File requests

Operations 3/0 (read), 3/2 (delete) and 3/3 (list) share one object.

```c
struct FileRequest {
    /* 0x000 */ uint8_t pad_000[8];
    /* 0x008 */ uint64_t owner;         // file owner, see below
    /* 0x010 */ char path[384];         // NUL-terminated path
    /* 0x190 */ uint32_t offset;        // read: offset; list: maximum count as a WORD
    /* 0x194 */ uint32_t length;        // read: maximum length
    /* 0x198 */ uint8_t pad_198[4];
    /* 0x19C */ uint32_t checksum;      // object checksum, see above
};
```

A delete (3/2) is 0x198 bytes: the members through `path`, four bytes of
padding and the checksum at 0x194.

The file owner packs the raw 41-bit POL identity in bits 0..40, the
directory in bits 41..47 and the volume in bits 48..63 (unverified). The
directory and volume come from the `D` and `V` values of the TeachDV reply
(see [startup.md](../flows/startup.md)). For the room's `b/g/PTL` the owner
is the room service ([directory-files.md](directory-files.md)).

A read returns `file[offset : offset + maximum]` followed by its checksum; a
missing file returns error 0x7E with no object. A delete (3/2) completes on
the response header alone. The client leaves the list request's padding
uninitialized, so a server must not require zero padding.

The Viewer's mailbox list (`O/m/`, 3/3) reads the 8-byte count without a
checksum and then, even for a zero count, reads four more checked bytes. An
empty mailbox reply is therefore 12 bytes: eight zero bytes and a zero
checksum. The game's file-count and file-list reads of the same operation
stop after the eight-byte count.

## File names

| Path | Size | Content |
|---|---|---|
| `U/g/TM0DataFile` | 0x3028 | the player's save, see [save-file.md](save-file.md) |
| `b/g/ZL` | 0x848 | zone list |
| `b/g/RL%03d` | up to 0xC848 | room list for zone `%03d` |
| `b/g/PTL` | 0xC050 | player and table list for the current room |
| `b/g/TM0RkData` | 28 | ranking metadata, seven DWORDs |
| `U/g/TM0_RANKLIST`, `U/g/TM0_RANKLIST_XXXXXXXX` | | ranking result rows, 0xE8 bytes each |
| `b/g/TM0AucData` | 20 | auction metadata, five DWORDs |
| `U/g/TM0_EXHIBITLIST`, `U/g/TM0_BIDLIST` | | auction search snapshots |
| `u/g/TM0_PM`, `u/g/TM0_BI`, `u/g/TM0_DI`, `u/g/TM0_RM` | 1 | auction pending markers, presence tested only |
| `b/g/TM0EventDataList` | 0x1308 | event list |
| `b/g/TM0EventMemberList` | 0x2808 | event member list |
| `O/m/` | | Viewer mailbox folder (3/3 only) |

`U/` files belong to the account; `b/` files are shared directory data. The
zone, room and table list layouts are in
[directory-files.md](directory-files.md); the ranking, auction and event
layouts are in [ranking-files.md](ranking-files.md),
[auction-files.md](auction-files.md) and [event-files.md](event-files.md).
