# Prize Center

The Prize Center exchanges prize points, a balance separate from money, for
one of thirty fixed cards. It runs on the shop service
([services.md](../irc/services.md#shop-service)) as a
[B2](../irc/G/B2.md) session, entered with its own request and initialised
with `ExcInit` instead of the shop's `Init`. The pack and card data are in
[catalogue.md](../client/catalogue.md). The flow starts from the Prize
Center entry in the online title and ends with Quit.

## Sequence

1. Client to the shop service: [70/00 CvReq](../irc/G/70.md#7000-cvreq)
   with the account identity. The server checks it against the
   authenticated account, credits any pending awards (see Rules), opens a
   session and answers [71/00 CvEnter](../irc/G/71.md#7100-cventer) and
   then [B2/13 ExcInit](../irc/G/B2.md#b213-init), which carries the point
   balance before awards, the Top 30, Lucky and Rookies awards and the
   Lucky Card models to announce. `/N` is the session every later request
   carries as argument. The client adds the awards to the balance itself
   and sends no acknowledgement for them.
2. Client to the shop service: [B2/14 Buy](../irc/G/B2.md#b214-buy) with a
   pack ID 45..74. The server answers `Card` with the generated card or
   `S=0` to refuse. After a positive `Card` the client debits the points
   locally and moves the card into its collection when it fits.
3. With a full collection the client first sends
   [B2/10 Erase](../irc/G/B2.md#b210-erase) Mode 1 and
   [B2/22 Decks](../irc/G/B2.md#b222-decks) edits exactly as in the shop,
   then Erase Mode 2. No reply.
4. Client to the shop service: [B2/15 Get](../irc/G/B2.md#b215-get). No
   reply. The server commits the staged card and the point debit.
5. Client to the shop service: [B2/17 Quit](../irc/G/B2.md#b217-quit),
   answered as in the shop. The reply closes the session.

## Rules

### Points

Prize points are a balance of their own in the save file
([save-file.md](../file/save-file.md#layout)), distinct from money, rating
and the VS. Rating points. Awards are added at entry with an unsigned
clamp at 99,999. Purchases debit points only; money is unchanged.

### Awards

Server choice:

- A server credits the awards to the saved balance before sending ExcInit.
  The reply shows the balance before the award; the client adds the award
  itself.
- A repeated entry during an open session replays the same reply.
- No automatic weekly or Lucky Card award schedule is known, so awards are
  an administrative input.

### Purchase staging

Server choice:

- The purchase is staged until Get.
- A repeated Buy of the same pack replays the identical Card; a different
  Buy while staged gets `S=0`.
- A server enforces the capacity of 1000 at Get.
- Erase and Decks are also accepted without a staged purchase and commit
  immediately.

## Exit

Quit closes the session. A disconnect discards the staged purchase.

Server choice: Quit with a staged purchase is refused; a repeated Quit for
the last closed session is answered again.
