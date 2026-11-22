# R Ranking

The ranking screen asks the [ranking service](services.md#ranking-service)
for one category at a time. The service answers with the name of a generated
binary file and a row count; the client reads that file, and a shared
metadata file, through the
[profile file service](services.md#profile-file-service), and deletes the
generated file when it leaves the screen. Five categories exist: 0 Grand
Total, 1 Weekly Total, 2 VS. Rating, 3 Top 30, 4 Best Rookies. The layout of
both files is in [ranking-files.md](../file/ranking-files.md) and the
statistics behind the scores are in
[ratings and statistics](../client/ratings-and-statistics.md). Nothing in
the client establishes how the original service tallied weekly prize money,
chose the Top 30 rating period, or qualified Best Rookies; the client only
displays what the files contain.

## Framing

A `GTM0R` payload is a sequence of commands, each `<XX>` BEL payload BEL,
where `XX` is a two-letter token; fields inside one payload are separated
by ACK. In the examples `\x07` is BEL and `\x06` is ACK. Other `GTM0R`
tokens (for example `GO`) exist in the client's command dictionary, but the
ranking screen never sends them and their layout is unknown.

## RR Request

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | ranking service | [RF Reply](#rf-reply) | `struct RankingRequest` |

```c
struct RankingRequest {
    uint32_t RR;             // ranking category, decimal 0..4
    struct {
        uint64_t owner;      // game identity of the requesting player, `0x` plus 16 hex digits
        uint32_t directory;  // profile directory selector, decimal that fits in 8 bits
        uint32_t version;    // profile version, decimal that fits in 16 bits
        uint32_t selector;   // decimal that fits in 32 bits; always 0 from this client
    } PI;
};
```

```
C2S GTM0R<RR>\x072\x07<PI>\x070x0000000000000001\x061\x061\x060\x07
```

Every request carries exactly two commands, `RR` then `PI`, for every
category. The owner in `PI` is the file owner for the result file, not the
nickname of the ranking service; a server answers from the nickname the
client addressed.

## RF Reply

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | ranking service | none | `struct RankingReply` |

```c
struct RankingReply {
    char     RF[0x180];  // path of the generated result file, read with the profile file service
    uint32_t LN;         // number of 0xE8-byte records in the file
};
```

```
S2C GTM0R<RF>\x07U/g/TM0_RANKLIST_00000001\x07<LN>\x072\x07
S2C GTM0R<RG>\x07-8811\x07
```

The first line is the success form, `RF` with `LN`. The second line is the
failure form: `RG` carries a signed code from [Error codes](#error-codes).
The client accepts `RF` or `RG` only while a request is pending. After `RF`
it reads the announced file through the profile file service with
`offset = first * 0xE8` and `length = count * 0xE8`, and keeps at most 100
rows.

Server choice: each successful query publishes a fresh, immutable,
account-scoped file under a unique name (`U/g/TM0_RANKLIST_` plus eight hex
digits), so a later query cannot change a file already announced. An empty
category still answers `RF` with `LN` 0 and a zero-length file.

Server choice: a server optionally tracks weeks that start
Monday 00:00 UTC, filling Weekly Total from prize money won in the current
week and Top 30 from each account's last rating of the previous week, and
names the previous week's prize-money leader for a champion shop pack. Best
Rookies requires an explicit rookie flag. Without that tracking those
categories answer `LN` 0.

## Sequence

1. On entering the ranking screen the client deletes the fixed name
   `U/g/TM0_RANKLIST` through the profile file service
   ([file delete](../file/service.md#file-requests)). A server
   that generates unique names treats a delete of the fixed name as clearing
   all of the requesting account's generated ranking files, and a delete of
   an exact generated name as removing that file only. Both are idempotent
   and account-scoped.
2. The client sends [RR with PI](#rr-request) for the selected category.
3. The service answers [RF with LN](#rf-reply), or `RG` with an error code.
4. The client reads the announced file and the metadata file
   `b/g/TM0RkData` through the profile file service.
5. When the client leaves the screen it deletes the generated file.

## Error codes

| Code | Meaning |
| --- | --- |
| -8811 | unknown category (unverified) |
| -8809 | the result file could not be created; the client reports a rank file write failure (unverified) |
