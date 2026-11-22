# Results and card prizes

When the last turn ends the table service sends
[43/0D ResultData](../irc/G/43.md#430d-resultdata). Each client answers
[43/0E Ready](../irc/G/43.md#430e-ready) and receives its
[43/1A TCList](../irc/G/43.md#431a-tclist), the final hands with their
current owners. If a winner holds more than one captured loser card the
client asks for a choice with
[43/0F GetSelect](../irc/G/43.md#430f-getselect). The result screen may then
send inventory edits, [43/10 Erase](../irc/G/43.md#4310-erase) and
[43/22 Decks](../irc/G/43.md#4322-decks), before
[43/11 Continue](../irc/G/43.md#4311-continue) closes the round. Stakes and
payouts are in [wagers.md](wagers.md); departed seats are in
[departures.md](departures.md).

## Sequence

1. Server to every human: [43/0D ResultData](../irc/G/43.md#430d-resultdata),
   argument the last sequence. Before sending it the server advances each
   seat's statistics (see Rules) and settles the payout
   ([wagers.md](wagers.md#rules)).
2. Each human to the server: [43/0E Ready](../irc/G/43.md#430e-ready). The
   server answers the first Ready of each human with that human's
   [43/1A TCList](../irc/G/43.md#431a-tclist); the prize screen sends a
   second Ready that the server does not answer.
3. For each winner and loser pair that needs a choice (see Rules), the
   winning client to the server:
   [43/0F GetSelect](../irc/G/43.md#430f-getselect) with the hand index.
   The server checks the index against the candidate list and answers every
   participant with the reply form. When the winner is a computer seat the
   server sends the reply form itself, after TCList, with the computer's
   seat as argument.
4. The result screen merges pending gains into the collection. Client to
   the server, no reply: [43/10 Erase](../irc/G/43.md#4310-erase) when the
   collection is full, and [43/22 Decks](../irc/G/43.md#4322-decks) for
   deck edits. The server applies them against the committed inventory plus
   the round's pending gains.
5. Each human to the server: [43/11 Continue](../irc/G/43.md#4311-continue)
   once its edits are flushed. The server answers every human with argument
   1 and every answer 1 (the table returns to card selection with the same
   seats and rules) or argument 2 and every answer 3 (the match ends). The
   computer menu's `Conf=1` is answered with
   [43/04 ComList](../irc/G/43.md#4304-comlist) instead
   ([computer-player.md](../client/computer-player.md)).

## Rules

### Result statistics

The client's offline bookkeeping defines the transitions a server
reproduces so that the saved profile and the next ResultData agree:

- Streak (`D3`): a draw resets to 0; a win after a negative streak becomes
  1, otherwise increments; a loss after a positive streak becomes -1,
  otherwise decrements. 16-bit overflow wraps, including the sign at 32767.
- Sequence (`D4`) increments modulo 65536.
- Count (`D5`) increments; the offline client adds `player_count - 1`.
- A fully tied round adds nothing to the reward sum or the maximum; any
  other credit adds to `D6` with 32-bit wrap.
- `T2` advances by one for a completed human match with 16-bit wrap;
  computer rounds leave it unchanged.

Rating (`D2`): offline, the client keeps a 32-entry ring of result classes
and averages `(class + 2) * 50`; a fresh ring (all entries 2) gives 200, and
one class 0 result makes it 196. Online, the ring is skipped and `D2` is
taken as sent. VS. Rating: offline, `T0` and `T1` are random (`r0 % 1000`
and `r1 % 1000` for first place, `-(r0 % 1500)` and 0 otherwise); online,
the received values are applied instead. The client clears `T1` before
applying it when the local class is at least 2. VS. Rating is displayed in
hundredths. `RN`: the offline result never writes it; the client preserves
`RN` when the field is omitted, clears it when empty, and replaces it when
supplied.

Server choice:

- The count saturates at 65535 instead of wrapping to 0.
- `T0` and `T1` are sent as 0.
- `RN` is the stored profile text.

### Prize resolution

Both clients run the same algorithm over the TCList owners, so the server
drives the same sequence of choices:

1. Order the seats by decreasing score, ties broken by lower seat.
2. For each (winner, loser) pair in that order where the loser's score is
   strictly lower, the candidates are the loser's remaining cards whose
   current owner is the winner, in ascending hand index.
3. Exactly one candidate, or a perfect winner (all cards on the board), is
   awarded without a choice. Otherwise the winner sends GetSelect.
4. The awarded card is removed from the loser's hand, which compacts, and
   its deck association becomes 255. For a nonperfect result the loser's
   other cards captured by this winner return to the loser, and the
   winner's cards captured by this loser return to the winner; both can
   change later pairs.
5. A perfect winner repeats the same pair while the loser still has cards.

A perfect winner in a three-player round therefore receives ten cards.
Awarded cards never enter the five selected slots; the client keeps them in
a separate pending-gain list.

Example (three players, scores 2, 3, 10, order seat 2, seat 1, seat 0):
seat 2 takes one of seat 1's cards at hand indexes 1..4, then one of seat
0's at 0, 1, 2 or 4; seat 1 then takes seat 0's card at index 3 without a
choice.

### Generated cards

A card produced by the CBC special tile is placed with base owner 5 (see
[board.md](../client/board.md)). At the result it does not enter the
five-card hands: only base owners below the player count are rebuilt. A
base-owner-5 card whose current owner is the local seat goes to the
pending-gain list with its stats as they stand (it receives no result
growth) and is then cleared from the board. An observer does not collect
it. Pending gains merge into the collection at the result screen; when the
collection is full the client sends Erase rather than a shop request.

### Result edits

Erase and Decks are sent from the result screen without a reply. Continue
flushes any attached Decks and Erase editors before it is sent. A deck
rename can close its editor earlier, so a Decks message may arrive after
Continue.

Server choice:

- A server refuses Erase after Continue.
- A server applies a late Decks only while the account is otherwise idle
  and unchanged.
- A server matches edits against the committed inventory plus the round's
  pending gains, and accepts at most 15 records per Erase and 13 per Decks.

### Continue and rematch

Each human votes with Continue once its edits are flushed. The client
checks the next stake and its card count before voting yes
([wagers.md](wagers.md#rules)).

Server choice: the positive reply waits for every participant's acceptance
and for the round's prize settlement to be stored; a stale or forged
acceptance from a client that could not afford the next stake is rejected.

## Exit

The match ends with the argument 2 Continue reply, after which the table
returns to its lobby state. A Quit from the result screen
([43/05](../irc/G/43.md#4305-quit), argument 10..13) or a disconnect ends
the match the same way for that player; the round's stored result and
payout stand ([wagers.md](wagers.md#exit)). An Erase that arrives after
Continue is refused and a late Decks is applied only to an idle account, as
under Rules.
