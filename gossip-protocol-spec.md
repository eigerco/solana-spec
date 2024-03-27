# Gossip protocol

This document describes the gossip protocol without implementation details which can be found [here](/implementation-details.md).

Solana nodes communicate with each other and share data using the gossip protocol. They send messages in a binary form which nodes need to deserialize. There are 6 types of messages:
* pull request
* pull response
* push message
* prune message
* ping
* pong

Each message contains data specific to its type: values that nodes share between them, filters, pruned nodes, etc. Nodes keep their data in [_Cluster Replicated Data Store_ (`crds`)](/implementation-details.md#cluster-replicated-data-store) which is synchronized between them via pull requests, push messages and pull responses.

## Message format

Each message is sent in a binary form with a maximum size of 1232 bytes (1280 is a minimum `IPv6 TPU`, 40 bytes is the size of `IPv6` header and 8 bytes is the size of the fragment header). Apart from the actual data, packet contains additional metadata:

 * size - size of the packet
 * addr - address of the origin
 * port - port of the origin
 * flags - additional flags

```rust
Packet {
    buffer: [u8; 1232],
    meta: Meta,
}

Meta {
    size: usize,
    addr: IpAddr,
    port: u16,
    flags: PacketFlags,
}

PacketFlags {
    const DISCARD        = 0b0000_0001;
    const FORWARDED      = 0b0000_0010;
    const REPAIR         = 0b0000_0100;
    const SIMPLE_VOTE_TX = 0b0000_1000;
    const TRACER_PACKET  = 0b0001_0000;
    const ROUND_COMPUTE_UNIT_PRICE = 0b0010_0000;
}
```

Data sent in the message is serialized from a `Protocol` type:

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

### `PullRequest`

Is sent by node to ask the cluster for new information. It contains a bloom filter with things node already has. Nodes receiving pull requests gather all new values from their `crds`, filter them using provided filters and send `PullResponse` to the origin of the request.

`PullRequest` message contains two values:
  * `CrdsFilter` - a bloom filter representing things node already has
  * `CrdsValue` - a [value](#data-shared-between-nodes), usually a `LegacyContactInfo` of the node who send the pull request containing nodes socket addresses for different protocols (gossip, tvu, tpu, rpc, etc.)

### `PullResponse`

These are sent in response to `PullRequest` and contain:
  * `Pubkey` - a public key of the origin
  * `[CrdsValue]` - a list of new values 

### `PushMessage` 
It is sent by nodes who want to share information with others. Node receiving the message checks for:
- duplication - duplicated messages are dropped, node response with prune message if message came from low staked node
- new data:
    - new information is stored in `crds` and replaces old value
    - stores message in `pushed_once` which is used for detecting duplicates
    - retransmits information to its peers
- expiration - messages older than `PUSH_MSG_TIMEOUT` are dropped

The `PushMessage` contains:
  *  `Pubkey` - a public key of the origin
  * `[CrdsValue]` - list of values to share

### `PruneMessage`
It is sent to peers with a list of nodes that should be pruned. The message contains:
  * `Pubkey` - a public key of the origin
  * `PruneData` - a structure which contains:
    - pubkey - public key of the origin
    - prunes - public keys of nodes that should be pruned
    - signature - signature of this message
    - destination - a public key of the destination node of this message
    - wallclock - wallclock of the node that generated that message

```rust
PruneData {
    pubkey: Pubkey,
    prunes: [Pubkey],
    signature: Signature,
    destination: Pubkey,
    wallclock: u64,
}
```
##### `Pubkey`
```rust
Pubkey = [u8; 32];
```
##### `Signature`
```rust
Signature = [u8, 64];
```

### `PingMessage`
Nodes are sending ping messages from time to time to other nodes to check whether they are active. `PingMessage` contains a `Ping` structure which consists of:
- from - public key of the origin
- token - 32 bytes token
- signature - signature of the message

```rust
Ping {
    from: Pubkey,
    token: [u8, 32],
    signature: Signature,
}
```

### `PongMessage` 
Sent by node as a [response](#pong) to `PingMessage`. Contains a `Pong` structure which consists of:
- from - public key of the origin
- hash - hash of the received ping token
- signature - signature of the message

```rust
Pong {
    from: Pubkey,
    hash: Hash,
    signature: Signature,
}
```
##### `Hash`
```rust
Hash = [u8; 32]
```

## Data shared between nodes

Values that are sent in push messages, pull requests & pull responses contain signature and the actual shared data:
```rust
CrdsValue {
    signature: Signature,
    data: CrdsData,
}
```
### `CrdsData`
Defines a data type that can be sent between nodes. Can be one of:
```rust
enum CrdsData
{
    LegacyContactInfo(LegacyContactInfo),
    Vote(u8, Vote),
    LowestSlot(u8, LowestSlot),
    LegacySnapshotHashes(LegacySnapshotHashes),
    AccountsHashes(AccountsHashes),
    EpochSlots(u8, EpochSlots),
    LegacyVersion(LegacyVersion),
    Version(Version),
    NodeInstance(NodeInstance),
    DuplicateShred(u16, DuplicateShred),
    SnapshotHashes(SnapshotHashes),
    ContactInfo(ContactInfo),
    RestartLastVotedForkSlots(RestartLastVotedForkSlots),
    RestartHeaviestFork(RestartHeaviestFork),
}
```

#### `LegacyContactInfo`
Basic info about the node. Nodes send this message to introduce themselves and provide all addresses and ports that can be used by their peers to communicate with them:
- id - public key of origin
- gossip - gossip protocol address
- tvu - address to connect to for replication
- tvu quic - TVU over QUIC protocol
- serve repair quic - repair service for QUIC protocol
- tpu - transactions address
- tpu forwards - address to forward unprocessed replications
- tpu vote - address for bank state requests
- rpc - address for JSON-RPC requests
- rpc pubsub - websocket for JSON-RPC push notifications
- serve repair - address for send repair requests
- wallclock - wallclock of the node that generated that message
- shred_version - the shred version node has been configured to use

```rust
LegacyContactInfo {
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
```
##### `SocketAddr`
A v4 or v6 socket address and port
```rust
enum SocketAddr {
    V4(SocketAddrV4 {
        ip: Ipv4Addr,
        port: u16
    }),
    V6(SocketAddrV6 {
        ip: Ipv6Addr,
        port: u16,
        flowinfo: u32,
        scope_id: u32
    })
}
```
##### `Ipv4Addr`
IP v4 address
```rust
Ipv4Addr {
    octets: [u8, 4]
}
```
##### `Ipv6Addr`
IP v6 address
```rust
Ipv6Addr {
    octets: [u8, 32]
}
```

#### `Vote`
It's a validators vote on a fork. Contains a one byte index from vote tower (range 0 to 31) and a `Vote` structure:
```rust
Vote(u8, Vote)
```
##### `Vote`
 - from - public key of origin
 - transaction - a vote transaction, an atomically-committed sequence of instructions
 - wallclock - wallclock of the node that generated that message
 - slot - the unit of time given to a leader for encoding a block, it is actually an `u64` type

 ```rust
 Vote {
    from: Pubkey,
    transaction: Transaction,
    wallclock: u64,
    slot: Slot | None,
}
 ```
 ##### `Slot`
```rust
Slot = u64
```
##### `Transaction`
An atomically-committed sequence of instructions.
 * signature - list of signatures equal to `num_required_signatures` for the message
 * message - transaction message containing instructions to invoke
```rust
Transaction {
    signature: [Signature],
    message: Message
}
```
##### `Message`
A transaction message.
* header - message header 
* account_keys - all account keys used by this transaction
* recent_blockhash - hash of a recent ledger entry
* instructions - list of programs to execute
```rust
Message {
    header: MessageHeader,
    account_keys: [Pubkey],
    recent_blockhash: Hash,
    instructions: [CompiledInstruction],
}
```
##### `MessageHeader`

* num_required_signatures - number of signatures required for this message to be considered valid
* num_readonly_signed_accounts - last `num_readonly_signed_accounts` of the signed keys are read-only accounts
* num_readonly_unsigned_accounts - last `num_readonly_unsigned_accounts` of the unsigned keys are read-only accounts
```rust
MessageHeader {
    num_required_signatures: u8,
    num_readonly_signed_accounts: u8,
    num_readonly_unsigned_accounts: u8,
}
```

##### `CompiledInstruction`
* program_id_index - index of the transaction keys array indicating the program account ID that executes the program,
* accounts: [u8] - indices of the transaction keys array indicating the accounts that are passed to a program
* data - program input data
```rust
struct CompiledInstruction {
    program_id_index: u8,
    accounts: [u8],
    data: [u8],
}
```


#### `LowestSlot`
It is the first available slot in Solana blockstore that contains any data. Contains a one byte index (deprecated) and a `LowestSlot` structure:
```rust
LowestSlot(u8, LowestSlot)
```
##### `LowestSlot`
- from - public key of origin
- root - deprecated
- lowest - the actual slot
- slots - deprecated
- stash - deprecated
- wallclock - wallclock of the node that generated that message

```rust
LowestSlot {
    from: Pubkey,
    root: Slot,
    lowest: Slot,
    slots: [Slot],
    stash: [EpochIncompleteSlots],
    wallclock: u64,
}
```

#### `LegacySnapshotHashes`, `AccountsHashes`
- from - public key of origin
- hashes - a list of slots and hashes
- wallclock - wallclock of the node that generated that message

```rust
LegacySnapshotHashes = AccountsHashes {
    from: Pubkey,
    hashes: [(Slot, Hash)],
    wallclock: u64,
}
```

#### `EpochSlots`
Contains a one byte index and a `EpochSlots` structure which holds the list of all slots from an epoch (epoch consists around 432000 slots). There can be 256 epoch slots in total:
```rust
EpochSlots(u8, EpochSlots)
```
##### `EpochSlots`
- from - public key of origin
- slots - list of epoch slots - can be either uncompressed or compressed with `Flate2` algorithm
- wallclock - wallclock of the node that generated that message

```rust
EpochSlots {
    from: Pubkey,
    slots: [CompressedSlots],
    wallclock: u64,
}
```
##### `CompressedSlots`
```rust
enum CompressedSlots {
   Flate2(Flate2),
   Uncompressed(Uncompressed),
}
```
##### `Flate2`
```rust
Flate2 {
    first_slot: Slot,
    num: usize,
    compressed: [u8]
}
```
##### `Uncompressed`
```rust
Uncompressed {
    first_slot: Slot,
    num: usize,
    slots: [u8],
}
```

#### `LegacyVersion`
- from - public key of origin
- wallclock - wallclock of the node that generated that message
- version - older version of the Solana used earlier 1.3.x releases
```rust
LegacyVersion {
    from: Pubkey,
    wallclock: u64,
    version: LegacyVersion1,
}
```
##### `LegacyVersion1`
```rust
LegacyVersion1 {
    major: u16,
    minor: u16,
    patch: u16,
    commit: u32 | None
}
```

#### `Version`
Version of Solana client the node is using
- from - public key of origin
- wallclock - wallclock of the node that generated that message
- version - version of the Solana
```rust
Version {
    from: Pubkey,
    wallclock: u64,
    version: LegacyVersion2,
}
```
##### `LegacyVersion2`
```rust
LegacyVersion2 {
    major: u16,
    minor: u16,
    patch: u16,
    commit: u32 | None,
    feature_set: u32
}
```

#### `NodeInstance`
Contains node creation timestamp and randomly generated token:
- from - public key of origin
- wallclock - wallclock of the node that generated that message
- timestamp - when the instance was created
- token - randomly generated value at node instantiation.
```rust
NodeInstance {
    from: Pubkey,
    wallclock: u64,
    timestamp: u64,
    token: u64,
}
```

#### `DuplicateShred`
```rust
DuplicateShred(u16, DuplicateShred)
```
Contains a 2 byte index and `DuplicateShred` structure:
- from - public key of origin
- wallclock - wallclock of the node that generated that message
- slot - unit of time for encoding a block
```rust
DuplicateShred {
    from: Pubkey,
    wallclock: u64,
    slot: Slot,
    _unused: u32,
    _unused_shred_type: ShredType,
    num_chunks: u8,
    chunk_index: u8,
    chunk: [u8],
}
```

#### `SnapshotHashes`
- from - public key of origin
- full - 
- incremental - 
- wallclock - wallclock of the node that generated that message
```rust
SnapshotHashes {
    from: Pubkey,
    full: (Slot, Hash),
    incremental: [(Slot, Hash)],
    wallclock: u64,
}
```

#### `ContactInfo`
- pubkey - public key of origin
- wallclock - wallclock of the node that generated that message
- outset - timestamp when node instance was first created
- shred_version - the shred version node has been configured to use
- version - Solana version
- addrs - list of unique IP addresses
- sockets - list of nique sockets
- extensions - future additions to ContactInfo will be added to Extensions instead of modifying ContactInfo, currently unused
- cache - cache of nodes socket addresses
```rust
ContactInfo {
    pubkey: Pubkey,
    wallclock: u64,
    outset: u64,
    shred_version: u16,
    version: Version,
    addrs: [IpAddr],
    sockets: [SocketEntry],
    extensions: [Extension],
    cache: [SocketAddr; 12],
}
```
##### `Extension`
```rust 
Extension {}
```
##### `IpAddr`
Is either v4 or v6 IP address
```rust
enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv4Addr)
}
```
##### `SocketEntry`
* key - protocol identifier (tvu, tpu, etc.)
* index - IpAddr index in the addrs list
* offset - port offset in respect to previous entry
```rust
SocketEntry {
    key: u8,
    index: u8,
    offset: u16
}
```
##### `Version`
```rust
Version {
    major: u16,
    minor: u16,
    patch: u16,
    commit: u32 | None,
    feature_set: u32,
    client: u16
}
```

#### `RestartLastVotedForkSlots`
- from - public key of origin
- wallclock - timestamp of data creation
- offsets - 
- last_voted_slot - 
- last_voted_hash - 
- shred_version - the shred version node has been configured to use
```rust
RestartLastVotedForkSlots {
    from: Pubkey,
    wallclock: u64,
    offsets: SlotsOffsets,
    last_voted_slot: Slot,
    last_voted_hash: Hash,
    shred_version: u16,
}
```
##### `SlotsOffsets`
```rust
enum SlotsOffsets {
    RunLengthEncoding([u16]),
    RawOffsets([u8]),
}
```

#### `RestartHeaviestFork`
- from - public key of origin
- wallclock - timestamp of data creation
- last_slot - 
- last_hash - 
- observed_stake - 
- shred_version - the shred version node has been configured to use
```rust
RestartHeaviestFork {
    from: Pubkey,
    wallclock: u64,
    last_slot: Slot,
    last_slot_hash: Hash,
    observed_stake: u64,
    shred_version: u16,
}
```
