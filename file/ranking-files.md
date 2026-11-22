# Ranking files

The ranking service ([services.md](../irc/services.md#ranking-service),
[R.md](../irc/R.md)) answers a category request with the name of a
generated result file and a row count. The client reads that file and the
shared metadata file `b/g/TM0RkData` through the profile file service
([service.md](service.md)) and draws one ranking list from them. A server
generates the result file and keeps the metadata file current.

## Access

### Result file

The [RF reply](../irc/R.md#rf-reply) names an account-owned file,
`U/g/TM0_RANKLIST_` followed by eight hex digits, and gives its row count.
The client reads the file after the reply. It has no header: rows of 0xE8
bytes, as many as the reply announced; an empty category is a zero-length
file. On entering the ranking screen the client deletes the fixed name
`U/g/TM0_RANKLIST`, and on leaving it deletes the announced name; a server
treats the fixed-name delete as clearing all of the account's generated
ranking files ([R.md](../irc/R.md#sequence)).

### TM0RkData

`b/g/TM0RkData` is shared data of 28 bytes, seven little-endian DWORDs, read
through the profile file service alongside the result file.

## Layout

### Result file

Only the fields below are read by the ranking screen; a server may leave
every other byte zero. All multi-byte values are little-endian.

```c
struct RankingEntry {
    /* 0x00 */ char     name[16];        // player name, NUL-terminated
    /* 0x10 */ uint64_t identity;        // account NN; with character_id marks the requester's own row
    /* 0x18 */ uint32_t character_id;    // character id
    /* 0x1C */ uint8_t  pad_1C[0x78];
    /* 0x94 */ uint32_t grand_total;     // Grand Total prize money (category 0)
    /* 0x98 */ uint8_t  pad_98[4];
    /* 0x9C */ uint32_t weekly_total;    // Weekly Total prize money (category 1)
    /* 0xA0 */ uint32_t vs_rating;       // VS. Rating (category 2), displayed in hundredths
    /* 0xA4 */ uint32_t rating;          // rating for Top 30 and Best Rookies (categories 3 and 4), displayed in hundredths
    /* 0xA8 */ uint8_t  pad_A8[0x18];
    /* 0xC0 */ uint8_t  anonymous;       // 1 hides the name and shows the anonymous resource string instead
    /* 0xC1 */ uint8_t  pad_C1[1];
    /* 0xC2 */ uint8_t  qualified;       // 1 when the row counts for the VS. Rating list
    /* 0xC3 */ int8_t   previous_rank_4; // previous rank, category 4
    /* 0xC4 */ int8_t   previous_rank_3; // previous rank, category 3
    /* 0xC5 */ int8_t   previous_rank_2; // previous rank, category 2
    /* 0xC6 */ int8_t   previous_rank_0; // previous rank, category 0
    /* 0xC7 */ int8_t   previous_rank_1; // previous rank, category 1
    /* 0xC8 */ uint8_t  pad_C8[0x20];
};
```

### TM0RkData

```c
struct TM0RkData {
    /* 0x00 */ uint32_t pad_00;              // not read; zero
    /* 0x04 */ uint32_t rating_population;   // population shown by the VS. Rating and Grand Total lists
    /* 0x08 */ uint32_t weekly_population;   // population shown by Weekly Total
    /* 0x0C */ uint32_t update_time;         // update timestamp the date labels derive from
    /* 0x10 */ uint32_t pad_10;              // not read; zero
    /* 0x14 */ uint32_t top30_population;    // population shown by Top 30
    /* 0x18 */ uint32_t rookie_population;   // population shown by Best Rookies
};
```

## Rules

### Result file

The client copies the name until NUL, so a server terminates it: 15 bytes
plus NUL at most. The VS. Rating list stops at the first row whose
`qualified` byte is 0.

Previous ranks are signed bytes; 0 means unknown and 1..127 are the
displayable positions. Rows are shown in file order; equal scores stay
adjacent and the screen assigns shared ranks itself.

The `anonymous` byte follows the player's `Ran` option
([save-file.md](save-file.md), also sent as `PR` on the result-screen
Continue): 0 displays the name, 1 hides it.

Scores are drawn signed. Prize totals are drawn as signed decimals; ratings
are drawn as hundredths (a value of 12345 reads as 123.45). A value with bit
31 set therefore displays as negative text (0xFFFFFFFF draws as `-1` for
prize money and `0.-1` for ratings); the client itself leaves every DWORD
unchanged.

Which profile field the original service placed in `vs_rating` is not
recovered; the profile's VS. Rating total is also displayed in hundredths.

Server choice:

- A server fills `grand_total` from the profile's reward sum and `rating`
  from the profile's average rating.
- A server sorts descending by score, then by identity.
- A server caps Top 30 at 30 rows and the other lists at 100.
- A server that derives the rating from the offline rating ring
  ([ratings-and-statistics.md](../client/ratings-and-statistics.md#offline-rating-ring))
  sorts the rating categories ascending, because lower ring ratings are
  better.

### TM0RkData

The populations are display totals, not row counts, so they may exceed the
rows in the capped files.

Date labels derive from `update_time` with bit 31 masked off. Weekly Total
draws `Tally Period: <t - 14 days> - <t - 8 days>`; Top 30 and Best Rookies
draw `Next update: <t>`. The format string contains a literal `PST`. When
the unsigned value is at most 0x38640900 no label is drawn.

Server choice: a server reports every provisioned account in
`rating_population` and the eligible population of each category in the
other three; a server that tracks weekly periods automatically writes 0 in
`update_time`, because one timestamp cannot label both a closed tally and
the next update truthfully.
