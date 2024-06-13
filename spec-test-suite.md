# Introduction

The purpose of this index is to provide an overview of the testing approaches for the gossip spec.
It is intended to evolve as the framework matures, leaving room for novel cases and extensions of existing cases,
as called for by any protocol idiosyncrasies that may come to light during the development process.

Some test cases have been consolidated when similar behaviour is tested with differing messages.
The final implementation of these cases will be subject to factors such as node setup and teardown details,
test run time (and potentially runtime) constraints, readability and maintainability.

> [!Tip]
> **Naming conventions used in this document**
> - _Node_ - a validator running the gossip
> - _Synthetic node_ - a lightweight gossip node that is used only for testing gossip protocol while interacting with the node via tests.
> - _Cluster_ - a network of validators with a leader that produces blocks

## Usage

The tests can be run with `cargo test` once the test suite setup is properly configured and dependencies (node instance to be tested) are satisfied.
See the [README](README.md) for details.

Tests are grouped into the following categories: conformance, performance, and resistance. Each test is named after the category it belongs to, in addition to what's being tested. For example, `c01_01_PING_upon_sending_expect_pong` is a conformance test that tests the `ping` message by expecting a `pong` response from the node.

The full naming convention is: `(test-group-id)(message-type-number)_(subtest-no)_(message_type_string)_(extra-test-desc)` where:

`test-group-id` is one of:
 - `c` - conformance,
 - `r` - resistance,
 - `p` - performance.

 `message-type-number` and `message-type-string` represents the category of tests:
 - `PING` is `01` and tests ping functionality,
 - `JOIN_CLUSTER` is `02` and tests node behaviour when joining the cluster,
 - `BLOOM_FILTER` is `03` and tests bloom filters,
 - `PRUNE` is `04` and tests prune functionality.

# Types of Tests

## Conformance

The conformance tests aim to verify the node adheres to the network protocol.
In addition, they include some naive error cases with malicious and fuzzing cases consigned to the resistance tests.
These tests also aim to evaluate the proper behaviour of a node when receiving unsolicited response messages.

## Performance

The performance tests aim to verify the node maintains a healthy throughput under pressure.
This is principally done through simulating load with synthetic peers and evaluating the node's responsiveness.
Synthetic peers will need to be able to simulate the behaviour of a full node by implementing handshaking, message sending and receiving.

These tests are intended to verify the node remains healthy under "reasonable load".
Additionally these tests will be pushed to the extreme for resistance testing with heavier loads.

## Resistance

The resistance tests are designed for the early detection and avoidance of weaknesses exploitable through malicious behaviour.
They attempt to probe boundary conditions with comprehensive fuzz testing and extreme load testing.
The nature of the peers in these cases will depend on how accurately they needs to simulate node behaviour.
It will likely be a mixture of simple sockets for the simple cases and peers used in the performance tests for the more advanced.

# Test Index

The test index makes use of symbolic language in describing connection and message sending directions.

| Symbol | Meaning |
|--------------|-----------|
| `-> A` | A synthetic node sends a message `A` to Solana node |
| `<- B` | Solana node sends a message `B` to a synthetic node |
| `>> C` | A synthetic node broadcasts a message `C` to set of nodes in the cluster |
| `<< D` | Solana node broadcasts a message `D` to set of nodes in the cluster |
| `<- [no-reply]` | Solana node doesn't answer with a reply |
| `<>` | A general sets of "handshake" messages after the node joins the cluster |
| `LNx` | Local node `x` |
| `SNx` | Synthetic node `x` |
| `...` | Any message |

## Network Protocol Test Coverage


> [!Tip]
> A quick table legend:
> - `Rx` - indicates if there are any tests that receive and parse this message.
> - `Tx` - indicates if there are any tests that send this message.
> - `Fuzz` - indicates if there are any tests that fuzz this message.
> - `Type` - message type.

