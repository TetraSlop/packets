# Computer player

The client carries a full local computer player: a roster of 30 named
opponents, a hand builder and a move chooser. Online, the client does not
run it: the table service makes every computer decision and announces moves
with the same [PutCard](../irc/G/43.md#430a-putdata-and-putcard) and
[BattleData](../irc/G/43.md#430c-battledata) as a human's. The data and
routines below are what a server reproduces; the computer menu messages are
[43/02 ComGameInit](../irc/G/43.md#4302-comgameinit),
[43/04 ComList](../irc/G/43.md#4304-comlist) and
[43/06 ComGame](../irc/G/43.md#4306-comgame).

## Opponent roster

PlPrm.BIN decompresses to 2964 bytes: a header of four DWORDs
`636, 31, 15, 0`, then 31 records of 20 bytes, record 0 unused.

```c
struct OpponentRecord {                 // PlPrm.BIN record, 20 bytes
    /* 0x00 */ uint16_t points;         // initial points shown for the opponent, not a running total
    /* 0x02 */ uint16_t threshold;      // collection points below which the client shows the hidden name instead of the real one
    /* 0x04 */ uint32_t rating;         // rating shown for the opponent; ComList `D` replaces it
    /* 0x08 */ uint8_t  score_chance;   // percent chance that a placement is scored
    /* 0x09 */ uint8_t  preference;     // scoring preference 0..4, see Ordinary move choice
    /* 0x0A */ uint8_t  preference_chance; // percent chance that the preference is applied
    /* 0x0B */ uint8_t  pack;           // catalogue pack the opponent's hand is generated from
    /* 0x0C */ uint32_t name;           // name text reference
    /* 0x10 */ uint32_t flavour;        // flavour text reference
};
```

| ID | Points | Threshold | Rating | score_chance, preference, preference_chance | Pack | Name |
|---|---|---|---|---|---|---|
| 1 | 0 | 0 | 300 | 65, 1, 50 | 25 | Flower Girl Natasha |
| 2 | 100 | 10 | 293 | 67, 3, 51 | 26 | Lumberjack Mick |
| 3 | 135 | 20 | 290 | 69, 1, 48 | 27 | Quina |
| 4 | 175 | 30 | 289 | 70, 2, 45 | 28 | Bogart the Gladiator |
| 5 | 190 | 50 | 282 | 70, 3, 42 | 29 | Agnes |
| 6 | 205 | 75 | 278 | 73, 2, 40 | 30 | Queen Brahne |
| 7 | 220 | 100 | 272 | 75, 4, 39 | 31 | Puppeteer Mina |
| 8 | 245 | 150 | 267 | 76, 3, 37 | 32 | Bartender Bill |
| 9 | 260 | 200 | 259 | 77, 1, 36 | 33 | Akbar the Hunter |
| 10 | 295 | 250 | 250 | 78, 2, 35 | 17 | Steiner |
| 11 | 310 | 300 | 199 | 80, 2, 34 | 34 | Jill |
| 12 | 345 | 350 | 192 | 82, 3, 33 | 14 | Stiltzkin |
| 13 | 360 | 400 | 189 | 83, 1, 32 | 35 | Fabius the Pirate |
| 14 | 400 | 450 | 182 | 85, 2, 31 | 16 | Amarant |
| 15 | 425 | 500 | 177 | 86, 3, 30 | 36 | Gardener Broidy |
| 16 | 455 | 550 | 170 | 87, 4, 29 | 37 | Lani |
| 17 | 470 | 600 | 169 | 88, 3, 28 | 38 | Eiko |
| 18 | 535 | 650 | 162 | 89, 2, 27 | 18 | Zidane |
| 19 | 560 | 700 | 156 | 90, 1, 26 | 39 | Chef Pierre |
| 20 | 590 | 750 | 149 | 92, 3, 25 | 40 | Freya |
| 21 | 615 | 800 | 145 | 93, 2, 20 | 41 | Revolutionary Carol |
| 22 | 675 | 850 | 143 | 94, 3, 19 | 11 | Vivi |
| 23 | 730 | 900 | 138 | 94, 2, 17 | 13 | Trade King Gohn |
| 24 | 785 | 950 | 137 | 95, 2, 16 | 19 | Kuja |
| 25 | 840 | 1000 | 135 | 97, 3, 15 | 15 | Garnet |
| 26 | 865 | 1050 | 131 | 98, 3, 12 | 42 | Artania |
| 27 | 895 | 1100 | 129 | 100, 2, 8 | 43 | Beatrix |
| 28 | 910 | 1250 | 126 | 100, 3, 5 | 44 | Doctor Tot |
| 29 | 950 | 1500 | 111 | 100, 2, 3 | 10 | Pilot Erin |
| 30 | 1000 | 2000 | 100 | 100, 0, 0 | 12 | Regent Cid |

The rating column is what [43/04 ComList](../irc/G/43.md#4304-comlist)
carries: `D[i]` replaces the rating of record `i+1`, so a server publishing
the defaults sends these 30 values in order. Several names are 19 bytes,
longer than the 16-byte `CN` slots of VsGameInit, so the computer menu never
goes through VsGameInit; the client takes the names from PlPrm itself.

## Hand construction

Inputs: the selected opponents in order, a guild filter 0..3 and raw CRT
draws ([rules.md](rules.md#crt-rand)). Result: a hand of five cards per
opponent.

1. Pool of 10: for each selected opponent in order, generate ten cards from
   that opponent's pack with [pack selection](catalogue.md#pack-selection)
   and [card generation](catalogue.md#card-generation). The guild filter
   selects the exclusion set `()`, `(2,4,6)`, `(2,3)` or `(3,4,6)`. All
   opponents' pools are generated before any hand is chosen.
2. Hand of 5: per opponent, pool index 0 is always kept, then four draws
   `rand() % remaining` remove cards from the shrinking pool (9, 8, 7, 6
   entries) without replacement.

A run from seed 57 for opponent 1 with guild 0 uses 103 generation draws
and 4 selection draws. Generating and selecting each opponent in turn gives
a different stream from pool-then-pool-then-select, so two-opponent games
must follow the order above. Whether the original service used this
builder is unknown; the online client never runs it.

Server choice: which guild filter to pass; the human's own guild is one
option.

## Pacing polls

Inputs: a denominator and one draw. Result: whether a decision is made on
this poll.

Every computer decision starts with a rate test that consumes one draw:
`rand() % denominator`, denominator 60 on the client's frame timer; a
remainder of 2 or more means no decision this poll and no further draws.
A server that decides immediately uses denominator 1, which always passes
but still consumes the draw.

Draw counts per successful decision: hand choice 2; placement 3 without
scoring, 4 with scoring; target choice with several candidates 2; sole
target 0. The default immediate move (hand then placement with scoring)
therefore uses 6 draws.

## Ordinary move choice

Inputs: the computer's hand, the board, the opponent's `score_chance`,
`preference` and `preference_chance`, and raw CRT draws. Result: the hand
card and the cell of the PutCard.

Hand: after the rate test, `rand() % remaining_hand_count`. Card stats are
not examined.

Placement:

1. Rate test.
2. Score test: `rand() % 100 < score_chance` (100..255 always passes).
3. If scoring, preference test: `rand() % 100 < preference_chance`;
   failure uses preference 0 but still scores.
4. Candidates: every owner-8 cell in ascending index.
5. If scoring, compute one score per candidate and keep every candidate
   equal to the maximum, in ascending order.
6. Choose `rand() % count` from the kept list.

Scoring a candidate cell: clear 16 marker bytes, then follow the hand
card's arrows. An opposing card with no arrow back gets bit 1 (direct
capture). An opposing card with an arrow back gets bit 2 (battle) and, one
hop on, every card of the same owner that it points at gets bit 4
(possible combo). Marks OR together. Every marked card also stores the
catalogue price tier shifted into the high nibble, `(tier << 4) & 0xFF`,
so tiers 16..24 wrap to 0..8. With `A` = cells with any low bit, `B` =
cells with bit 2, `C` = cells with bit 4 and `T` = the largest stored
tier nibble:

| Preference | Score |
|---|---|
| 0 or 5..255 | `A` |
| 1 | `16 + T` if `T > 1`, else `A` |
| 2 | `16 + B` if `B > 0`, else `A` |
| 3 | for every board cell without bit 2: 2, plus 1 if bit 1 is set (an unmarked board scores 32) |
| 4 | `16 + C` if `C > 0`, else `A` |

No battle strength is compared; the score counts opportunities only.

## Special-board move choice

Inputs: as for the ordinary move choice, on a board with any nonzero field
code or on every 5x5 board. Result: the hand card and the cell of the
PutCard.

Placement uses the special evaluator instead. It includes every owner-8 cell
(low tiles too), treats a card placed on F5 as having all arrows, honours F5
on opponent cards and combo hops, and predicts a spinner's ray for exactly
two wrapping steps, replacing an existing marker with a direct-capture mark
where the ray would hit. It uses the player count rather than battle
strength and ignores spinner chains. The rate, chance, preference and
maximum-tie draws follow the ordinary order.

## Target choice

Inputs: the reciprocal targets of the placed card in ascending cell order
and raw CRT draws. Result: the target of the BattleSelect.

With zero or one reciprocal target the client skips selection: no draw, one
target goes straight to battle. With two or more, after the rate test the
choice is `rand() % count` into the candidate list, on both 16 and 25 cell
boards.

## Two opponents

The computer menu offers one or two opponents. Choosing two sets the player
count to 3 and the 25-cell board with 15 placements; re-selecting the first
opponent clears that slot, so both IDs are always distinct. The
[43/06 ComGame](../irc/G/43.md#4306-comgame) argument is the player count
(2 or 3) and `/Com=` carries one or two IDs. ComGameInit and ComList have no
three-player fields; the client decides the count before CardSelect.

```
C2S GTM0G43000602@ComGame=/Rule=0|0|0|0|2|0|0|1/Com=1
C2S GTM0G43000603@ComGame=/Rule=0|0|0|0|2|0|0|1/Com=1|3
C2S GTM0G43000603@ComGame=/Rule=0|0|0|0|2|0|0|1/Com=30|1
```

`Rule` is the eight rule values `bm|st|cb|ca|tl|du|view|gs` of ComGame and
`Com` the opponent IDs 1..30.

Each computer seat keeps its own ID, personality and hand. A rematch
([Continue](../irc/G/43.md#4311-continue) `Conf=0`) rebuilds every computer
hand; `Conf=1` returns to the menu, where the count may change.
