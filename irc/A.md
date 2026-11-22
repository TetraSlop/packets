# A Auction

The auction has two halves. Listing, bidding and searching are family `A`
commands exchanged with the auction service
([services.md](services.md#auction-service)). Settled money and cards
reach the player later through a collection visit to the shop service,
which uses [B2](G/B2.md) with its own entry subtype; that visit is in
[Sequence](#sequence). Search results are files the client reads through
the [profile file service](../file/service.md); their layouts are in
[auction-files.md](../file/auction-files.md). Card tuples and the sale
formula are in [catalogue.md](../client/catalogue.md).

## Framing

A `GTM0A` payload is a sequence of commands, each `<XX>` BEL payload BEL,
where `XX` is a two-letter token from the client's command dictionary and
embedded fields are separated by ACK. The whole payload is at most 469
bytes. In the examples on this page `^G` is BEL (0x07) and `^F` is ACK
(0x06).

| Token | ID | Token | ID | Token | ID |
|---|---|---|---|---|---|
| ER | 0x34 | ES | 0x35 | EF | 0x36 |
| BR | 0x37 | BS | 0x38 | BF | 0x39 |
| SI | 0x3A | SS | 0x43 | SF | 0x44 |
| BH | 0x41 | HS | 0x45 | HF | 0x46 |
| SN | 0x47 | EI | 0x33 | CB | 0x2D |
| AI | 0x31 | SP | 0x28 | BI | 0x29 |
| ET | 0x26 | AM | 0x27 | AC | 0x2A |
| ED | 0x24 | NC | 0x25 | LC | 0x2B |
| CM | 0x2C | CD | 0x2E | BC | 0x2F |
| II | 0x30 | BM | 0x32 | IO | 0x3B |
| IB | 0x40 | MP | 0x3C | LT | 0x3D |
| IC | 0x3F | | | | |

The first command of every request carries the acting player as twelve
ACK-separated fields:

```c
struct AuctionActor {
    uint64_t nn;          // game identity, 0x + 16 hex; must match the authenticated account
    int32_t  character;   // character ID; must match
    int32_t  zero_2;      // purpose unknown (unverified); 0
    uint8_t  zero_3;      // purpose unknown (unverified); 0
    uint16_t zero_4;      // purpose unknown (unverified); 0
    char     name[16];    // character name, at most 16 bytes; must match
    int32_t  zero_6;      // purpose unknown (unverified); 0
    int32_t  zero_7;      // purpose unknown (unverified); 0
    uint16_t zero_8;      // purpose unknown (unverified); 0
    uint16_t zero_9;      // purpose unknown (unverified); 0
    uint8_t  zero_10;     // purpose unknown (unverified); 0
    uint8_t  zero_11;     // purpose unknown (unverified); 0
};
```

```
C2S GTM0A<BH>^G0x0000000000000001^F1^F0^F0^F0^FLocalPlayer^F0^F0^F0^F0^F0^F0^G<AI>^G1^G
```

Decimal 32-bit fields are serialized signed (`%d`); narrower fields are
unsigned. A server echoes the zero fields as zeros. In replies the same
twelve fields describe the seller (`EI`) and the current bidder (`CB`); an
absent bidder is all zeros with an empty name. In the examples below
`<actor>` stands for these twelve fields.

## ER Exhibit

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | auction service | [ES](#es-exhibit-success) or [EF](#ef-exhibit-failure) | `struct AuctionExhibit`, `struct AuctionItem` |

```c
struct AuctionExhibit {           // ER
    struct AuctionActor actor;
    int32_t SP;                   // opening bid, at least 1
    int32_t BI;                   // minimum increase; the UI offers 20, 50, 100, 200, 500, 1000
    uint8_t ET;                   // duration in hours, 1..168
    uint8_t AM;                   // automatic reoffers when unsold, 0..3
    struct AuctionItem II;        // descriptor of the card
};

struct AuctionItem {              // II: sixteen ACK-separated numbers, four 32-bit, four 16-bit, eight 8-bit
    uint32_t model;              // model
    uint32_t sale_value;         // sale value of the card (catalogue formula)
    uint32_t rating;             // seller rating
    uint32_t zero_3;             // purpose unknown (unverified); 0
    uint16_t zero_4;             // 0 (unverified: count)
    uint16_t zero_5;             // 0 (unverified: language)
    uint16_t collection_points;  // seller collection points
    uint16_t zero_7;             // purpose unknown (unverified); 0
    uint8_t  attack;             // attack
    uint8_t  physical;           // physical defense
    uint8_t  type;               // card type
    uint8_t  magical;            // magical defense
    uint8_t  points;             // card points
    uint8_t  arrows;             // arrow mask
    uint8_t  zero_14;            // purpose unknown (unverified); 0
    uint8_t  zero_15;            // purpose unknown (unverified); 0
};
```

```
C2S GTM0A<ER>^G<actor>^G<SP>^G500^G<BI>^G50^G<ET>^G24^G<AM>^G1^G<II>^G<descriptor>^G
```

`<descriptor>` stands for the sixteen fields of `struct AuctionItem`. The
same struct is stored in the listing file records. A server validates that
the model, stats, points and arrows name an owned copy and that the sale
value matches the catalogue; the descriptor is display data and grants
nothing. The client refuses to exhibit with 30 active exhibits or when it
would keep fewer than 5 cards. On a positive `ES` it removes one matching
card locally, so a server must take the card out of the inventory before
replying.

## ES Exhibit success

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | auction service | none | `struct AuctionListing` |

```c
struct AuctionListing {           // ES and BS
    struct AuctionActor EI;       // seller actor
    int32_t AI;                   // auction ID
    int32_t SP;                   // opening bid, as requested
    int32_t BI;                   // minimum increase, as requested
    int32_t ED;                   // created time
    int32_t NC;                   // expiry clock; the client subtracts the current clock for the remaining time
    int32_t LC;                   // last bid time, or created time before any bid
    uint8_t ET;                   // duration in hours, as requested
    uint8_t AM;                   // reoffers allowed, as requested
    int32_t AC;                   // reoffers used so far
    int32_t CM;                   // current amount: the leading bid, or the opening bid before any bid
    int32_t CD;                   // last bid time, 0 before any bid
    int32_t BC;                   // bid count; the client shows the bidder name only when nonzero
    struct AuctionActor CB;       // current bidder actor, zeros when none
    struct AuctionItem II;        // descriptor
};
```

```
S2C GTM0A<ES>^G^G<EI>^G<actor>^G<AI>^G1^G<SP>^G500^G<BI>^G50^G<ED>^G1700000000^G<NC>^G1700086400^G<LC>^G1700000000^G<ET>^G24^G<AM>^G1^G<AC>^G0^G<CM>^G500^G<CD>^G0^G<BC>^G0^G<CB>^G<actor or zeros>^G<II>^G<descriptor>^G
```

The success reply is the full listing.

Server choice:

- Timestamps are Unix seconds.
- `BC` is the number of accepted bids and the revision a bidder must quote.
- The full historical meaning of `ED` and `LC` is not recovered.

## EF Exhibit failure

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | auction service | none | `struct AuctionFailure` |

```c
struct AuctionFailure {           // EF, BF, SF and HF
    int32_t code;                 // error code, see Error codes
};
```

```
S2C GTM0A<EF>^G-8608^G
```

The codes are in [Error codes](#error-codes).

## BR Bid

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | auction service | [BS](#bs-bid-success) or [BF](#bf-bid-failure) | `struct AuctionBid` |

```c
struct AuctionBid {               // BR
    struct AuctionActor actor;
    int32_t AI;                   // auction ID
    int32_t BM;                   // full bid amount
    int32_t BC;                   // bid count the bidder saw in the listing
};
```

```
C2S GTM0A<BR>^G<actor>^G<AI>^G1^G<BM>^G550^G<BC>^G0^G
```

The client suggests `CM + BI` as the next bid. On a positive `BS` the
client deducts the full `BM` from its money; an outbid deposit comes back
later through the collection visit. Server choice: a bid must be at least
the opening bid before the first bid and at least `CM + BI` after it, quote
the current `BC`, and not come from the seller; the previous leading bid is
queued as a refund.

## BS Bid success

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | auction service | none | `struct AuctionListing` |

```c
struct AuctionListing;            // the same layout as ES
```

```
S2C GTM0A<BS>^G^G<EI>^G<actor>^G<AI>^G1^G<SP>^G500^G<BI>^G50^G<ED>^G1700000000^G<NC>^G1700086400^G<LC>^G1700003600^G<ET>^G24^G<AM>^G1^G<AC>^G0^G<CM>^G550^G<CD>^G1700003600^G<BC>^G1^G<CB>^G<actor>^G<II>^G<descriptor>^G
```

The example is constructed from the field list; no capture of it is recorded
(unverified).

The listing after the bid: `CM` is the accepted amount, `BC` counts the
bid, `CB` is the bidder and `LC` and `CD` carry the bid time.

## BF Bid failure

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | auction service | none | `struct AuctionFailure` |

```c
struct AuctionFailure;            // as in EF
```

```
S2C GTM0A<BF>^G-8611^G
```

The codes are in [Error codes](#error-codes).

## SI Search listings

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | auction service | [SS](#ss-search-success) or [SF](#sf-search-failure) | `struct AuctionSearch` |

```c
struct AuctionSearch {            // SI and BH
    struct AuctionActor actor;
    struct {
        char token[2];            // filter token from the table below
        char payload[];           // the filter payload: a name, or ACK-separated numbers
    } filters[];                  // each at most once except IC
};
```

```
C2S GTM0A<SI>^G<actor>^G<MP>^G0^F1000^G<IC>^G0^F120^F120^G
```

SI searches active listings. Filters follow the actor:

| Filter | Payload | Meaning |
|---|---|---|
| IO | name | Seller name, exact |
| IB | name | Bidder name, exact |
| AI | id | One auction |
| MP | lo ACK hi | Current amount range, inclusive |
| LT | lo ACK hi | Remaining whole hours, inclusive |
| IC | selector ACK lo ACK hi | Card selector 0 is model, 10 is type; other selectors are not supported |

Listing results are ordered by auction ID ascending.

Server choice:

- Each result file is an immutable snapshot private to the connection.
- A search matching more than 1000 records fails rather than truncates.
- A server keeps at most 64 snapshots per account.
- The client may delete `U/g/TM0_EXHIBITLIST` or `U/g/TM0_BIDLIST` to clear
  its snapshots.

## SS Search success

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | auction service | none | `struct AuctionResult` |

```c
struct AuctionResult {            // SS and HS
    char    file[];               // path of the result file on the profile file service
    int32_t SN;                   // record count in the file
};
```

```
S2C GTM0A<SS>^GU/g/TM0_EXHIBITLIST_00000001^G<SN>^G3^G
```

The reply names a file and its record count; the client then reads the
file through the profile file service. The record layout is in
[auction-files.md](../file/auction-files.md).

## SF Search failure

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | auction service | none | `struct AuctionFailure` |

```c
struct AuctionFailure;            // as in EF
```

```
S2C GTM0A<SF>^G-8621^G
```

The codes are in [Error codes](#error-codes).

## BH Bid history

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | auction service | [HS](#hs-history-success) or [HF](#hf-history-failure) | `struct AuctionSearch` |

```c
struct AuctionSearch;             // as in SI
```

```
C2S GTM0A<BH>^G<actor>^G<AI>^G1^G
```

BH asks for bid history; the client sends the filter `AI` naming the
auction. Results are ordered by amount descending.

## HS History success

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | auction service | none | `struct AuctionResult` |

```c
struct AuctionResult;             // as in SS
```

```
S2C GTM0A<HS>^GU/g/TM0_BIDLIST_00000002^G<SN>^G2^G
```

The client reads the named file through the profile file service. The
record layout is in [auction-files.md](../file/auction-files.md).

## HF History failure

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| S2C | auction service | none | `struct AuctionFailure` |

```c
struct AuctionFailure;            // as in EF
```

```
S2C GTM0A<HF>^G-8620^G
```

The codes are in [Error codes](#error-codes).

## Sequence

Settled proceeds, returned cards, won cards and refunds are delivered in
one collection visit to the shop service. The client sends
[B0/01 ShReq](G/B0.md#b001-shreq). A server replies, in order, with
[B2/13 Init](G/B2.md#b213-init), [B2/20 Data](G/B2.md#b220-data) carrying
the seller payments, and [B2/21 Card](G/B2.md#b221-card) pages carrying
the returned cards (unsold exhibits).

The client resolves overflow with [Erase](G/B2.md#b210-erase) and
[Decks](G/B2.md#b222-decks) edits (as in the shop) and sends
[B2/15 Get](G/B2.md#b215-get). The server answers the first Get with the
won-card pages, in the same format, and the second Get with the refund
Data and the [B2/17 Quit](G/B2.md#b217-quit) reply, which it sends without
a client Quit. All four stages are sent even when empty.

Server choice:

- A collection reserves the account exclusively until the second Get or
  disconnect.
- Each card stage is committed at its Get with any staged edits.
- Money is credited up to 99,999,999; any residual stays pending for a
  later visit.
- A disconnect keeps unaccepted cards and refunds pending.
- Later settlements wait for the next visit.

Server choice:

- Settlement charges no fees.
- A listing expires at `created + hours * 3600`. While unbid, it is
  reoffered automatically up to `AM` times, for another `hours` each.
- At final expiry an unbid card returns to the seller's queue.
- At final expiry with a bid, the amount moves to the seller's payment
  queue and the card to the bidder's won queue.
- A raised bid by the current leader deposits the full new amount and
  refunds the old one.
- There is no cancel message on the wire, so cancellation is only a trusted
  administrative action that queues the card as returned.

## Error codes

Codes sent in `EF`, `BF`, `SF` and `HF` by a server (which codes the client
maps to a message is unverified):

| Code | Used for |
|---|---|
| -680 | Auction not open, or stale `BC` |
| -8608 | Exhibited card not owned |
| -8609 | Descriptor disagrees with the owned card |
| -8610 | Not enough money |
| -8611 | Bid below the required minimum |
| -8616 | Other refusal |
| -8620 | Unknown auction ID |
| -8621 | Collection or snapshot limit |
