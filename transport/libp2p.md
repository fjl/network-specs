---
status: stable
---

# The libp2p Transport (Consensus Layer Profile)

This specification defines the libp2p transport as it is used by the Ethereum consensus
layer. [libp2p] is a modular networking stack. It supports many transports, security
protocols and stream multiplexers, which can be combined in many ways. This document picks
one combination of those components, called a *profile*, and describes it in enough
detail to explain what goes over the wire between two consensus-layer nodes.

The transport provides authenticated, encrypted connections between nodes. A connection
carries any number of independent, bidirectional, ordered byte *streams*. Each stream is
tagged with a *protocol ID* string that identifies the application protocol spoken on it.
The application protocols themselves, i.e. gossipsub and the Req/Resp protocols, are
specified in [Beacon Chain Networking](../consensus/beacon.md). Peer discovery is handled
by [Discovery v5](../discovery/discv5/discv5.md) and is also outside the scope of this
document.

This document summarizes the relevant parts of the upstream libp2p specifications and
states which options consensus-layer clients must use. Where it does not repeat every
detail of a component, the linked upstream specification is authoritative. Design
rationale can be found in the [consensus networking rationale](../consensus/rationale.md#transport).

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT" and "MAY" are to be interpreted
as described in [RFC 2119].

## Overview

Two transports are used: QUIC and TCP. QUIC is the primary transport. TCP is a fallback
for environments where UDP is unavailable or blocked.

The protocol stack for each transport is shown below. Both stacks provide the same
abstraction to the application: an authenticated connection to a peer, identified by its
peer ID, carrying multiplexed streams.

```text
        QUIC                                TCP
+-----------------------+       +-----------------------+
|  application protocol |       |  application protocol |
+-----------------------+       +-----------------------+
| multistream-select 1.0|       | multistream-select 1.0|  per stream
+-----------------------+       +-----------------------+
|                       |       |   yamux  /  mplex     |  stream multiplexing
|   QUIC v1 (RFC 9000)  |       +-----------------------+
|  streams + TLS 1.3    |       |        Noise XX       |  encryption, authentication
|                       |       +-----------------------+
|                       |       | multistream-select 1.0|  (negotiates layers above)
+-----------------------+       +-----------------------+
|          UDP          |       |          TCP          |
+-----------------------+       +-----------------------+
```

Establishing a connection over TCP proceeds in three steps:

1. Negotiate the security protocol using multistream-select. The only protocol is `/noise`.
2. Perform the Noise XX handshake, which authenticates both peers and encrypts all further
   traffic.
3. Negotiate the stream multiplexer using multistream-select, inside the encrypted
   channel. The multiplexer is `/yamux/1.0.0` or `/mplex/6.7.0`.

Over QUIC, encryption, authentication and stream multiplexing are all provided by QUIC
itself. Connection establishment is a single QUIC handshake.

Once a connection is up, either side may open streams. Every new stream begins with a
multistream-select exchange that selects the application protocol for that stream.

## Node Identity

### Identity Key

Every node has a long-term identity key pair. Consensus-layer nodes MUST use a
*secp256k1* key. The same key is used as the node's [ENR] identity key, so the libp2p
identity and the discovery identity of a node are always identical.

libp2p encodes public keys using the following protobuf message:

```protobuf
syntax = "proto2";

enum KeyType {
    RSA = 0;
    Ed25519 = 1;
    Secp256k1 = 2;
    ECDSA = 3;
}

message PublicKey {
    required KeyType Type = 1;
    required bytes Data = 2;
}
```

For a secp256k1 key, `Type` is `Secp256k1` and `Data` is the 33-byte compressed public key
as defined in [SEC1]. The encoding MUST be deterministic: both fields are present and
appear in field-number order. The encoded `PublicKey` message of a secp256k1 key is
therefore always 37 bytes long:

```text
08 02 12 21 <33 bytes compressed public key>
```

Signatures made with the identity key are ECDSA signatures over the SHA-256 hash of the
signed message, serialized in DER format.

### Peer ID

The *peer ID* is the identifier of a node in libp2p. It is derived from the encoded
`PublicKey` message. For keys whose encoding is at most 42 bytes long, which includes all
secp256k1 keys, the peer ID is the *identity* [multihash] of the encoded key, i.e. the key
itself, prefixed by the multihash code `0x00` and length:

```text
peer-id = 0x00 || 0x25 || encoded-public-key
```

The peer ID can be converted back into the public key without any further information.

In text form, peer IDs are represented in [base58btc] encoding. Due to the fixed prefix,
the text form of all consensus-layer peer IDs starts with `16Uiu2HA`. For example, the
peer ID of the public key `0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798`
is:

```text
16Uiu2HAm3cuhhRL2msUuLF62KRSfneFDx94RsuouyW25Ho42cFMq
```

Note that the peer ID is distinct from the [ENR] node ID, which is the keccak256 hash of
the uncompressed public key. Since both are derived from the same key, a node can convert
between them (if it has the public key).

## Addressing

libp2p addresses are expressed as [multiaddr]s. A multiaddr is a self-describing,
composable network address. Its text form is a sequence of `/protocol/value` components.
The following multiaddr forms are used by consensus-layer nodes:

| Transport | Multiaddr                                         |
|-----------|---------------------------------------------------|
| TCP, IPv4 | `/ip4/<address>/tcp/<port>/p2p/<peer-id>`         |
| TCP, IPv6 | `/ip6/<address>/tcp/<port>/p2p/<peer-id>`         |
| QUIC, IPv4| `/ip4/<address>/udp/<port>/quic-v1/p2p/<peer-id>` |
| QUIC, IPv6| `/ip6/<address>/udp/<port>/quic-v1/p2p/<peer-id>` |

The trailing `/p2p/<peer-id>` component is the expected identity of the remote node. When
dialing, the expected peer ID MUST be checked against the identity established by the
security handshake, and the connection MUST be closed if it does not match.

In the consensus layer, multiaddrs are not exchanged directly. Nodes learn about each other
through discovery, which yields [ENR]s, and convert them into multiaddrs as follows:

- The peer ID is derived from the `secp256k1` entry of the ENR.
- `ip` and `tcp` yield an IPv4 TCP address.
- `ip` and `quic` yield an IPv4 QUIC address.
- `ip6` and `tcp6` yield an IPv6 TCP address. If `tcp6` is absent, `tcp` is used.
- `ip6` and `quic6` yield an IPv6 QUIC address. If `quic6` is absent, `quic` is used.

Note that the `udp` and `udp6` entries of the ENR refer to the discovery port, not the
libp2p QUIC port.

## Transports

### Requirements

All implementations must support the QUIC transport and may support the TCP transport. Any
supported transport must be enabled for both dialing and listening, i.e. for both outbound
and inbound connections.

QUIC is the primary transport. When a peer is reachable over both TCP and QUIC, clients
should dial the QUIC address.

Clients must support listening on at least one of IPv4 or IPv6, and must be capable of
dialing both IPv4 and IPv6 addresses. Clients that do not support listening on IPv4 should
be aware that this may reduce their reachability.

All listening endpoints must be publicly dialable. The libp2p circuit relay, AutoNAT and
AutoRelay facilities are not used. Nodes operating behind a NAT or firewall, or which are
otherwise undialable by default, must have their infrastructure configured to allow
inbound traffic to the announced listening endpoint.

### QUIC

The QUIC transport uses QUIC version 1 as specified in [RFC 9000], running over UDP. Older
draft versions of QUIC (multiaddr component `/quic`) are not used.

QUIC connections are secured by TLS 1.3 as described in [RFC 9001]. The TLS handshake is
configured according to the [libp2p TLS specification][libp2p-tls], which is summarized
in the [TLS handshake](#tls-handshake) section below. The TLS ALPN protocol ID must be
`libp2p`.

QUIC natively provides multiplexed streams, so no further multiplexing layer is used.
Each libp2p stream is a bidirectional QUIC stream. Unidirectional QUIC streams are not
used. The stream operations described in [Stream Semantics](#stream-semantics) map onto
QUIC as follows:

- Closing the write side of a stream is done by sending a STREAM frame with the FIN bit.
- Resetting a stream is done by sending RESET_STREAM, and STOP_SENDING if the read side is
  still open.

Implementations MAY support 0-RTT. Note that application data sent in 0-RTT is not
protected against replay.

### TCP

On TCP, the libp2p connection upgrade process transforms the raw TCP connection into a
secure, multiplexed connection. All negotiation uses multistream-select (see
[Protocol Negotiation](#protocol-negotiation)). The dialer of the TCP connection is the
initiator of each negotiation.

1. **Security negotiation.** The initiator proposes `/noise`. The responder must accept it.
   Clients must NOT use any other security protocol. In particular, SecIO is not supported.
2. **Noise handshake.** The peers run the [Noise handshake](#noise-handshake). All
   subsequent bytes on the connection are encrypted.
3. **Multiplexer negotiation.** Inside the encrypted channel, the initiator proposes one or
   more stream multiplexers. Clients must support `/mplex/6.7.0` and may support
   `/yamux/1.0.0`. If a client supports both, it must prefer yamux, i.e. it must propose
   `/yamux/1.0.0` first when dialing.

After step 3 the connection is ready, and streams can be opened as specified by the chosen
multiplexer.

Implementations may perform step 3 during step 2 using Noise early muxer negotiation as
described in [Noise Extensions](#noise-extensions).

An example exchange of a TCP connection setup is shown below. Lines prefixed with `>` are
sent by the dialer, `<` by the listener. Each multistream-select message is sent with a
varint length prefix which is not shown.

```text
> /multistream/1.0.0\n
> /noise\n
< /multistream/1.0.0\n
< /noise\n
> [Noise handshake message 1]
< [Noise handshake message 2]
> [Noise handshake message 3]
  ---- all following messages are encrypted ----
> /multistream/1.0.0\n
> /yamux/1.0.0\n
< /multistream/1.0.0\n
< /yamux/1.0.0\n
  ---- connection is ready, yamux frames follow ----
```

## Security Handshakes

### Noise Handshake

On TCP, the secure channel is established using the [Noise protocol framework][Noise] as
profiled by the [libp2p Noise specification][libp2p-noise]. The Noise protocol name is

```text
Noise_XX_25519_ChaChaPoly_SHA256
```

This means the handshake uses the XX pattern, X25519 for Diffie-Hellman, ChaCha20-Poly1305
as the AEAD cipher and SHA-256 as the hash function. Clients must support this handshake.
The Noise prologue is empty.

The X25519 *static* key used by Noise is separate from the node's secp256k1 identity key.
It is bound to the identity key by a signature in the handshake payload. Implementations
should generate a fresh static key at startup or per connection.

#### Handshake messages

The XX pattern consists of three messages:

```text
XX:
  -> e
  <- e, ee, s, es
  -> s, se
```

Every handshake message is prefixed with its length, encoded as a 2-byte big-endian
unsigned integer.

The first message carries the initiator's ephemeral key and an empty payload. The second
message carries the responder's static key and handshake payload, and the third message
carries the initiator's static key and handshake payload. Both payloads are encrypted by
the Noise protocol.

The handshake payload has the following protobuf encoding:

```protobuf
syntax = "proto2";

message NoiseExtensions {
    repeated bytes webtransport_certhashes = 1;
    repeated string stream_muxers = 2;
}

message NoiseHandshakePayload {
    optional bytes identity_key = 1;
    optional bytes identity_sig = 2;
    optional NoiseExtensions extensions = 4;
}
```

- `identity_key` is the sender's identity public key in the `PublicKey` encoding defined in
  [Identity Key](#identity-key).
- `identity_sig` is a signature made with the sender's identity key over the message

  ```text
  "noise-libp2p-static-key:" || noise-static-public-key
  ```

  where `noise-static-public-key` is the sender's 32-byte X25519 static key.
- `extensions` is optional. See [Noise Extensions](#noise-extensions).

On receiving a handshake payload, a node must verify `identity_sig` against the Noise
static key `s` received in the same message and the public key given in `identity_key`.
If verification fails, the connection must be closed. The peer ID of the remote node is
derived from `identity_key`. If the remote peer ID was known in advance (i.e. when
dialing), it must match the derived peer ID.

#### Transport messages

When the handshake is complete, both sides derive a pair of cipher states, one for each
direction. All further data is sent in Noise transport messages. Each message is prefixed
by its length, encoded as a 2-byte big-endian unsigned integer. A Noise message is at most
65535 bytes long, including the 16-byte authentication tag, so a single message can carry
up to 65519 bytes of plaintext. Larger writes are split into multiple messages.

The AEAD nonce is a 64-bit counter which is incremented for every message. A node must
close the connection before the counter would wrap around.

#### Noise Extensions

The `stream_muxers` field of `NoiseExtensions` allows negotiating the stream multiplexer
during the Noise handshake, saving one round trip. The responder lists its supported
multiplexer protocol IDs in message 2, and the initiator lists its own in message 3, in
order of preference. The selected multiplexer is the first entry in the initiator's list
which also appears in the responder's list.

If either side does not send `stream_muxers`, or the lists have no entry in common, the
multiplexer is negotiated using multistream-select after the handshake, as described in
[TCP](#tcp).

The `webtransport_certhashes` field is not used.

### TLS Handshake

On QUIC, the secure channel is established using TLS 1.3 as profiled by the
[libp2p TLS specification][libp2p-tls]. Since TLS identity is normally based on a
certificate authority, libp2p uses a special self-signed certificate to convey the node
identity.

Each endpoint generates a self-signed X.509 certificate with a fresh, randomly generated
*certificate key*. The certificate key is not the identity key. It should be an ECDSA key
on the P-256 curve, or an Ed25519 key. The certificate carries a libp2p *public key
extension* with object identifier `1.3.6.1.4.1.53594.1.1`. The extension may be marked
critical. Its value is the DER encoding of the following ASN.1 structure:

```asn1
SignedKey ::= SEQUENCE {
    publicKey OCTET STRING,
    signature OCTET STRING
}
```

- `publicKey` is the node's identity public key in the `PublicKey` encoding defined in
  [Identity Key](#identity-key).
- `signature` is a signature made with the identity key over the message

  ```text
  "libp2p-tls-handshake:" || certificate-public-key
  ```

  where `certificate-public-key` is the DER-encoded `SubjectPublicKeyInfo` of the
  certificate key.

Both endpoints must present a certificate, i.e. the server must request a client
certificate. The certificate chain must consist of exactly one certificate. On receiving
a certificate, a node must

- check that the certificate is currently valid and its self-signature is correct,
- verify `signature` in the public key extension against `publicKey` and the certificate's
  `SubjectPublicKeyInfo`, and
- derive the remote peer ID from `publicKey`. If the remote peer ID was known in advance,
  it must match.

If any of these checks fails, the handshake must be aborted.

## Protocol Negotiation

libp2p uses [multistream-select 1.0][multistream-select] to agree on a protocol for a
connection or stream. On TCP, it negotiates the security protocol and multiplexer as
described above. On all transports, it is used on every new stream to select the
application protocol.

### Message Format

Every multistream-select message is a UTF-8 string terminated by a newline character
(`\n`), prefixed with the length of the message including the newline, encoded as an
unsigned [varint]:

```text
message = uvarint(len(text) + 1) || text || "\n"
```

### Negotiation

1. Both sides send the multistream-select protocol ID `/multistream/1.0.0` as the first
   message. Each side must check that the other sent the same header and abort otherwise.
2. The initiator sends a protocol ID that it wants to use.
3. If the responder supports the protocol, it echoes the protocol ID back, and negotiation
   is complete. All further data on the connection or stream belongs to the selected
   protocol.
4. If the responder does not support the protocol, it replies with `na`. The initiator may
   then propose another protocol ID by going back to step 2, or give up and close the
   stream.

The initiator MAY send its header and first proposal together without waiting for the
responder's header. Once the initiator has sent its proposal, it MAY also send
application data for the proposed protocol before receiving the response.

Clients must use exact equality when comparing protocol IDs. Consensus-layer protocol IDs
carry an explicit version, and clients MAY use the version number to prefer newer versions
of a protocol, e.g. by proposing them first.

Example of an application protocol negotiated on a new stream:

```text
> /multistream/1.0.0\n
> /eth2/beacon_chain/req/status/2/ssz_snappy\n
< /multistream/1.0.0\n
< /eth2/beacon_chain/req/status/2/ssz_snappy\n
> [request]
< [response]
```

Example of a rejected proposal:

```text
> /multistream/1.0.0\n
> /eth2/beacon_chain/req/status/3/ssz_snappy\n
< /multistream/1.0.0\n
< na\n
> /eth2/beacon_chain/req/status/2/ssz_snappy\n
< /eth2/beacon_chain/req/status/2/ssz_snappy\n
```

## Stream Multiplexing

On TCP connections, a stream multiplexer runs on top of the encrypted channel and divides
it into multiple independent streams. The two multiplexers used by consensus clients are
yamux and mplex.

### Yamux

[Yamux] is the preferred multiplexer, with protocol ID `/yamux/1.0.0`. It provides
per-stream flow control, which prevents a single busy stream from blocking the others.

All data is sent in frames. Each frame starts with a 12-byte header. All integers are
big-endian.

```text
+---------+--------+---------+-------------+------------+
| version |  type  |  flags  |  stream ID  |   length   |
| 1 byte  | 1 byte | 2 bytes |   4 bytes   |  4 bytes   |
+---------+--------+---------+-------------+------------+
```

`version` is always zero. `type` is one of:

| Type | Name          | Meaning of `length`                       |
|------|---------------|-------------------------------------------|
| 0x0  | Data          | number of payload bytes following the header |
| 0x1  | Window Update | increment of the receive window, in bytes |
| 0x2  | Ping          | opaque value, echoed in the response      |
| 0x3  | Go Away       | error code                                |

Only Data frames have a payload. `flags` is a combination of:

| Flag | Name | Meaning                                   |
|------|------|-------------------------------------------|
| 0x1  | SYN  | opens a new stream; on Ping, a ping request |
| 0x2  | ACK  | acknowledges a new stream; on Ping, a ping response |
| 0x4  | FIN  | closes the sender's write side of the stream |
| 0x8  | RST  | resets the stream                         |

Stream IDs are chosen by the side opening the stream. The dialer of the underlying
connection uses odd IDs, the listener uses even IDs. ID 0 refers to the session as a whole
and is used by Ping and Go Away frames.

A stream is opened by sending a Data or Window Update frame with the SYN flag. The
receiving side acknowledges it with the ACK flag on its first frame for the stream.

Each stream starts with a receive window of 256 KiB in each direction. A sender must NOT
send more data on a stream than the receiver's current window allows. The receiver grows
the window by sending Window Update frames as it consumes data.

Go Away frames terminate the session. The error code is 0 for normal termination, 1 for a
protocol error and 2 for an internal error.

### Mplex

[Mplex] is a simpler multiplexer with protocol ID `/mplex/6.7.0`. It does not provide
flow control and therefore suffers from head-of-line blocking. Mplex is deprecated in the
wider libp2p ecosystem but remains a requirement for consensus-layer clients, as described
in [TCP](#tcp).

Each mplex message has the form

```text
message = uvarint(stream-id << 3 | flag) || uvarint(len(data)) || data
```

where `flag` is one of:

| Flag | Name             | Sent by                                 |
|------|------------------|-----------------------------------------|
| 0    | NewStream        | opener of the stream                    |
| 1    | MessageReceiver  | side that did not open the stream       |
| 2    | MessageInitiator | opener of the stream                    |
| 3    | CloseReceiver    | side that did not open the stream       |
| 4    | CloseInitiator   | opener of the stream                    |
| 5    | ResetReceiver    | side that did not open the stream       |
| 6    | ResetInitiator   | opener of the stream                    |

Stream IDs are chosen by the opener of the stream. Since the flag identifies which side
opened a stream, both sides may use the same numeric ID for different streams. The data of
a NewStream message is an optional stream name and is otherwise ignored. The data of a
Close or Reset message is empty.

The data of a single mplex message must NOT exceed 1 MiB.

## Stream Semantics

Regardless of the transport, application protocols see streams with the following
properties:

- A stream is a bidirectional, reliable, ordered byte stream.
- Either side of a connection may open a stream at any time. The side opening the stream
  is the initiator of the multistream-select negotiation on that stream.
- Each side may *close* its write side of the stream independently. This signals EOF to the
  reader on the other side, while data may still be sent in the opposite direction. A
  stream in which only one side has closed is *half-closed*. The stream is fully closed when
  both sides have closed.
- Either side may *reset* the stream. This aborts the stream in both directions
  immediately. Data that was not yet delivered may be lost.

The consensus-layer Req/Resp protocols rely on half-closure: the requester closes its
write side after sending the request, and the responder closes its write side after
sending the response.

## Application Protocols

The following protocols run on streams of libp2p connections between consensus-layer
nodes:

| Protocol ID                                          | Description                 | Requirement |
|------------------------------------------------------|-----------------------------|-------------|
| `/meshsub/1.1.0`                                     | [gossipsub][beacon-gossip]  | MUST        |
| `/eth2/beacon_chain/req/<name>/<version>/ssz_snappy` | [Req/Resp][beacon-reqresp]  | MUST        |
| `/ipfs/id/1.0.0`                                     | [libp2p identify][identify] | MAY         |

Clients MAY use the libp2p identify protocol to exchange information such as client
version and listening addresses. This protocol is widely deployed, but it is not required
for interoperability, and its contents must not be relied upon for consensus-layer
functionality. The libp2p ping protocol (`/ipfs/ping/1.0.0`) is not required. Liveness is
checked using the consensus-layer `Ping` Req/Resp method instead.

## References

- [libp2p specifications][libp2p]
- [libp2p peer IDs and keys][peer-ids]
- [libp2p connection establishment][connections]
- [libp2p Noise specification][libp2p-noise]
- [libp2p TLS specification][libp2p-tls]
- [libp2p QUIC specification][libp2p-quic]
- [multistream-select 1.0][multistream-select]
- [Yamux specification][Yamux]
- [Mplex specification][Mplex]
- [The Noise Protocol Framework][Noise]
- [RFC 9000: QUIC: A UDP-Based Multiplexed and Secure Transport][RFC 9000]
- [RFC 9001: Using TLS to Secure QUIC][RFC 9001]

[libp2p]: https://github.com/libp2p/specs
[peer-ids]: https://github.com/libp2p/specs/blob/master/peer-ids/peer-ids.md
[connections]: https://github.com/libp2p/specs/blob/master/connections/README.md
[libp2p-noise]: https://github.com/libp2p/specs/blob/master/noise/README.md
[libp2p-tls]: https://github.com/libp2p/specs/blob/master/tls/tls.md
[libp2p-quic]: https://github.com/libp2p/specs/blob/master/quic/README.md
[identify]: https://github.com/libp2p/specs/blob/master/identify/README.md
[multistream-select]: https://github.com/multiformats/multistream-select
[multiaddr]: https://github.com/multiformats/multiaddr
[multihash]: https://github.com/multiformats/multihash
[varint]: https://github.com/multiformats/unsigned-varint
[base58btc]: https://datatracker.ietf.org/doc/html/draft-msporny-base58-03
[Yamux]: https://github.com/libp2p/specs/blob/master/yamux/README.md
[Mplex]: https://github.com/libp2p/specs/blob/master/mplex/README.md
[Noise]: https://noiseprotocol.org/noise.html
[RFC 2119]: https://www.rfc-editor.org/rfc/rfc2119
[RFC 9000]: https://www.rfc-editor.org/rfc/rfc9000
[RFC 9001]: https://www.rfc-editor.org/rfc/rfc9001
[SEC1]: https://www.secg.org/sec1-v2.pdf
[ENR]: ../discovery/enr.md
[beacon-gossip]: ../consensus/beacon.md#the-gossip-domain-gossipsub
[beacon-reqresp]: ../consensus/beacon.md#the-reqresp-domain
