# Phase 0 -- Networking

This document contains the networking specification for Bundler software.

It consists of four main sections:

1. A specification of the network fundamentals.
2. A specification of the three network interaction *domains* of the bundler: (a) the gossip domain, (b) the discovery domain, and (c) the Req/Resp domain.
3. The rationale and further explanation for the design choices made in the previous two sections.
4. An analysis of the maturity/state of the libp2p features required by this spec across the languages in which bundler software are being developed.

## Table of contents
<!-- TOC -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Network fundamentals](#network-fundamentals)
  - [Transport](#transport)
  - [Encryption and identification](#encryption-and-identification)
  - [Protocol Negotiation](#protocol-negotiation)
  - [Multiplexing](#multiplexing)
- [Bundler interaction domains](#bundler-interaction-domains)
  - [Configuration](#configuration)
  - [MetaData](#metadata)
  - [The gossip domain: gossipsub](#the-gossip-domain-gossipsub)
    - [Topics and messages](#topics-and-messages)
      - [Global topics](#global-topics)
        - [`topic_1`](#topic_1)
        - [`topic_2`](#topic_2)
    - [Encodings](#encodings)
  - [The Req/Resp domain](#the-reqresp-domain)
    - [Protocol identification](#protocol-identification)
    - [Req/Resp interaction](#reqresp-interaction)
      - [Requesting side](#requesting-side)
      - [Responding side](#responding-side)
    - [Encoding strategies](#encoding-strategies)
      - [SSZ-snappy encoding strategy](#ssz-snappy-encoding-strategy)
    - [Messages](#messages)
      - [Status](#status)
      - [Goodbye](#goodbye)
      - [Ping](#ping)
      - [GetMetaData](#getmetadata)
- [Design decision rationale](#design-decision-rationale)
- [libp2p implementations matrix](#libp2p-implementations-matrix)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
<!-- /TOC -->

# Network fundamentals

This section outlines the specification for the networking stack in ERC4337 bundlers.

## Transport

Even though libp2p is a multi-transport stack (designed to listen on multiple simultaneous transports and endpoints transparently),
we hereby define a profile for basic interoperability.

All implementations MUST support the TCP libp2p transport, and it MUST be enabled for both dialing and listening (i.e. outbound and inbound connections).
The libp2p TCP transport supports listening on IPv4 and IPv6 addresses (and on multiple simultaneously).

Bundlers must support listening on at least one of IPv4 or IPv6.
Bundlers that do _not_ have support for listening on IPv4 SHOULD be cognizant of the potential disadvantages in terms of
Internet-wide routability/support. Bundlers MAY choose to listen only on IPv6, but MUST be capable of dialing both IPv4 and IPv6 addresses.

All listening endpoints must be publicly dialable, and thus not rely on libp2p circuit relay, AutoNAT, or AutoRelay facilities.
(Usage of circuit relay, AutoNAT, or AutoRelay will be specifically re-examined soon.)

Nodes operating behind a NAT, or otherwise undialable by default (e.g. container runtime, firewall, etc.),
MUST have their infrastructure configured to enable inbound traffic on the announced public listening endpoint.

## Encryption and identification

The [Libp2p-noise](https://github.com/libp2p/specs/tree/master/noise) secure
channel handshake with `secp256k1` identities will be used for encryption.

As specified in the libp2p specification, Bundlers MUST support the `XX` handshake pattern.

## Protocol Negotiation

Bundlers MUST use exact equality when negotiating protocol versions to use and MAY use the version to give priority to higher version numbers.

Bundlers MUST support [multistream-select 1.0](https://github.com/multiformats/multistream-select/)
and MAY support [multiselect 2.0](https://github.com/libp2p/specs/pull/95) when the spec solidifies.
Once all Bundlers have implementations for multiselect 2.0, multistream-select 1.0 MAY be phased out.

## Multiplexing



# Bundler interaction domains

## Configuration

This section outlines constants that are used in this spec.

| Name | Value | Description |
|---|---|---|


## MetaData

Bundlers MUST locally store the following `MetaData`:

```
(
  
)
```

Where

- .
- .


## The gossip domain: gossipsub

Bundlers MUST support the [gossipsub v1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md) libp2p Protocol
including the [gossipsub v1.1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md) extension.

**Protocol ID:** `/meshsub/1.1.0`

**Gossipsub Parameters**

The following gossipsub [parameters](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md#parameters) will be used:

- `D` (topic stable mesh target count): 8
- `D_low` (topic stable mesh low watermark): 6
- `D_high` (topic stable mesh high watermark): 12
- `D_lazy` (gossip target): 6
- `heartbeat_interval` (frequency of heartbeat, seconds): 0.7
- `fanout_ttl` (ttl for fanout maps for topics we are not subscribed to but have published to, seconds): 60
- `mcache_len` (number of windows to retain full messages in cache for `IWANT` responses): 6
- `mcache_gossip` (number of windows to gossip about): 3
- `seen_ttl` (number of heartbeat intervals to retain message IDs): 550

*Note*: Gossipsub v1.1 introduces a number of
[additional parameters](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#overview-of-new-parameters)
for peer scoring and other attack mitigations.
These are currently under investigation and will be spec'd and released to mainnet when they are ready.

### Topics and messages

Topics are plain UTF-8 strings and are encoded on the wire as determined by protobuf (gossipsub messages are enveloped in protobuf messages).
Topic strings have form: `/eth2/ForkDigestValue/Name/Encoding`.
This defines both the type of data being sent on the topic and how the data field of the message is encoded.

- `ForkDigestValue` - the lowercase hex-encoded (no "0x" prefix) bytes of `compute_fork_digest(current_fork_version, genesis_validators_root)` where
    - `current_fork_version` is the fork version of the epoch of the message to be sent on the topic
    - `genesis_validators_root` is the static `Root` found in `state.genesis_validators_root`
- `Name` - see table below
- `Encoding` - the encoding strategy describes a specific representation of bytes that will be transmitted over the wire.
  See the [Encodings](#Encodings) section for further details.

*Note*: `ForkDigestValue` is composed of values that are not known until the genesis block/state are available.
Due to this, Bundlers SHOULD NOT subscribe to gossipsub topics until these genesis values are known.

Each gossipsub [message](https://github.com/libp2p/go-libp2p-pubsub/blob/master/pb/rpc.proto#L17-L24) has a maximum size of `GOSSIP_MAX_SIZE`.
Bundlers MUST reject (fail validation) messages that are over this size limit.
Likewise, Bundlers MUST NOT emit or propagate messages larger than this limit.

The optional `from` (1), `seqno` (3), `signature` (5) and `key` (6) protobuf fields are omitted from the message,
since messages are identified by content, anonymous, and signed where necessary in the application layer.
Starting from Gossipsub v1.1, Bundlers MUST enforce this by applying the `StrictNoSign`
[signature policy](https://github.com/libp2p/specs/blob/master/pubsub/README.md#signature-policy-options).

The `message-id` of a gossipsub message MUST be the following 20 byte value computed from the message data:
* If `message.data` has a valid snappy decompression, set `message-id` to the first 20 bytes of the `SHA256` hash of
  the concatenation of `MESSAGE_DOMAIN_VALID_SNAPPY` with the snappy decompressed message data,
  i.e. `SHA256(MESSAGE_DOMAIN_VALID_SNAPPY + snappy_decompress(message.data))[:20]`.
* Otherwise, set `message-id` to the first 20 bytes of the `SHA256` hash of
  the concatenation of `MESSAGE_DOMAIN_INVALID_SNAPPY` with the raw message data,
  i.e. `SHA256(MESSAGE_DOMAIN_INVALID_SNAPPY + message.data)[:20]`.

*Note*: The above logic handles two exceptional cases:
(1) multiple snappy `data` can decompress to the same value,
and (2) some message `data` can fail to snappy decompress altogether.

The payload is carried in the `data` field of a gossipsub message, and varies depending on the topic:

| Name                             | Message Type              |
|----------------------------------|---------------------------|
| `topic_1`                        | `SignedTopic1`            |
| `topic_2`                        | `SignedTopic2`            |


Bundlers MUST reject (fail validation) messages containing an incorrect type, or invalid payload.

When processing incoming gossip, Bundlers MAY descore or disconnect peers who fail to observe these constraints.

For any optional queueing, Bundlers SHOULD maintain maximum queue sizes to avoid DoS vectors.

Gossipsub v1.1 introduces [Extended Validators](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#extended-validators)
for the application to aid in the gossipsub peer-scoring scheme.
We utilize `ACCEPT`, `REJECT`, and `IGNORE`. For each gossipsub topic, there are application specific validations.
If all validations pass, return `ACCEPT`.
If one or more validations fail while processing the items in order, return either `REJECT` or `IGNORE` as specified in the prefix of the particular condition.

#### Global topics

There are two primary global topics used to propagate beacon blocks (`topic_1`)
and aggregate attestations (`topic_2`) to all nodes on the network.


##### `NewPooledUserOps`

The `NewPooledUserOps` topic is used solely for propagating to all the connected nodes on the networks. One or more UserOps that have appeared in the network and which have not yet been included in a block are propagated to a fraction of the nodes connected to the network.


##### `GetPooledUserOps`

The `GetPooledUserOps` 

##### `PooledUserOps`

The `PooledUserOps` 

