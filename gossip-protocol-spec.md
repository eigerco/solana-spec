# Gossip protocol

This document describes the gossip protocol without implementation details which can be found [here](/implementation-details.md).

Solana nodes communicate with each other and share data using the gossip protocol. They send messages in a binary form which need to be deserialized. There are 6 types of messages:
* pull request
* pull response
* push message
* prune message
* ping
* pong

Each message contains data specific to its type: values that nodes share between them, filters, pruned nodes, etc. Nodes keep their data in _Cluster Replicated Data Store_ (`crds`) which is synchronized between them via pull requests, push messages and pull responses.

### Type definitions
Fields described in the tables below have their types specified using Rust notation:
* `u8` - 1 byte of unsigned data (8-bit unsigned integer)
* `u16` - 16-bit unsigned integer
* `u32` - 32-bit unsigned integer, and so on...
* `[u8]` - dynamic size array of 1 byte elements
* `[u8, 32]` - static size array of 32 elements, with each element being 1 byte
* `[[u8, 64]]` - a two dimensional array containing an arrays of 64 1 byte elements 
* `(u64, [u8, 64])` - a tuple of two elements, one is 64-bit unsigned integer, the second one is a 64 1-byte elements array
* `u32 | None` - an optional type, can be either 32-bit unsigned integer or `None`
* `enum` - an enumeration type - it is a Rust `enum` which in other languages (C for example) can be implemented as an `union`
* `struct` - a complex type, consisting of many elements of different types

The **Size** column in tables contains the size of data in bytes. Size of dynamic arrays contains additional _plus_ (`+`) sign, e.g. `32+` which means the array has at least 32 bytes. Empty dynamic arrays always have 8 bytes which is the size of array header containing array length. 
In case size of a particular complex data is unknown it is marked with `?`. The limit however is always 1232 bytes for the whole data packet.

## Message format

Each message is sent in a binary form with a maximum size of 1232 bytes (1280 is a minimum `IPv6 TPU`, 40 bytes is the size of `IPv6` header and 8 bytes is the size of the fragment header). 


Data sent in the message is serialized from a `Protocol` type, which can be one of:

