# Wagers

A table's `bm` rule is the stake, 0..2000, and `du` doubles it on each
rematch (see [rules.md](../client/rules.md)). The client subtracts the
stake from its own balance at fixed points and then adopts whatever balance
the table service sends in
[43/01 VsGameInit](../irc/G/43.md#4301-vsgameinit) `/M`; the payout
arrives as [43/0D ResultData](../irc/G/43.md#430d-resultdata) `D1`. A
server therefore debits at the same points and never relies on a negative
`D1`. The flow starts with the advertised wager in the table metadata and
ends with the payout or a refund.

## Sequence

1. Room service ([services.md](../irc/services.md#room-service)) to client:
   `bm` and `du` in the table metadata
   ([L.md](../irc/L.md#table-metadata)). The client checks its money
   against `bm` before offering Join ([tables.md](tables.md#rules)).
2. Joining an existing table, on
   [41/02 GameEA](../irc/G/41.md#4102-gameea) with `EN` other than 1: the
   client subtracts the table's current wager; a negative result is clamped
   to 0. Creating a table, on `GameEA` with `EN=1`: the client does not
   debit yet; the rule editor stores the chosen wager, and confirming the
   table configuration subtracts it just before
   [14/00 Apply](../irc/G/14.md#1400-tet-and-tab) is sent. A new owner is
   not charged the empty table's old wager and then charged again; the one
   debit precedes the settings request.
3. Computer match: [43/02 ComGameInit](../irc/G/43.md#4302-comgameinit)
   `/M` is the pre-debit balance; the client subtracts the wager after
   choosing opponents, before [43/06 ComGame](../irc/G/43.md#4306-comgame).
4. Server to every human: [43/01 VsGameInit](../irc/G/43.md#4301-vsgameinit).
   The server debits the stake before sending it; `/M` replaces the local
   balance with the server's post-debit value.
5. Server to every human: [43/0D ResultData](../irc/G/43.md#430d-resultdata)
   with the credit in `D1`, computed under Rules.
6. Rematch: both Continue dialogs compute the next stake under Rules when
   they open. On a yes vote the client subtracts the next wager with no
   lower clamp, so balance 0 and wager 100 leave 4294967196 locally, and
   sends [43/11 Continue](../irc/G/43.md#4311-continue) `Ans=1` only if its
   balance is at least the next stake. The next round's VsGameInit `/M` is
   `balance - next`.

## Rules

### Server debit points

Server choice:

- A server debits a round's stake once per human, before VsGameInit,
  before the accepted ComGame selection, or before the positive Continue
  reply.
- A participant who cannot cover the stake or owns fewer than five cards is
  not admitted to the round.
- An unfinished round refunds the stake once.

### Next stake

Both Continue dialog constructors (human table and computer menu) compute
the next wager when the dialog opens, before the player answers:
`next = min(2000, bm * (1 + du))`. Wagers 1000 and above with `du=1` all
become 2000. Opening the dialog changes no balance; a no vote costs
nothing. The computer menu's `Conf=1` path skips the balance check.

### Payout

`D1` is a credit. The client adds a positive `D1` to its post-stake
balance and raises the saved maximum reward; zero is coherent for a round
without stakes, and a negative value does not take money. `T0` and `T1`
are VS. Rating components, not money.

The client contains an offline reward formula, used when it plays without
a server. With `L` the number of strictly lower scores, `r` the truncated
mean of the other players' ratings, and `m` 3 for a perfect result or 1
otherwise:

```text
fully tied:  reward = stake
otherwise:   factor = 100 + trunc((400 - r) / 2)
             reward = trunc(factor * (stake + 100) * m * L / 100)
```

At rating 300 and stake 100: a two-player win pays 300, a perfect win 900,
a draw 100, a loss 0; a sole three-player win pays 600 and a perfect one
1800. A high rating can make the value negative. This is the offline
client's arithmetic; nothing shows the original service used it online.

Server choice: computer rounds credit `max(0, reward)` from this formula,
capped at the 99,999,999 balance limit. Human-versus rounds are zero-sum:
every participant pays the same stake into a pot, the highest scorers
divide it by integer division, and any remainder or cap overflow is
refunded to contributors in ascending seat order; a fully tied round
refunds every stake.

## Exit

The client's table cleanup refunds the table wager locally only when its
argument is 0 and the table id is nonzero, capped at 99,999,999. That
refund is a local predicate of the client's disconnect path and says
nothing about what the original service refunded.

Server choice: a human who sends Quit during the turn or battle phase of a
wagered computer round forfeits the stake; every other interruption
(disconnect, timeout, Break, Quit from the menu, selection or result
screens) refunds it. No card is taken on a forfeit. A completed round keeps
its payout even if the prize choice or the exit dialog is interrupted.
