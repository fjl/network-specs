# Ethereum Consensus Layer Networking -- Design Rationale

This document collects the non-normative design rationale for the consensus-layer
networking protocol. It answers "why" questions about the transport, gossip, Req/Resp,
discovery and encoding choices.

The content was originally part of the Phase 0, Bellatrix and Deneb networking
specifications in
[consensus-specs](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#design-decision-rationale).
Questions that were added with a later fork are marked with the version that
introduced them.

<!-- toc start -->

- [Transport](#transport)
  - [Why are we defining specific transports?](#why-are-we-defining-specific-transports)
  - [Can clients support other transports/handshakes than the ones mandated by the specification?](#can-clients-support-other-transportshandshakes-than-the-ones-mandated-by-the-specification)
  - [What are the advantages of using TCP/QUIC/Websockets?](#what-are-the-advantages-of-using-tcpquicwebsockets)
  - [Why do we not just support a single transport?](#why-do-we-not-just-support-a-single-transport)
  - [Why is QUIC the primary transport?](#why-is-quic-the-primary-transport)
- [Multiplexing](#multiplexing)
  - [Why are we using mplex/yamux?](#why-are-we-using-mplexyamux)
- [Protocol negotiation](#protocol-negotiation)
  - [When is multiselect 2.0 due and why do we plan to migrate to it?](#when-is-multiselect-20-due-and-why-do-we-plan-to-migrate-to-it)
  - [What is the difference between connection-level and stream-level protocol negotiation?](#what-is-the-difference-between-connection-level-and-stream-level-protocol-negotiation)
- [Encryption](#encryption)
  - [Why are we not supporting SecIO?](#why-are-we-not-supporting-secio)
  - [Why are we using Noise?](#why-are-we-using-noise)
  - [Why are we using encryption at all?](#why-are-we-using-encryption-at-all)
- [Gossipsub](#gossipsub)
  - [Why are we using a pub/sub algorithm for block and attestation propagation?](#why-are-we-using-a-pubsub-algorithm-for-block-and-attestation-propagation)
  - [Why are we using topics to segregate encodings, yet only support one encoding?](#why-are-we-using-topics-to-segregate-encodings-yet-only-support-one-encoding)
  - [How do we upgrade gossip channels (e.g. changes in encoding, compression)?](#how-do-we-upgrade-gossip-channels-eg-changes-in-encoding-compression)
  - [Why must all clients use the same gossip topic instead of one negotiated between each peer pair?](#why-must-all-clients-use-the-same-gossip-topic-instead-of-one-negotiated-between-each-peer-pair)
  - [Why are the topics strings and not hashes?](#why-are-the-topics-strings-and-not-hashes)
  - [Why are we using the `StrictNoSign` signature policy?](#why-are-we-using-the-strictnosign-signature-policy)
  - [Why are we overriding the default libp2p pubsub `message-id`?](#why-are-we-overriding-the-default-libp2p-pubsub-message-id)
  - [Why are these specific gossip parameters chosen?](#why-are-these-specific-gossip-parameters-chosen)
  - [Why is there `MAXIMUM_GOSSIP_CLOCK_DISPARITY` when validating slot ranges of messages in gossip subnets?](#why-is-there-maximum_gossip_clock_disparity-when-validating-slot-ranges-of-messages-in-gossip-subnets)
  - [Why are there `ATTESTATION_SUBNET_COUNT` attestation subnets?](#why-are-there-attestation_subnet_count-attestation-subnets)
  - [Why are attestations limited to be broadcast on gossip channels within `SLOTS_PER_EPOCH` slots?](#why-are-attestations-limited-to-be-broadcast-on-gossip-channels-within-slots_per_epoch-slots)
  - [Why are aggregate attestations broadcast to the global topic as `AggregateAndProof`s rather than just as `Attestation`s?](#why-are-aggregate-attestations-broadcast-to-the-global-topic-as-aggregateandproofs-rather-than-just-as-attestations)
  - [Why are we sending entire objects in the pubsub and not just hashes?](#why-are-we-sending-entire-objects-in-the-pubsub-and-not-just-hashes)
  - [Should clients gossip blocks if they *cannot* validate the proposer signature due to not yet being synced, not knowing the head block, etc?](#should-clients-gossip-blocks-if-they-cannot-validate-the-proposer-signature-due-to-not-yet-being-synced-not-knowing-the-head-block-etc)
  - [How are we going to discover peers in a gossipsub topic?](#how-are-we-going-to-discover-peers-in-a-gossipsub-topic)
  - [How should fork version be used in practice?](#how-should-fork-version-be-used-in-practice)
  - [Why was the max gossip message size increased at Bellatrix?](#why-was-the-max-gossip-message-size-increased-at-bellatrix)
  - [Why are blobs relayed as a sidecar, separate from beacon blocks?](#why-are-blobs-relayed-as-a-sidecar-separate-from-beacon-blocks)
- [Req/Resp](#reqresp)
  - [Why segregate requests into dedicated protocol IDs?](#why-segregate-requests-into-dedicated-protocol-ids)
  - [Why are messages length-prefixed with a protobuf varint in the SSZ-encoding?](#why-are-messages-length-prefixed-with-a-protobuf-varint-in-the-ssz-encoding)
  - [Why do we version protocol strings with ordinals instead of semver?](#why-do-we-version-protocol-strings-with-ordinals-instead-of-semver)
  - [Why is it called Req/Resp and not RPC?](#why-is-it-called-reqresp-and-not-rpc)
  - [What is a typical rate limiting strategy?](#what-is-a-typical-rate-limiting-strategy)
  - [Why do we allow empty responses in block requests?](#why-do-we-allow-empty-responses-in-block-requests)
  - [Why does `BeaconBlocksByRange` let the server choose which branch to send blocks from?](#why-does-beaconblocksbyrange-let-the-server-choose-which-branch-to-send-blocks-from)
  - [Why are `BlocksByRange` requests only required to be served for the latest `compute_min_epochs_for_block_requests()` epochs?](#why-are-blocksbyrange-requests-only-required-to-be-served-for-the-latest-compute_min_epochs_for_block_requests-epochs)
  - [Why must the proposer signature be checked when backfilling blocks in the database?](#why-must-the-proposer-signature-be-checked-when-backfilling-blocks-in-the-database)
  - [What's the effect of empty slots on the sync algorithm?](#whats-the-effect-of-empty-slots-on-the-sync-algorithm)
  - [Why was the max chunk response size increased at Bellatrix?](#why-was-the-max-chunk-response-size-increased-at-bellatrix)
  - [Why allow invalid payloads on the P2P network?](#why-allow-invalid-payloads-on-the-p2p-network)
- [Discovery](#discovery)
  - [Why are we using discv5 and not libp2p Kademlia DHT?](#why-are-we-using-discv5-and-not-libp2p-kademlia-dht)
  - [What is the difference between an ENR and a multiaddr, and why are we using ENRs?](#what-is-the-difference-between-an-enr-and-a-multiaddr-and-why-are-we-using-enrs)
  - [Why do we not form ENRs and find peers until genesis block/state is known?](#why-do-we-not-form-enrs-and-find-peers-until-genesis-blockstate-is-known)
- [Compression/Encoding](#compressionencoding)
  - [Why are we using SSZ for encoding?](#why-are-we-using-ssz-for-encoding)
  - [Why are we compressing, and at which layers?](#why-are-we-compressing-and-at-which-layers)
  - [Why are we using Snappy for compression?](#why-are-we-using-snappy-for-compression)
  - [Can I get access to unencrypted bytes on the wire for debugging purposes?](#can-i-get-access-to-unencrypted-bytes-on-the-wire-for-debugging-purposes)
  - [What are SSZ type size bounds?](#what-are-ssz-type-size-bounds)
  - [Why is the message size defined in terms of application payload?](#why-is-the-message-size-defined-in-terms-of-application-payload)
  - [Why is there a limit on message sizes at all?](#why-is-there-a-limit-on-message-sizes-at-all)

<!-- toc end -->

## Transport

### Why are we defining specific transports?

libp2p peers can listen on multiple transports concurrently, and these can
change over time. Multiaddrs encode not only the address but also the transport
to be used to dial.

Due to this dynamic nature, agreeing on specific transports like TCP, QUIC, or
WebSockets on paper becomes irrelevant.

However, it is useful to define a minimum baseline for interoperability
purposes.

### Can clients support other transports/handshakes than the ones mandated by the specification?

Beyond the mandated QUIC transport and the TCP fallback, clients may support
other transports such as WebSockets and WebRTC, if available in the language of
choice. While interoperability shall not be harmed by lack of such support, the
advantages are desirable:

- Paving the way for interfacing with future light clients (WebSockets, WebRTC).

The usage of one handshake procedure or the other shall be transparent to the
application layer, once the libp2p Host/Node object has been configured
appropriately.

### What are the advantages of using TCP/QUIC/Websockets?

TCP is a reliable, ordered, full-duplex, congestion-controlled network protocol
that powers much of the Internet as we know it today. HTTP/1.1 and HTTP/2 run
atop TCP.

QUIC is a UDP-based transport standardized by the IETF in
[RFC 9000](https://www.rfc-editor.org/rfc/rfc9000). It’s reliable, ordered,
multiplexed, natively secure (TLS 1.3), reduces connection establishment latency
vs. TCP, offers stream-level flow control (thus removing head-of-line blocking),
0-RTT connection establishment, and endpoint migration, amongst other features.
UDP also has better NAT traversal properties than TCP—something we desperately
pursue in peer-to-peer networks. These characteristics are why QUIC is the
primary transport for the consensus-layer network.

QUIC is also the underlying protocol for HTTP/3. This has the potential to award
us censorship resistance against deep packet inspection for free. Provided that
we use the same port numbers and encryption mechanisms as HTTP/3, our traffic
may be indistinguishable from standard web traffic, and we may only become
subject to standard IP-based firewall filtering—something we can counteract via
other mechanisms.

WebSockets and/or WebRTC transports are necessary for interaction with browsers,
and will become increasingly important as we incorporate browser-based light
clients to the Ethereum network.

### Why do we not just support a single transport?

Networks evolve. Hardcoding design decisions leads to ossification, preventing
the evolution of networks alongside the state of the art. Introducing changes on
an ossified protocol is very costly, and sometimes, downright impracticable
without causing undesirable breakage.

Modeling for upgradeability and dynamic transport selection from the get-go lays
the foundation for a future-proof stack.

Clients can adopt new transports without breaking old ones, and the
multi-transport ability enables constrained and sandboxed environments (e.g.
browsers, embedded devices) to interact with the network as first-class citizens
via suitable/native transports (e.g. WSS), without the need for proxying or
trust delegation to servers.

### Why is QUIC the primary transport?

When this networking specification was first written, the QUIC standard was not
yet finalized (working draft 22 at the time), and mainstream runtimes lacked
mature, interoperable QUIC support. TCP was therefore mandated as the baseline
transport, with QUIC offered as an optional improvement.

QUIC has since been standardized as
[RFC 9000](https://www.rfc-editor.org/rfc/rfc9000) and is widely implemented
across the libp2p ecosystem. Its native stream multiplexing, built-in TLS 1.3
security, lower connection-establishment latency (including 0-RTT), and superior
NAT-traversal properties make it a better fit for the consensus-layer p2p
network than TCP. It is therefore now the primary transport, with TCP retained
as a fallback for environments where QUIC or UDP is unavailable or blocked.

## Multiplexing

### Why are we using mplex/yamux?

[Yamux](https://github.com/hashicorp/yamux/blob/master/spec.md) is a multiplexer
invented by Hashicorp that supports stream-level congestion control.
Implementations exist in a limited set of languages, and it’s not a trivial
piece to develop.

Conscious of that, the libp2p community conceptualized
[mplex](https://github.com/libp2p/specs/blob/master/mplex/README.md) as a
simple, minimal multiplexer for usage with libp2p. It does not support
stream-level congestion control and is subject to head-of-line blocking.

Overlay multiplexers are not necessary with QUIC since the protocol provides
native multiplexing, but they need to be layered atop TCP, WebSockets, and other
transports that lack such support.

## Protocol negotiation

### When is multiselect 2.0 due and why do we plan to migrate to it?

multiselect 2.0 is currently being conceptualized. The debate started
[on this issue](https://github.com/libp2p/specs/pull/95), but it got
overloaded—as it tends to happen with large conceptual OSS discussions that
touch the heart and core of a system.

At some point in 2020, we expect a renewed initiative to first define the
requirements, constraints, assumptions, and features, in order to lock in basic
consensus upfront and subsequently build on that consensus by submitting a
specification for implementation.

We plan to eventually migrate to multiselect 2.0 because it will:

1. Reduce round trips during connection bootstrapping and stream protocol
   negotiation.
2. Enable efficient one-stream-per-request interaction patterns.
3. Leverage *push data* mechanisms of underlying protocols to expedite
   negotiation.
4. Provide the building blocks for enhanced censorship resistance.

### What is the difference between connection-level and stream-level protocol negotiation?

All libp2p connections must be authenticated, encrypted, and multiplexed.
Connections using network transports unsupportive of native
authentication/encryption and multiplexing (e.g. TCP) need to undergo protocol
negotiation to agree on a mutually supported:

1. authentication/encryption mechanism (such as SecIO, TLS 1.3, Noise).
2. overlay multiplexer (such as mplex, Yamux, spdystream).

In this specification, we refer to these two as *connection-level negotiations*.
Transports supporting those features natively (such as QUIC) omit those
negotiations.

After successfully selecting a multiplexer, all subsequent I/O happens over
*streams*. When opening streams, peers pin a protocol to that stream, by
conducting *stream-level protocol negotiation*.

At present, multistream-select 1.0 is used for both types of negotiation, but
multiselect 2.0 will use dedicated mechanisms for connection bootstrapping
process and stream protocol negotiation.

## Encryption

### Why are we not supporting SecIO?

SecIO has been the default encryption layer for libp2p for years. It is used in
IPFS and Filecoin. And although it will be superseded shortly, it is proven to
work at scale.

Although SecIO has wide language support, we won’t be using it for mainnet
because, amongst other things, it requires several round trips to be sound, and
doesn’t support early data (0-RTT data), a mechanism that multiselect 2.0 will
leverage to reduce round trips during connection bootstrapping.

SecIO is not considered secure for the purposes of this specification.

### Why are we using Noise?

Copied from the Noise Protocol Framework
[website](http://www.noiseprotocol.org):

> Noise is a framework for building crypto protocols. Noise protocols support
> mutual and optional authentication, identity hiding, forward secrecy, zero
> round-trip encryption, and other advanced features.

Noise in itself does not specify a single handshake procedure, but provides a
framework to build secure handshakes based on Diffie-Hellman key agreement with
a variety of tradeoffs and guarantees.

Noise handshakes are lightweight and simple to understand, and are used in major
cryptographic-centric projects like WireGuard, I2P, and Lightning.
[Various](https://www.wireguard.com/papers/kobeissi-bhargavan-noise-explorer-2018.pdf)
[studies](https://eprint.iacr.org/2019/436.pdf) have assessed the stated
security goals of several Noise handshakes with positive results.

### Why are we using encryption at all?

Transport level encryption secures message exchange and provides properties that
are useful for privacy, safety, and censorship resistance. These properties are
derived from the following security guarantees that apply to the entire
communication between two peers:

- Peer authentication: the peer I’m talking to is really who they claim to be
  and who I expect them to be.
- Confidentiality: no observer can eavesdrop on the content of our messages.
- Integrity: the data has not been tampered with by a third-party while in
  transit.
- Non-repudiation: the originating peer cannot dispute that they sent the
  message.
- Depending on the chosen algorithms and mechanisms (e.g. continuous HMAC), we
  may obtain additional guarantees, such as non-replayability (this byte
  could’ve only been sent *now;* e.g. by using continuous HMACs), or perfect
  forward secrecy (in the case that a peer key is compromised, the content of a
  past conversation will not be compromised).

Note that transport-level encryption is not exclusive of application-level
encryption or cryptography. Transport-level encryption secures the communication
itself, while application-level cryptography is necessary for the application’s
use cases (e.g. signatures, randomness, etc.).

## Gossipsub

### Why are we using a pub/sub algorithm for block and attestation propagation?

Pubsub is a technique to broadcast/disseminate data across a network rapidly.
Such data is packaged in fire-and-forget messages that do not require a response
from every recipient. Peers subscribed to a topic participate in the propagation
of messages in that topic.

The alternative is to maintain a fully connected mesh (all peers connected to
each other 1:1), which scales poorly (O(n^2)).

### Why are we using topics to segregate encodings, yet only support one encoding?

For future extensibility with almost zero overhead now (besides the extra bytes
in the topic name).

### How do we upgrade gossip channels (e.g. changes in encoding, compression)?

Changing gossipsub/broadcasts requires a coordinated upgrade where all clients
start publishing to the new topic together, during an upgrade.

When a node is preparing for upcoming tasks (e.g. validator duty lookahead) on a
gossipsub topic, the node should join the topic of the future epoch in which the
task is to occur in addition to listening to the topics for the current epoch.

### Why must all clients use the same gossip topic instead of one negotiated between each peer pair?

Supporting multiple topics/encodings would require the presence of relayers to
translate between encodings and topics so as to avoid network fragmentation
where participants have diverging views on the gossiped state, making the
protocol more complicated and fragile.

Gossip protocols typically remember what messages they've seen for a finite
period of time-based on message identity -- if you publish the same message
again after that time has passed, it will be re-broadcast—adding a relay delay
also makes this scenario more likely.

One can imagine that in a complicated upgrade scenario, we might have peers
publishing the same message on two topics/encodings, but the price here is
pretty high in terms of overhead -- both computational and networking -- so we'd
rather avoid that.

It is permitted for clients to publish data on alternative topics as long as
they also publish on the network-wide mandatory topic.

### Why are the topics strings and not hashes?

Topic names have a hierarchical structure. In the future, gossipsub may support
wildcard subscriptions (e.g. subscribe to all children topics under a root
prefix) by way of prefix matching. Enforcing hashes for topic names would
preclude us from leveraging such features going forward.

No security or privacy guarantees are lost as a result of choosing plaintext
topic names, since the domain is finite anyway, and calculating a digest's
preimage would be trivial.

Furthermore, the topic names are shorter than their digest equivalents (assuming
SHA-256 hash), so hashing topics would bloat messages unnecessarily.

### Why are we using the `StrictNoSign` signature policy?

The policy omits the `from` (1), `seqno` (3), `signature` (5) and `key` (6)
fields. These fields would:

- Expose origin of sender (`from`), type of sender (based on `seqno`)
- Add extra unused data to the gossip, since message IDs are based on `data`,
  not on the `from` and `seqno`.
- Introduce more message validation than necessary, e.g. no `signature`.

### Why are we overriding the default libp2p pubsub `message-id`?

For our current purposes, there is no need to address messages based on source
peer, or track a message `seqno`. By overriding the default `message-id` to use
content-addressing we can filter unnecessary duplicates before hitting the
application layer.

Some examples of where messages could be duplicated:

- A validator client connected to multiple beacon nodes publishing duplicate
  gossip messages
- Attestation aggregation strategies where clients partially aggregate
  attestations and propagate them. Partial aggregates could be duplicated
- Clients re-publishing seen messages

### Why are these specific gossip parameters chosen?

- `D`, `D_low`, `D_high`, `D_lazy`: recommended defaults.
- `heartbeat_interval`: 0.7 seconds, recommended for the beacon chain in the
  [GossipSub evaluation report by Protocol Labs](https://gateway.ipfs.io/ipfs/QmRAFP5DBnvNjdYSbWhEhVRJJDFCLpPyvew5GwCCB4VxM4).
- `fanout_ttl`: 60 seconds, recommended default. Fanout is primarily used by
  committees publishing attestations to subnets. This happens once per epoch per
  validator and the subnet changes each epoch so there is little to gain in
  having a `fanout_ttl` be increased from the recommended default.
- `mcache_len`: 6, increase by one to ensure that mcache is around for long
  enough for `IWANT`s to respond to `IHAVE`s in the context of the shorter
  `heartbeat_interval`. If `mcache_gossip` is increased, this param should be
  increased to be at least `3` (~2 seconds) more than `mcache_gossip`.
- `mcache_gossip`: 3, recommended default. This can be increased to 5 or 6 (~4
  seconds) if gossip times are longer than expected and the current window does
  not provide enough responsiveness during adverse conditions.
- `seen_ttl`: `SLOTS_PER_EPOCH * SLOT_DURATION_MS / 1000 / heartbeat_interval`.
  Attestation gossip validity is bounded by an epoch, so this is the safe max
  bound.

### Why is there `MAXIMUM_GOSSIP_CLOCK_DISPARITY` when validating slot ranges of messages in gossip subnets?

For some gossip channels (e.g. those for Attestations and BeaconBlocks), there
are designated ranges of slots during which particular messages can be sent,
limiting messages gossiped to those that can be reasonably used in the consensus
at the current time/slot. This is to reduce optionality in DoS attacks.

`MAXIMUM_GOSSIP_CLOCK_DISPARITY` provides some leeway in validating slot ranges
to prevent the gossip network from becoming overly brittle with respect to clock
disparity. For minimum and maximum allowable slot broadcast times,
`MAXIMUM_GOSSIP_CLOCK_DISPARITY` MUST be subtracted and added respectively,
marginally extending the valid range. Although messages can at times be eagerly
gossiped to the network, the node's fork choice prevents integration of these
messages into the actual consensus until the _actual local start_ of the
designated slot.

### Why are there `ATTESTATION_SUBNET_COUNT` attestation subnets?

Depending on the number of validators, it may be more efficient to group shard
subnets and might provide better stability for the gossipsub channel. The exact
grouping will be dependent on more involved network tests. This constant allows
for more flexibility in setting up the network topology for attestation
aggregation (as aggregation should happen on each subnet). The value is
currently set to be equal to `MAX_COMMITTEES_PER_SLOT` if/until network tests
indicate otherwise.

### Why are attestations limited to be broadcast on gossip channels within `SLOTS_PER_EPOCH` slots?

Attestations can only be included on chain within an epoch's worth of slots so
this is the natural cutoff. There is no utility to the chain to broadcast
attestations older than one epoch, and because validators have a chance to make
a new attestation each epoch, there is minimal utility to the fork choice to
relay old attestations as a new latest message can soon be created by each
validator.

In addition to this, relaying attestations requires validating the attestation
in the context of the `state` during which it was created. Thus, validating
arbitrarily old attestations would put additional requirements on which states
need to be readily available to the node. This would result in a higher resource
burden and could serve as a DoS vector.

### Why are aggregate attestations broadcast to the global topic as `AggregateAndProof`s rather than just as `Attestation`s?

The dominant strategy for an individual validator is to always broadcast an
aggregate containing their own attestation to the global channel to ensure that
proposers see their attestation for inclusion. Using a private selection
criteria and providing this proof of selection alongside the gossiped aggregate
ensures that this dominant strategy will not flood the global channel.

Also, an attacker can create any number of honest-looking aggregates and
broadcast them to the global pubsub channel. Thus without some sort of proof of
selection as an aggregator, the global channel can trivially be spammed.

### Why are we sending entire objects in the pubsub and not just hashes?

Entire objects should be sent to get the greatest propagation speeds. If only
hashes are sent, then block and attestation propagation is dependent on
recursive requests from each peer. In a hash-only scenario, peers could receive
hashes without knowing who to download the actual contents from. Sending entire
objects ensures that they get propagated through the entire network.

### Should clients gossip blocks if they *cannot* validate the proposer signature due to not yet being synced, not knowing the head block, etc?

The prohibition of unverified-block-gossiping extends to nodes that cannot
verify a signature due to not being fully synced to ensure that such (amplified)
DOS attacks are not possible.

### How are we going to discover peers in a gossipsub topic?

In Phase 0, peers for attestation subnets will be found using the `attnets`
entry in the ENR.

Although this method will be sufficient for early upgrade of the beacon chain,
we aim to use the more appropriate discv5 topics for this and other similar
tasks in the future. ENRs should ultimately not be used for this purpose. They
are best suited to store identity, location, and capability information, rather
than more volatile advertisements.

### How should fork version be used in practice?

Fork versions are to be manually updated (likely via incrementing) at each
upgrade. This is to provide native domain separation for signatures as well as
to aid in usefulness for identifying peers (via ENRs) and versioning network
protocols (e.g. using fork version to naturally version gossipsub topics).

`BeaconState.genesis_validators_root` is mixed into signature and ENR fork
domains (`ForkDigest`) to aid in the ease of domain separation between chains.
This allows fork versions to safely be reused across chains except for the case
of contentious forks using the same genesis. In these cases, extra care should
be taken to isolate fork versions (e.g. flip a high order bit in all future
versions of one of the chains).

A node locally stores all previous and future planned fork versions along with
the each fork epoch. This allows for handling sync and processing messages
starting from past forks/epochs.

### Why was the max gossip message size increased at Bellatrix?

*Since: version 2 (Bellatrix).*

With the addition of `ExecutionPayload` to `BeaconBlock`s, there is a dynamic
field -- `transactions` -- which can validly exceed the `MAX_PAYLOAD_SIZE` limit
(1 MiB) put in place at Phase 0, so MAX_PAYLOAD_SIZE has increased to 10 MiB on
the network. At the `GAS_LIMIT` (~30M) currently seen on mainnet in 2021, a
single transaction filled entirely with data at a cost of 16 gas per byte can
create a valid `ExecutionPayload` of ~2 MiB. Thus we need a size limit to at
least account for current mainnet conditions.

Note, that due to additional size induced by the `BeaconBlock` contents (e.g.
proposer signature, operations lists, etc) this does reduce the theoretical max
valid `ExecutionPayload` (and `transactions` list) size as slightly lower than
10 MiB. Considering that `BeaconBlock` max size is on the order of 128 KiB in
the worst case and the current gas limit (~30M) bounds max blocksize to less
than 2 MiB today, this marginal difference in theoretical bounds will have zero
impact on network functionality and security.

### Why are blobs relayed as a sidecar, separate from beacon blocks?

*Since: version 3 (Deneb).*

This "sidecar" design provides forward compatibility for further data increases
by black-boxing `is_data_available()`: with full sharding `is_data_available()`
can be replaced by data-availability-sampling (DAS) thus avoiding all blobs
being downloaded by all beacon nodes on the network.

Such sharding design may introduce an updated `BlobSidecar` to identify the
shard, but does not affect the `BeaconBlock` structure.

## Req/Resp

### Why segregate requests into dedicated protocol IDs?

Requests are segregated by protocol ID to:

1. Leverage protocol routing in libp2p, such that the libp2p stack will route
   the incoming stream to the appropriate handler. This allows the handler
   function for each request type to be self-contained. For an analogy, think
   about how you attach HTTP handlers to a REST API server.
2. Version requests independently. In a coarser-grained umbrella protocol, the
   entire protocol would have to be versioned even if just one field in a single
   message changed.
3. Enable clients to select the individual requests/versions they support. It
   would no longer be a strict requirement to support all requests, and clients,
   in principle, could support a subset of requests and variety of versions.
4. Enable flexibility and agility for clients adopting specification changes
   that impact the request, by signalling to peers exactly which subset of
   new/old requests they support.
5. Enable clients to explicitly choose backwards compatibility at the request
   granularity. Without this, clients would be forced to support entire versions
   of the coarser request protocol.
6. Parallelise RFCs (or EIPs). By decoupling requests from one another, each RFC
   that affects the request protocol can be deployed/tested/debated
   independently without relying on a synchronization point to version the
   general top-level protocol.
   1. This has the benefit that clients can explicitly choose which RFCs to
      deploy without buying into all other RFCs that may be included in that
      top-level version.
   2. Affording this level of granularity with a top-level protocol would imply
      creating as many variants (e.g. /protocol/43-{a,b,c,d,...}) as the
      cartesian product of RFCs in-flight, O(n^2).
7. Allow us to simplify the payload of requests. Request-id’s and method-ids no
   longer need to be sent. The encoding/request type and version can all be
   handled by the framework.

**Caveat**: The protocol negotiation component in the current version of libp2p
is called multistream-select 1.0. It is somewhat naïve and introduces overhead
on every request when negotiating streams, although implementation-specific
optimizations are possible to save this cost. Multiselect 2.0 will eventually
remove this overhead by memoizing previously selected protocols, and modeling
shared protocol tables. Fortunately, this req/resp protocol is not the expected
network bottleneck in the protocol so the additional overhead is not expected to
significantly hinder this domain.

### Why are messages length-prefixed with a protobuf varint in the SSZ-encoding?

We are using single-use streams where each stream is closed at the end of the
message. Thus, libp2p transparently handles message delimiting in the underlying
stream. libp2p streams are full-duplex, and each party is responsible for
closing their write side (like in TCP). We can therefore use stream closure to
mark the end of the request and response independently.

Nevertheless, in the case of `ssz_snappy`, messages are still length-prefixed
with the length of the underlying data:

- A basic reader can prepare a correctly sized buffer before reading the message
- A more advanced reader can stream-decode SSZ given the length of the SSZ data.
- Alignment with protocols like gRPC over HTTP/2 that prefix with length
- Sanity checking of message length, and enabling much stricter message length
  limiting based on SSZ type information, to provide even more DOS protection
  than the global message length already does. E.g. a small `Status` message
  does not nearly require `MAX_PAYLOAD_SIZE` bytes.

[Protobuf varint](https://developers.google.com/protocol-buffers/docs/encoding#varints)
is an efficient technique to encode variable-length (unsigned here) ints.
Instead of reserving a fixed-size field of as many bytes as necessary to convey
the maximum possible value, this field is elastic in exchange for 1-bit overhead
per byte.

### Why do we version protocol strings with ordinals instead of semver?

Using semver for network protocols is confusing. It is never clear what a change
in a field, even if backwards compatible on deserialization, actually implies.
Network protocol agreement should be explicit. Imagine two peers:

- Peer A supporting v1.1.1 of protocol X.
- Peer B supporting v1.1.2 of protocol X.

These two peers should never speak to each other because the results can be
unpredictable. This is an oversimplification: imagine the same problem with a
set of 10 possible versions. We now have 10^2 (100) possible outcomes that peers
need to model for. The resulting complexity is unwieldy.

For this reason, we rely on negotiation of explicit, verbatim protocols. In the
above case, peer B would provide backwards compatibility by supporting and
advertising both v1.1.1 and v1.1.2 of the protocol.

Therefore, semver would be relegated to convey expectations at the human level,
and it wouldn't do a good job there either, because it's unclear if "backwards
compatibility" and "breaking change" apply only to wire schema level, to
behavior, etc.

For this reason, we remove and replace semver with ordinals that require
explicit agreement and do not mandate a specific policy for changes.

### Why is it called Req/Resp and not RPC?

Req/Resp is used to avoid confusion with JSON-RPC and similar user-client
interaction mechanisms.

### What is a typical rate limiting strategy?

The responder typically will want to rate limit requests to protect against spam
and to manage resource consumption, while the requester will want to maximise
performance based on its own resource allocation strategy. For the network, it
is beneficial if available resources are used optimally.

Broadly, the requester does not know the capacity / limit of each server but can
derive it from the rate of responses for the purpose of selecting the next peer
for a request.

Because the server withholds the response until capacity is available, a client
can optimistically send requests without risking running into negative scoring
situations or sub-optimal rate polling.

A typical approach for the requester is to implement a timeout on the request
that depends on the nature of the request and on connectivity parameters in
general - for example when requesting blocks, a peer might choose to send a
request to a second peer if the first peer does not respond within a reasonable
time, and to reset the request to the first peer if the second peer responds
faster. Clients may use past response performance to reward fast peers when
implementing peer scoring.

A typical approach for the responder is to implement a two-level token/leaky
bucket with a per-peer limit and a global limit. The granularity of rate
limiting may be based either on full requests or individual chunks with the
latter being preferable. A token cost may be assigned to the request itself and
separately each chunk in the response so as to remain protected both against
large and frequent requests.

For requesters, rate limiting is not distinguishable from other conditions
causing slow responses (slow peers, congestion etc) and since the latter
conditions must be handled anyway, including rate limiting in this strategy
keeps the implementation simple.

### Why do we allow empty responses in block requests?

When requesting blocks by range or root, it may happen that there are no blocks
in the selected range or the responding node does not have the requested blocks.

Thus, it may happen that we need to transmit an empty list - there are several
ways to encode this:

0. Close the stream without sending any data
1. Add a `null` option to the `success` response, for example by introducing an
   additional byte
2. Respond with an error result, using a specific error code for "No data"

Semantically, it is not an error that a block is missing during a slot making
option 2 unnatural.

Option 1 allows the responder to signal "no block", but this information may be
wrong - for example in the case of a malicious node.

Under option 0, there is no way for a client to distinguish between a slot
without a block and an incomplete response, but given that it already must
contain logic to handle the uncertainty of a malicious peer, option 0 was
chosen. Clients should mark any slots missing blocks as unknown until they can
be verified as not containing a block by successive blocks.

Assuming option 0 with no special `null` encoding, consider a request for slots
`2, 3, 4` -- if there was no block produced at slot 4, the response would be
`2, 3, EOF`. Now consider the same situation, but where only `4` is requested --
closing the stream with only `EOF` (without any `response_chunk`) is consistent.

Failing to provide blocks that nodes "should" have is reason to trust a peer
less -- for example, if a particular peer gossips a block, it should have access
to its parent. If a request for the parent fails, it's indicative of poor peer
quality since peers should validate blocks before gossiping them.

### Why does `BeaconBlocksByRange` let the server choose which branch to send blocks from?

When connecting, the `Status` message gives an idea about the sync status of a
particular peer, but this changes over time. By the time a subsequent
`BeaconBlocksByRange` request is processed, the information may be stale, and
the responder might have moved on to a new finalization point and pruned blocks
around the previous head and finalized blocks.

To avoid this race condition, we allow the responder to choose which branch to
send to the requester. The requester then goes on to validate the blocks and
incorporate them in their own database -- because they follow the same rules,
they should at this point arrive at the same canonical chain.

### Why are `BlocksByRange` requests only required to be served for the latest `compute_min_epochs_for_block_requests()` epochs?

Due to economic finality and weak subjectivity requirements of a proof-of-stake
blockchain, for a new node to safely join the network the node must provide a
recent checkpoint found out-of-band. This checkpoint can be in the form of a
`root` & `epoch` or it can be the entire beacon state and then a simple block
sync from there to the head. We expect the latter to be the dominant UX
strategy.

These checkpoints *in the worst case* (i.e. very large validator set and maximal
allowed safety decay) must be from the most recent
`compute_min_epochs_for_block_requests()` epochs, and thus a user must be able
to block sync to the head from this starting point. Thus, this defines the epoch
range outside which nodes may prune finalized blocks, and the epoch range that a
new node syncing from a checkpoint must backfill.

`compute_min_epochs_for_block_requests()` is calculated using the arithmetic
from `compute_weak_subjectivity_period` found in the
[weak subjectivity guide](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/weak-subjectivity.md). Specifically to find this max
epoch range, we use the worst case event of a very large validator size
(`>= MIN_PER_EPOCH_CHURN_LIMIT * CHURN_LIMIT_QUOTIENT`).

### Why must the proposer signature be checked when backfilling blocks in the database?

When backfilling blocks in a database from a known safe block/state (e.g. when
starting from a weak subjectivity state), the node not only must ensure the
`BeaconBlock`s form a chain to the known safe block, but also must check that
the proposer signature is valid in the `SignedBeaconBlock` wrapper.

This is because the signature is not part of the `BeaconBlock` hash chain, and
thus could be corrupted by an attacker serving valid `BeaconBlock`s but invalid
signatures contained in `SignedBeaconBlock`.

Although in this particular use case this does not represent a decay in safety
(due to the assumptions of starting at a weak subjectivity checkpoint), it would
represent invalid historic data and could be unwittingly transmitted to
additional nodes.

### What's the effect of empty slots on the sync algorithm?

When syncing one can only tell that a slot has been skipped on a particular
branch by examining subsequent blocks and analyzing the graph formed by the
parent root. Because the server side may choose to omit blocks in the response
for any reason, clients must validate the graph and be prepared to fill in gaps.

For example, if a peer responds with blocks [2, 3] when asked for [2, 3, 4],
clients may not assume that block 4 doesn't exist -- it merely means that the
responding peer did not send it (they may not have it yet or may maliciously be
trying to hide it) and successive blocks will be needed to determine if there
exists a block at slot 4 in this particular branch.

### Why was the max chunk response size increased at Bellatrix?

*Since: version 2 (Bellatrix).*

Similar to the discussion about the maximum gossip size increase, the
`ExecutionPayload` type can cause `BeaconBlock`s to exceed the 1 MiB bounds put
in place during Phase 0.

As with the gossip limit, 10 MiB is selected because this is firmly above any
valid block sizes in the range of gas limits expected in the medium term.

As with both gossip and req/rsp maximum values, type-specific limits should
always by simultaneously respected.

### Why allow invalid payloads on the P2P network?

*Since: version 2 (Bellatrix).*

The specification allows blocks with invalid execution payloads to propagate
across gossip and via RPC calls. The reasoning for this is as follows:

1. Optimistic nodes must listen to block gossip to obtain a view of the head of
   the chain.
2. Therefore, optimistic nodes must propagate gossip blocks. Otherwise, they'd
   be censoring.
3. If optimistic nodes will propagate blocks via gossip, then they must respond
   to requests for the parent via RPC.
4. Therefore, optimistic nodes must send optimistic blocks via RPC.

So, to prevent network segregation from optimistic nodes inadvertently sending
invalid execution payloads, nodes should never downscore/disconnect nodes due to
such invalid payloads. This does open the network to some DoS attacks from
invalid execution payloads, but the scope of actors is limited to validators who
can put those payloads in valid (and slashable) beacon blocks. Therefore, it is
argued that the DoS risk introduced in tolerable.

More complicated schemes are possible that could restrict invalid payloads from
RPC. However, it's not clear that complexity is warranted.

## Discovery

### Why are we using discv5 and not libp2p Kademlia DHT?

discv5 is a standalone protocol, running on UDP on a dedicated port, meant for
peer and service discovery only. discv5 supports self-certified, flexible peer
records (ENRs) and topic-based advertisement, both of which are, or will be,
requirements in this context.

On the other hand, libp2p Kademlia DHT is a fully-fledged DHT
protocol/implementations with content routing and storage capabilities, both of
which are irrelevant in this context.

Ethereum execution-layer nodes will evolve to support discv5. By sharing the
discovery network between Ethereum consensus-layer and execution-layer clients,
we benefit from the additive effect on network size that enhances resilience and
resistance against certain attacks, to which smaller networks are more
vulnerable. It should also help light clients of both networks find nodes with
specific capabilities.

discv5 is in the process of being audited.

### What is the difference between an ENR and a multiaddr, and why are we using ENRs?

Ethereum Node Records are self-certified node records. Nodes craft and
disseminate ENRs for themselves, proving authorship via a cryptographic
signature. ENRs are sequentially indexed, enabling conflicts to be resolved.

ENRs are key-value records with string-indexed ASCII keys. They can store
arbitrary information, but EIP-778 specifies a pre-defined dictionary, including
IPv4 and IPv6 addresses, secp256k1 public keys, etc.

Comparing ENRs and multiaddrs is like comparing apples and oranges. ENRs are
self-certified containers of identity, addresses, and metadata about a node.
Multiaddrs are address strings with the peculiarity that they’re
self-describing, composable and future-proof. An ENR can contain multiaddrs, and
multiaddrs can be derived securely from the fields of an authenticated ENR.

discv5 uses ENRs and we will presumably need to:

1. Add `multiaddr` to the dictionary, so that nodes can advertise their
   multiaddr under a reserved namespace in ENRs. – and/or –
2. Define a bi-directional conversion function between multiaddrs and the
   corresponding denormalized fields in an ENR (ip, ip6, tcp, tcp6, etc.), for
   compatibility with nodes that do not support multiaddr natively (e.g.
   Ethereum execution-layer nodes).

### Why do we not form ENRs and find peers until genesis block/state is known?

Although client software might very well be running locally prior to the
solidification of the beacon-chain genesis state and block, clients cannot form
valid ENRs prior to this point. ENRs contain `fork_digest` which utilizes the
`genesis_validators_root` for a cleaner separation between chains so prior to
knowing genesis, we cannot use `fork_digest` to cleanly find peers on our
intended chain. Once genesis data is known, we can then form ENRs and safely
find peers.

When using a proof-of-work deposit contract for deposits, `fork_digest` will be
known `GENESIS_DELAY` (7 days in mainnet configuration) before `genesis_time`,
providing ample time to find peers and form initial connections and gossip
subnets prior to genesis.

## Compression/Encoding

### Why are we using SSZ for encoding?

SSZ is used at the consensus layer, and all implementations should have support
for SSZ-encoding/decoding, requiring no further dependencies to be added to
client implementations. This is a natural choice for serializing objects to be
sent across the wire. The actual data in most protocols will be further
compressed for efficiency.

SSZ has well-defined schemas for consensus objects (typically sent across the
wire) reducing any serialization schema data that needs to be sent. It also has
defined all required types that are required for this network specification.

### Why are we compressing, and at which layers?

We compress on the wire to achieve smaller payloads per-message, which, in
aggregate, result in higher efficiency, better utilization of available
bandwidth, and overall reduction in network-wide traffic overhead.

At this time, libp2p does not have an out-of-the-box compression feature that
can be dynamically negotiated and layered atop connections and streams, but it
is [being considered](https://github.com/libp2p/libp2p/issues/81).

This is a non-trivial feature because the behavior of network IO loops, kernel
buffers, chunking, and packet fragmentation, amongst others, need to be taken
into account. libp2p streams are unbounded streams, whereas compression
algorithms work best on bounded byte streams of which we have some prior
knowledge.

Compression tends not to be a one-size-fits-all problem. A lot of variables need
careful evaluation, and generic approaches/choices lead to poor size shavings,
which may even be counterproductive when factoring in the CPU and memory
tradeoff.

For all these reasons, generically negotiating compression algorithms may be
treated as a research problem at the libp2p community, one we’re happy to tackle
in the medium-term.

At this stage, the wisest choice is to consider libp2p a messenger of bytes, and
to make application layer participate in compressing those bytes. This looks
different depending on the interaction layer:

- Gossip domain: since gossipsub has a framing protocol and exposes an API, we
  compress the payload (when dictated by the encoding token in the topic name)
  prior to publishing the message via the API. No length-prefixing is necessary
  because protobuf takes care of bounding the field in the serialized form.
- Req/Resp domain: since we define custom protocols that operate on byte
  streams, implementers are encouraged to encapsulate the encoding and
  compression logic behind MessageReader and MessageWriter components/strategies
  that can be layered on top of the raw byte streams.

### Why are we using Snappy for compression?

Snappy is used in Ethereum 1.0. It is well maintained by Google, has good
benchmarks, and can calculate the size of the uncompressed object without
inflating it in memory. This prevents DOS vectors where large uncompressed data
is sent.

### Can I get access to unencrypted bytes on the wire for debugging purposes?

Yes, you can add loggers in your libp2p protocol handlers to log incoming and
outgoing messages. It is recommended to use programming design patterns to
encapsulate the logging logic cleanly.

If your libp2p library relies on frameworks/runtimes such as Netty (jvm) or
Node.js (javascript), you can use logging facilities in those
frameworks/runtimes to enable message tracing.

For specific ad-hoc testing scenarios, you can use the
[plaintext/2.0.0 secure channel](https://github.com/libp2p/specs/blob/master/plaintext/README.md)
(which is essentially no-op encryption or message authentication), in
combination with tcpdump or Wireshark to inspect the wire.

### What are SSZ type size bounds?

The SSZ encoding outputs of each type have size bounds: each dynamic type, such
as a list, has a "limit", which can be used to compute the maximum valid output
size. Note that for some more complex dynamic-length objects, element offsets (4
bytes each) may need to be included. Other types are static, they have a fixed
size: no dynamic-length content is involved, and the minimum and maximum bounds
are the same.

For reference, the type bounds can be computed ahead of time,
[as per this example](https://gist.github.com/protolambda/db75c7faa1e94f2464787a480e5d613e).
It is advisable to derive these lengths from the SSZ type definitions in use, to
ensure that version changes do not cause out-of-sync type bounds.

### Why is the message size defined in terms of application payload?

When transmitting messages over gossipsub and/or the req/resp domain, we want to
ensure that the same payload sizes are supported regardless of the underlying
transport, decoupling the consensus layer from libp2p-induced overhead and the
particular transmission strategy.

To derive "encoded size limits" from desired application sizes, we take into
account snappy compression and framing overhead.

In the case of gossipsub, the protocol supports sending multiple application
payloads as well as mixing application data with control messages in each
gossipsub frame. The limit is set such that at least one max-sized
application-level message together with a small amount (1 KiB) of gossipsub
overhead is allowed. Implementations are free to pack multiple smaller
application messages into a single gossipsub frame, and/or combine it with
control messages as they see fit.

The limit is set on the uncompressed payload size in particular to protect
against decompression bombs.

### Why is there a limit on message sizes at all?

The message size limit protects against several forms of DoS and network-based
amplification attacks and provides upper bounds for resource (network, memory)
usage in the client based on protocol requirements to decode, buffer, cache,
store and re-transmit messages which in turn translate into performance and
protection tradeoffs, ensuring capacity to handle worst cases during recovery
from network instability.

In particular, blocks—-currently the only message type without a practical
SSZ-derived upper bound on size—-cannot be fully verified synchronously as part
of gossipsub validity checks. This means that there exist cases where invalid
messages signed by a validator may be amplified by the network.