|  Message                          | Type                  | Rx | Tx | Fuzz | Tests that focus on this message                                                    |
|-----------------------------------|-----------------------|----|----|------|-------------------------------------------------------------------------------------|
| IpEchoServerMessage               | IP echo server        | ✅ | ✅ |      | most tests that use `shred_version` |
| IpEchoServerResponse              | IP echo server        | ✅ | ✅ |      | most tests that use `shred_version` |
| PullRequest                       | Gossip Protocol       | ✅ | ✅ |      | `C02-02`, `c02-04` |
| PullResponse                      | Gossip Protocol       | ✅ | ✅ |      | `C02-02`, `c02-04` |
| PushMessage                       | Gossip Protocol       | ✅ | ✅ |      | `C02-02`, `c02-04` |
| PruneMessage                      | Gossip Protocol       | ✅ | ✅ |      | `C04-01`, `C04-02`, `C04-03` |
| PingMessage                       | Gossip Protocol       | ✅ | ✅ |      | `C01-01`, `C01-02`, `C01-03`, `C01-04`, all other tests |
| PongMessage                       | Gossip Protocol       | ✅ | ✅ |      | `C01-01`, `C01-02`, `C01-03`, `C01-04`, all other tests |
| LegacyContactInfo                 | CRDS                  | ✅ | ✅ |      | `C02-02`, `C02-03`, `C02-05` |
| Vote                              | CRDS                  | ✅ |    |      | `C02-02` |
| LowestSlot                        | CRDS                  |    |    |      | |
| EpochSlots                        | CRDS                  | ✅ |    |      | `C02-02` |
| LegacyVersion                     | CRDS                  |    |    |      | |
| Version                           | CRDS                  |    | ✅ |      | `C02-02` |
| NodeInstance                      | CRDS                  | ✅ | ✅ |      | `C02-02` |
| DuplicateShred                    | CRDS                  |    |    |      | |
| SnapshotHashes                    | CRDS                  | ✅ | ✅ |      | `C02-02`, `C02-03` |
| ContactInfo                       | CRDS                  | ✅ |    |      | `C02-02` |
| RestartLastVotedForkSlots         | CRDS                  |    |    |      | |
| RestartHeaviestFork               | CRDS                  |    |    |      | |
| LegacySnapshotHashes (Deprecated) | CRDS                  | x  | x  |  x   | _depricated_ |
| AccountsHashes       (Deprecated) | CRDS                  | x  | x  |  x   | _depricated_ |


The table below is oriented on particular functionality within the gossip protocol.

|  Functionality                    | Tests                                                                |
|-----------------------------------|----------------------------------------------------------------------|
| Pruning                           | `C04-01`, `C04-02`, `C04-03` |
| Bloom Filters                     | `C03-01` |

## Conformance

### GOSSIP PING C01-01

    The node correctly replies with a pong message to a ping request.

    -> ping (token)
    <- pong (correct token hash)

    Assert: the received token hash is correct.

### GOSSIP PING C01-02

    The node correctly successfully exchanges ping/pong messages.

    <> SN joins the cluster
    <- ping (token-1)
    -> pong (correct token-1 hash)
    -> ping (token-2)
    <- pong (correct token-2 hash)

    Assert: the received token hash is correct.

### GOSSIP PING C01-03

    The node correctly rejects ping request after receiving an invalid hash in the prior pong response.

    <>
    <- ping (token-1)
    -> pong (invalid token-1 hash)
    -> ping (token-2)
    <- [no-reply]

    Assert: the node’s ignores the ping message.

### GOSSIP PING C01-04

    The node correctly rejects ping request after receiving an unsolicited prior pong message.

    -> pong (unsolicited token-1 hash)
    -> ping (token-2)
    <- [no-reply]

    Assert: the node’s ignores the ping message.

### GOSSIP JOIN-CLUSTER C02-01

    The node correctly replies to the IP echo server request message (a pre-gossip protocol service)
    used for obtaining the shred version and IP address and testing open ports.

    -> IP echo server request
    <- IP echo server response (shred_version, IP address)

    Note: A synthetic node is not even used in this test.

### GOSSIP JOIN-CLUSTER C02-02

    The synthetic node joins the cluster and expects the usual message traffic from the cluster.

    -> PushMessage[NodeInstance]
    -> PushMessage[Version]
    -> Ping (token1)
    -> PullRequest[LegacyContactInfo]

    Expect messages:
    <- Pong (token1 hash)
    <- Ping (token2)
    <- PullRequest[...]
    <- PushMessage[X]

    Where X represents the commonly observed set of CRDS messages:
    - EpochSlots
    - Vote
    - LegacyContactInfo
    - ContactInfo
    - NodeInstance
    - SnapshotHashes

    Assert: all listed messages should be received.

