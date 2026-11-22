# Card bytes and rule arithmetic

A card is eight values that the client stores, sends in CardSelect, PutCard
and the shop, trade and auction messages, and recomputes after every battle.
This page gives the card record in its wire and stored forms, the random
generator every match draw comes from, and the arithmetic a server
reproduces for points, battles, growth, sale value, collection points and
result class. The per-model baselines these formulas use are in
[catalogue.md](catalogue.md#card-records).

## Card record

A card on the wire is eight decimals in this order. The same values, as
bytes, make up the client's 16-byte stored card.

```c
struct Card {
    uint16_t model;     // 0..249, catalogue row (CardPrm.BIN); 0xFFFF is an empty slot
    uint8_t  attack;    // 0..255
    uint8_t  type;      // 0..3: 0 P (physical), 1 M (magical), 2 X, 3 A
    uint8_t  physical;  // 0..255, physical defence
    uint8_t  magical;   // 0..255, magical defence
    uint8_t  points;    // 0..255, derived, see Points
    uint8_t  arrows;    // bit mask N=1, NE=2, E=4, SE=8, S=16, SW=32, W=64, NW=128
    uint8_t  deck;      // deck slot 0..49, 255 when unassigned
};
```

The full record is `D=` in CardSelect, PutCard and CBC, the shop and Prize
Center Card replies, event CardSelect and EcmInit. Edits and listings carry a
reduced record without points or deck:

```c
struct CardReduced {
    uint16_t model;     // catalogue model
    uint8_t  attack;    // attack
    uint8_t  type;      // type 0..3
    uint8_t  physical;  // physical defence
    uint8_t  magical;   // magical defence
    uint8_t  arrows;    // arrow mask
};
```

TCList, Erase, Sell, trade offers and auction card pages use it as is; Decks
appends the deck slot as a seventh value.

Stored card, 16 bytes:

```c
struct StoredCard {
    /* 0x00 */ uint8_t  attack;   // attack
    /* 0x01 */ uint8_t  type;     // type 0..3
    /* 0x02 */ uint8_t  physical; // physical defence
    /* 0x03 */ uint8_t  magical;  // magical defence
    /* 0x04 */ uint8_t  points;   // card points, see Points
    /* 0x05 */ uint8_t  arrows;   // arrow mask
    /* 0x06 */ uint16_t model;    // catalogue model
    /* 0x08 */ uint32_t sale;     // sale value, see Sale value
    /* 0x0C */ uint16_t index;    // position in the client's card list
    /* 0x0E */ uint8_t  deck;     // deck slot, 255 when unassigned
    /* 0x0F */ uint8_t  state;    // purpose unknown (unverified)
};
```

The baseline per model is `struct CatalogueCard` in
[catalogue.md](catalogue.md#card-records); a generated card starts at half
to full baseline per stat ([catalogue.md](catalogue.md#card-generation)).

## CRT rand

The client uses the Microsoft CRT generator for every match draw:

```
state = (state * 214013 + 2531011) mod 2^32
rand  = (state >> 16) & 0x7FFF
```

Values are 0..32767.

Server choice: the seed and the draw schedule, since the original service's
are unknown.

## Points

Inputs: a card's attack, physical and magical and its model's baselines.
Result: the points byte.

```
points = ((attack + physical + magical) * base_points
          / (base_attack + base_physical + base_magical)) & 0xFF
```

Integer division. The client recomputes points when it parses a final card
list, so a server does not need to trust a received value.

## Battle

Inputs: the attacker (the placed card), the defender, the two sides' signed
modifiers from F6 and F7 tiles, and the two cells' properties, whose low
nibble is 1 for an attacker on F3 and 2 for a defender on F4
([board.md](board.md#field-codes)). Run once for each reciprocal target.
Result: an attacker win, a defender win or a tie, and the two finals the
BattleData carries.

1. Choose the compared stats by attacker type:
   - 0: attacker attack against defender physical.
   - 1: attacker attack against defender magical.
   - 2: attacker attack against the smaller defender defence.
   - 3: attacker's largest of attack/physical/magical against the
     defender's smallest of the three.
   Equal stats prefer magical, then physical, then attack. The chosen stat
   sets the selector bit 8, 4 or 1.
2. Add the side's signed modifier; clamp the result to at least 1. This is
   `start`.
3. Multiplier: attacker property low nibble 1, or defender low nibble 2,
   each independently draws `rand() % 4 + 2`; otherwise 1.
4. Four draws `r0..r3`. With `m = start * multiplier + 1`:
   `final_A = ((r0 & 0xFFFF) % m_A + (r1 & 0xFFFF) % m_A) / 2`, and the
   same for the defender with `r2, r3`.
5. If both finals are 0, two more draws replace them with `rand() & 1`.
   There is no further reroll.

The attacker wins on a strictly greater final and the defender on a
strictly smaller one. Equal finals are a tie, reported as such, and the
server sends another BattleData for the same target.

## Growth

Inputs: a card, its catalogue baseline, a mode and one random byte `R`.
Result: the card with its type or one stat changed, and its points and sale
value recomputed.

Mode 0 (after every decisive battle, on the winning card, with `R[0]` of
that BattleData): type 0 or 1 becomes 2 when `R < 4`; type 2 becomes 3 when
`R < 1`; type 3 is unchanged.

Mode 1 (at result, for cards of seats with result class below 3): pick
attack, physical or magical by `R % 3` and add 1 only while that stat is
below the catalogue baseline. Values already above the baseline are kept.

Both modes recompute points and sale value even without a stat change.
The result-phase `R` index is `R[(hand_slot - 3*seat + 2) & 7]` where
`hand_slot` is the position in the hand rebuilt from the board (see
[43/1A TCList](../irc/G/43.md#431a-tclist)). Losers keep battle promotions
but get no stat increment.

## Sale value

Inputs: a card's stats, its model's baselines and type, and the model's
price tier value from [catalogue.md](catalogue.md#price-tiers). Result: the
sale value in T.

```
sale = floor(( attack/base_attack + physical/base_physical
             + magical/base_magical + max(type,1)/2 ) * price_tier_value / 4)
```

The client computes this in x87 arithmetic; the rational form above agrees
with every recorded case. Every model has positive baselines (all 250), so
no division by zero occurs.

## Collection points

Inputs: a card list. Result: the `CP` value shown in Init replies and
compared by table eligibility, summed as a WORD.

```
sum over models of the best points held for that model
+ sum over cards of max(type - 1, 0)
+ 5 * number of distinct arrow masks
```

Slots with model 0xFFFF are skipped.

## Result class

Inputs: each seat's score at the end of the round, the number of board cards
it owns. Result: a class per seat.
`class = 2 + (seats with higher score) - (seats with lower score)`;
class 1 is a sole or shared high, 2 the middle, 3 the low. Class below 3
qualifies for mode 1 growth. The full three-player table is in
[ratings-and-statistics.md](ratings-and-statistics.md#result-classes) and
the departure variants in [departures.md](../flows/departures.md).
