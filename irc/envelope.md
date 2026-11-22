# Service envelope

Every service message is the trailing argument of an IRC `NOTICE` or
`PRIVMSG` on the game connection ([session.md](session.md)), addressed to
a service or player nickname ([services.md](services.md)). It begins with
the literal `GTM0` and a one-letter family, followed by one of two body
grammars: the header and named sections of family `G`, or the command
codec of families `A`, `L`, `P` and `R`. Above this layer sit the family
and opcode pages.

## Prefix and family

```c
struct Envelope {
    char magic[4];      // `GTM0`, matched case-sensitively by the client
    char family;        // family letter: G named sections, A auction, L rooms, P character profile, R ranking
    char body[];        // family-specific, see below
};
```

```
C2S GTM0G43000A11@PutData=/P=1/H=2/F=3
C2S GTM0P<CR>\x070x0000000000000001\x061\x07<AN>\x07LocalPlayer\x07
```

The client checks only the four literal bytes and skips the family letter
without checking it, so `GTM0g`, `GTM0Z` and `GTM0L` reach the same receiver
as `GTM0G`.

## Family G header

The body after `GTM0G` starts with eight hex digits, then one or more
sections.

```c
struct NamedMessage {
    uint8_t opcode;         // 2 hex, the opcode, for example `43`
    uint8_t reserved;       // 2 hex, see below
    uint8_t subtype;        // 2 hex, second selector, for example `0A`
    uint8_t argument;       // 2 hex, per-message argument: a session number, a seat, a status or a sequence
    char sections[447];     // `@Name=` followed by `/Key=value` fields
};
```

The client writes the reserved byte as `00`, except in the heartbeat literal
which carries `0D`, and ignores it on receive: `43FF0A11` parses as 43/0A
with argument 0x11.

## Family G sections

A section is `@Name=` followed by zero or more `/Key=value` fields; a second
section begins at the next `@`. Values are printable ASCII without `/` or
`@`. List values use `|` as the element separator (for example
`/AC=1|1`). Hex identities are 16 upper-case digits; names are hex byte
pairs; numbers are decimal.

```
C2S GTM0G43000A11@X=/A=1@Y=/B=2
C2S GTM0G42000700@Chat=/Dt=#CHAT#hello @X=/A=1
```

`X`, `Y`, `A` and `B` stand for any section and field name. How the client
reads a body:

- A section's value is the text after `@Name=` up to the next `@`; for `@Y`
  in the first example that is `/B=2`.
- Chat text is not split at `/`: the client takes everything after `@Chat=`
  as the text, in the second example `/Dt=#CHAT#hello @X=/A=1`, so a server
  must not append a section after `@Chat`.
- A duplicated section name hides both copies (`@X=/A=1@X=/B=2` finds no
  `@X`), so a server never repeats a section name.
- One IRC line is one message.

## Length limit

The client refuses to send a payload longer than 469 bytes. With the
longest service prefix (`:UUIRT71SR!p@local NOTICE UUIRT71SS :`, 37 bytes)
and the six framing bytes that is exactly the 512-byte IRC line. A family
`G` body is at most 447 characters, so a complete family `G` message is at
most 5 + 8 + 447 = 460 bytes.

There is no fragmentation. The client can slice a longer body into several
messages, but each slice repeats the same eight-character header with no
fragment number or count, and the receiver treats each slice as a separate
message. A 448-character body therefore arrives as a 447-character message
followed by a one-character message with the same header, which is parsed
on its own. Nothing follows the body: there is no checksum or fragment
index after it.

```
S2C GTM0G43000A11@X=AAAA...(447 chars)
S2C GTM0G43000A11A
```

Services that need more than 447 characters use application-level paging
(chat rosters, card lists), which each opcode page documents.

## Command codec

The body of families `A`, `L`, `P` and `R` is a sequence of commands, each
`<XX>` + BEL (0x07) + payload + BEL. Fields inside a payload are separated
by ACK (0x06) where the command defines fields; a payload may also be empty
or opaque text. Commands are concatenated without separators.

```
C2S GTM0P<PG>\x070x0000000000000001\x061\x07
S2C GTM0P<CS>\x07\x07
C2S GTM0A<BR>\x07<actor>\x07<AI>\x071\x07<BM>\x07550\x07<BC>\x070\x07
```

`<actor>` is the twelve-field actor of [A.md](A.md). The two-character
token is one of 102 distinct tokens in a dictionary of 103 entries; `PO`
appears at 0x60 and 0x65 and the lookup returns the first. A token's name
says nothing about what it does; each family page documents its commands.

```
00 RI  01 MR  02 MC  03 RE  04 CF  05 WH  06 ZI  07 NN  08 US  09 DP  0A MN  0B MA
0C MS  0D CN  0E GN  0F TN  10 RS  11 MF  12 RO  13 PD  14 PC  15 TD  16 ST  17 TS
18 TP  19 DR  1A DD  1B DO  1C DN  1D DC  1E RD  1F RN  20 DE  21 DS  22 DF  23 EP
24 ED  25 NC  26 ET  27 AM  28 SP  29 BI  2A AC  2B LC  2C CM  2D CB  2E CD  2F BC
30 II  31 AI  32 BM  33 EI  34 ER  35 ES  36 EF  37 BR  38 BS  39 BF  3A SI  3B IO
3C MP  3D LT  3E IN  3F IC  40 IB  41 BH  42 BD  43 SS  44 SF  45 HS  46 HF  47 SN
48 CR  49 AN  4A CI  4B CS  4C NS  4D CP  4E GR  4F PN  50 PV  51 PR  52 GK  53 RK
54 PF  55 GG  56 PE  57 GO  58 OK  59 OG  5A RR  5B RF  5C RG  5D PI  5E LN  5F PG
60 PO  61 PM  62 NO  63 VO  64 PP  65 PO  66 FO
```

The whole payload, including `GTM0` and the family letter, counts against the
469-byte send limit.

## Relay guard

The client does not authenticate every message it receives:

| Receiver | Behaviour |
|---|---|
| opcode 2B, header only | any sender; opens the maintenance dialog that leads to exit |
| opcode 2C, header only | any sender; changes the status override |
| [A1/00 TrEntAns](G/A1.md#a100-trentans) | adopts the sender as the trade service |
| [A2/18 Init](G/A2.md#a218-init) | replaces the trade service and session, resets offers |

Because the client does not check the family letter and accepts lower-case
hex and any argument, a server cannot protect these by filtering only the
canonical `GTM0G2B` and `GTM0G2C` strings. A server must reserve the whole
`GTM0` prefix.

The client sends another player only these `GTM0` messages:

- chat, opcode [42](G/42.md), which a server rewrites with the authenticated
  sender's name before delivery;
- direct trade controls 21, 22, 23, 25, 2A and
  [A0/00 TrID](G/A0.md#a000-trid-and-trent), which a server relays only
  between a pair that has consented through the trade service
  ([trade.md](../flows/trade.md)).

Everything else beginning with `GTM0` and addressed to a player or a channel
is rejected, including channel-addressed trade controls that would bypass the
pair check. A server relays payloads that do not begin with `GTM0`
unchanged.

Server choice: a rejected relay closes the sending session and does not
notify the recipient.
