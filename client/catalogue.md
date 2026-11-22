# Card catalogue and packs

The client carries its own card, price and pack data in three compressed
resources: CardPrm.BIN (250 records of 12 bytes), CardPri.BIN (25 records
of 4 bytes) and PackPrm.BIN (78 records of 52 bytes). The decoded data, with
the hash of each decompressed file, is
[economy_catalog.json](economy_catalog.json). No wire message names, prices
or describes a pack: the shop service
([services.md](../irc/services.md#shop-service)), the Prize Center, the
trade service ([services.md](../irc/services.md#trade-service)) and the
auction service ([services.md](../irc/services.md#auction-service)) refer to
this data by model ID and pack ID, and the client reads prices from its own
copy. The arithmetic over card values (points, sale value, growth, type
promotion) is in [rules.md](rules.md).

## Card records

A card model is a CardPrm record, indexed 0..249.

```c
struct CatalogueCard {
    /* 0x00 */ uint8_t  attack;            // baseline attack
    /* 0x01 */ uint8_t  type;              // baseline type 0..3
    /* 0x02 */ uint8_t  physical_defense;  // baseline physical defense
    /* 0x03 */ uint8_t  magical_defense;   // baseline magical defense
    /* 0x04 */ uint8_t  base_card_points;  // points of a card at baseline stats
    /* 0x05 */ uint8_t  price_tier;        // index into the 25 CardPri values
    /* 0x06 */ uint8_t  category_6;        // pack filter key, 0..29
    /* 0x07 */ uint8_t  category_7;        // pack filter key and guild exclusion key, 1..8
    /* 0x08 */ uint16_t unused_08;         // FFFF in every record; not used by selection, generation, points or sale
    /* 0x0A */ uint16_t name_reference;    // name text reference; not used by selection, generation, points or sale
};
```

125 models have type 0, 92 type 1, 21 type 2 and 12 type 3. Model 148
(Hippaul) is the only tier 0 model and is the fallback of pack selection.
Model 65535 is the empty-slot value in card arrays and in Prize Center `LC`
fields; collection points skip it.

## Price tiers

CardPri holds 25 prices, indexed by a model's price tier:

| Tier | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Price | 0 | 25 | 50 | 100 | 250 | 500 | 750 | 1000 | 1500 | 2500 | 5000 | 10000 | 2500 |

| Tier | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Price | 100 | 250 | 500 | 750 | 1000 | 2000 | 5000 | 7500 | 10000 | 12500 | 15000 | 17500 |

The values restart at tier 12: tiers 13..24 form a second ladder used by
every model 150..249 and by four FFIX models (54, 99, 130, 149). The
computer player's placement scoring stores `(price_tier << 4) & 255`, so
tiers 16..24 wrap to nibbles 0..8 there; see
[computer-player.md](computer-player.md#ordinary-move-choice).

## Pack records

A pack is a PackPrm record, indexed 0..77.

```c
struct CataloguePack {
    /* 0x00 */ uint16_t unused_00;         // FFFF, not used
    /* 0x02 */ uint16_t name_reference;    // name text reference, not used
    /* 0x04 */ uint32_t price;             // in T (shop) or points (Prize Center packs 45..74)
    /* 0x08 */ uint8_t  category;          // availability or category, see below
    /* 0x09 */ uint8_t  card_count;        // cards generated per purchase, 1..10
    /* 0x0A */ uint16_t unused_0A;         // 0000, not used
    /* 0x0C */ struct {
                   uint8_t price_tier;     // price tier filter, 255 is any
                   uint8_t category_6;     // category 6 filter, 0 is any
                   uint8_t category_7;     // category 7 filter, 0 is any
                   uint8_t threshold;      // inclusive roll threshold, 0 marks an unused bin
               } bins[4];                  // generation bins
    /* 0x1C */ uint16_t range_first;       // first model
    /* 0x1E */ uint16_t range_end;         // end model, exclusive
    /* 0x20 */ struct {
                   uint16_t model;         // 9999 marks an unused entry
                   uint8_t  threshold;     // inclusive roll threshold, 0 marks an unused entry
                   uint8_t  unused;        // not used
               } explicit_models[5];       // explicit models
};
```

The category byte says where the pack is offered; the shop Init reply may
rewrite it (see [shop.md](../flows/shop.md)):

| Category | Meaning |
|---|---|
| 0 | Hidden: starter and computer opponent decks, the Chance Card |
| 1 | Ordinary purchasable pack |
| 2 | Pauper's Pack: visible only with fewer than 5 cards and less than 1000 T |
| 3 | Visible special pack (pack 5 when Init `SP=3`) |
| 4 | Highlighted but unavailable (packs 76, 77; `HD` indexes with `CMP=0`) |
| 5 | Prize Center exchange, paid in points |
| 7, 8, 9 | Guild pack for Warriors (guild 1), Voyagers (guild 2), Summoners (guild 3) |

The full table. "Selection" lists the bins in order as "filter <= threshold",
then explicit models. "tier n" is a price tier filter, "c6" and "c7" are the
category filters, "any" is tier filter 255 with no category filter.

| ID | Name | Price | Category | Cards | Range | Selection |
|---|---|---|---|---|---|---|
| 0 | Pauper's Pack | 0 | 2 | 5 | 0..149 | tier 0 c6=1 <=254; tier 23 c6=15 <=255; any <=255 |
| 1 | Beginner's Pack | 1000 | 1 | 10 | 0..149 | tier 1 <=100; tier 2 <=200; tier 3 <=250; tier 5 <=255 |
| 2 | Standard Pack | 1500 | 1 | 5 | 0..149 | tier 3 <=101; tier 4 <=240; tier 7 <=245; tier 6 <=255 |
| 3 | Collector's Pack | 9000 | 1 | 5 | 0..149 | tier 4 <=75; tier 5 <=175; tier 6 <=250; tier 9 <=255 |
| 4 | Hero's Pack | 30000 | 1 | 3 | 0..149 | tier 7 <=90; tier 8 <=180; tier 9 <=230; tier 10 <=255 |
| 5 | rank_1 Pack | 100000 | 3 | 3 | 0..149 | tier 10 <=5; tier 11 <=105; any <=255 |
| 6 | Starter's Pack | 0 | 0 | 10 | 0..149 | tier 1 <=255; any <=255 |
| 7 | COM's Lv. 1 Pack | 0 | 0 | 10 | 0..149 | tier 1 <=254; tier 4 <=255; any <=255 |
| 8 | COM's Lv. 2 Pack | 0 | 0 | 10 | 0..149 | tier 2 <=248; tier 3 <=253; tier 4 <=255; any <=255 |
| 9 | COM's Lv. 3 Pack | 0 | 0 | 10 | 0..149 | tier 3 <=251; tier 4 <=255; any <=255 |
| 10 | Pilot Erin | 0 | 0 | 10 | 0..149 | tier 12 c6=4 <=255; any <=255 |
| 11 | Vivi | 0 | 0 | 10 | 0..149 | tier 7 c6=2 <=50; any c6=22 <=100; tier 9 <=199; any c6=16 <=200; model 127 <=255 |
| 12 | Regent Cid | 0 | 0 | 10 | 0..149 | tier 9 <=30; tier 10 <=80; any c6=12 c7=4 <=198; any c6=27 <=200; model 112 <=255 |
| 13 | Trade King Gohn | 0 | 0 | 10 | 0..149 | tier 7 <=50; tier 8 <=100; any c6=3 c7=3 <=200; tier 8 c7=3 <=255 |
| 14 | Stiltzkin | 0 | 0 | 10 | 0..149 | any c6=3 c7=5 <=150; tier 5 <=220; tier 6 <=254; any c6=25 <=255 |
| 15 | Garnet | 0 | 0 | 10 | 0..149 | any c6=2 c7=2 <=100; any c6=29 <=158; tier 8 <=168; any c6=11 <=170; model 107 <=225; model 111 <=255 |
| 16 | Amarant | 0 | 0 | 10 | 0..149 | tier 4 c6=3 <=120; tier 5 c6=3 <=170; tier 6 c6=3 <=200; any c6=21 <=255 |
| 17 | Steiner | 0 | 0 | 10 | 0..149 | tier 2 c6=3 <=70; tier 3 c6=3 <=170; any c6=28 <=220; any c6=23 <=221; model 108 <=255 |
| 18 | Zidane | 0 | 0 | 10 | 0..149 | tier 7 <=140; any c6=6 <=200; tier 8 c6=9 <=252; any c6=10 <=255 |
| 19 | Kuja | 0 | 0 | 10 | 0..149 | tier 7 <=130; any c7=7 <=170; any c6=18 <=254; any c6=17 <=255 |
| 20 | Premium Pack | 14000 | 1 | 3 | 0..149 | tier 6 <=175; tier 7 <=230; tier 9 <=255; any <=255 |
| 21 | Summoner's Pack | 6500 | 9 | 3 | 0..149 | any c6=2 c7=2 <=255; any <=255 |
| 22 | Warrior's Pack | 3000 | 7 | 3 | 0..149 | tier 3 c7=3 <=60; tier 4 c7=3 <=120; tier 5 c7=3 <=200; any c6=3 c7=3 <=255 |
| 23 | Voyager's Pack | 4500 | 8 | 3 | 0..149 | any c6=12 c7=4 <=255; any <=255 |
| 24 | FFIX Celebrity Pack | 20000 | 1 | 3 | 0..149 | tier 6 c7=7 <=100; tier 7 c7=7 <=200; any c7=7 <=255; any <=255 |
| 25 | Natasha66/0 | 0 | 0 | 10 | 0..149 | tier 1 <=254; tier 7 <=255; any <=255 |
| 26 | Mick83/20 | 0 | 0 | 10 | 0..149 | tier 1 <=170; tier 2 <=150; tier 3 <=151; tier 7 <=152; model 33 <=200; model 13 <=255 |
| 27 | Quina97/40 | 0 | 0 | 10 | 0..149 | tier 2 <=50; tier 3 <=80; any c6=20 <=150; tier 1 <=160; model 91 <=245; model 114 <=255 |
| 28 | Bogart105/40 | 0 | 0 | 10 | 0..149 | tier 1 <=100; tier 2 <=190; tier 3 <=250; tier 7 <=255 |
| 29 | Agnes140/100 | 0 | 0 | 10 | 0..149 | tier 1 <=30; tier 2 <=160; tier 3 <=250; tier 7 <=255 |
| 30 | Brahne254/150 | 0 | 0 | 10 | 0..149 | tier 2 <=110; tier 3 <=120; tier 4 <=139; tier 11 c6=13 <=140; model 121 <=175; model 122 <=210; model 100 <=255 |
| 31 | Mina276/200 | 0 | 0 | 10 | 0..149 | tier 2 <=100; tier 3 <=190; tier 4 <=250; tier 7 <=255 |
| 32 | Bill334/250 | 0 | 0 | 10 | 0..149 | tier 2 <=30; tier 3 <=160; tier 4 <=250; tier 7 <=255 |
| 33 | Akbar360/300 | 0 | 0 | 10 | 0..149 | tier 3 <=130; tier 4 <=220; tier 5 <=250; tier 7 <=255 |
| 34 | Jill509/400 | 0 | 0 | 10 | 0..149 | tier 3 <=30; tier 4 <=160; tier 5 <=250; tier 8 <=255 |
| 35 | Fabius612/500 | 0 | 0 | 10 | 0..149 | tier 4 <=110; tier 5 <=170; tier 6 <=195; tier 8 <=197; model 11 <=255 |
| 36 | Broidy727/620 | 0 | 0 | 10 | 0..149 | tier 5 <=130; tier 6 <=220; tier 7 <=253; tier 11 c6=8 <=255 |
| 37 | Lani750/690 | 0 | 0 | 10 | 0..149 | tier 5 <=80; tier 6 <=150; tier 7 <=195; tier 8 <=200; model 123 <=255 |
| 38 | Eiko873/760 | 0 | 0 | 10 | 0..149 | any c6=2 c7=2 <=70; tier 5 c6=3 c7=5 <=190; any c6=24 <=244; any c6=26 <=245; model 118 <=255 |
| 39 | Pierre1016/900 | 0 | 0 | 10 | 0..149 | tier 6 <=90; tier 7 <=150; tier 8 <=195; tier 9 <=200; model 103 <=255 |
| 40 | Freya1128/100 | 0 | 0 | 10 | 0..149 | tier 7 <=100; tier 8 <=130; tier 9 <=150; any c6=19 <=205; model 56 <=255 |
| 41 | Carol1210/1100 | 0 | 0 | 10 | 0..149 | tier 7 <=150; tier 8 <=200; tier 9 <=255; any <=255 |
| 42 | Artania1789/1600 | 0 | 0 | 10 | 0..149 | tier 7 <=40; tier 7 c7=6 <=100; tier 8 <=254; any c6=13 <=255 |
| 43 | Beatrix1870/1700 | 0 | 0 | 10 | 0..149 | tier 7 <=40; tier 8 <=130; any c6=7 <=250; tier 10 <=255 |
| 44 | Tot1930/1800 | 0 | 0 | 10 | 0..149 | tier 8 <=90; tier 9 <=230; tier 10 <=253; any c6=14 <=255 |
| 45 | Prize Center Card 1 | 50 | 5 | 1 | 0..149 | model 198 <=255 |
| 46 | Prize Center Card 2 | 60 | 5 | 1 | 0..149 | model 197 <=255 |
| 47 | Prize Center Card 3 | 70 | 5 | 1 | 0..149 | model 167 <=255 |
| 48 | Prize Center Card 4 | 85 | 5 | 1 | 0..149 | model 172 <=255 |
| 49 | Prize Center Card 5 | 100 | 5 | 1 | 0..149 | model 202 <=255 |
| 50 | Prize Center Card 6 | 125 | 5 | 1 | 0..149 | model 175 <=255 |
| 51 | Prize Center Card 7 | 150 | 5 | 1 | 0..149 | model 199 <=255 |
| 52 | Prize Center Card 8 | 175 | 5 | 1 | 0..149 | model 188 <=255 |
| 53 | Prize Center Card 9 | 175 | 5 | 1 | 0..149 | model 187 <=255 |
| 54 | Prize Center Card 10 | 200 | 5 | 1 | 0..149 | model 186 <=255 |
| 55 | Prize Center Card 11 | 230 | 5 | 1 | 0..149 | model 182 <=255 |
| 56 | Prize Center Card 12 | 250 | 5 | 1 | 0..149 | model 133 <=255 |
| 57 | Prize Center Card 13 | 300 | 5 | 1 | 0..149 | model 128 <=255 |
| 58 | Prize Center Card 14 | 300 | 5 | 1 | 0..149 | model 151 <=255 |
| 59 | Prize Center Card 15 | 350 | 5 | 1 | 0..149 | model 154 <=255 |
| 60 | Prize Center Card 16 | 375 | 5 | 1 | 0..149 | model 159 <=255 |
| 61 | Prize Center Card 17 | 425 | 5 | 1 | 0..149 | model 193 <=255 |
| 62 | Prize Center Card 18 | 500 | 5 | 1 | 0..149 | model 150 <=255 |
| 63 | Prize Center Card 19 | 525 | 5 | 1 | 0..149 | model 97 <=255 |
| 64 | Prize Center Card 20 | 525 | 5 | 1 | 0..149 | model 99 <=255 |
| 65 | Prize Center Card 21 | 550 | 5 | 1 | 0..149 | model 120 <=255 |
| 66 | Prize Center Card 22 | 625 | 5 | 1 | 0..149 | model 222 <=255 |
| 67 | Prize Center Card 23 | 700 | 5 | 1 | 0..149 | model 160 <=255 |
| 68 | Prize Center Card 24 | 775 | 5 | 1 | 0..149 | model 161 <=255 |
| 69 | Prize Center Card 25 | 800 | 5 | 1 | 0..149 | model 185 <=255 |
| 70 | Prize Center Card 26 | 1250 | 5 | 1 | 0..149 | model 211 <=255 |
| 71 | Prize Center Card 27 | 2000 | 5 | 1 | 0..149 | model 54 <=255 |
| 72 | Prize Center Card 28 | 3000 | 5 | 1 | 0..149 | model 215 <=255 |
| 73 | Prize Center Card 29 | 5000 | 5 | 1 | 0..149 | model 149 <=255 |
| 74 | Prize Center Card 30 | 10000 | 5 | 1 | 0..149 | model 152 <=255 |
| 75 | Chance Card | 0 | 0 | 1 | 216..220 | any <=255 |
| 76 | FFX Blitzball Edition | 300000 | 4 | 3 | 150..249 | any c6=1 <=254; model 153 <=255 |
| 77 | FFX Monster Edition | 300000 | 4 | 4 | 150..249 | tier 13 c6=2 c7=1 <=104; tier 14 c7=1 <=199; tier 15 c7=1 <=251; model 201 <=255 |

Pack 75 (Chance Card) is generated by the table service for the F18 special
tile, not sold; see [board.md](board.md#field-codes). Packs 7..9
(`COM's Lv. 1 Pack` to `COM's Lv. 3 Pack`) are the decks of the unnamed
computer opponents, and packs 10..19 and 25..44 the decks of the named
computer opponents
([computer-player.md](computer-player.md#opponent-roster)).

## Card values on the wire

The shop service, the Prize Center, the trade service and the auction
collection carry a card as `|`-separated decimals in a `/D=` field. Shop and
Prize Center Card replies, event CardSelect and EcmInit use the eight-field
`struct Card`. Erase, Sell, trade offers and auction card pages use
`struct CardReduced`, and Decks appends the deck slot to it. Both are
defined in [rules.md](rules.md#card-record).

Arrows are an 8-bit mask, one bit per direction. Deck 255 means no deck. A
server never accepts points, sale value or price from the client; it
recomputes them from the model and stats with the formulas in
[rules.md](rules.md). Reduced records carry no instance ID, so a server
matches an edit against owned copies by value and consumes one copy per
record.

Server choice: deck slots 0..49 are accepted.

## Pack selection

Inputs: a pack `p`, a guild `g` and `draw()`, the 15-bit random value of
[rules.md](rules.md#crt-rand). Result: one model.

1. Repeat at most 256 times:
   1. `roll = draw() & 255`.
   2. Take the first bin with nonzero threshold and `roll <= threshold`.
   3. If no bin matches, take the first explicit entry with nonzero threshold,
      `roll <= threshold` and model < 250; return that model. Otherwise
      continue the loop.
   4. Candidates are the models in the pack's range whose category 7 is not in
      the guild's exclusion set and which match the bin's tier filter (255 is
      any) and nonzero category 6 and category 7 filters.
   5. If any candidate exists, return `candidates[draw() % count]`; otherwise
      continue.
2. After 256 failed rounds return model 148.

Guild exclusion sets by category 7:

| Guild | Excluded category 7 |
|---|---|
| 0 (unregistered) | none |
| 1 Warriors | 2, 4, 6 |
| 2 Voyagers | 2, 3 |
| 3 Summoners | 3, 4, 6 |

Bins are tested in record order, so a later bin whose threshold is below an
earlier one (pack 26) is unreachable.

## Card generation

Inputs: a model with baselines bA, bT, bP, bM and `draw()`. Result: a card
record with the stats below, points from [rules.md](rules.md#points) and
deck slot 255.

```
attack   = bA/2 + (draw() % bA)/2        (integer divisions)
type     = 2 if draw() % 256 < 2 else bT
physical = bP/2 + (draw() % bP)/2
magical  = bM/2 + (draw() % bM)/2
```

Arrows: for model 148, one arrow, `1 << (draw() % 8)`. Otherwise
`count = table[draw() % 100]` where the 100-entry table yields counts
0..8 with frequencies 1, 4, 16, 31, 27, 10, 6, 3, 2, in ascending order; then
`count` times remove `pool[draw() % len(pool)]` from the pool of the eight
directions and set that bit.

## Starting inventory

The ranked and computer opponent packs are never sold.

Server choice: a new account holds 10,000 T and ten cards generated from the
Starter's Pack (pack 6) with its guild's exclusion set; an account that
registers from guild 0 receives them at registration.
