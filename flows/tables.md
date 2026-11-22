# Tables

The room service ([services.md](../irc/services.md#room-service))
advertises each table with a state and a 49-byte metadata string, in the
room's PTL file and in its `TD` deltas. The client decodes the
metadata, decides locally which actions to offer (Join, Computer, the
password dialog) and only then talks to the table service. This page covers
the advertisement, the client's local gates, the rule query, the settings
editor and the channel probe. The admission handshake itself is in
[match-admission.md](match-admission.md).

## Sequence

1. Room service to client: the table's state and metadata, in the PTL
   ([directory-files.md](../file/directory-files.md#layout))
   and in each `TD` delta
   ([L.md](../irc/L.md#td-table-update)); the metadata layout is
   under [Table metadata](../irc/L.md#table-metadata). Before publishing a
   table a server sets its state, occupancy, owner, password and rules, so
   that the gates under Rules reach the intended decision.
2. The client decodes the record and enables or disables Join and Computer
   by the gates under Rules. A disabled action sends nothing. On a password
   table the Join action opens the password dialog first; a mismatch sends
   nothing.
3. Client to the table service: the admission handshake of
   [match-admission.md](match-admission.md#sequence), starting with
   [41/01 GameEN](../irc/G/41.md#4101-gameen). Spectator entry and Computer
   entry instead send
   [2D/00 CheckJoinTable](../irc/G/2D.md#2d00-checkjointable) before
   changing to the table channel; the server answers
   [2E/00 CheckJoinTable_Ans](../irc/G/2E.md#2e00-checkjointable_ans), which
   the client reads only on its recovery path (see Exit).
4. Client to the table service: [12/00 Rule](../irc/G/12.md#1200-rule),
   sent after admission to the admitted table only. The server answers
   [13/00](../irc/G/13.md#1300-tet-and-tab) with the table's current game
   rules and table options. The reply feeds the settings editor and leaves
   the client's saved personal defaults untouched.
5. The admitted owner edits the settings. On `GameEA /EN=1` the client
   copies the selected table's decoded rules, options and room context into
   its current settings. In a fixed room (room context 0) only the table
   options editor opens and the game rules stay as advertised; in a custom
   room (context 1) the game rule editor opens first, with the player's
   personal defaults, and its commit opens the table options editor. Initial
   table options come from the saved defaults. A later configuration action
   from the table opens the same editor by the same context.
6. Client to the table service:
   [14/00 Apply](../irc/G/14.md#1400-tet-and-tab) with both `Tet` and `Tab`.
   No reply is parsed. After accepting an Apply the server publishes the
   `TD` with the new metadata and owner, then
   [41/04 GameML](../irc/G/41.md#4104-gameml) and
   [1A/00 GameOwner](../irc/G/1A.md#1a00-gameowner) to the members.
7. Client to the table service, optional:
   [24/00 Save](../irc/G/24.md#2400-save) stores the flagged sections as
   personal defaults and does not change the current table. No
   acknowledgement is parsed.

## Rules

### Table states

| State | Meaning |
| --- | --- |
| 0 | Empty; selecting it takes the claim (new owner) branch |
| 1 | Open with players; Join takes the existing-table branch |
| 2 | Playing; Join disabled |
| 3, 4, 5 | Join disabled |
| 6 | Closed; disabled by a separate check |

An owner who has entered but not yet applied settings shows as state 0
with a nonzero count; the client disables every table action on such a
record. A server advertises an untouched table as state 0, count 0, owner 0.

### Table types

The Join action accepts metadata type `A` and type `B`; the Computer
action needs `A`. A mixed room advertises a human-only table as `B`.

The client reads the rule bytes only from an occupied state 1 record. For
an empty or state 3 record it zeroes the table options; in a custom room it
also zeroes the game rules, so a custom room's first owner starts from an
empty rule set, while a fixed room's rules remain available from the RL
record.

### Join eligibility

These gates are evaluated by the client. The messages they lead to carry no
proof of any of them, so a server that wants the same rules checks its own
committed inventory and balance on `GameEN` and `Req`.

Existing table (state 1), player not yet admitted:

- at least five owned cards;
- occupancy below 3 and state outside 2..5 (6 is handled separately);
- money at least the advertised wager;
- `ll` <= collection points <= `lu`, inclusive.

The bounded value is the player's collection points
([catalogue.md](../client/rules.md#collection-points)), although the
client labels the range "Card Level"; it is neither the rating nor the
level byte. The client's help text 20 reports insufficient money and 13
fewer than five cards.

Empty table (state 0): five cards are required and `ll`/`lu` are not read.
In a fixed room money is checked against the advertised wager; in a custom
room the money check is skipped, because the owner chooses the wager
afterwards.

Computer action on an empty `A` table: metadata type `A`, the room's
computer byte `A` in the RL record, and five cards. Money and wager are not
checked. Help texts: 22 available, 23 fewer than five cards, 25 disallowed
room or table, 19 when the client is busy.

Already admitted members use different button modes and bypass these
checks; a server should not re-check bounds against a member when the owner
changes them later. Observers have their own path and no inventory
requirement. The rank bounds `au`/`al` are never checked.

Server choice: reject a failed check with the `GameEA`/`PLAYACK` failure
codes listed under [41/02](../irc/G/41.md#4102-gameea) and
[02/00](../irc/G/02.md#0200-playack); no distinct rejection code per
condition is known.

### Password

The password is public: it is in the metadata every room member receives.
When the password flag is set, the Join action opens a password dialog and
the client compares the typed text case-sensitively with the advertised
password. A match continues to the normal admission; a mismatch shows an
error and sends nothing. No message carries the password or a proof of it,
so a server cannot authenticate it; it can only advertise it.

The editor's input buffer allows twelve bytes, but the advertised field
holds eight.

Server choice: reject an Apply whose password cannot be advertised
faithfully rather than truncate it.

## Exit

A disabled action and a password mismatch send nothing, so there is nothing
to undo.

When the client's recovery path runs after a channel probe, it reads the
`2E` answer: `Join` and `Mem` both 0 skip the table-channel cleanup;
anything else sends [20/00 ExitCh](../irc/G/20.md#2000-exitch) to the table
and returns to the room channel. A server answers `ExitCh` with nothing, or
with [41/10 GameWa](../irc/G/41.md#4110-gamewa) to the others, and
releases whatever the probing client held at that table.

Leaving a table before the match starts is in
[match-admission.md](match-admission.md#exit).
