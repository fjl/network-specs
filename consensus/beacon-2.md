---
fork: bellatrix
status: legacy
---

# Ethereum Consensus Layer Networking -- Version 2

This document is the complete specification of the consensus-layer
peer-to-peer networking protocol as of Capella. Bellatrix introduced no change
to the set of topics, methods or configuration and is therefore folded into
this version. It covers the libp2p transport
profile, the gossipsub configuration and topics, the Req/Resp protocols, and
the discv5 integration. The application-level *validation* of gossip messages
is specified as executable Python in
[consensus-specs](https://github.com/ethereum/consensus-specs) and is linked
from the relevant sections rather than repeated here. Non-normative design
rationale lives in [rationale.md](rationale.md).

<!-- toc start -->

- [Network fundamentals](#network-fundamentals)
  - [Transport](#transport)
  - [Encryption and identification](#encryption-and-identification)
  - [Protocol negotiation](#protocol-negotiation)
  - [Multiplexing](#multiplexing)
- [Configuration](#configuration)
  - [Constants](#constants)
  - [Configs](#configs)
- [Types](#types)
  - [`Attnets`](#attnets)
  - [`BeaconBlockRoots`](#beaconblockroots)
  - [`ErrorMessage`](#errormessage)
  - [`NodeID`](#nodeid)
  - [`SignedBeaconBlocks`](#signedbeaconblocks)
  - [`SubnetID`](#subnetid)
  - [`Syncnets`](#syncnets)
  - [`LightClientUpdates`](#lightclientupdates)
- [Fork digest](#fork-digest)
  - [`compute_fork_digest`](#compute_fork_digest)
- [Maximum message sizes](#maximum-message-sizes)
  - [`max_compressed_len`](#max_compressed_len)
  - [`max_message_size`](#max_message_size)
- [MetaData](#metadata)
- [The gossip domain: gossipsub](#the-gossip-domain-gossipsub)
  - [Gossipsub parameters](#gossipsub-parameters)
  - [Topics and messages](#topics-and-messages)
    - [Message ID](#message-id)
    - [Topic table](#topic-table)
    - [Global topics](#global-topics)
      - [`beacon_block`](#beacon_block)
      - [`beacon_aggregate_and_proof`](#beacon_aggregate_and_proof)
      - [`voluntary_exit`](#voluntary_exit)
      - [`proposer_slashing`](#proposer_slashing)
      - [`attester_slashing`](#attester_slashing)
      - [`bls_to_execution_change`](#bls_to_execution_change)
      - [`sync_committee_contribution_and_proof`](#sync_committee_contribution_and_proof)
    - [Attestation subnets](#attestation-subnets)
      - [`beacon_attestation_{subnet_id}`](#beacon_attestation_subnet_id)
    - [Attestations and Aggregation](#attestations-and-aggregation)
    - [Sync committee subnets](#sync-committee-subnets)
      - [`sync_committee_{subnet_id}`](#sync_committee_subnet_id)
    - [Sync committees and aggregation](#sync-committees-and-aggregation)
  - [Transitioning the gossip](#transitioning-the-gossip)
  - [Encodings](#encodings)
  - [Gossipsub size limits](#gossipsub-size-limits)
- [The Req/Resp domain](#the-reqresp-domain)
  - [Protocol identification](#protocol-identification)
  - [Req/Resp interaction](#reqresp-interaction)
    - [`ForkDigest`-context](#forkdigest-context)
    - [Requesting side](#requesting-side)
    - [Responding side](#responding-side)
  - [Encoding strategies](#encoding-strategies)
    - [SSZ-snappy encoding strategy](#ssz-snappy-encoding-strategy)
  - [Messages](#messages)
    - [Status v1](#status-v1)
    - [Goodbye v1](#goodbye-v1)
    - [BeaconBlocksByRange v2](#beaconblocksbyrange-v2)
      - [`compute_min_epochs_for_block_requests`](#compute_min_epochs_for_block_requests)
    - [BeaconBlocksByRoot v2](#beaconblocksbyroot-v2)
    - [Ping v1](#ping-v1)
    - [GetMetaData v2](#getmetadata-v2)
- [The discovery domain: discv5](#the-discovery-domain-discv5)
  - [Integration into libp2p stacks](#integration-into-libp2p-stacks)
  - [ENR structure](#enr-structure)
    - [Attestation subnet bitfield](#attestation-subnet-bitfield)
    - [Sync committee bitfield](#sync-committee-bitfield)
    - [`eth2` field](#eth2-field)
- [Subnet subscription](#subnet-subscription)
  - [Attestation subnet subscription](#attestation-subnet-subscription)
  - [Sync committee subnet subscription](#sync-committee-subnet-subscription)
- [Light client networking](#light-client-networking)
  - [Light client gossip topics](#light-client-gossip-topics)
    - [`light_client_finality_update`](#light_client_finality_update)
    - [`light_client_optimistic_update`](#light_client_optimistic_update)
  - [Light client Req/Resp methods](#light-client-reqresp-methods)
    - [GetLightClientBootstrap v1](#getlightclientbootstrap-v1)
    - [LightClientUpdatesByRange v1](#lightclientupdatesbyrange-v1)
    - [GetLightClientFinalityUpdate v1](#getlightclientfinalityupdate-v1)
    - [GetLightClientOptimisticUpdate v1](#getlightclientoptimisticupdate-v1)
  - [Light client and full node behaviour](#light-client-and-full-node-behaviour)
- [Change log](#change-log)
  - [Version 2 (Bellatrix, September 2022; Capella, April 2023)](#version-2-bellatrix-september-2022-capella-april-2023)
  - [Version 1 (Altair, October 2021)](#version-1-altair-october-2021)
  - [Version 0 (Phase 0, December 2020)](#version-0-phase-0-december-2020)

<!-- toc end -->

## Network fundamentals

This section outlines the specification for the networking stack in Ethereum
consensus-layer clients.

### Transport

Even though libp2p is a multi-transport stack (designed to listen on multiple
simultaneous transports and endpoints transparently), we hereby define a profile
for basic interoperability.

All implementations MUST support the QUIC (UDP) libp2p transport and the TCP
libp2p transport. Any supported transport MUST be enabled for both dialing and
listening (i.e. outbound and inbound connections). The libp2p QUIC (UDP) and TCP
transports support listening on IPv4 and IPv6 addresses (and on multiple
simultaneously).

QUIC is the primary transport. When a peer is reachable over both TCP and QUIC,
clients SHOULD prioritise peer's QUIC addresses.

Clients must support listening on at least one of IPv4 or IPv6. Clients that do
_not_ have support for listening on IPv4 SHOULD be cognizant of the potential
disadvantages in terms of Internet-wide routability/support. Clients MAY choose
to listen only on IPv6, but MUST be capable of dialing both IPv4 and IPv6
addresses.

All listening endpoints must be publicly dialable, and thus not rely on libp2p
circuit relay, AutoNAT, or AutoRelay facilities. (Usage of circuit relay,
AutoNAT, or AutoRelay will be specifically re-examined soon.)

Nodes operating behind a NAT, or otherwise undialable by default (e.g. container
runtime, firewall, etc.), MUST have their infrastructure configured to enable
inbound traffic on the announced public listening endpoint.

### Encryption and identification

The QUIC transport is secured using TLS 1.3, configured according to the
[libp2p TLS specification](https://github.com/libp2p/specs/blob/master/tls/tls.md).

For the TCP fallback transport, the
[Libp2p-noise](https://github.com/libp2p/specs/tree/master/noise) secure channel
handshake with `secp256k1` identities will be used for encryption. As specified
in the libp2p specification, clients MUST support the `XX` handshake pattern.

### Protocol negotiation

Clients MUST use exact equality when negotiating protocol versions to use and
MAY use the version to give priority to higher version numbers.

Clients MUST support
[multistream-select 1.0](https://github.com/multiformats/multistream-select/)
and MAY support [multiselect 2.0](https://github.com/libp2p/specs/pull/95) when
the specification solidifies. Once all clients have implementations for
multiselect 2.0, multistream-select 1.0 MAY be phased out.

### Multiplexing

During connection bootstrapping, libp2p dynamically negotiates a mutually
supported multiplexing method to conduct parallel conversations. This applies to
transports that are natively incapable of multiplexing (e.g. TCP, WebSockets,
WebRTC), and is omitted for capable transports (e.g. QUIC).

Two multiplexers are commonplace in libp2p implementations:
[mplex](https://github.com/libp2p/specs/tree/master/mplex) and
[yamux](https://github.com/libp2p/specs/blob/master/yamux/README.md). Their
protocol IDs are, respectively: `/mplex/6.7.0` and `/yamux/1.0.0`.

Because the primary QUIC transport multiplexes natively, a stream multiplexer is
only required for the TCP fallback transport. Clients that support the TCP
transport MUST support
[mplex](https://github.com/libp2p/specs/tree/master/mplex) and MAY support
[yamux](https://github.com/libp2p/specs/blob/master/yamux/README.md). If both
are supported by the client, yamux MUST take precedence during negotiation. See
the [rationale](rationale.md#multiplexing) for tradeoffs.

## Configuration

### Constants

| Name           | Value         |
| -------------- | ------------- |
| `NODE_ID_BITS` | `Uint64(256)` |

### Configs

This section outlines configurations that are used in this specification.

| Name                                 | Value                                       | Description                                                                       |
| ------------------------------------ | ------------------------------------------- | --------------------------------------------------------------------------------- |
| `MAX_PAYLOAD_SIZE`                   | `Uint64(10 * 2**20)` (= 10,485,760, 10 MiB) | Maximum allowed size of uncompressed payload in gossipsub messages and RPC chunks |
| `MAX_REQUEST_BLOCKS`                 | `Uint64(2**10)` (= 1,024)                   | Maximum number of blocks in a single request                                      |
| `EPOCHS_PER_SUBNET_SUBSCRIPTION`     | `Epoch(2**8)` (= 256)                       | Number of epochs on a subnet subscription                                         |
| `ATTESTATION_PROPAGATION_SLOT_RANGE` | `Slot(32)`                                  | Maximum number of slots during which an attestation can be propagated             |
| `MAXIMUM_GOSSIP_CLOCK_DISPARITY`     | `Uint64(500)`                               | Maximum **milliseconds** of clock disparity assumed between honest nodes          |
| `MESSAGE_DOMAIN_INVALID_SNAPPY`      | `DomainType("0x00000000")`                  | 4-byte domain for gossip message-id isolation of *invalid* snappy messages        |
| `MESSAGE_DOMAIN_VALID_SNAPPY`        | `DomainType("0x01000000")`                  | 4-byte domain for gossip message-id isolation of *valid* snappy messages          |
| `SUBNETS_PER_NODE`                   | `Uint64(2)`                                 | Number of long-lived subnets a beacon node should be subscribed to                |
| `ATTESTATION_SUBNET_COUNT`           | `Uint64(2**6)` (= 64)                       | Number of attestation subnets used in the gossipsub protocol                      |
| `ATTESTATION_SUBNET_EXTRA_BITS`      | `Uint64(0)`                                 | Number of extra bits of a NodeId to use when mapping to a subscribed subnet       |
| `MAX_CONCURRENT_REQUESTS`            | `Uint64(2)`                                 | Maximum number of concurrent requests per protocol ID that a client may issue     |
| `MAX_REQUEST_LIGHT_CLIENT_UPDATES`   | `Uint64(2**7)` (= 128)                      | Maximum number of `LightClientUpdate` instances in a single request               |

The number of sync committee subnets, `SYNC_COMMITTEE_SUBNET_COUNT` (= 4), is a
preset defined in the
[Altair validator specification](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/validator.md#constants).

## Types

The following SSZ types are used on the wire in addition to the consensus types
defined in the
[beacon chain](https://github.com/ethereum/consensus-specs/blob/master/specs/capella/beacon-chain.md) and
[validator](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/validator.md#containers) specifications.

### `Attnets`

```python
class Attnets(BitVector):
    """
    The attestation subnets a node is subscribed to, one bit per subnet.
    """

    LENGTH = ATTESTATION_SUBNET_COUNT
```

### `BeaconBlockRoots`

```python
class BeaconBlockRoots(List[Root]):
    """
    Beacon block roots requested in a ``BeaconBlocksByRoot`` request.
    """

    LIMIT = MAX_REQUEST_BLOCKS
```

### `ErrorMessage`

```python
class ErrorMessage(List[Byte]):
    """
    The error message of an unsuccessful response chunk.
    """

    LIMIT = 256
```

### `NodeID`

```python
class NodeID(Uint256):
    """
    A node identifier on the discovery network.
    """
```

### `SignedBeaconBlocks`

```python
class SignedBeaconBlocks(List[SignedBeaconBlock]):
    """
    Signed beacon blocks returned in a ``BeaconBlocksByRange`` or
    ``BeaconBlocksByRoot`` response.
    """

    LIMIT = MAX_REQUEST_BLOCKS
```

### `SubnetID`

```python
class SubnetID(Uint64):
    """
    The identifier of a gossip subnet, like an attestation subnet.
    """
```

### `Syncnets`

```python
class Syncnets(BitVector):
    """
    The sync committee subnets a node is subscribed to, one bit per subnet.
    """

    LENGTH = SYNC_COMMITTEE_SUBNET_COUNT
```

### `LightClientUpdates`

```python
class LightClientUpdates(List[LightClientUpdate]):
    """
    Light client updates returned in a ``LightClientUpdatesByRange`` response.
    """

    LIMIT = MAX_REQUEST_LIGHT_CLIENT_UPDATES
```

## Fork digest

Gossip topics, the `Status` handshake and the ENR `eth2` entry are all
parametrised by a 4-byte *fork digest* which separates networks and forks from
each other.

`compute_fork_version(epoch)` returns the fork version in effect at `epoch`
according to the network's fork schedule: `CAPELLA_FORK_VERSION` from
`CAPELLA_FORK_EPOCH` on, `BELLATRIX_FORK_VERSION` from `BELLATRIX_FORK_EPOCH`,
`ALTAIR_FORK_VERSION` from `ALTAIR_FORK_EPOCH`, `GENESIS_FORK_VERSION` before.
Every later fork adds a branch to this function. The authoritative definition
is
[`compute_fork_version`](https://github.com/ethereum/consensus-specs/blob/master/specs/capella/p2p-interface.md#modified-compute_fork_version)
in consensus-specs.

### `compute_fork_digest`

```python
def compute_fork_digest(
    genesis_validators_root: Root,
    epoch: Epoch,
) -> ForkDigest:
    """
    Return the 4-byte fork digest for the ``genesis_validators_root`` at a given ``epoch``.

    This is a digest primarily used for domain separation on the p2p layer.
    4-bytes suffices for practical separation of forks/chains.
    """
    fork_version = compute_fork_version(epoch)
    base_digest = compute_fork_data_root(fork_version, genesis_validators_root)
    return ForkDigest(base_digest[:4])
```

## Maximum message sizes

Maximum message sizes are derived from the maximum payload size that the network
can carry according to the following functions:

### `max_compressed_len`

```python
def max_compressed_len(n: Uint64) -> Uint64:
    # Worst-case compressed length for a given payload of size n when using snappy:
    # https://github.com/google/snappy/blob/32ded457c0b1fe78ceb8397632c416568d6714a0/snappy.cc#L218C1-L218C47
    return 32 + n + n // 6
```

### `max_message_size`

```python
def max_message_size() -> Uint64:
    # Allow 1024 bytes for framing and encoding overhead but at least 1MiB in case MAX_PAYLOAD_SIZE is small.
    return max(max_compressed_len(MAX_PAYLOAD_SIZE) + 1024, Uint64(1024 * 1024))
```

## MetaData

Clients MUST locally store the following `MetaData`:

```
(
  seq_number: Uint64
  attnets: Attnets
  syncnets: Syncnets
)
```

Where

- `seq_number` is a `Uint64` starting at `0` used to version the node's
  metadata. If any other field in the local `MetaData` changes, the node MUST
  increment `seq_number` by 1.
- `attnets` is a `BitVector` representing the node's persistent attestation
  subnet subscriptions.
- `syncnets` is a `BitVector` representing the node's sync committee subnet
  subscriptions. This field should mirror the data in the node's ENR as outlined
  in the
  [validator guide](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/validator.md#sync-committee-subnet-stability).

*Note*: `MetaData.seq_number` is used for versioning of the node's metadata, is
entirely independent of the ENR sequence number, and will in most cases be out
of sync with the ENR sequence number.

## The gossip domain: gossipsub

Clients MUST support the
[gossipsub v1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md)
libp2p Protocol including the
[gossipsub v1.1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md)
extension.

**Protocol ID:** `/meshsub/1.1.0`

### Gossipsub parameters

The following gossipsub
[parameters](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md#parameters)
will be used:

- `D` (topic stable mesh target count): 8
- `D_low` (topic stable mesh low watermark): 6
- `D_high` (topic stable mesh high watermark): 12
- `D_lazy` (gossip target): 6
- `heartbeat_interval` (frequency of heartbeat, seconds): 0.7
- `fanout_ttl` (ttl for fanout maps for topics we are not subscribed to but have
  published to, seconds): 60
- `mcache_len` (number of windows to retain full messages in cache for `IWANT`
  responses): 6
- `mcache_gossip` (number of windows to gossip about): 3
- `seen_ttl` (expiry time for cache of seen message ids, seconds):
  `milliseconds_to_seconds(SLOT_DURATION_MS * SLOTS_PER_EPOCH * 2)`

*Note*: Gossipsub v1.1 introduces a number of
[additional parameters](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#overview-of-new-parameters)
for peer scoring and other attack mitigations. These are currently under
investigation and will be specified and released to mainnet when they are ready.

### Topics and messages

Topics are plain UTF-8 strings and are encoded on the wire as determined by
protobuf (gossipsub messages are enveloped in protobuf messages). Topic strings
have form: `/eth2/ForkDigestValue/Name/Encoding`. This defines both the type of
data being sent on the topic and how the data field of the message is encoded.

- `ForkDigestValue` - the lowercase hex-encoded (no "0x" prefix) bytes of
  `compute_fork_digest(genesis_validators_root, epoch)` where
  - `genesis_validators_root` is the static `Root` found in
    `state.genesis_validators_root`
  - `epoch` is the context epoch of the message to be sent on the topic
- `Name` - see table below
- `Encoding` - the encoding strategy describes a specific representation of
  bytes that will be transmitted over the wire. See the [Encodings](#encodings)
  section for further details.

Clients MUST reject messages with an unknown topic.

*Note*: `ForkDigestValue` is composed of values that are not known until the
genesis block/state are available. Due to this, clients SHOULD NOT subscribe to
gossipsub topics until these genesis values are known.

The optional `from` (1), `seqno` (3), `signature` (5) and `key` (6) protobuf
fields are omitted from the message, since messages are identified by content,
anonymous, and signed where necessary in the application layer. Starting from
Gossipsub v1.1, clients MUST enforce this by applying the `StrictNoSign`
[signature policy](https://github.com/libp2p/specs/blob/master/pubsub/README.md#signature-policy-options).

#### Message ID

The `message-id` of a gossipsub message incorporates the message `topic` along
with the message `data`. These are fields of the `Message` Protobuf, and
interpreted as empty byte strings if missing. The `message-id` MUST be the
following 20 byte value computed from the message:

- If `message.data` has a valid snappy decompression, set `message-id` to the
  first 20 bytes of the `SHA256` hash of the concatenation of the following
  data: `MESSAGE_DOMAIN_VALID_SNAPPY`, the length of the topic byte string
  (encoded as little-endian `Uint64`), the topic byte string, and the snappy
  decompressed message data: i.e.
  `SHA256(MESSAGE_DOMAIN_VALID_SNAPPY + uint_to_bytes(Uint64(len(message.topic))) + message.topic + snappy_decompress(message.data))[:20]`.
- Otherwise, set `message-id` to the first 20 bytes of the `SHA256` hash of the
  concatenation of the following data: `MESSAGE_DOMAIN_INVALID_SNAPPY`, the
  length of the topic byte string (encoded as little-endian `Uint64`), the topic
  byte string, and the raw message data: i.e.
  `SHA256(MESSAGE_DOMAIN_INVALID_SNAPPY + uint_to_bytes(Uint64(len(message.topic))) + message.topic + message.data)[:20]`.

Where relevant, clients MUST reject messages with `message-id` sizes other than
20 bytes.

*Note*: The above logic handles two exceptional cases: (1) multiple snappy
`data` can decompress to the same value, and (2) some message `data` can fail to
snappy decompress altogether.

*Note*: Messages on topics with the Phase 0 fork digest use the
[version 0 `message-id` procedure](beacon-0.md#message-id), which
does not include the topic. Messages on topics with the Altair (or a later)
fork digest use the procedure defined here. If an implementation only supports
a single `message-id` function, it can define a switch inline; for example,
`if topic in phase0_topics: return phase0_msg_id_fn(message) else return altair_msg_id_fn(message)`.

#### Topic table

The payload is carried in the `data` field of a gossipsub message, and varies
depending on the topic. The last column links to the gossip validation function
for the topic in consensus-specs.

| Name                             | Message Type              | Gossip validation                                                                                                                     |
| -------------------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `beacon_block`                   | `SignedBeaconBlock`       | [`validate_beacon_block_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/capella/p2p-interface.md#modified-beacon_block)                              |
| `beacon_aggregate_and_proof`     | `SignedAggregateAndProof` | [`validate_beacon_aggregate_and_proof_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#beacon_aggregate_and_proof) |
| `beacon_attestation_{subnet_id}` | `Attestation`             | [`validate_beacon_attestation_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#beacon_attestation_subnet_id)       |
| `voluntary_exit`                 | `SignedVoluntaryExit`     | [`validate_voluntary_exit_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#voluntary_exit)                         |
| `proposer_slashing`              | `ProposerSlashing`        | [`validate_proposer_slashing_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#proposer_slashing)                   |
| `attester_slashing`              | `AttesterSlashing`        | [`validate_attester_slashing_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#attester_slashing)                   |
| `sync_committee_contribution_and_proof` | `SignedContributionAndProof` | [`validate_sync_committee_contribution_and_proof_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/p2p-interface.md#new-sync_committee_contribution_and_proof) |
| `sync_committee_{subnet_id}`     | `SyncCommitteeMessage`    | [`validate_sync_committee_message_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/p2p-interface.md#new-sync_committee_subnet_id) |
| `bls_to_execution_change`        | `SignedBLSToExecutionChange` | [`validate_bls_to_execution_change_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/capella/p2p-interface.md#new-bls_to_execution_change) |

The light client topics `light_client_finality_update` and
`light_client_optimistic_update` are listed in the
[light client networking](#light-client-networking) section.

Note that the `ForkDigestValue` path segment of the topic separates the
pre-fork and post-fork variants of a topic whose payload type changes at a fork
(e.g. `beacon_block`, whose `SignedBeaconBlock` payload changes in Altair).

Clients MUST reject (fail validation) messages containing an incorrect type, or
invalid payload.

When processing incoming gossip, clients MAY descore or disconnect peers who
fail to observe these constraints.

For any optional queueing, clients SHOULD maintain maximum queue sizes to avoid
DoS vectors.

Gossipsub v1.1 introduces
[Extended Validators](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#extended-validators)
for the application to aid in the gossipsub peer-scoring scheme. We utilize
`ACCEPT`, `REJECT`, and `IGNORE`. For each gossipsub topic, there are
application specific validations, specified as `validate_*_gossip` functions in
consensus-specs and linked from the table above and from each topic below. If
all validations pass, return `ACCEPT`. If one or more validations fail while
processing the items in order, return either `REJECT` or `IGNORE` as specified
by the failing condition.

#### Global topics

There are two primary global topics used to propagate beacon blocks
(`beacon_block`) and aggregate attestations (`beacon_aggregate_and_proof`) to
all nodes on the network.

There are four additional global topics that are used to propagate lower
frequency validator messages (`voluntary_exit`, `proposer_slashing`,
`attester_slashing`, and `bls_to_execution_change`), and one global topic to
propagate partially aggregated sync committee messages
(`sync_committee_contribution_and_proof`) to all potential proposers of beacon
blocks.

##### `beacon_block`

The `beacon_block` topic is used solely for propagating new signed beacon blocks
to all nodes on the networks. Signed blocks are sent in their entirety. The
payload type is the `SignedBeaconBlock` of the fork identified by the topic's
fork digest.

*Note*: Since Bellatrix, blocks with execution enabled are permitted to
propagate regardless of the validity of the execution payload. This prevents
network segregation between
[optimistic](https://github.com/ethereum/consensus-specs/blob/master/specs/bellatrix/optimistic-sync.md) and non-optimistic nodes.

**Validation**:
[`validate_beacon_block_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/capella/p2p-interface.md#modified-beacon_block)

##### `beacon_aggregate_and_proof`

The `beacon_aggregate_and_proof` topic is used to propagate aggregated
attestations (as `SignedAggregateAndProof`s) to subscribing nodes (typically
validators) to be included in future blocks.

**Validation**:
[`validate_beacon_aggregate_and_proof_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#beacon_aggregate_and_proof)

##### `voluntary_exit`

The `voluntary_exit` topic is used solely for propagating signed voluntary
validator exits to proposers on the network. Signed voluntary exits are sent in
their entirety.

**Validation**:
[`validate_voluntary_exit_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#voluntary_exit)

##### `proposer_slashing`

The `proposer_slashing` topic is used solely for propagating proposer slashings
to proposers on the network. Proposer slashings are sent in their entirety.

**Validation**:
[`validate_proposer_slashing_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#proposer_slashing)

##### `attester_slashing`

The `attester_slashing` topic is used solely for propagating attester slashings
to proposers on the network. Attester slashings are sent in their entirety.

**Validation**:
[`validate_attester_slashing_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#attester_slashing)

##### `bls_to_execution_change`

The `bls_to_execution_change` topic is used solely for propagating signed BLS to
execution change messages on the network. Signed messages are sent in their
entirety.

**Validation**:
[`validate_bls_to_execution_change_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/capella/p2p-interface.md#new-bls_to_execution_change)

##### `sync_committee_contribution_and_proof`

This topic is used to propagate partially aggregated sync committee messages to
be included in future blocks.

**Validation**:
[`validate_sync_committee_contribution_and_proof_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/p2p-interface.md#new-sync_committee_contribution_and_proof)

#### Attestation subnets

Attestation subnets are used to propagate unaggregated attestations to
subsections of the network.

##### `beacon_attestation_{subnet_id}`

The `beacon_attestation_{subnet_id}` topics are used to propagate unaggregated
attestations to the subnet `subnet_id` (typically beacon and persistent
committees) to be aggregated before being gossiped to
`beacon_aggregate_and_proof`.

**Validation**:
[`validate_beacon_attestation_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md#beacon_attestation_subnet_id)

#### Attestations and Aggregation

Attestation broadcasting is grouped into subnets defined by a topic. The number
of subnets is defined via `ATTESTATION_SUBNET_COUNT`. The correct subnet for an
attestation can be calculated with
[`compute_subnet_for_attestation`](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/validator.md#broadcast-attestation).
`beacon_attestation_{subnet_id}` topics, are rotated through throughout the
epoch in a similar fashion to rotating through shards in committees (future
beacon-chain upgrade). The subnets are rotated through with
`committees_per_slot = get_committee_count_per_slot(state, attestation.data.target.epoch)`
subnets per slot.

Unaggregated attestations are sent as `Attestation`s to the subnet topic,
`beacon_attestation_{compute_subnet_for_attestation(committees_per_slot, attestation.data.slot, attestation.data.index)}`
as `Attestation`s.

Aggregated attestations are sent to the `beacon_aggregate_and_proof` topic as
`AggregateAndProof`s.

#### Sync committee subnets

Sync committee subnets are used to propagate unaggregated sync committee
messages to subsections of the network.

##### `sync_committee_{subnet_id}`

The `sync_committee_{subnet_id}` topics are used to propagate unaggregated sync
committee messages to the subnet `subnet_id` to be aggregated before being
gossiped to the global `sync_committee_contribution_and_proof` topic.

**Validation**:
[`validate_sync_committee_message_gossip`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/p2p-interface.md#new-sync_committee_subnet_id)

#### Sync committees and aggregation

The aggregation scheme closely follows the design of the attestation aggregation
scheme. Sync committee messages are broadcast into "subnets" defined by a topic.
The number of subnets is defined by `SYNC_COMMITTEE_SUBNET_COUNT` in the
[Altair validator guide](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/validator.md#constants). Sync committee
members are divided into "subcommittees" which are then assigned to a subnet
for the duration of tenure in the sync committee. Individual validators can be
duplicated in the broader sync committee such that they are included multiple
times in a given subcommittee or across multiple subcommittees.

Unaggregated messages (along with metadata) are sent as `SyncCommitteeMessage`s
on the `sync_committee_{subnet_id}` topics. The subnets for a validator are
given by
[`compute_subnets_for_sync_committee`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/validator.md#broadcast-sync-committee-message).

Aggregated sync committee messages are packaged into (signed)
`SyncCommitteeContribution` along with proofs and gossiped to the
`sync_committee_contribution_and_proof` topic.

### Transitioning the gossip

With any fork, the fork version, and thus the `ForkDigestValue`, change. Message
types are unique per topic, and so for a smooth transition a node must
temporarily subscribe to both the old and new topics.

The topics that are not removed in a fork are updated with a new
`ForkDigestValue`. In advance of the fork, a node SHOULD subscribe to the
post-fork variants of the topics.

Subscriptions are expected to be well-received, all updated nodes should
subscribe as well. Topic-meshes can be grafted quickly as the nodes are already
connected and exchanging gossip control messages.

Messages SHOULD NOT be re-broadcast from one fork to the other. A node's
behavior before the fork and after the fork are as follows:

Pre-fork:

- Peers who propagate messages on the post-fork topics MAY be scored negatively
  proportionally to time till fork, to account for clock discrepancy.
- Messages can be IGNORED on the post-fork topics, with a
  `MAXIMUM_GOSSIP_CLOCK_DISPARITY` margin.

Post-fork:

- Peers who propagate messages on the pre-fork topics MUST NOT be scored
  negatively. Lagging IWANT may force them to.
- Messages on pre and post-fork variants of topics share application-level
  caches. E.g. an attestation on the both the old and new topic is ignored like
  any duplicate.
- Two epochs after the fork, pre-fork topics SHOULD be unsubscribed from. This
  is well after the configured `seen_ttl`.

### Encodings

Topics are post-fixed with an encoding. Encodings define how the payload of a
gossipsub message is encoded.

- `ssz_snappy` - All objects are SSZ-encoded and then compressed with
  [Snappy](https://github.com/google/snappy) block compression. Example: The
  beacon aggregate attestation topic string is
  `/eth2/446a7232/beacon_aggregate_and_proof/ssz_snappy`, the fork digest is
  `446a7232` and the data field of a gossipsub message is an `AggregateAndProof`
  that has been SSZ-encoded and then compressed with Snappy.

Snappy has two formats: "block" and "frames" (streaming). Gossip messages remain
relatively small (100s of bytes to 100s of kilobytes) so
[basic snappy block compression](https://github.com/google/snappy/blob/master/format_description.txt)
is used to avoid the additional overhead associated with snappy frames.

Implementations MUST use a single encoding for gossip. Changing an encoding will
require coordination between participating implementations.

### Gossipsub size limits

Size limits are placed both on the
[`RPCMsg`](https://github.com/libp2p/specs/blob/b5f7fce29b32d4c7d0efe37b019936a11e5db872/pubsub/README.md#the-rpc)
frame as well as the encoded payload in each
[`Message`](https://github.com/libp2p/specs/blob/b5f7fce29b32d4c7d0efe37b019936a11e5db872/pubsub/README.md#the-message).

Clients MUST reject and MUST NOT emit or propagate messages whose size exceed
the following limits:

- The size of the encoded `RPCMsg` (including control messages, framing, topics,
  etc) must not exceed `max_message_size()`.
- The size of the compressed payload in the `Message.data` field must not exceed
  `max_compressed_len(MAX_PAYLOAD_SIZE)`.
- The size of the uncompressed payload must not exceed `MAX_PAYLOAD_SIZE` or the
  [type-specific SSZ bound](rationale.md#what-are-ssz-type-size-bounds),
  whichever is lower.

## The Req/Resp domain

### Protocol identification

Each message type is segregated into its own libp2p protocol ID, which is a
case-sensitive UTF-8 string of the form:

```
/ProtocolPrefix/MessageName/SchemaVersion/Encoding
```

With:

- `ProtocolPrefix` - messages are grouped into families identified by a shared
  libp2p protocol name prefix. In this case, we use `/eth2/beacon_chain/req`.
- `MessageName` - each request is identified by a name consisting of English
  alphabet, digits and underscores (`_`).
- `SchemaVersion` - an ordinal version number (e.g. 1, 2, 3…). Each schema is
  versioned to facilitate backward and forward-compatibility when possible.
- `Encoding` - while the schema defines the data types in more abstract terms,
  the encoding strategy describes a specific representation of bytes that will
  be transmitted over the wire. See the [Encodings](#encoding-strategies)
  section for further details.

This protocol segregation allows libp2p `multistream-select 1.0` /
`multiselect 2.0` to handle the request type, version, and encoding negotiation
before establishing the underlying streams.

### Req/Resp interaction

We use ONE stream PER request/response interaction. Streams are closed when the
interaction finishes, whether in success or in error.

Request/response messages MUST adhere to the encoding specified in the protocol
name and follow this structure (relaxed BNF grammar):

```
request   ::= <encoding-dependent-header> | <encoded-payload>
response  ::= <response_chunk>*
response_chunk  ::= <result> | <context-bytes> | <encoding-dependent-header> | <encoded-payload>
result    ::= “0” | “1” | “2” | "3" | [“128” ... ”255”]
```

The encoding-dependent header may carry metadata or assertions such as the
encoded payload length, for integrity and attack proofing purposes. Because
req/resp streams are single-use and stream closures implicitly delimit the
boundaries, it is not strictly necessary to length-prefix payloads; however,
certain encodings like SSZ do, for added security.

The `<context-bytes>` field functions as a short meta-data, defined per
req-resp method, and can parametrize the payload decoder. For the version 0
methods (`/1/` schema versions) `<context-bytes>` is empty. On a non-zero
`<result>` with `ErrorMessage` payload, the `<context-bytes>` is also empty.

#### `ForkDigest`-context

SSZ type definitions may change from fork to fork. For this common case, we
define the `ForkDigest`-context:

A fixed-width 4 byte `<context-bytes>`, set to the `ForkDigest` matching the
chunk: `compute_fork_digest(genesis_validators_root, epoch)`. The method
definitions below state how the context `epoch` is derived for each chunk, and
which SSZ type is selected by each fork version.

A `response` is formed by zero or more `response_chunk`s. Responses that consist
of a single SSZ-list (such as `BlocksByRange` and `BlocksByRoot`) send each list
item as a `response_chunk`. All other response types (non-Lists) send a single
`response_chunk`.

For both `request`s and `response`s, the `encoding-dependent-header` MUST be
valid, and the `encoded-payload` must be valid within the constraints of the
`encoding-dependent-header`. This includes type-specific bounds on payload size
for some encoding strategies. Regardless of these type specific bounds, a global
maximum uncompressed byte size of `MAX_PAYLOAD_SIZE` MUST be applied to all
method response chunks.

Clients MUST ensure that lengths are within these bounds; if not, they SHOULD
reset the stream immediately. Clients tracking peer reputation MAY decrement the
score of the misbehaving peer under this circumstance.

#### Requesting side

Once a new stream with the protocol ID for the request type has been negotiated,
the full request message SHOULD be sent immediately. The request MUST be encoded
according to the encoding strategy.

The requester MUST close the write side of the stream once it finishes writing
the request message. At this point, the stream will be half-closed.

The requester MUST NOT make more than `MAX_CONCURRENT_REQUESTS` concurrent
requests with the same protocol ID.

If a timeout occurs or the response is no longer relevant, the requester SHOULD
reset the stream.

A requester SHOULD read from the stream until either:

1. An error result is received in one of the chunks (the error payload MAY be
   read before stopping).
2. The responder closes the stream.
3. Any part of the `response_chunk` fails validation.
4. The maximum number of requested chunks are read.

For requests consisting of a single valid `response_chunk`, the requester SHOULD
read the chunk fully, as defined by the `encoding-dependent-header`, before
closing the stream.

#### Responding side

Once a new stream with the protocol ID for the request type has been negotiated,
the responder SHOULD process the incoming request and MUST validate it before
processing it. Request processing and validation MUST be done according to the
encoding strategy, until EOF (denoting stream half-closure by the requester).

The responder MUST:

1. Use the encoding strategy to read the optional header.
2. If there are any length assertions for length `N`, it should read exactly `N`
   bytes from the stream, at which point an EOF should arise (no more bytes).
   Should this not be the case, it should be treated as a failure.
3. Deserialize the expected type, and process the request.
4. Write the response which may consist of zero or more `response_chunk`s
   (result, optional header, payload).
5. Close their write side of the stream. At this point, the stream will be fully
   closed.

If steps (1), (2), or (3) fail due to invalid, malformed, or inconsistent data,
the responder MUST respond in error. Clients tracking peer reputation MAY record
such failures, as well as unexpected events, e.g. early stream resets.

The responder MAY rate-limit chunks by withholding each chunk until capacity is
available. The responder MUST NOT respond with an error or close the stream when
rate limiting.

When rate limiting, the responder MUST send each `response_chunk` in full
promptly but may introduce delays between each chunk.

Chunks start with a **single-byte** response code which determines the contents
of the `response_chunk` (`result` particle in the BNF grammar above). For
multiple chunks, only the last chunk is allowed to have a non-zero error code
(i.e. The chunk stream is terminated once an error occurs).

The response code can have one of the following values, encoded as a single
unsigned byte:

- 0: **Success** -- a normal response follows, with contents matching the
  expected message schema and encoding specified in the request.
- 1: **InvalidRequest** -- the contents of the request are semantically invalid,
  or the payload is malformed, or could not be understood. The response payload
  adheres to the schema (described below).
- 2: **ServerError** -- the responder encountered an error while processing the
  request. The response payload adheres to the schema (described below).
- 3: **ResourceUnavailable** -- the responder does not have requested resource.
  The response payload adheres to the schema (described below). *Note*: This
  response code is only valid as a response where specified.

Clients MAY use response codes above `128` to indicate alternative, erroneous
request-specific responses.

The range `[4, 127]` is RESERVED for future usages, and should be treated as
error if not recognized expressly.

```
(
  error_message: ErrorMessage
)
```

*Note*: By convention, the `error_message` is a sequence of bytes that MAY be
interpreted as a UTF-8 string (for debugging purposes). Clients MUST treat as
valid any byte sequences.

The responder MAY penalise peers that concurrently open more than
`MAX_CONCURRENT_REQUESTS` streams for the same request type, for the protocol
IDs defined in this specification.

### Encoding strategies

The token of the negotiated protocol ID specifies the type of encoding to be
used for the req/resp interaction. Only one value is possible at this time:

- `ssz_snappy`: The contents are first SSZ-encoded and then compressed with
  [Snappy](https://github.com/google/snappy) frames compression. For objects
  containing a single field, only the field is SSZ-encoded not a container with
  a single field. For example, the `BeaconBlocksByRoot` request is an
  SSZ-encoded list of `Root`'s. This encoding type MUST be supported by all
  clients.

#### SSZ-snappy encoding strategy

The [SimpleSerialize (SSZ) specification](https://github.com/ethereum/ssz-specs)
outlines how objects are SSZ-encoded.

To achieve snappy encoding on top of SSZ, we feed the serialized form of the
object to the Snappy compressor on encoding. The inverse happens on decoding.

Snappy has two formats: "block" and "frames" (streaming). To support large
requests and response chunks, snappy-framing is used.

Since snappy frame contents
[have a maximum size of `65536` bytes](https://github.com/google/snappy/blob/master/framing_format.txt#L104)
and frame headers are just `identifier (1) + checksum (4)` bytes, the expected
buffering of a single frame is acceptable.

**Encoding-dependent header:** Req/Resp protocols using the `ssz_snappy`
encoding strategy MUST encode the length of the raw SSZ bytes, encoded as an
unsigned
[protobuf varint](https://developers.google.com/protocol-buffers/docs/encoding#varints).

*Writing*: By first computing and writing the SSZ byte length, the SSZ encoder
can then directly write the chunk contents to the stream. When Snappy is
applied, it can be passed through a buffered Snappy writer to compress frame by
frame.

*Reading*: After reading the expected SSZ byte length, the SSZ decoder can
directly read the contents from the stream. When snappy is applied, it can be
passed through a buffered Snappy reader to decompress frame by frame.

Before reading the payload, the header MUST be validated:

- The length-prefix MUST be encoded as an unsigned protobuf varint. It SHOULD be
  minimally encoded (i.e., without any redundant bytes) and MUST not exceed 10
  bytes in length, which is sufficient to represent any `Uint64` value. The
  length-prefix MUST be decoded into a type which supports the full range of
  `Uint64` values.
- The length-prefix is within the expected
  [size bounds derived from the payload SSZ type](rationale.md#what-are-ssz-type-size-bounds)
  or `MAX_PAYLOAD_SIZE`, whichever is smaller.

After reading a valid header, the payload MAY be read, while maintaining the
size constraints from the header.

A reader MUST NOT read more than `max_compressed_len(n)` bytes after reading the
SSZ length-prefix `n` from the header.

A reader MUST consider the following cases as invalid input:

- Any remaining bytes, after having read the `n` SSZ bytes. An EOF is expected
  if more bytes are read than required.
- An early EOF, before fully reading the declared length-prefix worth of SSZ
  bytes.

In case of an invalid input (header or payload), a reader MUST:

- From requests: send back an error message, response code `InvalidRequest`. The
  request itself is ignored.
- From responses: ignore the response, the response MUST be considered bad
  server behavior.

All messages that contain only a single field MUST be encoded directly as the
type of that field and MUST NOT be encoded as an SSZ container.

Responses that are SSZ-lists (for example `SignedBeaconBlocks`) send their
constituents individually as `response_chunk`s. For example, the
`SignedBeaconBlocks` response type sends zero or more `response_chunk`s. Each
_successful_ `response_chunk` contains a single `SignedBeaconBlock` payload.

### Messages

Non-faulty, [optimistic](https://github.com/ethereum/consensus-specs/blob/master/specs/bellatrix/optimistic-sync.md) nodes may send
blocks which result in an INVALID response from an execution engine. To prevent
network segregation between optimistic and non-optimistic nodes, transmission of
an INVALID execution payload via the Req/Resp domain SHOULD NOT cause a node to
be down-scored or disconnected. Transmission of a block which is invalid due to
any consensus-layer rules (i.e., *not* execution-layer rules) MAY result in
down-scoring or disconnection.

#### Status v1

**Protocol ID:** `/eth2/beacon_chain/req/status/1/`

Request, Response Content:

```
(
  fork_digest: ForkDigest
  finalized_root: Root
  finalized_epoch: Epoch
  head_root: Root
  head_slot: Slot
)
```

As seen by the client at the time of sending the message:

- `fork_digest`: The node's `ForkDigest`
  (`compute_fork_digest(genesis_validators_root, epoch)`) where
  - `genesis_validators_root` is the static `Root` found in
    `state.genesis_validators_root`
  - `epoch` is the node's current epoch defined by the wall-clock time (not
    necessarily the epoch to which the node is sync).
- `finalized_root`: `store.finalized_checkpoint.root` according to
  [fork choice](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/fork-choice.md).
  (Note this defaults to `Root(b'\x00' * 32)` for the genesis finalized
  checkpoint).
- `finalized_epoch`: `store.finalized_checkpoint.epoch` according to
  [fork choice](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/fork-choice.md).
- `head_root`: The `hash_tree_root` root of the current head block
  (`BeaconBlock`).
- `head_slot`: The slot of the block corresponding to the `head_root`.

The dialing client MUST send a `Status` request upon connection.

The request/response MUST be encoded as an SSZ-container.

The response MUST consist of a single `response_chunk`.

Clients SHOULD immediately disconnect from one another following the handshake
above under the following conditions:

1. If `fork_digest` does not match the node's local `fork_digest`, since the
   client’s chain is on another fork.
2. If the (`finalized_root`, `finalized_epoch`) shared by the peer is not in the
   client's chain at the expected epoch. For example, if Peer 1 sends (root,
   epoch) of (A, 5) and Peer 2 sends (B, 3) but Peer 1 has root C at epoch 3,
   then Peer 1 would disconnect because it knows that their chains are
   irreparably disjoint.

Once the handshake completes, the client with the lower `finalized_epoch` or
`head_slot` (if the clients have equal `finalized_epoch`s) SHOULD request beacon
blocks from its counterparty via the `BeaconBlocksByRange` request.

*Note*: Under abnormal network condition or after some rounds of
`BeaconBlocksByRange` requests, the client might need to send `Status` request
again to learn if the peer has a higher head. Implementers are free to implement
such behavior in their own way.

#### Goodbye v1

**Protocol ID:** `/eth2/beacon_chain/req/goodbye/1/`

Request, Response Content:

```
(
  Uint64
)
```

Client MAY send goodbye messages upon disconnection. The reason field MAY be one
of the following values:

- 1: Client shut down.
- 2: Irrelevant network.
- 3: Fault/error.

Clients MAY use reason codes above `128` to indicate alternative, erroneous
request-specific responses.

The range `[4, 127]` is RESERVED for future usage.

The request/response MUST be encoded as a single SSZ-field.

The response MUST consist of a single `response_chunk`.

#### BeaconBlocksByRange v2

**Protocol ID:** `/eth2/beacon_chain/req/beacon_blocks_by_range/2/`

Request Content:

```
(
  start_slot: Slot
  count: Uint64
  step: Uint64 # Deprecated, must be set to 1
)
```

Response Content:

```
(
  SignedBeaconBlocks
)
```

Requests beacon blocks in the slot range `[start_slot, start_slot + count)`,
leading up to the current head block as selected by fork choice. For example,
requesting blocks starting at `start_slot=2` and `count=4` would return the
blocks at slots `[2, 3, 4, 5]`. In cases where a slot is empty for a given slot
number, no block is returned. For example, if slot 4 were empty in the previous
example, the returned array would contain `[2, 3, 5]`.

`step` is deprecated and must be set to 1. Clients may respond with a single
block if a larger step is returned during the deprecation transition period.

`BeaconBlocksByRange` is primarily used to sync historical blocks.

The request MUST be encoded as an SSZ-container.

The response MUST consist of zero or more `response_chunk`. Each _successful_
`response_chunk` MUST contain a single `SignedBeaconBlock` payload.

Clients MUST keep a record of signed blocks seen on the epoch range
`[max(GENESIS_EPOCH, current_epoch - compute_min_epochs_for_block_requests()), current_epoch]`
where `current_epoch` is defined by the current wall-clock time, and clients
MUST support serving requests of blocks on this range.

*Note*: The epoch range above is based on the current wall-clock time and does
not account for finality. Clients MUST also keep a record of all blocks more
recent than their latest finalized checkpoint and MUST support serving requests
of these blocks, even if outside of this range, as peers need them to sync
during an extended period of non-finality.

##### `compute_min_epochs_for_block_requests`

```python
def compute_min_epochs_for_block_requests() -> Epoch:
    """
    Return the minimum epoch range over which a node must serve blocks.
    """
    return MIN_VALIDATOR_WITHDRAWABILITY_DELAY + CHURN_LIMIT_QUOTIENT // 2
```

Peers that are unable to reply to block requests within the
`compute_min_epochs_for_block_requests()` epoch range SHOULD respond with error
code `3: ResourceUnavailable`. Such peers that are unable to successfully reply
to this range of requests MAY get descored or disconnected at any time.

*Note*: The above requirement implies that nodes that start from a recent weak
subjectivity checkpoint MUST backfill the local block database to at least epoch
`current_epoch - compute_min_epochs_for_block_requests()` to be fully compliant
with `BlocksByRange` requests. To safely perform such a backfill of blocks to
the recent state, the node MUST validate both (1) the proposer signatures and
(2) that the blocks form a valid chain up to the most recent block referenced in
the weak subjectivity state.

*Note*: Although clients that bootstrap from a weak subjectivity checkpoint can
begin participating in the networking immediately, other peers MAY disconnect
and/or temporarily ban such an un-synced or semi-synced client.

Clients MUST respond with at least the first block that exists in the range, if
they have it, and no more than `MAX_REQUEST_BLOCKS` blocks.

The following blocks, where they exist, MUST be sent in consecutive order.

Clients MAY limit the number of blocks in the response.

The response MUST contain no more than `count` blocks.

Clients MUST respond with blocks from their view of the current fork choice --
that is, blocks from the single chain defined by the current head. Of note,
blocks from slots before the finalization MUST lead to the finalized block
reported in the `Status` handshake.

Clients MUST respond with blocks that are consistent from a single chain within
the context of the request. This applies to any `step` value. In particular when
`step == 1`, each `parent_root` MUST match the `hash_tree_root` of the preceding
block.

After the initial block, clients MAY stop in the process of responding if their
fork choice changes the view of the chain in the context of the request.

A `ForkDigest`-context is used to select the fork namespace of the Response
type. For each successful `response_chunk`, the `ForkDigest` context epoch is
determined by `compute_epoch_at_slot(signed_beacon_block.message.slot)`.

Per `fork_version = compute_fork_version(epoch)`:

| `fork_version`           | Chunk SSZ type                |
| ------------------------ | ----------------------------- |
| `GENESIS_FORK_VERSION`   | `phase0.SignedBeaconBlock`    |
| `ALTAIR_FORK_VERSION`    | `altair.SignedBeaconBlock`    |
| `BELLATRIX_FORK_VERSION` | `bellatrix.SignedBeaconBlock` |
| `CAPELLA_FORK_VERSION`   | `capella.SignedBeaconBlock`   |

#### BeaconBlocksByRoot v2

**Protocol ID:** `/eth2/beacon_chain/req/beacon_blocks_by_root/2/`

Request Content:

```
(
  BeaconBlockRoots
)
```

Response Content:

```
(
  SignedBeaconBlocks
)
```

Requests blocks by block root (= `hash_tree_root(SignedBeaconBlock.message)`).
The response is a list of `SignedBeaconBlock` whose length is less than or equal
to the number of requested blocks. It may be less in the case that the
responding peer is missing blocks.

No more than `MAX_REQUEST_BLOCKS` may be requested at a time.

`BeaconBlocksByRoot` is primarily used to recover recent blocks (e.g. when
receiving a block or attestation whose parent is unknown).

The request MUST be encoded as an SSZ-field.

The response MUST consist of zero or more `response_chunk`. Each _successful_
`response_chunk` MUST contain a single `SignedBeaconBlock` payload.

Clients MUST support requesting blocks on the epoch range
`[max(GENESIS_EPOCH, current_epoch - compute_min_epochs_for_block_requests()), current_epoch]`.
If any root in the request content references a block earlier than this range,
peers MAY respond with error code `3: ResourceUnavailable` or not include the
block in the response.

Clients MUST respond with at least one block, if they have it. Clients MAY limit
the number of blocks in the response.

Clients MAY include a block in the response as soon as it passes the gossip
validation rules. Clients SHOULD NOT respond with blocks that fail the beacon
chain state transition.

A `ForkDigest`-context is used to select the fork namespace of the Response
type. For each successful `response_chunk`, the `ForkDigest` context epoch is
determined by `compute_epoch_at_slot(signed_beacon_block.message.slot)`.

Per `fork_version = compute_fork_version(epoch)`:

| `fork_version`           | Chunk SSZ type                |
| ------------------------ | ----------------------------- |
| `GENESIS_FORK_VERSION`   | `phase0.SignedBeaconBlock`    |
| `ALTAIR_FORK_VERSION`    | `altair.SignedBeaconBlock`    |
| `BELLATRIX_FORK_VERSION` | `bellatrix.SignedBeaconBlock` |
| `CAPELLA_FORK_VERSION`   | `capella.SignedBeaconBlock`   |

#### Ping v1

**Protocol ID:** `/eth2/beacon_chain/req/ping/1/`

Request Content:

```
(
  Uint64
)
```

Response Content:

```
(
  Uint64
)
```

Sent intermittently, the `Ping` protocol checks liveness of connected peers.
Peers request and respond with their local metadata sequence number
(`MetaData.seq_number`).

If the peer does not respond to the `Ping` request, the client MAY disconnect
from the peer.

A client can then determine if their local record of a peer's MetaData is up to
date and MAY request an updated version via the `MetaData` RPC method if not.

The request MUST be encoded as an SSZ-field.

The response MUST consist of a single `response_chunk`.

#### GetMetaData v2

**Protocol ID:** `/eth2/beacon_chain/req/metadata/2/`

No Request Content.

Response Content:

```
(
  MetaData
)
```

Requests the MetaData of a peer, using the [`MetaData`](#metadata) definition
given above. The request opens and negotiates the stream without sending any
request content. Once established the receiving peer responds with it's local
most up-to-date MetaData.

The response MUST be encoded as an SSZ-container.

The response MUST consist of a single `response_chunk`.

## The discovery domain: discv5

Discovery Version 5 ([discv5](../discovery/discv5/discv5.md)) (Protocol
version v5.1) is used for peer discovery.

`discv5` is a standalone protocol, running on UDP on a dedicated port, meant for
peer discovery only. `discv5` supports self-certified, flexible peer records
(ENRs) and topic-based advertisement, both of which are (or will be)
requirements in this context.

### Integration into libp2p stacks

`discv5` SHOULD be integrated into the client’s libp2p stack by implementing an
adaptor to make it conform to the
[service discovery](https://github.com/libp2p/go-libp2p-core/blob/master/discovery/discovery.go)
and
[peer routing](https://github.com/libp2p/go-libp2p-core/blob/master/routing/routing.go#L36-L44)
abstractions and interfaces (go-libp2p links provided).

Inputs to operations include peer IDs (when locating a specific peer) or
capabilities (when searching for peers with a specific capability), and the
outputs will be multiaddrs converted from the ENR records returned by the discv5
backend.

This integration enables the libp2p stack to subsequently form connections and
streams with discovered peers.

### ENR structure

The Ethereum Node Record (ENR) for an Ethereum consensus client MUST contain the
following entries (exclusive of the sequence number and signature, which MUST be
present in an ENR):

- The compressed secp256k1 publickey, 33 bytes (`secp256k1` field).

The ENR MAY contain the following entries:

- An IPv4 address (`ip` field) and/or IPv6 address (`ip6` field).
- An IPv4 QUIC port (`quic` field) representing the local libp2p QUIC (UDP)
  listening port and/or the corresponding IPv6 port (`quic6` field).
- An IPv4 TCP port (`tcp` field) representing the local libp2p TCP listening
  port and/or the corresponding IPv6 port (`tcp6` field).
- An IPv4 UDP port (`udp` field) representing the local discv5 listening port
  and/or the corresponding IPv6 port (`udp6` field).

Specifications of these parameters can be found in the
[ENR Specification](../discovery/enr.md).

In addition, the ENR carries the consensus-layer specific entries described
below.

| Key        | Value       | Requirement |
| ---------- | ----------- | ----------- |
| `eth2`     | `ENRForkID` | MUST        |
| `attnets`  | `Attnets`   | see below   |
| `syncnets` | `Syncnets`  | see below   |

#### Attestation subnet bitfield

The ENR `attnets` entry signifies the attestation subnet bitfield with the
following form to more easily discover peers participating in particular
attestation gossip subnets.

| Key       | Value     |
| --------- | --------- |
| `attnets` | `Attnets` |

If a node's `MetaData.attnets` has any non-zero bit, the ENR MUST include the
`attnets` entry with the same value as `MetaData.attnets`.

If a node's `MetaData.attnets` is composed of all zeros, the ENR MAY optionally
include the `attnets` entry or leave it out entirely.

#### Sync committee bitfield

An additional bitfield is added to the ENR under the key `syncnets` to
facilitate sync committee subnet discovery. The length of this bitfield is
`SYNC_COMMITTEE_SUBNET_COUNT` where each bit corresponds to a distinct
`subnet_id` for a specific sync committee subnet. The `i`th bit is set in this
bitfield if the validator is currently subscribed to the `sync_committee_{i}`
topic.

| Key        | Value      |
| ---------- | ---------- |
| `syncnets` | `Syncnets` |

See the
[validator document](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/validator.md#sync-committee-subnet-stability)
for further details on how the new bits are used.

#### `eth2` field

ENRs MUST carry a generic `eth2` key with an 16-byte value of the node's current
fork digest, next fork version, and next fork epoch to ensure connections are
made with peers on the intended Ethereum network.

| Key    | Value       |
| ------ | ----------- |
| `eth2` | `ENRForkID` |

Specifically, the value of the `eth2` key MUST be the following SSZ encoded
object (`ENRForkID`)

```
(
  fork_digest: ForkDigest
  next_fork_version: Version
  next_fork_epoch: Epoch
)
```

The fields of `ENRForkID` are defined as:

- `fork_digest` is
  `compute_fork_digest(genesis_validators_root, current_epoch)`.
- `next_fork_version` is the fork version that will be in effect at
  `next_fork_epoch`: `compute_fork_version(current_epoch)` if
  `next_fork_epoch == FAR_FUTURE_EPOCH`, otherwise
  `compute_fork_version(next_fork_epoch)`.
- `next_fork_epoch` is the epoch at which the next fork is planned. If no future
  fork is planned, set `next_fork_epoch = FAR_FUTURE_EPOCH` to signal this fact.

*Note*: In the definitions above, `genesis_validators_root` is the static `Root`
found in `state.genesis_validators_root` and `current_epoch` is the node's
current epoch defined by the wall-clock time (not necessarily the epoch to which
the node is sync).

*Note*: `fork_digest` is composed of values that are not known until the genesis
block/state are available. Due to this, clients SHOULD NOT form ENRs and begin
peer discovery until genesis values are known. One notable exception to this
rule is the distribution of bootnode ENRs prior to genesis. In this case,
bootnode ENRs SHOULD be initially distributed with `eth2` field set as
`ENRForkID(fork_digest=compute_fork_digest(b'\x00'*32, GENESIS_EPOCH), next_fork_version=GENESIS_FORK_VERSION, next_fork_epoch=FAR_FUTURE_EPOCH)`.
After genesis values are known, the bootnodes SHOULD update ENRs to participate
in normal discovery operations.

Clients SHOULD connect to peers with `fork_digest`, `next_fork_version`, and
`next_fork_epoch` that match local values.

Clients MAY connect to peers with the same `fork_digest` but a different
`next_fork_version`/`next_fork_epoch`. Unless `ENRForkID` is manually updated to
matching prior to the earlier `next_fork_epoch` of the two clients, these
connecting clients will be unable to successfully interact starting at the
earlier `next_fork_epoch`.

## Subnet subscription

### Attestation subnet subscription

Because Phase 0 does not have shards and thus does not have Shard Committees,
there is no stable backbone to the attestation subnets
(`beacon_attestation_{subnet_id}`). To provide this stability, each beacon node
should:

- Remain subscribed to `SUBNETS_PER_NODE` for `EPOCHS_PER_SUBNET_SUBSCRIPTION`
  epochs.
- Maintain advertisement of the selected subnets in their node's ENR `attnets`
  entry by setting the selected `subnet_id` bits to `True` (e.g.
  `ENR["attnets"][subnet_id] = True`) for all persistent attestation subnets.
- Select these subnets based on their node-id as specified by the following
  `compute_subscribed_subnets(node_id, epoch)` function.

```python
def compute_attestation_subnet_prefix_bits() -> Uint64:
    """
    Return the number of NodeId bits to use when mapping to a subscribed subnet.
    """
    return ceillog2(ATTESTATION_SUBNET_COUNT) + ATTESTATION_SUBNET_EXTRA_BITS
```

```python
def compute_subscribed_subnet(node_id: NodeID, epoch: Epoch, index: int) -> SubnetID:
    prefix_bits = int(compute_attestation_subnet_prefix_bits())
    node_id_prefix = node_id >> int(NODE_ID_BITS - prefix_bits)
    node_offset = Uint64(node_id % Uint256(EPOCHS_PER_SUBNET_SUBSCRIPTION))
    permutation_seed = sha256(
        uint_to_bytes(Uint64((epoch + node_offset) // EPOCHS_PER_SUBNET_SUBSCRIPTION))
    )
    permutated_prefix = compute_shuffled_index(
        Uint64(node_id_prefix),
        Uint64(1 << prefix_bits),
        permutation_seed,
    )
    return SubnetID((permutated_prefix + index) % ATTESTATION_SUBNET_COUNT)
```

```python
def compute_subscribed_subnets(node_id: NodeID, epoch: Epoch) -> Sequence[SubnetID]:
    return [compute_subscribed_subnet(node_id, epoch, index) for index in range(SUBNETS_PER_NODE)]
```

*Note*: When preparing for an upgrade, a node must select and subscribe to
subnets of the future fork versioning at least `EPOCHS_PER_SUBNET_SUBSCRIPTION`
epochs in advance of the upgrade. These new subnets for the fork are maintained
in addition to those for the current fork until the fork occurs. After the fork
occurs, let the subnets from the previous fork reach the end of life with no
replacements.

### Sync committee subnet subscription

Nodes with validators in the current or next sync committee subscribe to the
`sync_committee_{subnet_id}` topics of their subcommittees and advertise them
in the ENR `syncnets` entry, as specified in
[Sync committee subnet stability](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/validator.md#sync-committee-subnet-stability)
of the Altair validator guide.

## Light client networking

*This section is optional.* It extends the networking specification with
additional messages, topics and data in the Req/Resp and gossip domains that
allow light clients to stay in sync with the network. Full nodes SHOULD support
it; light clients using libp2p MUST.

### Light client gossip topics

Gossip meshes are added to allow light clients to stay in sync with the network.
New global topics are added to provide light clients with the latest updates.

| Name                             | Message Type                  |
| -------------------------------- | ----------------------------- |
| `light_client_finality_update`   | `LightClientFinalityUpdate`   |
| `light_client_optimistic_update` | `LightClientOptimisticUpdate` |

#### `light_client_finality_update`

This topic is used to propagate the latest `LightClientFinalityUpdate` to light
clients, allowing them to keep track of the latest `finalized_header`.

The following validations MUST pass before forwarding the `finality_update` on
the network.

- _[IGNORE]_ The `finalized_header.beacon.slot` is greater than that of all
  previously forwarded `finality_update`s, or it matches the highest previously
  forwarded slot and also has a `sync_aggregate` indicating supermajority (>
  2/3) sync committee participation while the previously forwarded
  `finality_update` for that slot did not indicate supermajority
- _[IGNORE]_ The `finality_update` is received after the block at
  `signature_slot` was given enough time to propagate through the network --
  i.e. validatate that `get_sync_message_due_ms()` milliseconds (with a
  `MAXIMUM_GOSSIP_CLOCK_DISPARITY` allowance) has transpired since the start of
  `signature_slot`.

For full nodes, the following validations MUST additionally pass before
forwarding the `finality_update` on the network.

- _[IGNORE]_ The received `finality_update` matches the locally computed one
  exactly (as defined in
  [`create_light_client_finality_update`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/full-node.md#create_light_client_finality_update))

For light clients, the following validations MUST additionally pass before
forwarding the `finality_update` on the network.

- _[REJECT]_ The `finality_update` is valid -- i.e. validate that
  [`process_light_client_finality_update`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/sync-protocol.md#process_light_client_finality_update)
  does not indicate errors
- _[IGNORE]_ The `finality_update` advances the `finalized_header` of the local
  `LightClientStore` -- i.e. validate that processing `finality_update`
  increases `store.finalized_header.beacon.slot`

Light clients SHOULD call `process_light_client_finality_update` even if the
message is ignored.

The `ForkDigest` context epoch is determined by
`compute_epoch_at_slot(finality_update.attested_header.beacon.slot)`.

Per `fork_version = compute_fork_version(epoch)`:

| `fork_version`                  | Message SSZ type                   |
| ------------------------------- | ---------------------------------- |
| `GENESIS_FORK_VERSION`          | n/a                                |
| `ALTAIR_FORK_VERSION` through `BELLATRIX_FORK_VERSION` | `altair.LightClientFinalityUpdate` |
| `CAPELLA_FORK_VERSION` and later | `capella.LightClientFinalityUpdate` |

#### `light_client_optimistic_update`

This topic is used to propagate the latest `LightClientOptimisticUpdate` to
light clients, allowing them to keep track of the latest `optimistic_header`.

The following validations MUST pass before forwarding the `optimistic_update` on
the network.

- _[IGNORE]_ The `attested_header.beacon.slot` is greater than that of all
  previously forwarded `optimistic_update`s
- _[IGNORE]_ The `optimistic_update` is received after the block at
  `signature_slot` was given enough time to propagate through the network --
  i.e. validatate that `get_sync_message_due_ms()` milliseconds (with a
  `MAXIMUM_GOSSIP_CLOCK_DISPARITY` allowance) has transpired since the start of
  `optimistic_update.signature_slot`.

For full nodes, the following validations MUST additionally pass before
forwarding the `optimistic_update` on the network.

- _[IGNORE]_ The received `optimistic_update` matches the locally computed one
  exactly (as defined in
  [`create_light_client_optimistic_update`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/full-node.md#create_light_client_optimistic_update))

For light clients, the following validations MUST additionally pass before
forwarding the `optimistic_update` on the network.

- _[REJECT]_ The `optimistic_update` is valid -- i.e. validate that
  [`process_light_client_optimistic_update`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/sync-protocol.md#process_light_client_optimistic_update)
  does not indicate errors
- _[IGNORE]_ The `optimistic_update` either matches corresponding fields of the
  most recently forwarded `LightClientFinalityUpdate` (if any), or it advances
  the `optimistic_header` of the local `LightClientStore` -- i.e. validate that
  processing `optimistic_update` increases `store.optimistic_header.beacon.slot`

Light clients SHOULD call `process_light_client_optimistic_update` even if the
message is ignored.

The `ForkDigest` context epoch is determined by
`compute_epoch_at_slot(optimistic_update.attested_header.beacon.slot)`.

Per `fork_version = compute_fork_version(epoch)`:

| `fork_version`                  | Message SSZ type                     |
| ------------------------------- | ------------------------------------ |
| `GENESIS_FORK_VERSION`          | n/a                                  |
| `ALTAIR_FORK_VERSION` through `BELLATRIX_FORK_VERSION` | `altair.LightClientOptimisticUpdate` |
| `CAPELLA_FORK_VERSION` and later | `capella.LightClientOptimisticUpdate` |

### Light client Req/Resp methods

#### GetLightClientBootstrap v1

**Protocol ID:** `/eth2/beacon_chain/req/light_client_bootstrap/1/`

Request Content:

```
(
  Root
)
```

Response Content:

```
(
  LightClientBootstrap
)
```

Requests the `LightClientBootstrap` structure corresponding to a given
post-Altair beacon block root.

The request MUST be encoded as an SSZ-field.

Peers SHOULD provide results as defined in
[`create_light_client_bootstrap`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/full-node.md#create_light_client_bootstrap).
To fulfill a request, the requested block and its post state need to be known.

When a `LightClientBootstrap` instance cannot be produced for a given block
root, peers SHOULD respond with error code `3: ResourceUnavailable`.

For each successful `response_chunk`, the `ForkDigest` context epoch is
determined by `compute_epoch_at_slot(bootstrap.header.beacon.slot)`.

Per `fork_version = compute_fork_version(epoch)`:

| `fork_version`                  | Response SSZ type             |
| ------------------------------- | ----------------------------- |
| `GENESIS_FORK_VERSION`          | n/a                           |
| `ALTAIR_FORK_VERSION` through `BELLATRIX_FORK_VERSION` | `altair.LightClientBootstrap` |
| `CAPELLA_FORK_VERSION` and later | `capella.LightClientBootstrap` |

#### LightClientUpdatesByRange v1

**Protocol ID:** `/eth2/beacon_chain/req/light_client_updates_by_range/1/`

Request Content:

```
(
  start_period: Uint64
  count: Uint64
)
```

Response Content:

```
(
  LightClientUpdates
)
```

Requests the `LightClientUpdate` instances in the sync committee period range
`[start_period, start_period + count)`, leading up to the current head sync
committee period as selected by fork choice.

The request MUST be encoded as an SSZ-container.

The response MUST consist of zero or more `response_chunk`. Each _successful_
`response_chunk` MUST contain a single `LightClientUpdate` payload.

Peers SHOULD provide results as defined in
[`create_light_client_update`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/full-node.md#create_light_client_update).
They MUST respond with at least the earliest known result within the requested
range, and MUST send results in consecutive order (by period). The response
MUST NOT contain more than `min(MAX_REQUEST_LIGHT_CLIENT_UPDATES, count)`
results.

For each successful `response_chunk`, the `ForkDigest` context epoch is
determined by `compute_epoch_at_slot(update.attested_header.beacon.slot)`. Note
that the context epoch may differ from the one used to verify the
`update.sync_aggregate`, which is based on `update.signature_slot`.

Per `fork_version = compute_fork_version(epoch)`:

| `fork_version`                  | Response chunk SSZ type    |
| ------------------------------- | -------------------------- |
| `GENESIS_FORK_VERSION`          | n/a                        |
| `ALTAIR_FORK_VERSION` through `BELLATRIX_FORK_VERSION` | `altair.LightClientUpdate` |
| `CAPELLA_FORK_VERSION` and later | `capella.LightClientUpdate` |

#### GetLightClientFinalityUpdate v1

**Protocol ID:** `/eth2/beacon_chain/req/light_client_finality_update/1/`

No Request Content.

Response Content:

```
(
  LightClientFinalityUpdate
)
```

Requests the latest `LightClientFinalityUpdate` known by a peer.

Peers SHOULD provide results as defined in
[`create_light_client_finality_update`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/full-node.md#create_light_client_finality_update).

When no `LightClientFinalityUpdate` is available, peers SHOULD respond with
error code `3: ResourceUnavailable`.

For each successful `response_chunk`, the `ForkDigest` context epoch is
determined by
`compute_epoch_at_slot(finality_update.attested_header.beacon.slot)`. Note that
the context epoch may differ from the one used to verify the
`finality_update.sync_aggregate`, which is based on
`finality_update.signature_slot`.

Per `fork_version = compute_fork_version(epoch)`:

| `fork_version`                  | Response SSZ type                  |
| ------------------------------- | ---------------------------------- |
| `GENESIS_FORK_VERSION`          | n/a                                |
| `ALTAIR_FORK_VERSION` through `BELLATRIX_FORK_VERSION` | `altair.LightClientFinalityUpdate` |
| `CAPELLA_FORK_VERSION` and later | `capella.LightClientFinalityUpdate` |

#### GetLightClientOptimisticUpdate v1

**Protocol ID:** `/eth2/beacon_chain/req/light_client_optimistic_update/1/`

No Request Content.

Response Content:

```
(
  LightClientOptimisticUpdate
)
```

Requests the latest `LightClientOptimisticUpdate` known by a peer.

Peers SHOULD provide results as defined in
[`create_light_client_optimistic_update`](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/full-node.md#create_light_client_optimistic_update).

When no `LightClientOptimisticUpdate` is available, peers SHOULD respond with
error code `3: ResourceUnavailable`.

For each successful `response_chunk`, the `ForkDigest` context epoch is
determined by
`compute_epoch_at_slot(optimistic_update.attested_header.beacon.slot)`. Note
that the context epoch may differ from the one used to verify the
`optimistic_update.sync_aggregate`, which is based on
`optimistic_update.signature_slot`.

Per `fork_version = compute_fork_version(epoch)`:

| `fork_version`                  | Response SSZ type                    |
| ------------------------------- | ------------------------------------ |
| `GENESIS_FORK_VERSION`          | n/a                                  |
| `ALTAIR_FORK_VERSION` through `BELLATRIX_FORK_VERSION` | `altair.LightClientOptimisticUpdate` |
| `CAPELLA_FORK_VERSION` and later | `capella.LightClientOptimisticUpdate` |

### Light client and full node behaviour

Light clients using libp2p to stay in sync with the network SHOULD subscribe to
the [`light_client_finality_update`](#light_client_finality_update) and
[`light_client_optimistic_update`](#light_client_optimistic_update) pubsub
topics and validate all received messages while the
[light client sync process](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/light-client.md#light-client-sync-process)
supports processing `LightClientFinalityUpdate` and
`LightClientOptimisticUpdate` structures.

Light clients MAY also collect historic light client data and make it available
to other peers. If they do, they SHOULD advertise supported message endpoints in
the [light client Req/Resp domain](#light-client-reqresp-methods), and MAY also
update the contents of their [`Status`](#status-v1) message to reflect the
locally available light client data.

If only limited light client data is locally available, the light client SHOULD
use data based on `genesis_block` and `GENESIS_SLOT` in its `Status` message.
Hybrid peers that also implement full node functionality MUST only incorporate
data based on their full node sync progress into their `Status` message.

All full nodes SHOULD subscribe to and provide stability on the
`light_client_finality_update` and `light_client_optimistic_update` pubsub
topics by validating all received messages.

Whenever fork choice selects a new head block with a sync aggregate
participation `>= MIN_SYNC_COMMITTEE_PARTICIPANTS` and a post-Altair parent
block, full nodes with at least one validator assigned to the current sync
committee at the block's `slot` SHOULD broadcast derived light client data as
follows:

- If `finalized_header.beacon.slot` increased, a `LightClientFinalityUpdate`
  SHOULD be broadcasted to the pubsub topic `light_client_finality_update` if no
  matching message has not yet been forwarded as part of gossip validation.
- If `attested_header.beacon.slot` increased, a `LightClientOptimisticUpdate`
  SHOULD be broadcasted to the pubsub topic `light_client_optimistic_update` if
  no matching message has not yet been forwarded as part of gossip validation.

These messages SHOULD be broadcasted `get_sync_message_due_ms()` milliseconds
after the start of the slot. To ensure that the corresponding block was given
enough time to propagate through the network, they SHOULD NOT be sent earlier.
Note that this is different from how other messages are handled, e.g.,
attestations, which may be sent early.

## Change log

### Version 2 (Bellatrix, September 2022; Capella, April 2023)

Bellatrix ([EIP-3675](https://eips.ethereum.org/EIPS/eip-3675), the Merge)
introduced no change to the set of topics, methods or configuration. The
`beacon_block` payload type changed to the Bellatrix `SignedBeaconBlock`
carrying an execution payload, blocks with invalid execution payloads are
allowed to propagate over gossip and Req/Resp, and a Bellatrix row was added to
the `ForkDigest`-context tables. See the
[rationale](rationale.md#gossipsub) for the message size discussion
around Bellatrix.

Capella ([EIP-4895](https://eips.ethereum.org/EIPS/eip-4895), withdrawals)
adds:

- The global topic `bls_to_execution_change` carrying
  `SignedBLSToExecutionChange` messages.
- The `beacon_block` payload type changes to the Capella `SignedBeaconBlock`; a
  Capella row is added to the `ForkDigest`-context tables of
  `BeaconBlocksByRange` v2 and `BeaconBlocksByRoot` v2 and to the light client
  type tables.
- The version 0 methods deprecated in version 1 (`/1/` variants of
  `BeaconBlocksByRange`, `BeaconBlocksByRoot` and `GetMetaData`) are removed
  from this specification.

### Version 1 (Altair, October 2021)

Altair adds new messages, topics and data to the Req/Resp, gossip and discovery
domains:

- The gossipsub `message-id` now covers the topic string in addition to the
  message data.
- New global topic `sync_committee_contribution_and_proof` and new subnet topics
  `sync_committee_{subnet_id}` carry the sync committee protocol. The
  `beacon_block` payload type changes to the Altair `SignedBeaconBlock`.
- Req/Resp response chunks gain a `<context-bytes>` field. `BeaconBlocksByRange`
  and `BeaconBlocksByRoot` move to schema version 2 with a `ForkDigest`-context
  selecting the block type per chunk; `GetMetaData` moves to version 2 to carry
  the extended `MetaData`. The `/1/` variants are deprecated.
- `MetaData` and the ENR gain a `syncnets` bitfield.
- Optional light client networking: topics `light_client_finality_update` and
  `light_client_optimistic_update`, and the `GetLightClientBootstrap`,
  `LightClientUpdatesByRange`, `GetLightClientFinalityUpdate` and
  `GetLightClientOptimisticUpdate` methods.
