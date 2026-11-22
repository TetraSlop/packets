# Event cards

The event card service ([services.md](../irc/services.md#event-card-service))
hands event members the cards and money an event awarded them. The client
treats it as a collection screen: it asks to enter, receives the pending
items, confirms the cards with `Get`, animates the money without
acknowledging it, and leaves with `Quit`. It never starts a match through
this service. The service is advertised in the event room's `PTL`
([event-rooms.md](event-rooms.md#rules)). The flow starts from the
collection action in the event room and ends with Quit.

## Sequence

1. Client to the event card service:
   [90/00 CardReq](../irc/G/90.md#9000-cardreq) with the account identity.
   The server checks it against the authenticated account, credits the
   pending money (see Rules) and answers
   [91/00 CardEnt](../irc/G/91.md#9100-cardent). A positive `Ans` opens the
   collection and the server follows with
   [B2/13 EcmInit](../irc/G/B2.md#b213-init) with argument 0: `N` is the
   session the client repeats as the argument of every later `B2` message,
   `M` the prior balance, `PM` the money credited by this visit and `C` the
   pending cards. The card records are described in
   [catalogue.md](../client/catalogue.md#card-values-on-the-wire).
2. Client to the event card service: [B2/15 Get](../irc/G/B2.md#b215-get),
   sent before the cards are revealed when they fit the inventory. With
   `C=0` no `Get` is sent. The server marks the batch claimed.
3. The client reveals every card; there is no choose-one step. It then
   animates `PM` into its balance, capped at 99,999,999, and sends no money
   acknowledgement.
4. The client checks capacity again. A fitting batch is moved into the
   collection and compacted without another `Get`. An overflowing batch
   opens the overflow editor, which sends
   [B2/10 Erase](../irc/G/B2.md#b210-erase) and
   [B2/22 Decks](../irc/G/B2.md#b222-decks) edits followed by
   [B2/15 Get](../irc/G/B2.md#b215-get). The server validates the edits and
   applies the batch with them.
5. Client to the event card service:
   [B2/17 Quit](../irc/G/B2.md#b217-quit). The client waits for the reply
   and polls `Dead` on failure.

## Rules

### Claim semantics

What the client does fixes what a server commits and when:

- Money is credited before `EcmInit` is sent, because nothing acknowledges
  it. `M` is the prior balance and `PM` the amount credited by this visit.
  Money that would exceed the cap stays pending for a later visit.
- Cards stay pending until `Get`.
- A repeated `90` during a visit replays the same `91` and `EcmInit`; after
  `Get` the replay reports the current balance with `PM=0` and `C=0`.

### Grants

Server choice: grants are trusted administrative input (at most 30
undecked cards and 99,999,999 money per grant) keyed by a stable grant
identifier, and the account, membership and claim offsets are persisted
together before replying.

## Exit

Quit closes the session; a Quit with unclaimed cards (`C` above 0 and no
`Get`) is an error, and a visit with `C=0` may Quit directly. A disconnect
before `Get` drops the staged batch and edits; cards already claimed stay
owned. Reconnecting collects the rest without crediting money twice.
