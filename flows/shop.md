# Shop

The shop service ([services.md](../irc/services.md#shop-service)) sells
packs for money, buys cards back, and takes discards and deck edits. The
same service also runs the [Prize Center](prize-center.md) and the
[auction collection](../irc/A.md#sequence); all three use the
[B2 opcode](../irc/G/B2.md). Pack contents, prices and the money and
points formulas are in [catalogue.md](../client/catalogue.md). The flow
starts from the shop entry in the online title and ends with Quit.

## Sequence

1. Client to the shop service: [B0/00 ShReq](../irc/G/B0.md#b000-shreq)
   with the account identity. The server checks the identity against the
   authenticated account, opens a session and answers
   [B1/00 ShEnter](../irc/G/B1.md#b100-shenter) and then
   [B2/13 Init](../irc/G/B2.md#b213-init), whose `/N` is the session that
   every later request carries as argument.
2. Client to the shop service: [B2/14 Buy](../irc/G/B2.md#b214-buy) with a
   pack ID from the visible list. The server answers `Card` with the
   generated cards, or `S=0` to refuse. On a positive `Card` the client
   debits its displayed balance by the catalogue price and holds the cards
   in its temporary buffer until Get.
3. With a full inventory the client opens the overflow editor, which sends
   [B2/10 Erase](../irc/G/B2.md#b210-erase) Mode 1 and
   [B2/22 Decks](../irc/G/B2.md#b222-decks) edits, then Erase Mode 2. No
   reply. The server holds the edits with the staged purchase.
4. Client to the shop service: [B2/15 Get](../irc/G/B2.md#b215-get), once
   the working inventory fits 1000 cards. No reply. The server commits the
   staged purchase and every edit received since Buy together.
5. Client to the shop service, any time in the session:
   [B2/16 Sell](../irc/G/B2.md#b216-sell) and
   [B2/22 Decks](../irc/G/B2.md#b222-decks) edits. No reply. The server
   applies Sell by crediting the sale value and removing the cards.
6. Client to the shop service: [B2/17 Quit](../irc/G/B2.md#b217-quit). The
   reply closes the session.

## Rules

### Entry

Server choice: an account still in guild 0 with no cards is refused the
shop until it registers. Entry while a purchase is pending is refused; a
repeated entry reuses the open session.

### Purchase staging

The debit and the cards become durable at Get, not at Buy.

Server choice: a server stages the purchase and commits the debit and the
cards only at Get. Repeating Buy for the same pack while staged replays the
identical Card reply; a Buy for a different pack while staged is refused
with `S=0`.

## Exit

Quit closes the session. A disconnect discards the staged purchase and any
edits not yet committed by Get; the client reloads its profile on
reconnect.

Server choice: Quit while a purchase is staged is refused.
