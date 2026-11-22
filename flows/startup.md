# Startup

What the client sends between the Game IRC login and the online title, and
what a server answers at each step. The profile file service
([services.md](../irc/services.md#profile-file-service)), the directory
service and the shop service ([services.md](../irc/services.md#shop-service))
take part. The flow starts when the IRC login completes and ends at the
online title.

## Sequence

1. Client to server: the Game IRC login, challenge, USER, NICK and 422
   ([session.md](../irc/session.md)).
2. Client to the profile file service: the handle, character and status
   reads ([service.md](../file/service.md)). The registration and read
   commands of the `P` family, CR and PG, are also sent during startup
   ([P.md](../irc/P.md#cr-registration)).
3. Client to the directory service:
   [E1/00 TeachDV](../irc/G/E1.md#e100-teachdv), empty body. The server
   answers [E2/00 TeachDVAns](../irc/G/E2.md#e200-teachdvans) with the
   profile directory and volume, the auction and ranking file locations and
   the shop service identity. The client uses `D` and `V` as the file owner
   of every later profile request and as `Dm` and `Vol` of Init, and
   addresses the returned identity for Init and the shop.
4. Client to the profile file service: a read of the save file
   `U/g/TM0DataFile` ([save-file.md](../file/save-file.md)). The read runs
   concurrently with the next step.
5. Client to the shop service: [80/00 Init](../irc/G/80.md#8000-init). It
   is sent before the save read has finished, so this first Init always
   carries `GLD=0`, even for a registered account. The server answers
   [81/00 InitAns](../irc/G/81.md#8100-initans); `Ans=2` makes the client
   reread the save after the first read completes.
6. Guild choice, see Rules. A guild 0 save opens the guild picker; the
   picker's confirm sends a second Init with the chosen `GLD`, answered with
   a second InitAns.
7. The client shows the online title.

## Rules

### Guild choice

After a positive InitAns the client reads the guild byte of the save file
([save-file.md](../file/save-file.md#layout)). Guild 0 opens the guild
picker; guild 1..3 skips to the online title. The picker's three choices set
guild 1, 2 or 3, and its confirm button sends a second Init with the chosen
`GLD`. The client rereads the save after any positive answer to that second
Init.

Server choice:

- `GLD=0` is a read-only probe answered `Ans=1`.
- A nonzero `GLD` on a guild 0 account registers the guild and is answered
  `Ans=2`.
- Repeating the same nonzero guild is also answered `Ans=2`, so a lost
  reply is safe to retry.
- A server rejects a different nonzero guild by closing the connection.
- Registration changes only the guild byte and never resets money,
  statistics, preferences or cards.

### Later saves

The options editor saves through [84/00 Opt](../irc/G/84.md#8400-opt) and
the table and game editors through [24/00 Save](../irc/G/24.md#2400-save).
Neither is part of the startup order.

## Exit

Startup has no quit message. The only rejection is the server closing the
connection on a guild conflict, after which the client has to log in again.
Nothing has to be undone: a refused Init changes no account state, and a
completed registration changed only the guild byte.
