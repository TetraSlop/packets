# Writing guide

TetraPackets documents the PlayOnline Tetra Master client/server protocol as
recovered from the PC client (TM 1.6.11.0, image base 0x10000000) and from
captured traffic. It is a protocol reference for someone building a server.
Every page follows the template for its kind, uses the same vocabulary, and
reads top to bottom without the reader having seen the other pages.

## Voice

- Describe what the client sends, expects and does, and what a server must
  do to satisfy it. Keep the two apart: "the client", "a server".
- Where the original service's behaviour is unknown and a compatible server
  has to choose, say so once, in a paragraph that starts "Server choice:",
  placed last in its section. When there are several choices, follow
  "Server choice:" with a list, one choice per item.
- Plain sentences. No bold for emphasis, no em dashes, no rhetorical hedges.
- No working-notes language: a fact is stated, marked `(unverified)`, or
  left out.
- No client internals as explanation. Scene names, child and parent
  controllers, state numbers, flag addresses and memory offsets do not
  explain a message. Say what the client does ("the client waits for the
  Quit reply before returning to the room"), not where it stores a byte.
- A struct member comment says what the field is for. If nobody knows, the
  comment is `purpose unknown (unverified)`. A memory offset, a routine
  address or a register is never a field's meaning.
- Every field that appears in an example is a struct member or is explained
  in the sentence after the example.

## Vocabulary

Use these words and no synonyms.

- The five IRC families are `G`, `L`, `P`, `R`, `A`; the word "family"
  means the letter only.
- A family `G` header is `GTM0G` followed by the opcode (two hex digits),
  the reserved byte, the subtype (two hex digits) and the argument (two hex
  digits). Write them opcode, subtype, argument. Never "header byte",
  "session byte", "status byte" or "family" for the opcode. When a message
  uses the argument as a session number, a seat or a status, say so in that
  message's Argument line.
- A session is the number a service assigns at entry and the client repeats
  in the argument of every later message of the visit.
- Services are named in plain words, always the same way: the room service,
  the shop service, the Prize Center (which runs on the shop service), the
  trade service, the auction service, the ranking service, the event match
  service, the event card service, the registration service, the profile
  file service. The nickname tokens the client addresses them by (`Shm`,
  `E`, `F` and so on) are given once in [irc/services.md](irc/services.md)
  and otherwise appear only inside packet examples.
- Peers are "the inviter" and "the invitee", seats are "seat 0..2", the two
  trade sides are "side 0" and "side 1".
- Direction is `C2S` (client to server), `S2C` (server to client) or
  `P2P` (peer to peer, relayed). The service column names the service in
  plain words and, for `S2C`, the recipients when they are not simply the
  client ("room service to every human").
- Opcodes in hex with the subtype: `43/0A`. Numerics in decimal unless they
  are offsets, masks or addresses. Wire tokens as on the wire.

## Templates

Every page is one of six kinds and follows that kind's template exactly:
the same `##` sections, in the same order, with the same names. Optional
sections are marked; nothing else may be added.

### Opcode page, `irc/G/XX.md`

```
# XX Name

One paragraph: what the opcode is for, which service sends or receives it,
which flow pages use it (links).

## XX/YY Name

| Direction | Service | Argument | Reply | Body |
| --- | --- | --- | --- | --- |
| C2S | <service> | <session number, seat, 0, status: ...> | <link> or none | `struct Name` |

```c
struct Name { ... };
```

```
C2S GTM0G...
S2C GTM0G...
```

Notes: rules a server applies, what the client does with the message,
`Server choice:` sentences last.

### <Service>          (only when the body or a rule differs per service;
                        then every service gets one, with the same inner
                        order: struct, example, notes)

## Sequence            (optional, once, last: a worked exchange of this
                        opcode's messages in order)
```

Rules: one `## XX/YY` section per subtype, in subtype order, each subtype in
exactly one page across the repo. The table always comes first. Its Body
cell names the struct in the block that follows, or is `none` for a message
without a body (then no struct block), a link when the struct is defined
elsewhere, `unknown (unverified)` when the body is not recovered (then no
example), or `per service, below` when each `###` subsection has its own
struct. A subtype that carries both a request and a broadcast gets one row
per leg. When one subtype carries messages with different section names
(`TrID` and `TrEnt`), the section is `## XX/YY TrID and TrEnt` with one
`###` per message name, each with its own table.

### Family page, `irc/A.md`, `irc/L.md`, `irc/P.md`, `irc/R.md`

```
# X Name

One paragraph: what the family is for, which service, which flow pages.

## Framing

How a command is encoded in this family (tokens, separators, limits).

## <TOKEN> Name        (one per command, in the order the client sends them)

| Direction | Service | Reply | Body |
| --- | --- | --- | --- |
| C2S | <service> | <link> or none | `struct Name` |

```c
struct Name { ... };
```

```
C2S ...
S2C ...
```

Notes.

## Sequence            (optional)
## Error codes         (optional, last)
```

### Transport page, `irc/session.md`, `irc/envelope.md`, `irc/services.md`, `file/service.md`

```
# Name

One paragraph: what layer this is and what sits above and below it.

## <Step>              (one per step of the handshake or framing, in wire
                        order; each with its struct and example where a
                        record exists)
```

### Flow page, `flows/<topic>.md`

```
# Name

One paragraph: what the flow achieves, which services, where it starts.

## Sequence

A numbered list. Each step: direction, a link to the message section, what
it carries, what the server must do before the next step.

## Rules

Rules that span messages: eligibility, money, state the server holds between
steps, `Server choice:` sentences. One `###` per rule when there are several.

## Exit

How the flow ends early: quit, timeout, disconnect, rejection, and what the
server must undo.
```

A flow page never contains a struct or restates a layout.

### File page, `file/<name>.md` other than `service.md`

```
# Name

One paragraph: what the file holds and who reads or writes it.

## Access

Path pattern, the file service operations used, size, when the client
reads or deletes it.

## Layout

The structs, outermost first.

## Rules

What a server fills in, value bounds, `Server choice:` sentences.
```

### Client page, `client/<name>.md`

```
# Name

One paragraph: what data or computation this is and which messages depend
on it.

## <Data set or routine>   (one per item; a routine section states inputs,
                            steps and result; a data section gives the
                            struct and the value tables)
```

## Layout rules

- Field layouts are C structs in a `c` fenced block: one member per field,
  a trailing `//` comment for its meaning, `/* 0xNN */` offset prefixes when
  the record has fixed offsets, explicit `pad` members for gaps, member names
  taken from the wire tokens or the page's field names. Structs are
  byte-packed, little-endian unless the comment says otherwise. For a text
  message the struct is the decoded form: decimal token `uint32_t`
  (`int32_t` when it can be negative), hex string a byte array, repeated
  section an array sized to its documented maximum.
- Tables are for the message header (Direction, Service, Argument, Reply,
  Body), enumerations and value lists only.
- Packet examples go in fenced code blocks, exactly as on the wire, each
  line prefixed `C2S ` or `S2C ` (or `P2P ` for a relayed peer message).
- No citation or evidence lines. Routine addresses appear inline only where
  a sentence needs them. The only quoted source material is a verbatim
  decompiled or disassembled excerpt in a fenced block.
- Cross-link with relative links; link a message by its section
  (`B2.md#b213-init`).
