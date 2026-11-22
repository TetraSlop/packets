# Board and special tiles

The board is 4x4 (16 cells) for two players and 5x5 (25 cells) for three.
[StartData](../irc/G/43.md#4308-startdata) `F` lists one field code per
cell; code 0 is an empty cell and 1..18 are the special tiles below. The
client's ordinary generator draws walls even when every special rule is off,
so a server that wants an empty board sends an all-zero `F` explicitly. The
cell numbering, the tile effects and the generator a server reproduces for
StartData and the first [TurnData](../irc/G/43.md#4309-turndata) are given
here.

## Cell numbering

Cells are numbered row-major from the top-left: `index = y*width + x`.
Neighbours do not wrap. Arrow bits on a card are N=1, NE=2, E=4, SE=8,
S=16, SW=32, W=64, NW=128, with N pointing to `index - width`.

```
4x4            5x5
 0  1  2  3     0  1  2  3  4
 4  5  6  7     5  6  7  8  9
 8  9 10 11    10 11 12 13 14
12 13 14 15    15 16 17 18 19
               20 21 22 23 24
```

Cell 0 has neighbours E=1, SE=5, S=4 on a 4x4 board; cell 3 has no east
neighbour.

## Field codes

Every nonzero code becomes a cell with model `0x8000+F` (9..16 share
`0x8009`, 17 is `0x800A`, 18 is `0x800B`), base owner 4, and the property
and arrows below. Owner 8 means a card may be placed there; owner 4 is a
tile that cards interact with through arrows.

| F | Tile | Property | Owner | Arrows | Effect |
|---|---|---|---|---|---|
| 0 | empty | 0 | 8 | 0 | ordinary cell |
| 1, 2 | wall | 0 | 4 | 0 | blocks the cell for the whole game |
| 3 | low tile 1 | 1 | 8 | 0 | card placed here gets an attack multiplier roll; consumed at turn end if the property is exactly 1 |
| 4 | low tile 2 | 2 | 8 | 0 | card placed here gets a defence multiplier roll; stays on the card |
| 5 | low tile 3 | 3 | 8 | 0 | card placed here has all eight arrows for targeting and combos; stays |
| 6 | minus | 0x10 | 4 | 255 | every card the active seat owns gets modifier -10 (signed byte, wraps); tile removed |
| 7 | plus | 0x20 | 4 | 255 | same with +10; tile removed |
| 8 | ownership | 0x30 | 4 | 255 | `R[1]` picks a beneficiary, then every player-owned card adjacent to the tile flips to it without arrow checks; tile removed |
| 9..16 | spinner | 0 | 4 | `1<<(F-9)` | direction `F-9`; see below |
| 17 | shuffle | 0x40 | 4 | 255 | all ordinary cards are redistributed; tile removed, combat ends |
| 18 | CBC | 0x50 | 4 | 255 | tile is replaced by a freshly generated card owned by the active seat (base owner 5); see [results-and-prizes.md](../flows/results-and-prizes.md) |

Only the low nibble of a property reaches the battle calculation; the high
nibble selects the tile effect. A tile is targeted like a battle: it gets
its own BattleData with selectors 0, and the effect applies after that
packet. Tiles 6, 7, 8 and 17 appear with `cb=1`, spinners with `ca=1`, and
3..5 with `st=1`.

Ownership (F8) beneficiary from `R[1]`: below 179 the active seat; otherwise
the next seat, except that on a 5x5 board a value of 217 or more picks the
seat after that. If the ownership flip costs the active seat its placed
card, pending direct captures are abandoned.

Shuffle (F17): ordinary cards are lifted in ascending cell order, then each
is dropped onto the ascending list of owner-8 cells at position
`R[old_index & 7] mod remaining`. Owner, selection slot, property and
modifier travel with the card.

Spinner (F9..16): the arrow points in direction `F-9` and the ray runs the
opposite way, wrapping around the board, through empty and wall cells. The
ray length is `min(start/30 + 1, players)` where `start` is the displayed
attacker value (start times multiplier). Opponent cards on the ray flip to
the active seat; another spinner on the ray fires on the next wave. A
spinner fires once per turn, and every spinner rotates one step clockwise
at the end of every turn, including a pass.

## Ordinary board generator

Inputs: the player count, the toggles `st`, `cb`, `ca` from the table rules
and raw CRT `rand()` values (0..32767, see [rules.md](rules.md#crt-rand)),
consumed in the order below with `mode` 0 for two players and 1 for three.
Result: the `F` list, `B`, `E` and the first seat `S` of StartData, and
`R[0..7]` of the first TurnData. The client also has a local mode that
forces all three toggles on, which no online match uses (unverified).

1. `B = BLOCKS[mode][rand() % 100]`.
2. If `ca`: `spinners = B / 4` (integer), otherwise 0.
3. If `st`: `E = EFFECTS[mode][rand() % 100]`, otherwise 0 with no draw.
4. `S = rand() % players` (first seat).
5. `B - spinners` times: if `cb` and `rand() % 100 < 50`, one more draw
   `v = rand() % 100` gives F6 (`v<25`), F7 (`<50`), F8 (`<75`) or F17;
   otherwise `F = (rand() & 1) + 1`. Then `rand() % empty_count` picks the
   cell from the ascending list of still-empty cells.
6. `spinners` times: `rand() % empty_count` picks the cell, then
   `F = (rand() & 7) + 9`.
7. `E` times: `v = rand() % 100` gives F3 (`<25`), F4 (`<75`) or F5, then
   `rand() % empty_count` picks the cell.
8. Eight draws; the low byte of each becomes `R[0..7]` of the first
   TurnData.

Count tables, as 100-entry lookups (value times bucket size):

| Table | Buckets |
|---|---|
| BLOCKS 2p | 0x1, 1x2, 2x3, 3x5, 4x9, 5x30, 6x50 |
| BLOCKS 3p | 0x1, 1x1, 2x1, 3x3, 4x2, 5x3, 6x9, 7x15, 8x15, 9x25, 10x25 |
| EFFECTS 2p | 0x3, 1x20, 2x70, 3x5, 4x2 |
| EFFECTS 3p | 0x3, 1x2, 2x3, 3x70, 4x3, 5x2, 6x2, then 0x15 |

The three-player effect table ends in fifteen zero entries. A server keeps
them as zero, so 18 of 100 three-player rolls (3 + 15) place no low tile.
Placement is without replacement, so StartData
`B` equals the number of walls, cb tiles and spinners placed and `E` the
number of low tiles; at most 6 or 10 blocks leave at least 10 or 15
placeable cells. The generator never emits F18. Draws are consumed even on
branches that place a plain wall, so a replay must follow the exact order
above.