| Message                        | Data                      | Description |
|--------------------------------|---------------------------|------------|
| [pull request](#pullrequest)   | `CrdsFilter`, `CrdsValue` | sent by node to ask for new information |
| [pull response](#pullresponse) | `Pubkey`, `[CrdsValue]`   | response to a pull request |
| [push message](#pushmessage)   | `Pubkey`, `CrdsValue`     | sent by node to share its data |
| [prune message](#prunemessage) | `Pubkey`, `PruneData`     | sent to peers with a list of nodes which should be pruned |
| [ping message](#pingmessage)   | `Ping`                    | a ping message |
| [pong message](#pongmessage)   | `Pong`                    | response to a ping |

```mermaid
block-beta
    block
        columns 3
        block
            columns 1
            b["Pull request"]
            c["Pull response"]
            d["Push message"]
            e["Prune message"]
            f["Ping message"]
            g["Pong message"]
        end
        right<["Serialize"]>(right)
        a["Packet\n(1232 bytes)"]
    end
```

<details>
  <summary>Solana client Rust implementation</summary>

``` rust
enum Protocol
{
    PullRequest(CrdsFilter, CrdsValue),
    PullResponse(Pubkey, [CrdsValue]),
    PushMessage(Pubkey, [CrdsValue]),
    PruneMessage(Pubkey, PruneData),
    PingMessage(Ping),
    PongMessage(Pong)
}
```

</details>

### PullRequest

| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `CrdsFilter` | `struct` | 37+ | a bloom filter representing things node already has |
| `CrdsValue` | `struct` | ? | a [value](#data-shared-between-nodes), usually a `LegacyContactInfo` of the node who send the pull request containing nodes socket addresses for different protocols (gossip, tvu, tpu, rpc, etc.) |

It is sent by node to ask the cluster for new information. It contains a bloom filter with things node already has. Nodes receiving pull requests gather all new values from their `crds`, filter them using provided filters and send `PullResponse` to the origin of the request.

<details>
  <summary>Solana client Rust implementation</summary>

``` rust
struct CrdsFilter {
    filter: Bloom,
    mask: u64,
    mask_bits: u32,
}

struct Bloom {
    keys: Vec<u64>,
    bits: BitVec<u64>,
    num_bits_set: u64,
}
```

</details>

### PullResponse

| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `Pubkey` | `[u8, 32]` | 32 | a public key of the origin |
| `[CrdsValue]` | `[struct]` | 8+ | a list of new values  |

These are sent in response to a `PullRequest`.


### PushMessage

| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `Pubkey` | `[u8, 32]` | 32 | a public key of the origin |
| `[CrdsValue]` | `[struct]` | 8+ | a list of values to share  |

It is sent by nodes who want to share information with others. Node receiving the message checks for:
- duplication - duplicated messages are dropped, node responses with prune message if message came from low staked node
- new data:
    - new information is stored in `crds` and replaces old value
    - stores message in `pushed_once` which is used for detecting duplicates
    - retransmits information to its peers
- expiration - messages older than `PUSH_MSG_TIMEOUT` are dropped

### PruneMessage
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `Pubkey` | `[u8, 32]` | 32 | a public key of the origin |
| `PruneData` | `struct` | 144+ | a structure which contains |

It is sent to peers with a list of nodes that should be pruned.

#### PruneData
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `pubkey` |`[u8, 32]` | 32 | public key of the origin |
| `prunes` | `[[u8, 32]]` | 8+ | public keys of nodes that should be pruned |
| `signature` | `[u8, 64]` | 64 | signature of this message |
| `destination` | `[u8, 32]` | 32 | a public key of the destination node of this message |
| `wallclock` | `u64` | 8 | wallclock of the node that generated that message |

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct PruneData {
    pubkey: Pubkey,
    prunes: Vec<Pubkey,
    signature: Signature,
    destination: Pubkey,
    wallclock: u64,
}
```

</details>


### PingMessage
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `from` |`[u8, 32]` | 32 | public key of the origin |
| `token` |`[u8, 32]` | 32 | 32 bytes token |
| `signature` |`[u8, 64]` | 64 | signature of the message |

Nodes send ping messages frequently to their peers to check whether they are active. 

<details>
  <summary>Solana client Rust implementation</summary>

```rust
Ping {
    from: Pubkey,
    token: [u8, 32],
    signature: Signature,
}
```

</details>


### PongMessage
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `from` |`[u8, 32]` | 32 | public key of the origin |
| `hash` |`[u8, 32]` | 32 | hash of the received ping token |
| `signature` |`[u8, 64]` | 64 | signature o f the message |

Sent by node as a response to `PingMessage`.

<details>
  <summary>Solana client Rust implementation</summary>

```rust
Pong {
    from: Pubkey,
    hash: Hash,
    signature: Signature,
}
```

</details>

## Data shared between nodes

The `CrdsValue` values that are sent in push messages, pull requests & pull responses contain signature and the actual shared data:

| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `signature` | `[u8, 64]` | 64 | signature of origin |
| `data` | `enum` | ? | data  |

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct CrdsValue {
    signature: Signature,
    data: CrdsData,
}

enum CrdsData {
    LegacyContactInfo(LegacyContactInfo),
    Vote(VoteIndex, Vote),
    LowestSlot(LowestSlotIndex, LowestSlot),
    EpochSlots(EpochSlotsIndex, EpochSlots),
    LegacyVersion(LegacyVersion),
    Version(Version),
    NodeInstance(NodeInstance),
    DuplicateShred(DuplicateShredIndex, DuplicateShred),
    SnapshotHashes(SnapshotHashes),
    ContactInfo(ContactInfo),
    RestartLastVotedForkSlots(RestartLastVotedForkSlots),
    RestartHeaviestFork(RestartHeaviestFork),
}
```
</details>

### CrdsData
The `CrdsValue` data (`CrdsData`) can be one of:
* [LegacyContactInfo](#legacycontactinfo)
* [Vote](#vote)
* [LowestSlot](#lowestslot)
* [LegacySnapshotHashes](#legacysnapshothashes-accountshashes) (_deprecated_)
* [AccountsHashes](#legacysnapshothashes-accountshashes) (_deprecated_)
* [EpochSlots](#epochslots)
* [LegacyVersion](#legacyversion)
* [Version](#version)
* [NodeInstance](#nodeinstance)
* [DuplicateShred](#duplicateshred)
* [SnapshotHashes](#snapshothashes)
* [ContactInfo](#contactinfo)
* [RestartLastVotedForkSlots](#restartlastvotedforkslots)
* [RestartHeaviestFork](#restartheaviestfork)

<details>
  <summary>Solana client Rust implementation</summary>

```rust
enum CrdsData
{
    LegacyContactInfo(LegacyContactInfo),
    Vote(VoteIndex, Vote),
    LowestSlot(LowestSlotIndex, LowestSlot),
    LegacySnapshotHashes(LegacySnapshotHashes),
    AccountsHashes(AccountsHashes),
    EpochSlots(EpochSlotsIndex, EpochSlots),
    LegacyVersion(LegacyVersion),
    Version(Version),
    NodeInstance(NodeInstance),
    DuplicateShred(DuplicateShredIndex, DuplicateShred),
    SnapshotHashes(SnapshotHashes),
    ContactInfo(ContactInfo),
    RestartLastVotedForkSlots(RestartLastVotedForkSlots),
    RestartHeaviestFork(RestartHeaviestFork),
}
```
</details>

#### LegacyContactInfo

| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `id` | `[u8, 32]` | 32 | public key of the origin |
| `gossip` | `enum` | 10 or 22 * |  gossip protocol address |
| `tvu` | `enum` | 10 or 22 | address to connect to for replication |
| `tvu_quic` | `enum` | 10 or 22 | TVU over QUIC protocol |
| `serve_repair_quic` | `enum` | 10 or 22 | repair service for QUIC protocol |
| `tpu` | `enum` | 10 or 22 | transactions address |
| `tpu_forwards` | `enum` | 10 or 22 | address to forward unprocessed transactions |
| `tpu_vote` | `enum` | 10 or 22 | address for sending votes |
| `rpc` | `enum` | 10 or 22 | address for JSON-RPC requests |
| `rpc_pubsub` | `enum` | 10 or 22 | websocket for JSON-RPC push notifications |
| `serve_repair` | `enum` | 10 or 22 | address for sending repair requests |
| `wallclock` | `u64` | 8 | wallclock of the node that generated that message |
| `shred_version` | `u16` | 2 | the shred version node has been configured to use |

\* _size of the address field is 10 bytes for IPv4 address and 22 bytes for IPv6 address_

Basic info about the node. Nodes send this message to introduce themselves and provide all addresses and ports that can be used by their peers to communicate with them. 

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct LegacyContactInfo {
    id: Pubkey,
    gossip: SocketAddr,
    tvu: SocketAddr,
    tvu_quic: SocketAddr,
    serve_repair_quic: SocketAddr,
    tpu: SocketAddr,
    tpu_forwards: SocketAddr,
    tpu_vote: SocketAddr,
    rpc: SocketAddr,
    rpc_pubsub: SocketAddr,
    serve_repair: SocketAddr,
    wallclock: u64,
    shred_version: u16,
}

enum SocketAddr {
    V4(SocketAddrV4),
    V6(SocketAddrV6)
}

struct SocketAddrV4 {
    ip: Ipv4Addr,
    port: u16,
}

struct SocketAddrV6 {
    ip: Ipv6Addr,
    port: u16,
    flowinfo: u32,
    scope_id: u32
}

struct Ipv4Addr {
    octets: [u8; 4]
}

struct Ipv6Addr {
    octets: [u8; 16]
}
```

</details>

#### Vote

| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `index` | `u8` | 1 | vote tower index | 
| `from` | `[u8, 32]` | 32 | public key of the origin |
| [`transaction`](#transaction)  | `struct` | 59+ | a vote transaction, an atomically-committed sequence of instructions |
| `wallclock`  | `u64` | 8 |  wallclock of the node that generated that message |
| `slot`  | `u64` | 8 |  slot in which the vote was created |

It's a validators vote on a fork. Contains a one byte index from vote tower (range 0 to 31) and vote transaction to execute by the leader.

##### Transaction
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `signature` | `[[u8, 64]]` | 8+ | list of signatures equal to `num_required_signatures` for the message |
| `message` | `struct` | 51+ | transaction message containing instructions to invoke |


A vote transaction, contains a signature and a message with sequence of instructions:
 * `signature` - list of signatures equal to `num_required_signatures` for the message
 * `message` - transaction message containing instructions to invoke:

 ##### Message

| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| [`header`](#message-header) | `struct` | 3 | message header |
| `account_keys` | `[[u8, 32]]` | 32+ | all account keys used by this transaction |
| `recent_blockhash` | `[u8, 32]` | 32 |hash of a recent ledger entry |
| [`instructions`](#compiled-instruction) | `[struct]` | 17+ | list of compiled instructions to execute |

##### Message header

| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `num_required_signatures` | `u8` | 1 | number of signatures required for this message to be considered valid |
| `num_readonly_signed_accounts` | `u8` | 1 | last `num_readonly_signed_accounts` of the signed keys are read-only accounts |
| `num_readonly_unsigned_accounts` | `u8` | 1 | last `num_readonly_unsigned_accounts` of the unsigned keys are read-only accounts |

##### Compiled instruction

| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `program_id_index` | `u8` | 1 | index of the transaction keys array indicating the program account ID that executes the program |
| `accounts` |`[u8]` | 8+ | indices of the transaction keys array indicating the accounts that are passed to a program |
| `data` | `[u8]` | 8+ | program input data |

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct Vote {
    from: Pubkey,
    transaction: Transaction,
    wallclock: u64,
    slot: Option<Slot>,
}

type Slot = u64

struct Transaction {
    signature: Vec<Signature>,
    message: Message
}

struct Message {
    header: MessageHeader,
    account_keys: Vec<Pubkey>,
    recent_blockhash: Hash,
    instructions: Vec<CompiledInstruction>,
}

struct MessageHeader {
    num_required_signatures: u8,
    num_readonly_signed_accounts: u8,
    num_readonly_unsigned_accounts: u8,
}

struct CompiledInstruction {
    program_id_index: u8,
    accounts: Vec<u8>,
    data: Vec<u8>,
}
```
</details>

#### LowestSlot

| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `index` | `u8` | 1 | _deprecated_ | 
| `from` | `[u8, 32]`| 32 | public key of the origin |
| `root` | `u64` | 8 | _deprecated_ |
| `lowest` | `u64` | 8 | the actual slot |
| `slots` | `[u64]` | 8+ | _deprecated_ |
| `stash` | `[struct]` | 8+ | _deprecated_ |
| `wallclock` | `u64` | 8 | wallclock of the node that generated that message |

It is the first available slot in Solana [blockstore](https://docs.solanalabs.com/validator/blockstore) that contains any data. Contains a one byte index (deprecated) and the lowest slot number.

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct LowestSlot {
    from: Pubkey,
    root: Slot,
    lowest: Slot,
    slots: BTreeSet<Slot>,
    stash: Vec<EpochIncompleteSlots>,
    wallclock: u64,
}

struct EpochIncompleteSlots {
    first: Slot,
    compression: CompressionType,
    compressed_list: Vec<u8>,
}

enum CompressionType {
    Uncompressed,
    GZip,
    BZip2,
}
```
</details>

#### LegacySnapshotHashes, AccountsHashes
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `from` | `[u8, 32]`| 32 | public key of the origin |
| `hashes` | `[(u64, [u8, 32])]`| 8+ | a list of hashes grouped by slots |
| `wallclock` | `u64`| 8 | wallclock of the node that generated that message |

_Deprecated_

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct AccountsHashes {
    from: Pubkey,
    hashes: Vec<(Slot, Hash)>,
    wallclock: u64,
}

type LegacySnapshotHashes = AccountsHashes;
```
</details>

#### EpochSlots
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `index` | `u8` | 1 | index | 
| `from` | `[u8, 32]`| 32 | public key of origin | 
| `slots` | `[enum]`| 8+ | list of epoch slots - either uncompressed or compressed with a `Flate2` algorithm |
| `wallclock` | `u64`| 64 | wallclock of the node that generated that message |

Contains a one byte index and a `EpochSlots` structure which holds the list of all slots from an epoch (epoch consists around 432000 slots). There can be 256 epoch slots in total.


##### Flate2
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `first_slot` | `u64` | 8 | first slot number |
| `num` | `u32` or `u64` | 4 or 8* | number of slots |
| `compressed` | `[u8]` | 8+ | bytes array of compressed slots |

_the `num` field size depends on architecture (32bit or 64bit)_


##### Uncompressed
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `first_slot` | `u64` | 8 | first slot number |
| `num` | `u32` or `u64` | 4 or 8 | number of slots |
| `slots` | `[u8]` | 9+ | bits array of slots |


<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct EpochSlots {
    from: Pubkey,
    slots: [CompressedSlots],
    wallclock: u64,
}

enum CompressedSlots {
   Flate2(Flate2),
   Uncompressed(Uncompressed),
}

struct Flate2 {
    first_slot: Slot,
    num: usize,
    compressed: Vec<u8>
}

struct Uncompressed {
    first_slot: Slot,
    num: usize,
    slots: BitVec<u8>,
}
```

</details>

#### LegacyVersion
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `from` | `[u8, 32]`| 32 | public key of origin |
| `wallclock` | `u64`| 8 | wallclock of the node that generated that message |
| `version` | `struct`| 7+ | older version of the Solana used in 1.3.x and earlier releases |

The older version of Solana client the node is using.

##### LegacyVersion 1
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `major` | `u16`| 2 | major part of version |
| `minor` | `u16`| 2 | minor part of version |
| `patch` | `u16`| 2 | patch |
| `commit` | `u32 \| None`| 1+ | commit |

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct LegacyVersion {
    from: Pubkey,
    wallclock: u64,
    version: LegacyVersion1,
}

struct LegacyVersion1 {
    major: u16,
    minor: u16,
    patch: u16,
    commit: Option<u32>
}
```
</details>

#### Version
| Data | Type | Size | Description |
|------|:----:|:-----:|-------------|
| `from` | `[u8, 32]` | 32 | public key of origin |
| `wallclock` | `u64` | 8 | wallclock of the node that generated that message |
| `version` | `LegacyVersion2` | 11+ | [version](#legacyversion-2) of the Solana |

Version of Solana client the node is using.

##### LegacyVersion 2
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `major` | `u16`| 2 | major part of version |
| `minor` | `u16`| 2 | minor part of version |
| `patch` | `u16`| 2 | patch |
| `commit` | `u32 \| None`| 1+ | commit |
| `feature_set` | `u32`| 4 | feature set |

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct Version {
    from: Pubkey,
    wallclock: u64,
    version: LegacyVersion2,
}

struct LegacyVersion2 {
    major: u16,
    minor: u16,
    patch: u16,
    commit: Option<u32>,
    feature_set: u32
}
```
</details>

#### NodeInstance
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `from` | `[u8, 32]`| 32 | public key of origin |
| `wallclock` | `u64`| 8 | wallclock of the node that generated that message |
| `timestamp` | `u64`| 8 | when the instance was created |
| `token` | `u64`| 8 | randomly generated value at node instantiation |

Contains node creation timestamp and randomly generated token.

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct NodeInstance {
    from: Pubkey,
    wallclock: u64,
    timestamp: u64,
    token: u64,
}
```
</details>

#### DuplicateShred
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `index` | `u16` | 2 | index |
| `from` | `[u8, 32]`| 32 | public key of origin |
| `wallclock` | `u64`| 8 | wallclock of the node that generated that message |
| `slot` | `u64`| 8 | slot when shreds where created |
| `_unused` | `u32`| 32 | _unused_ |
| `_unused_shred_type` | `enum`| 1 | _unused_ |
| `num_chunks` | `u8`| 1 | number of chunks available |
| `chunk_index` | `u8`| 1 | index of the chunk |
| `chunk` | `[u8]`| 8+ | shred data |

A duplicated shred proof. Contains a 2 byte index and `DuplicateShred` structure.

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct DuplicateShred {
    from: Pubkey,
    wallclock: u64,
    slot: Slot,
    _unused: u32,
    _unused_shred_type: ShredType,
    num_chunks: u8,
    chunk_index: u8,
    chunk: Vec<u8>,
}

#[serde(into = "u8", try_from = "u8")]
enum ShredType {
    Data = 0b1010_0101,
    Code = 0b0101_1010,
}
```
</details>

#### SnapshotHashes
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `from` | `[u8, 32]`| 32 | public key of origin |
| `full` | `(u64, [u8, 32])`| 40 | hash and slot number of the full snapshot |
| `incremental` | `[u64, [u8, 32]]`| 8+ | list of hashes and slot numbers of incremental snapshots |
| `wallclock` | `u64`| 8 | wallclock of the node that generated that message |

Contains hashes of full and incremental snapshots.

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct SnapshotHashes {
    from: Pubkey,
    full: (Slot, Hash),
    incremental: Vec<(Slot, Hash)>,
    wallclock: u64,
}
```
</details>

#### ContactInfo
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `pubkey` | `[u8, 32]`| 32 | public key of origin |
| `wallclock` | `u64`| 8 | wallclock of the node that generated that message |
| `outset` | `u64`| 8 | timestamp when node instance was first created |
| `shred_version` | `u16`| 2 | the shred version node has been configured to use |
| `version` | `Version`| 13+ | Solana [version](#version-1) |
| `addrs` | `[IpAddr]`| 8+ | list of unique IP addresses |
| `sockets` | `[SocketEntry]`| 8+ | list of unique [sockets](#socketentry)  |
| `extensions` | `[Extension]`| 8+ | future additions to ContactInfo will be added to Extensions instead of modifying ContactInfo, currently unused |
| `cache` | `[SocketAddr, 12]`| 120+ | cache of nodes socket addresses |

Contact info of the node.

##### Version
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `major` | `u16`| 2 | major part of version |
| `minor` | `u16`| 2 | minor part of version |
| `patch` | `u16`| 2 | patch |
| `commit` | `u32 \| None`| 1+ | commit |
| `feature_set` | `u32`| 4 | feature set |
| `client` | `u16`| 2 | client |
##### SocketEntry
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `key` | `u8`| 1 | protocol identifier (tvu, tpu, etc.) |
| `index` | `u8`| 1 | IpAddr index in the addrs list |
| `offset` | `u16`| 2 | port offset in respect to previous entry |

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct ContactInfo {
    pubkey: Pubkey,
    wallclock: u64,
    outset: u64,
    shred_version: u16,
    version: Version,
    addrs: Vec<IpAddr>,
    sockets: Vec<SocketEntry>,
    extensions: Vec<Extension>,
    cache: [SocketAddr; 12],
}

enum Extension {}

enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv4Addr)
}

struct SocketEntry {
    key: u8,
    index: u8,
    offset: u16
}

struct Version {
    major: u16,
    minor: u16,
    patch: u16,
    commit: Option<u32>,
    feature_set: u32,
    client: u16
}
```
</details>

#### RestartLastVotedForkSlots
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `from` | `[u8, 32]`| 32 | public key of origin |
| `wallclock` | `u64`| 8 | timestamp of data creation |
| `offsets` | `enum`| 12+ | list of slots |
| `last_voted_slot` | `u64`| 8 | last voted slot |
| `last_voted_hash` | `[u8, 32]`| 32 | |
| `shred_version` | `u16`| 2 | the shred version node has been configured to use |

Contains a list of last voted fork slots.

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct RestartLastVotedForkSlots {
    from: Pubkey,
    wallclock: u64,
    offsets: SlotsOffsets,
    last_voted_slot: Slot,
    last_voted_hash: Hash,
    shred_version: u16,
}

enum SlotsOffsets {
    RunLengthEncoding(RunLengthEncoding),
    RawOffsets(RawOffsets),
}

struct RunLengthEncoding(Vec<u16>);
struct RawOffsets(BitVec<u8>);
```
</details>


#### RestartHeaviestFork
| Data | Type | Size | Description |
|------|:----:|:----:|-------------|
| `from` | `[u8, 32]`| 32 | public key of origin |
| `wallclock` | `u64`| 8 | timestamp of data creation |
| `last_slot` | `u64`| 8 | last slot of the fork |
| `last_hash` | `[u8, 32]`| 32 | hash of the last slot |
| `observed_stake` | `u64`| 8 | |
| `shred_version` | `u16`| 2 | the shred version node has been configured to use |

Contains heaviest fork.

<details>
  <summary>Solana client Rust implementation</summary>

```rust
struct RestartHeaviestFork {
    from: Pubkey,
    wallclock: u64,
    last_slot: Slot,
    last_slot_hash: Hash,
    observed_stake: u64,
    shred_version: u16,
}
```
</details>
