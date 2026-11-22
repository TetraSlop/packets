# Save file

`U/g/TM0DataFile` is the player's save: identity, money, statistics,
options, saved table defaults and the card collection. The client reads it
through the profile file service ([service.md](service.md)) and never writes
it; a server rewrites it whenever a result, purchase, trade, prize or option
save changes the player.

## Access

The path is `U/g/TM0DataFile`, owned by the account. The client reads the
whole file with one file read
([service.md](service.md#file-requests)) at login. Option changes
reach a server as [84/00 Opt](../irc/G/84.md#8400-opt), not as a file write.
The file is always 0x3028 bytes: a 0x148-byte header followed by 1000 card
slots of 12 bytes. All multibyte integers are little-endian. Bytes not listed
below are opaque and a server preserves them when it rewrites the file.

## Layout

```c
struct SaveHeader {
    /* 0x000 */ uint32_t header;            // file format marker, always 300; never a rating
    /* 0x004 */ uint32_t character_id;      // character id, the `GID` of Init (unverified)
    /* 0x008 */ uint64_t NN;                // account identity used in `NN=` fields (unverified)
    /* 0x010 */ char name[16];              // player name, may fill all 16 bytes without a terminator
    /* 0x020 */ uint8_t pad_020[16];
    /* 0x030 */ uint32_t rating;            // average rating; ResultData `D2` replaces it
    /* 0x034 */ int32_t money;              // money in T
    /* 0x038 */ uint16_t collection_points; // collection points (unverified)
    /* 0x03A */ uint8_t pad_03A[1];
    /* 0x03B */ uint8_t guild;              // 0 none, 1 Warriors, 2 Voyagers, 3 Summoners; 0 opens the guild picker at startup
    /* 0x03C */ uint16_t card_count;        // number of occupied card slots
    /* 0x03E */ uint8_t pad_03E[2];
    /* 0x040 */ uint16_t match_count;       // completed match count, divisor of the average reward; ResultData `D5` replaces it
    /* 0x042 */ uint8_t pad_042[2];
    /* 0x044 */ uint16_t sequence;          // result sequence, wraps; ResultData `D4` replaces it
    /* 0x046 */ uint8_t pad_046[2];
    /* 0x048 */ uint32_t max_reward;        // largest single reward
    /* 0x04C */ uint32_t average_reward;    // reward sum divided by match count
    /* 0x050 */ uint32_t total_prize;       // reward sum; the ranking Grand Total value
    /* 0x054 */ uint8_t pad_054[36];
    /* 0x078 */ int16_t streak;             // current streak, positive wins, negative losses
    /* 0x07A */ int16_t best_streak;        // best streak reached
    /* 0x07C */ uint8_t pad_07C[1];
    /* 0x07D */ char RN[32];                // Memorable Win text, NUL-terminated, at most 31 bytes
    /* 0x09D */ uint8_t pad_09D[19];
    /* 0x0B0 */ uint8_t Ar;                 // option `Ar`, 0 or 1, purpose unknown (unverified)
    /* 0x0B1 */ uint8_t Cu;                 // option `Cu`, purpose unknown (unverified)
    /* 0x0B2 */ uint8_t Vi;                 // option `Vi`, purpose unknown (unverified)
    /* 0x0B3 */ uint8_t Se;                 // option `Se`, sound effect volume 0..3
    /* 0x0B4 */ uint8_t Bgm;                // option `Bgm`, music volume 0..3
    /* 0x0B5 */ uint8_t Per;                // option `Per`, 1 accepts trade requests
    /* 0x0B6 */ uint8_t Ran;                // option `Ran`, 1 hides the name in rankings; also sent as `PR` in Continue
    /* 0x0B7 */ uint8_t pad_0B7[1];
    /* 0x0B8 */ uint32_t tet_bm;            // saved game default: wager `bm`
    /* 0x0BC */ uint8_t tet_du;             // saved game default: `du` toggle
    /* 0x0BD */ uint8_t pad_0BD[3];
    /* 0x0C0 */ uint8_t tet_st;             // saved game default: `st`, low tiles
    /* 0x0C1 */ uint8_t pad_0C1[3];
    /* 0x0C4 */ uint8_t tet_cb;             // saved game default: `cb`, effect tiles
    /* 0x0C5 */ uint8_t pad_0C5[3];
    /* 0x0C8 */ uint8_t tet_ca;             // saved game default: `ca`, spinners
    /* 0x0C9 */ uint8_t pad_0C9[3];
    /* 0x0CC */ uint8_t tet_gs;             // saved game default: `gs`, departure rule 0..2
    /* 0x0CD */ uint8_t pad_0CD[3];
    /* 0x0D0 */ uint8_t tet_tl;             // saved game default: `tl`, time limit 0..5
    /* 0x0D1 */ uint8_t pad_0D1[7];
    /* 0x0D8 */ uint8_t tab_in;             // saved table default: `in` 0..2
    /* 0x0D9 */ uint8_t pad_0D9[3];
    /* 0x0DC */ uint32_t tab_lu;            // saved table default: upper collection-point bound
    /* 0x0E0 */ uint32_t tab_ll;            // saved table default: lower collection-point bound
    /* 0x0E4 */ uint32_t tab_au;            // saved table default: upper rank bound
    /* 0x0E8 */ uint32_t tab_al;            // saved table default: lower rank bound
    /* 0x0EC */ uint8_t tab_co;             // saved table default: comment index 0..8
    /* 0x0ED */ uint8_t pad_0ED[3];
    /* 0x0F0 */ uint8_t tab_pa;             // saved table default: 1 when a password is set
    /* 0x0F1 */ uint8_t pad_0F1[3];
    /* 0x0F4 */ char tab_pw[10];            // saved table default: password, at most 9 bytes plus NUL
    /* 0x0FE */ uint8_t pad_0FE[2];
    /* 0x100 */ uint8_t CMD;                // option `CMD`, 0 or 1, purpose unknown (unverified)
    /* 0x101 */ uint8_t CAD;                // option `CAD`, 0 or 1, purpose unknown (unverified)
    /* 0x102 */ uint8_t CL;                 // option `CL`, chat lines 0..20
    /* 0x103 */ uint8_t CT;                 // option `CT`, chat transparency 30..70
    /* 0x104 */ char deck1_name[17];        // name of deck 1, NUL-terminated (unverified)
    /* 0x115 */ char deck2_name[17];        // name of deck 2, NUL-terminated (unverified)
    /* 0x126 */ uint8_t HNSS;               // option `HNSS`, 0 or 1, purpose unknown (unverified)
    /* 0x127 */ uint8_t pad_127[1];
    /* 0x128 */ uint32_t rt_points;         // VS. Rating total, displayed in hundredths
    /* 0x12C */ uint8_t pad_12C[4];
    /* 0x130 */ uint16_t vs_player_games;   // matches completed against human players
    /* 0x132 */ uint8_t pad_132[2];
    /* 0x134 */ uint32_t prize_points;      // Prize Center balance
    /* 0x138 */ uint8_t pad_138[16];
};
```

Card slots start at 0x148 and are 12 bytes each. `card_count` says how many
are occupied; the rest of the file is zero.

```c
struct SaveCard {
    /* 0x00 */ uint16_t model;          // catalogue model 0..249
    /* 0x02 */ uint8_t attack;          // attack
    /* 0x03 */ uint8_t kind;            // type 0..3
    /* 0x04 */ uint8_t physical;        // physical defence
    /* 0x05 */ uint8_t magical;         // magical defence
    /* 0x06 */ uint8_t points;          // card points
    /* 0x07 */ uint8_t arrows;          // arrow mask, one bit per direction
    /* 0x08 */ uint8_t deck;            // deck slot, 255 when not in a deck
    /* 0x09 */ uint8_t pad_09[3];       // zero
};
```

The card values and their arithmetic are in [rules.md](../client/rules.md)
and the catalogue in [catalogue.md](../client/catalogue.md).

## Rules

Two cards never share a deck slot. The limits below are what a server
respects when it rewrites the file; a limit that is a server's policy rather
than something the client enforces is marked as such.

| Field | Bound | Source |
|---|---|---|
| money | 0..99,999,999 | server choice; the client displays eight digits |
| cards | at most 1000 | file capacity |
| prize points | 0..99999 | unsigned cap in the Prize Center |
| match count | saturates at 65535 | the client divides by it, so it must not wrap to 0 |
| sequence | wraps at 65536 | offline bookkeeping |
| streaks | signed 16-bit | offline bookkeeping |
| RN | at most 31 bytes | 32-byte field with terminator |
| player name | at most 16 bytes, no NUL needed | field width |
| deck names | at most 17 bytes | field width (unverified) |

Option bytes:

| Option | Range | Notes |
|---|---|---|
| Ar | 0..1 | |
| Cu, Vi | 0..255 | preserved as raw bytes |
| Se, Bgm | 0..3 | volume |
| Per | 0..1 | trade requests |
| Ran | 0..1 | 0 display name, 1 hide name in rankings |
| CMD, CAD | 0..1 | |
| CL | 0..20 | chat lines; an invalid value falls back to 8 |
| CT | 30..70 | chat transparency |
| HNSS | 0..1 | |

Saved game defaults: `bm` 0..2000, `du`, `st`, `cb`, `ca` 0..1, `gs` 0..2,
`tl` 0..5. Saved table defaults: `in` 0..2, `lu`, `ll`, `au`, `al` 0..99999
with `ll` at most `lu` and `al` at most `au`, `co` 0..8, `pa` 0..1, `pw` at
most 9 bytes. The client copies at most 10 password bytes into its editor
without clearing what follows, so a password that fills all 10 bytes shows
stale characters after it; a server keeps the password to 9 bytes plus NUL.
When the password is disabled the client writes `********` followed by NULs.

Before the file is read, the client's built-in option values are Se 2, Bgm
2, Per 1, CL 5, CT 60 and 0 for every other option. Its built-in rating 200,
sequence 0 and guild 0 are overwritten by the file at login, so they are not
file defaults. The header value 300 is never used as a rating.

Server choice: a new account's file carries header 300, the option defaults
above, rating 0, sequence 0 and the server's chosen starting money and cards;
the original service's first-account values are not recovered.