### GOSSIP JOIN-CLUSTER C02-03

    The synthetic node represents the single-node cluster and the node uses synthetic node's address
    as an entrypoint to join the cluster.

    <- PushMessage[NodeInstance & LegacyContactInfo & ContactInfo]
    <- Ping
    <- PullRequest[...]

    Assert: all listed messages should be received from the node that is joining our cluster.

    Note: This test is using another real node's RPC server, since the synthetic node doesn't have an RPC server implemented (yet).

### GOSSIP JOIN-CLUSTER C02-04

    Two synthetic node joins the cluster of two nodes.
    Each synthetic node joins the cluster using the separate node as an entrypoint node.

    <> LN1 with SN1
    <> LN2 with SN2
    SN2: LN2<- PushMessage[LegacyContactInfo(SN1)]

    Assert: The legacy contact info belonging to one synthetic node is shared with the
    second synthetic node through the cluster.

### GOSSIP JOIN-CLUSTER C02-05

    Two synthetic node are started.

    Only the first synthetic node joins the cluster, while the second synthetic idly waits for connections,
    unaware of any cluster. The first synthetic node shares the legacy contact info from the second synthetic
    node with the cluster. The nodes from the cluster try to connect to the second synthetic node, which is
    how the second synthetic node joins the cluster.

    <> SN1
    ... start idle SN2 in the background
    SN1: LN-> PushMessage[LegacyContactInfo(SN2)]
    <>: LN connects to SN2

    Assert: The node from the cluster contacts synthetic node after it receives contact information
    from another synthetic node.

### GOSSIP BLOOM-FILTERS C03-01

    The synthetic node joins the cluster and asks for CRDS data from the node.
    After obtaining the data, the synthetic node sends a `PullRequest` message
    with the same bloom filter (exactly the same filter it received from the node)
    and expects to get no response back, since it has all data (according to the bloom filter).

    <> SN1
    <- PullRequest with the bloom filter (*)
    -> PullResponse[LegacyContactInfo] with LegacyContactInfo the node has already received
    -> PullRequest with the exactly the same bloom filter (*) the node sent us
    <- ...

    Assert: Expect no response to our PullRequest, since the bloom filter implies we have the identical CRDS table.

### GOSSIP PRUNE C04-01

    The synthetic nodes joins the cluster and starts re-sedning received push messages to one of the nodes.
    After some time the synthetic node should be pruned and should receive the prune message.
    To keep the connection with the cluster synthetic node should respond to any pull request.

    <> SN1
    <- PushMessage
    -> PushMessage - send previously received PushMessage to one of the nodes
    <- PullRequest
    -> PullResponse with any value
    <- PruneMessage

    Assert: Expect receiving PruneMessage with destination set to the synthetic nodes public key.

### GOSSIP PRUNE C04-02

    The synthetic nodes joins the cluster of 4 nodes and gathers push messages from all of them.
    When it receives push messages from all the nodes coming from all origins (e.g. messages sent by node 0 
    originating from nodes 0, 1, 2 and 3, and so on) it sends prune messages to all of the nodes: 
    e.g. node 0 will be pruned for messages originating from nodes 1, 2 and 3. After sending the prunes 
    the synthetic node should only receive push messages coming directly from the origins 
    (e.g. message from node 0 originating from node 0).

    <> SN1
    <- PushMessage from all nodes
    -> PruneMessage to all nodes
    <- PushMessage - only messages coming directly from origins

    Assert: Expect pruned nodes stop sending push messages to the synthetic node

### GOSSIP PRUNE C04-03

    Two synthetic nodes join the cluster, SN1 and SN2. SN2 sends a push message to local node LN1. 
    SN1 should receive a data from this message sent by LN1. Then, SN1 prunes SN2 by sending a prune
    message to LN1. SN2 sends the push message to LN1 again, but this time SN1 should not receive any message
    originating from SN2 sent by LN1.

    <> SN1
    <> SN2
    -> PushMessage sent by SN2 to LN1
    <- PushMessage from LN1 received by SN1, originating from SN2
    -> PruneMessage sent by SN1 to LN1 pruning SN2
    -> PushMessage sent by SN2 to LN1

    Assert: Expect SN1 to not receive any message originating from SN2 sent by LN1 after sending the prune
