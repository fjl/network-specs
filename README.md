<p align="center"><img src="img/etherdog.png"></p>

This repository contains specifications for the peer-to-peer networking protocols used by
Ethereum. The issue tracker here is for discussions of protocol changes. It's also OK to
open an issue if you just have a question.

Protocol level security issues are valuable! Please report serious issues responsibly
through the [Ethereum Foundation Bounty Program](https://bounty.ethereum.org).

## Specifications

There are multiple layers of specification, from the basic transport and discovery up to
the application-level protocols for consensus and execution.

### Transport

- [RLPx](./transport/rlpx.md) -- the legacy transport of the execution layer.

### Discovery

- [Ethereum Node Records](./discovery/enr.md) -- the node descriptor format.
- [Discovery v5](./discovery/discv5/discv5.md)
- [Discovery v4](./discovery/discv4/discv4.md) -- the legacy discovery protocol of the execution layer.
- [DNS node lists](./discovery/dnsdisc.md)

### Execution Layer

- [eth Protocol](./execution/eth.md)
- [snap Protocol](./execution/snap.md)

### Consensus Layer

- [Beacon Chain Networking](./consensus/beacon.md)

## Specification File Versioning

The links above point to the most recent version of each spec. As a general rule, the
'current' version of the spec is available in a Markdown file with just the name of the
spec document.

Previous versions of each spec are stored as versioned files alongside the main spec
document. For example, the most recent version of the execution-layer `eth` protocol is in
`execution/eth.md`, and prior versions are in `execution/eth-63.md`,
`execution/eth-64.md`, etc.

Future versions of each spec may also exist in the repository:

- If a spec update is proposed via an EIP, the future spec file has `-eipXXXX` appended to
  its basename.
- For working drafts which do not have an associated EIP, or protocols which are not
  tracked by the EIP process, the future spec file has `-draftYY-TITLE` appended to its
  basename, where `YY` is the future spec version number and `TITLE` is an arbitrary
  identifier for the proposal.

## Specification Metadata

Specifications have metadata attributes defined in Markdown front matter. The following
attributes are commonly used:

| Attribute | Values                             | Usage                                           |
|-----------|------------------------------------|-------------------------------------------------|
| `status`  | `legacy`, `stable`, `experimental` | reflects the stability/current-ness of the spec |
| `fork`    | Ethereum mainnet fork name         | for specs that require a blockchain fork        |

