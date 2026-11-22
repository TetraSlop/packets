# Ratings and statistics

The save file ([save-file.md](../file/save-file.md)) carries the player's
result statistics. The client reads them at login, the
[43/0D ResultData](../irc/G/43.md#430d-resultdata) of an ordinary match
(fields `D`, `T` and `RN`) updates some of them and the client derives the
rest. This page lists the fields, how the client updates and displays them,
and which values a server must therefore produce. How the values reach the
ranking lists is in [ranking-files.md](../file/ranking-files.md) and
[R.md](../irc/R.md).

Three ratings exist and must not be confused: the average rating (`rating`,
replaced by `D2`), the VS. Rating total (`vs_rating`, adjusted by `T0` and
`T1`) and the Memorable Win text (`RN`).

## Profile statistics fields

All values are little-endian. The struct covers the first 0x138 bytes of
the save file; the rest of the file is in
[save-file.md](../file/save-file.md).

```c
struct ProfileStatistics {
    /* 0x000 */ uint8_t  pad_000[0x30];
    /* 0x030 */ uint32_t rating;            // average rating; D2 replaces it
    /* 0x034 */ uint8_t  pad_034[0xC];
    /* 0x040 */ uint16_t count;             // completed result count, divisor for the average reward; D5 replaces it
    /* 0x042 */ uint8_t  pad_042[2];
    /* 0x044 */ uint16_t sequence;          // result sequence, ring index for the offline rating; D4 replaces it
    /* 0x046 */ uint8_t  pad_046[2];
    /* 0x048 */ uint32_t max_reward;        // maximum reward; the client derives it from D1
    /* 0x04C */ uint32_t average_reward;    // average reward; the client derives it as D6 / D5
    /* 0x050 */ uint32_t reward_sum;        // cumulative reward sum; D6 replaces it
    /* 0x054 */ uint8_t  pad_054[0x24];
    /* 0x078 */ int16_t  streak;            // current streak, positive wins, negative losses; D3 replaces it
    /* 0x07A */ int16_t  best_streak;       // best streak; the client derives it from D3
    /* 0x07C */ uint8_t  pad_07C[1];
    /* 0x07D */ char     memorable_win[32]; // Memorable Win text, NUL-terminated; RN replaces it
    /* 0x09D */ uint8_t  pad_09D[0x8B];
    /* 0x128 */ uint32_t vs_rating;         // VS. Rating total; T0 and T1 are deltas
    /* 0x12C */ uint8_t  pad_12C[4];
    /* 0x130 */ uint16_t vs_player_games;   // VS. Player Games; T2 replaces it
    /* 0x132 */ uint8_t  pad_132[2];
    /* 0x134 */ uint32_t prize_points;      // Prize Points balance; not touched by ResultData
};
```

The client reads only these scalars; the file holds no separate win, loss,
draw or perfect-win counters. Reading the file does not rebuild the rating
ring from `rating` and `sequence`.

## Result classes

Inputs: each player's board score at the end of a completed match. Result:
a class per player, in one consistent seat order.

```
class = 2 + count(other > own) - count(other < own)
```

| Result | Two players | Three players |
|---|---|---|
| Sole highest | 1 | 0 |
| Tied highest, above a third player | | 1 |
| All tied, or strictly middle | 2 | 2 |
| Tied lowest, below a third player | | 3 |
| Sole lowest | 3 | 4 |

Lower is better. The presentation flags (1 win, 2 draw, 3 loss for two
players) and the perfect state come from the same scores, not from `D0`.
Perfect wins carry no separate weight in the rating ring.

## Offline rating ring

Inputs: a ring of 32 result classes, the saved sequence and a new result
class. Result: the new average rating and sequence. In offline play the
client keeps the ring itself:

1. Read the sequence as an unsigned WORD; the slot is `sequence & 31`.
2. Store the result class in that slot. Ties store class 2; they do not skip
   the update.
3. `rating = floor(sum(50 * (entry + 2)) / 32)` over all 32 entries.
4. Increment the sequence, wrapping 65535 to 0.

A fresh client fills the ring with class 2, rating 200 and sequence 0. From
that state one two-player win gives 198 and one loss 201; the five classes
give 196, 198, 200, 201, 203. The range for valid classes is 100 (32
three-player sole wins) to 300 (32 sole losses).

Online, this arithmetic is bypassed: the client stores the received `D2` as
the rating and leaves the ring and sequence unchanged.

## Streaks, counts and rewards

Inputs: the saved streak, count, reward sum and maximum reward, and the
`D1`, `D3`, `D4`, `D5` and `D6` of ResultData. Result: the saved values the
client displays.

Online, `D3` is stored as the current streak and the client only derives the
best streak from it: a winning setup compares `D3` with the saved best as
signed 16-bit. The offline transitions are the model a server reproduces: a
draw resets the streak to 0; a win after a negative streak becomes 1,
otherwise adds 1; a loss after a positive streak becomes -1, otherwise
subtracts 1. Overflow wraps at 32767 and -32768.

`D5` must be nonzero in a completed ordinary result: the client divides `D6`
by it without a zero check to produce the average reward. `D6` must equal the
saved cumulative sum. A positive `D1` raises the maximum reward in a non-draw
setup; fully tied results add nothing to the sum or the maximum.

`D4` is the sequence. The client treats a sequence of exactly 1 as the
first result, so a server carries the sequence across tables and reconnects
rather than restarting it.

Server choice:

- A server counts one completed match per result and saturates the count
  at 65535 instead of wrapping. The offline client adds players minus one.
- A server preserves the rating.
- A server that applies the offline ring to hosted results instead enrols
  an account with 32 neutral entries at its saved sequence and sends the
  ring rating as `D2`.

## VS. Rating

Inputs: the saved VS. Rating total and the `T0`, `T1` and `T2` of
ResultData. Result: the new total and VS. Player Games count.

The total is displayed in hundredths: 12345 reads as 123.45. `T0` and `T1`
are signed WORD deltas applied to the total; a zero `T0` and a nonpositive
`T1` each skip their addition. `T2` is the final VS. Player Games count,
which the client stores as received. Neutral components `T=0|0|<count>`
therefore preserve the saved total exactly, including values with bit 31
set.

The client has an offline producer for `T0` and `T1`, driven by the local
result order (higher score first, equal scores by seat) and two CRT random
draws `r0`, `r1` in 0..32767:

| Local result order | T0 | T1 |
|---|---|---|
| 0 (first) | `r0 % 1000` | `r1 % 1000` |
| 1 or 2 | `-(r0 % 1500)` | 0 |

The client clears `T1` whenever the local result class is at least 2,
before it is applied. Online the producer is skipped and the received
components are used; nothing shows the original service used the offline
producer.

A fresh client starts the total at 1,000,000 (10000.00); the save file
overwrites it. The event replies `EInit`, `EcmInit` and `Event` assign a
received `/RT=` total directly.

Server choice: a server sends neutral `T0` and `T1` and advances `T2` by
one for a completed human match, leaving the VS. Rating total unchanged.

## Memorable Win

`RN` is a server-supplied string displayed under Memorable Win. The client
copies it into a 32-byte field without a length check, so a server sends at
most 31 bytes so the terminator fits. Changing the numeric rating does not
change this text, and no rule converting a rating into an `RN` string is
recovered.

## What a server must produce

| Value | Source |
|---|---|
| `D1` reward | Server. The client animates positive values only. |
| `D2` rating | Server; it is authoritative online. |
| `D3` streak, `D4` sequence, `D5` count, `D6` sum | Server, consistent with the saved profile. |
| Maximum reward, average reward, best streak | Client derives; the saved profile must agree. |
| `T0`, `T1` | Server deltas; 0 is neutral. |
| `T2` | Server, the final count. |
| `RN` | Server, at most 31 bytes. |
| Prize Points | Separate flow; not part of `ResultData`. |

`D0` has no established reader and should stay neutral.
