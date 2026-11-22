# Auction files

The auction service ([services.md](../irc/services.md#auction-service),
[A.md](../irc/A.md)) answers a search with the name of a result file, and
the client reads that file, a shared count file and four delivery markers
through the profile file service ([service.md](service.md)). A server
writes the result files when it answers a search and keeps the counts and
markers current.

## Access

### Result files

The `SS` reply to a listing search names `U/g/TM0_EXHIBITLIST_` followed by
eight hex digits, and the `HS` reply to a bid history search names
`U/g/TM0_BIDLIST_` followed by eight hex digits
([A.md](../irc/A.md#si-search-listings)). Both are account-owned
and carry as many records as the reply announced, with no header. The client
reads a file after the reply and may delete `U/g/TM0_EXHIBITLIST` or
`U/g/TM0_BIDLIST` to clear its snapshots.

### TM0AucData

`b/g/TM0AucData` is shared data of 20 bytes read through the profile file
service.

### Delivery markers

`u/g/TM0_PM`, `u/g/TM0_BI`, `u/g/TM0_DI` and `u/g/TM0_RM` are account-owned
files the client probes for existence only; it reads no bytes from them.

## Layout

### Result files

An actor record is 0x38 bytes, zero outside the three members:

```c
struct AuctionActorRecord {
    /* 0x00 */ uint64_t nn;              // account NN
    /* 0x08 */ uint32_t character;       // character id
    /* 0x0C */ uint8_t  pad_0C[12];
    /* 0x18 */ char     name[16];        // character name, NUL-terminated
    /* 0x28 */ uint8_t  pad_28[16];
};
```

A listing file record is 0xC0 bytes, the same values as the `ES` listing
reply ([A.md](../irc/A.md#es-exhibit-success)):

```c
struct AuctionListingRecord {
    /* 0x00 */ struct AuctionActorRecord seller;
    /* 0x38 */ uint32_t auction_id;      // auction ID
    /* 0x3C */ uint32_t opening;         // opening bid
    /* 0x40 */ uint32_t increase;        // minimum increase
    /* 0x44 */ uint32_t ED;              // created time
    /* 0x48 */ uint32_t NC;              // expiry clock; the client subtracts the current clock for the remaining time
    /* 0x4C */ uint32_t LC;              // last bid time, or created time before any bid
    /* 0x50 */ uint8_t  hours;           // duration in hours
    /* 0x51 */ uint8_t  reoffers_allowed; // automatic reoffers when unsold
    /* 0x52 */ uint8_t  reoffers_used;   // reoffers used so far
    /* 0x53 */ uint8_t  pad_53[5];
    /* 0x58 */ struct AuctionActorRecord bidder; // current bidder, zeros when none
    /* 0x90 */ uint32_t CM;              // current amount: the leading bid, or the opening bid before any bid
    /* 0x94 */ uint32_t CD;              // last bid time, 0 before any bid
    /* 0x98 */ uint32_t BC;              // bid count
    /* 0x9C */ uint8_t  pad_9C[4];
    /* 0xA0 */ struct AuctionItem descriptor;   // card descriptor, see A.md
};
```

`struct AuctionItem` is the `II` descriptor of
[A.md](../irc/A.md#er-exhibit): four 32-bit, four 16-bit and eight
8-bit values.

A bid history record is 0x48 bytes:

```c
struct AuctionBidRecord {
    /* 0x00 */ struct AuctionActorRecord bidder;
    /* 0x38 */ uint32_t auction_id;      // auction ID
    /* 0x3C */ uint32_t amount;          // bid amount
    /* 0x40 */ uint32_t time;            // bid time
    /* 0x44 */ uint32_t zero;            // 0
};
```

### TM0AucData

Counts over the active listings:

```c
struct AuctionData {
    /* 0x00 */ uint32_t all;             // every active listing
    /* 0x04 */ uint32_t upto_1000;       // current amount 0..1000
    /* 0x08 */ uint32_t upto_2000;       // current amount 1001..2000
    /* 0x0C */ uint32_t upto_5000;       // current amount 2001..5000
    /* 0x10 */ uint32_t above_5000;      // current amount above 5000
};
```

### Delivery markers

A marker has no layout the client reads; when present it is one zero byte.

## Rules

Listing records are ordered by auction ID ascending and bid history records
by amount descending.

Server choice:

- Timestamps are Unix seconds.
- Each result file is an immutable snapshot private to the connection, as
  in [A.md](../irc/A.md#si-search-listings).
- All four delivery markers exist while the account has any pending
  delivery and are absent otherwise.
