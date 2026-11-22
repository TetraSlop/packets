# Game IRC session

The game connection is a PlayOnline IRC dialect on TCP port 51241. Below it
is a plain TCP stream; above it are the service messages of
[envelope.md](envelope.md), each the trailing argument of an ordinary
`NOTICE` or `PRIVMSG` to a service nickname ([services.md](services.md)).
Every line is checksummed, and after the key exchange every line is
encrypted. The profile file service on a separate port is described in
[service.md](../file/service.md).

## Connection

| Service | Host seen in the client | TCP port |
|---|---|---|
| Game IRC | gi000.pol.com | 51241 |
| Profile file service | pp000.pol.com | 51220 |
| Main Viewer IRC | not used by the game client | 51240 |

After login the client closes and reopens Game IRC when the player selects
Play, using the host named in the zone file record, and reopens it again
with the selected zone's host (see
[directory-files.md](../file/directory-files.md)). Each reopen runs the
whole handshake below again with a new session key.

## Line framing

Each line is `message + checksum(4 chars) + CRLF`, at most 512 bytes
including the six framing bytes. The checksum is the sum of the message's
little-endian 32-bit words; trailing bytes (fewer than four) are shifted into
the high end of a final word (`tail = (tail >> 8) | (byte << 24)` per byte).
Four characters are emitted from bit offsets 26, 20, 14 and 8 of the sum, each
`((sum >> offset) & 0x3F) + 0x3F`.

```
C2S PING aiExH\r\n
```

The message is `PING a` and `iExH` is its checksum. A server should drop a
line whose checksum does not match and close after repeated mismatches.
Messages never contain NUL, CR or LF.

## Alphabets

| Encoding | Alphabet | Used for |
|---|---|---|
| base64 | `TSG8IncW3HFKokOg79qzeCmZs2yBYEQVAUxR5rbwi4P@jMDLtpvad0f_J1hlN6uX` | RSA modulus in USER, wrapped key in 300 |
| base32 | `N43OVHBJ1Y2C0WSXED5QFILRZMUTAPGK` | challenge |
| base36 | `EFKAOYMJVNGTDSWBQLPCIRHZXU6328401795` | POL ID strings and scrambled nicknames |

Base64 groups of fewer than three bytes are padded with `T` (value 0) rather
than `=`; the client sends the 32-byte modulus unpadded as 43 characters.

## Challenge

The server's first line is a plaintext `NOTICE AUTH` carrying a 40-character
base32 challenge of a 25-byte blob.

```c
struct Challenge {
    /* 0x00 */ uint32_t timestamp;      // big-endian
    /* 0x04 */ uint32_t address;        // client IPv4 address, big-endian
    /* 0x08 */ uint8_t pad_08[12];      // zero
    /* 0x14 */ uint16_t port;           // server port, big-endian
    /* 0x16 */ uint8_t pad_16[3];       // zero
};
```

```
S2C :srv NOTICE AUTH <40 base32 chars>
```

The client does not decode the challenge; it only hashes the 40 characters
into the NICK digest.

## USER

The client replies in plaintext with `USER` whose trailing argument is its
256-bit RSA public modulus, little-endian, custom base64, 43 characters. The
three leading arguments are not interpreted by a server.

```
C2S USER x 8 * :<43 base64 chars>
```

## 300 Session key

The server generates an 8-byte Blowfish session key and returns numeric 300.

```
S2C :srv 300 * <44 base64 chars>
```

The 32-byte block is PKCS#1 type 2: `00 02`, 29 nonzero padding bytes, `00`,
then the session key reversed. It is raised to exponent 65535 modulo the
client's modulus and the 32-byte big-endian result is base64 encoded. The
client decrypts, reverses the last eight bytes and keys SqBlowfish. From the
next line on, both directions are encrypted.

## SqBlowfish and per-line OFB

The key schedule is not standard Blowfish:

1. Extend the 8-byte key to 0x1000 bytes by repeated MD5: the first digest
   uses the standard MD5 IV, every later digest starts from an all-zero IV
   (the finalizer zeroes the context) and each digest is appended to the
   buffer.
2. Read P[0..17] as big-endian words at byte offsets 0..17 (overlapping, one
   byte stride) and S[b][j] as the big-endian word at `4 * (256 * b + j)`.
3. XOR P[i] with the big-endian word at `4 * i`.
4. Replace each P pair and each S pair with the encryption of the zero block
   (the input is zero every time; outputs are not chained).

Rounds and the F function are standard 16-round Blowfish.

Per line the cipher runs in OFB with reset: the state starts at the IV, which
is the first eight bytes of the RSA modulus as a little-endian integer; each
8-byte block of keystream is `state = E(state)`, and byte `i` is XORed with
keystream byte `i mod 8`. A plaintext byte that is CR or LF, or whose
ciphertext would be CR or LF, is passed through unchanged so line framing
survives. The keystream restarts from the IV at every line.

## NICK

The first encrypted line is `NICK` with one colon-separated argument.

```c
struct Nick {
    char id[9];         // scrambled POL ID, `U` + 8 base36 digits, see below
    char digest[32];    // lowercase hex MD5 of `challenge(40 chars) + password`
    char info[];        // opaque client string
};
```

```
C2S NICK <scrambled POL ID>:<md5 hex>:<client info>
S2C :srv 422 <nick> :no motd
```

The scrambled ID is derived from the 41-bit POL identity: OR 0xC00000000000,
then for k = 4 down to 0 `v ^= (v >> 8) & (0xFF << 8k)`, mask with
0xFFFF3FFFFFFFFFFF, and write eight base36 digits (most significant first)
with the POL ID alphabet. The plain 8-character login ID uses the same digit
alphabet without scrambling. The same scrambled form names every service and
player a message is addressed to ([services.md](services.md)).

A server must check that the account exists, that the digest matches the
challenge it issued, and that the account is not already connected, before
accepting the session. On success it answers numeric 422, which the client
treats as login complete.

## Verbs after login

| Verb | Direction | Notes |
|---|---|---|
| `PING <token>` | either | answered with `PONG <token>` |
| `QUIT` | client | closes the session |
| `JOIN #channel` | client | server echoes `:<nick>!p@local JOIN :#channel` to every member, then 353 name lists and 366 |
| `PART #channel` | client | server echoes PART to every member |
| `NOTICE <target> :<payload>` | either | service request, reply or peer relay |
| `PRIVMSG <target> :<payload>` | either | same handling as NOTICE |

The target of a `NOTICE` or `PRIVMSG` is a channel or the scrambled
nickname of a 64-bit identity, which names a service or a player. Service
replies carry the service nickname in the prefix:

```
S2C :UUIRT71SR!p@local NOTICE UGITW5T4U :GTM0P<CS>\x07\x07
```

`UUIRT71SR` is the service's nickname and `UGITW5T4U` the player's. The
client extracts the sender from that prefix, so the prefix nickname must be
at most nine characters. A 353 reply lists whole names and is split across
lines to stay inside 512 bytes. Numeric 442 answers a message to a channel
the sender has not joined; 421 answers an unknown verb.

## Routing

What a server validates before it routes a line:

1. The session has completed NICK; nothing is routed before 422.
2. The trailing payload is at most 469 bytes and contains no NUL, CR or LF
   (see [envelope.md](envelope.md#length-limit) for where 469 comes from).
3. A channel target is one the sender has joined.
4. Service messages (payloads addressed to a service nickname) are handled
   before any relay and are never forwarded to another player.
5. Peer-to-peer payloads that begin with `GTM0` are rejected unless they are
   normalized chat or consent-checked trade controls; see the relay guard in
   [envelope.md](envelope.md#relay-guard).
6. A server relays payloads that do not begin with `GTM0` unchanged.

Server choice: a server that rejects a relayed payload closes the sender and
emits no control message to the recipient.
