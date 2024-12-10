# Introduction

The purpose of this index is to provide an overview of the testing approaches for the [gossip spec][spec].
It is intended to evolve as the framework matures, leaving room for novel cases and extensions of existing cases,
as called for by any protocol idiosyncrasies that may come to light during the development process.

Some test cases have been consolidated when similar behaviour is tested with differing messages.
The final implementation of these cases will be subject to factors such as node setup and teardown details,
test run time (and potentially runtime) constraints, readability and maintainability.

> [!Tip]
> **Naming conventions used in this document**
> - _Node_ - a validator running gossip undergoing testing
> - _Synthetic node_ - a lightweight gossip node that is used only for testing the gossip protocol while interacting with the node via tests
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
 - `01`: `PING` - Tests ping functionality
 - `02`: `JOIN_CLUSTER` - Tests node behavior when joining the cluster
 - `03`: `BLOOM_FILTER` - Tests bloom filters
 - `04`: `PRUNE` - Tests prune functionality

# Types of Tests

## Conformance

The conformance tests aim to verify the node adheres to the network [spec protocol][spec].
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
The tests will likely be a mixture of simple sockets for the simple cases and peers used in the performance tests for the more advanced.

# Test Index

The test index makes use of symbolic language in describing connection and message sending directions.

| Symbol | Meaning |
|--------------|-----------|
| `A <> B` | Node `A` joins the cluster by contacting the node `B` |
| `A -> B: MSG` | Node `A` sends a message `MSG` to the node `B` |
| `LN` | Local node undergoing testing |
| `SN` | Synthetic node running tests against `LN` |
| `LNx` | Local node `x` |
| `SNx` | Synthetic node `x` |
| `...` | Any message |

Note: The ContactInfo in all below tests refers to the CrdsData::ContactInfo and not the CrdsData::LegacyContactInfo unless otherwise specified.

## Network Protocol Test Coverage

> [!Tip]
> A quick table legend:
> - `Rx` - indicates if there are any tests that receive and parse this message.
> - `Tx` - indicates if there are any tests that send this message.
> - `Fuzz` - indicates if there are any tests that fuzz this message or the related message functionality.
> - `Type` - message type.

|  Message                          | Type                  | Rx | Tx | Fuzz | Tests that focus on this message                                                    |
|-----------------------------------|-----------------------|----|----|------|-------------------------------------------------------------------------------------|
| IpEchoServerMessage               | IP echo server        | ✅ | ✅ |  ✅  | most tests that use `shred_version` |
| IpEchoServerResponse              | IP echo server        | ✅ | ✅ |      | most tests that use `shred_version` |
| PullRequest                       | Gossip Protocol       | ✅ | ✅ |  ✅  | `C02-02`, `C02-04`, `R03-01` |
| PullResponse                      | Gossip Protocol       | ✅ | ✅ |      | `C02-02`, `C02-04` |
| PushMessage                       | Gossip Protocol       | ✅ | ✅ |  ✅  | `C02-02`, `C02-04`, `R02-07tX` |
| PruneMessage                      | Gossip Protocol       | ✅ | ✅ |  ✅  | `C04-01`, `C04-02`, `C04-03`, `R04-01`, `R04-02` `R04-03` |
| PingMessage                       | Gossip Protocol       | ✅ | ✅ |  ✅  | `C01-01`, `C01-02`, `R01-01`, `R01-02`, all other tests |
| PongMessage                       | Gossip Protocol       | ✅ | ✅ |  ✅  | `C01-01`, `C01-02`, `R01-01`, `R01-02`, all other tests |
| Vote                              | CRDS                  | ✅ |    |  ✅  | `C02-02`, `R02-07t1` |
| LowestSlot                        | CRDS                  | ✅ |    |  ✅  | `C02-02`, `R02-07t3` |
| EpochSlots                        | CRDS                  | ✅ |    |  ✅  | `C02-02`, `R02-07t4` |
| NodeInstance                      | CRDS                  | ✅ | ✅ |  ✅  | `C02-02`, `R02-01`, `R02-07t5` |
| DuplicateShred                    | CRDS                  |    |    |      | |
| SnapshotHashes                    | CRDS                  | ✅ | ✅ |  ✅  | `C02-02`, `C02-03`, `R02-07t6` |
| ContactInfo                       | CRDS                  | ✅ | ✅ |  ✅  | `C02-02`, `R02-07t2`, and most of the tests |
| RestartLastVotedForkSlots         | CRDS                  |    |    |      | |
| RestartHeaviestFork               | CRDS                  |    |    |      | |
| Version                           | CRDS                  | x  | x  |  x   | _depricated_ |
| LegacyContactInfo    (Deprecated) | CRDS                  | x  | x  |  x   | _depricated_ |
| LegacyVersion        (Deprecated) | CRDS                  | x  | x  |  x   | _depricated_ |
| LegacySnapshotHashes (Deprecated) | CRDS                  | x  | x  |  x   | _depricated_ |
| AccountsHashes       (Deprecated) | CRDS                  | x  | x  |  x   | _depricated_ |

The CRDS data type is the actual data that is stored in the CRDS table of the node.

## List of tests

| Type | Number | Status | Description |
|------|--------|--------|-------------|
| Conformance | [C01-01](#gossip-ping-c01-01) |  ✅ | The node correctly responds to a simple ping request. |
| Conformance | [C01-02](#gossip-ping-c01-02) |  ✅ | The node correctly initiates ping-pong exchange after another node joins the cluster. |
| Conformance | [C01-03](#gossip-ping-c01-03) |  ✅ | The test ensures that the node will ignore pull requests from nodes that have neither responded to nor sent a ping message. |
| Conformance | [C01-04](#gossip-ping-c01-04) |  ✅ | The test ensures that the node will not ignore pull requests from nodes that have responded to a ping message. |
| Conformance | [C02-01](#gossip-join-cluster-c02-01) |  ✅ | Test ensures the node correctly implements the IP echo server functionality. |
| Conformance | [C02-02](#gossip-join-cluster-c02-02) |  ✅ | Test ensures the node broadcasts expected messages to new nodes that joined the cluster. |
| Conformance | [C02-03](#gossip-join-cluster-c02-03) |  ✅ | The test ensures the node knows how to join the cluster. |
| Conformance | [C02-04](#gossip-join-cluster-c02-04) |  ✅ | The test ensures that nodes will share ContactInfo of new nodes with the cluster. |
| Conformance | [C02-05](#gossip-join-cluster-c02-05) |  ✅ | The test ensures that the node will try to contact newly discovered nodes. |
| Conformance | [C03-01](#gossip-bloom-filters-c03-01) |  ✅ | The test ensures the node will respond to the pull request only with the data that is not contained in the sent pull request's bloom filter. |
| Conformance | [C03-02](#gossip-bloom-filters-c03-02) |  ✅ | The test ensures the node will reply with exact CRDS data that is specified as missing in the bloom filter. |
| Conformance | [C03-03](#gossip-bloom-filters-c03-03) |  ✅ | The test ensures the node will send a created CRDS value that is missing in the bloom filter. |
| Conformance | [C03-04](#gossip-bloom-filters-c03-04) |  ✅ | The test ensures the node will not send a created CRDS value that is contained in the bloom filter. |
| Conformance | [C04-01](#gossip-prune-c04-01) |  ✅ | The test ensures the node will send a prune message when it starts receiving too many push messages from another node. |
| Conformance | [C04-02](#gossip-prune-c04-02) |  ✅ | The test ensures the node will stop sending specified push messages after it receives a prune message from another node. |
| Conformance | [C04-03](#gossip-prune-c04-03) |  ✅ | The test ensures the node will stop sending specified push messages after it receives a prune message from another node. |
| Performance | [P01-01t1](#gossip-ping-p01-01-t1) |  ✅ | The test ensures the node can handle lots of incoming ping messages from another node. |
| Performance | [P01-01t2](#gossip-ping-p01-01-t1) |  ✅ | The test ensures the node can handle lots of incoming ping messages from another node. |
| Performance | [P01-02](#gossip-ping-p01-02) |  ✅ | The test ensures the node can handle lots of incoming ping messages from multiple nodes. |
| Performance | [P01-03](#gossip-ping-p01-03) |  ✅ | The test ensures the node can handle a few incoming ping messages from many nodes in a large cluster. |
| Performance | [P01-04](#gossip-ping-p01-04) |  ✅ | The test ensures the node can handle lots of incoming ping messages from a large cluster at great intensity. |
| Resistance | [R01-01](#gossip-ping-r01-01) |  ✅ | The test checks whether the node will ignore nodes that fail to respond correctly to pong messages. |
| Resistance | [R01-02](#gossip-ping-r01-02) |  ✅ | The test checks if the node will ignore other nodes that send unsolicited pong messages. |
| Resistance | [R01-03](#gossip-ping-r01-03) |  ✅ | The test checks whether the node will respond to a ping that contains an invalid signature. |
| Resistance | [R01-04](#gossip-ping-r01-04) |  ✅ | The test checks whether the node will ignore nodes that send pongs with the wrong signature. |
| Resistance | [R02-01](#gossip-join-cluster-r02-01) |  ✅ | The test checks whether the node will accept any randomly generated NodeInstance. |
| Resistance | [R02-02](#gossip-join-cluster-r02-02) |  ✅ | Test ensures the node correctly checks for open ports listed in the IP echo request message. |
| Resistance | [R02-03](#gossip-join-cluster-r02-03) |  ✅ | The test ensures that the node will ignore pull requests from nodes that contain ContactInfo with an invalid signature. |
| Resistance | [R02-04](#gossip-join-cluster-r02-04) |  ✅ | The test ensures that the node will ignore pull requests from nodes that contain ContactInfo with an invalid wallclock. |
| Resistance | [R02-05](#gossip-join-cluster-r02-05) |  ✅ | The test ensures that nodes will not share ContactInfo of new nodes with the cluster that contain an expired wallclock. |
| Resistance | [R02-06](#gossip-join-cluster-r02-06) |  ✅ | The test ensures that nodes will reject push messages containing a ContactInfo with an invalid signature. |
| Resistance | [R02-07t1](#gossip-join-cluster-r02-07-t1) |  ✅ | The test ensures that nodes will reject push messages containing a fuzzed Vote data. |
| Resistance | [R02-07t2](#gossip-join-cluster-r02-07-t2) |  ✅ | The test ensures that nodes will reject push messages containing a fuzzed ContactInfo data. |
| Resistance | [R02-07t3](#gossip-join-cluster-r02-07-t3) |  ✅ | The test ensures that nodes will reject push messages containing a fuzzed LowestSlot data. |
| Resistance | [R02-07t4](#gossip-join-cluster-r02-07-t4) |  ✅ | The test ensures that nodes will reject push messages containing a fuzzed EpochSlots data. |
| Resistance | [R02-07t5](#gossip-join-cluster-r02-07-t5) |  ✅ | The test ensures that nodes will reject push messages containing a fuzzed NodeInstance data. |
| Resistance | [R02-07t6](#gossip-join-cluster-r02-07-t6) |  ✅ | The test ensures that nodes will reject push messages containing a fuzzed SnapshotHashes data. |
| Resistance | [R02-08](#gossip-join-cluster-r02-08) |  ✅ | The test checks how the node will react in case of receiving some random bytes from newly connected synthetic node. |
| Resistance | [R02-09](#gossip-join-cluster-r02-09) |  ✅ | The test checks how the node will react in case of receiving some random bytes in a pull response from synthetic node. |
| Resistance | [R03-01](#gossip-bloom-filters-r03-01) |  ✅ | The test ensures that nodes will ignore pull requests with invalid bloom filters. |
| Resistance | [R04-01](#gossip-prune-r04-01) |  ✅ | The goal of this test is to check whether the node who signs the prune message also has to be the node that sends that message. |
| Resistance | [R04-02](#gossip-prune-r04-02) |  ✅ | Check whether the node will reject prune messages signed and sent by the wrong node (malicious node). |
| Resistance | [R04-03](#gossip-prune-r04-03) |  ✅ | Check behavior of the nodes in the cluster after ignoring incoming prune messages. |
| Resistance | [R04-04](#gossip-prune-r04-04) |  ✅ | The test ensures the node will not stop sending specified push messages after it receives a prune message with an invalid signature. |
| Resistance | [R04-05](#gossip-prune-r04-05) |  ✅ | The test ensures the node will not stop sending specified push messages after it receives a prune message with an invalid wallclock. |
| Resistance | [R04-06](#gossip-prune-r04-06) |  ✅ | The test ensures the node will not stop sending specified push messages after it receives a prune message with an invalid destination. |


## Conformance

### GOSSIP PING C01-01

    Goal: The node correctly responds to a simple ping request.

    LN1 correctly replies with a pong message to a ping request.

    SN -> LN: ping (token)
    LN -> SN: pong (correct token hash)

    Assert: the received token hash is correct.

### GOSSIP PING C01-02

    Goal: The node correctly initiates ping-pong exchange after another node joins the cluster.

    LN successfully exchanges ping/pong messages after SN joins the cluster and performs the handshake.

    SN <> LN
    LN -> SN: ping (token-1)
    SN -> LN: pong (correct token-1 hash)
    SN -> LN: ping (token-2)
    LN -> SN: pong (correct token-2 hash)

    Assert: the received token hash is correct.

### GOSSIP PING C01-03

    Goal: The test ensures that the node will ignore pull requests from nodes that have neither responded to nor sent a ping message.

    SN sends only pull requests and never responds to LN's ping requests.

    SN -> LN: pull request
    LN -> SN: ping
    SN -> LN: pull request

    Assert: LN won't reply to a pull request nor send any other message until SN replies with a pong message.

### GOSSIP PING C01-04

    Goal: The test ensures that the node will not ignore pull requests from nodes that have responded to a ping message.

    SN sends a pull request to initiate the connection.
    LN ignores the pull request but sends a ping message request instead.
    SN responds with a pong message and then sends another pull request.

    SN -> LN: pull request
    LN -> SN: ping
    SN -> LN: pong
    SN -> LN: pull request
    LN -> SN: pull response

    Assert: LN only replies to a pull request if SN has responded to a previously sent ping message with a pong message.
    Note: In the test above, SN never sends a ping message request to LN. It only replies to a ping message request from LN.

### GOSSIP JOIN-CLUSTER C02-01

    Goal: Test ensures the node correctly implements the IP echo server functionality.

    LN correctly replies to the IP echo server request message (a pre-gossip protocol service)
    used for obtaining the shred version and IP address and testing open ports.

    TCP socket -> LN: IP echo server request
    LN -> TCP socket: IP echo server response (shred_version, IP address)

    Note: A synthetic node is not used in this test.

### GOSSIP JOIN-CLUSTER C02-02

    Goal: Test ensures the node broadcasts expected messages to new nodes that joined the cluster.

    SN joins the cluster and expects the usual message traffic from LN representing the entrypoint node in the cluster.

    SN -> LN: PushMessage[NodeInstance]
    SN -> LN: PushMessage[Version]
    SN -> LN: Ping (token1)
    SN -> LN: PullRequest[ContactInfo]

    Expect messages:
    LN -> SN: Pong (token1 hash)
    LN -> SN: Ping (token2)
    SN -> LN: Pong (token2 hash)
    LN -> SN: PullRequest[...]
    LN -> SN: PullResponse[X]
    LN -> SN: PushMessage[X]

    Where X represents the commonly observed set of CRDS messages:
    - EpochSlots
    - Vote
    - LegacyContactInfo
    - ContactInfo
    - NodeInstance
    - SnapshotHashes
    - LowestSlot

    Assert: SN should receive all listed messages.

### GOSSIP JOIN-CLUSTER C02-03

    Goal: The test ensures the node knows how to join the cluster.

    SN represents the cluster and LN2 uses SN's address as an entrypoint to join the cluster.

    SN <> LN1
    LN1 -> SN: ContactInfo with the RPC server address

    SN crafts ContactInfo which uses LN1's RPC server address.
    LN2 is started with the SN's gossip address as an entrypoint address.
    LN2 -> SN: PushMessage[NodeInstance & LegacyContactInfo & ContactInfo]
    LN2 -> SN: Ping
    LN2 -> SN: PullRequest[...]

    Assert: LN2 should receive all listed messages.

    Note: This test is using another LN's RPC server, since SN doesn't have an RPC server implemented (yet).

### GOSSIP JOIN-CLUSTER C02-04

    Goal: The test ensures that nodes will share ContactInfo of new nodes with the cluster.

    Two SNs join the cluster of LNs.
    Each SN joins the cluster using the separate LN as an entrypoint node.

    SN1 <> LN1
    SN2 <> LN2
    LN2 -> SN2: PushMessage[ContactInfo(SN1)]

    Assert: The ContactInfo belonging to one SN1 is shared with the SN2 node through the cluster.

### GOSSIP JOIN-CLUSTER C02-05

    Goal: The test ensures that the node will try to contact newly discovered nodes.

    Two SNs are started.
    Only SN1 joins the cluster, while SN2 idly waits for connections, unaware of any cluster.
    SN1 shares the ContactInfo belonging to SN2 with the cluster.
    LN node from the cluster will try to connect to SN2, which is how SN2 joins the cluster.

    SN1 <> LN
    SN2 is started idly in the background
    SN1 -> LN: PushMessage[ContactInfo(SN2)]
    LN <> SN2

    Assert: LN contacts SN2 after it receives the ContactInfo from another node (SN1).

### GOSSIP BLOOM-FILTERS C03-01

    Goal: The test ensures the node will respond to the pull request only with the data that
          is not contained in the sent pull request's bloom filter.

    SN joins the cluster and asks for CRDS data from the node via the pull request message.
    After obtaining the data, SN sends a `PullRequest` message with the same bloom filter
    (exactly the same bloom filter it received from LN) and expects to get no response back,
    since SN already has all data (according to the bloom filter).

    SN <> LN
    LN -> SN: PullRequest with the bloom filter
    SN -> LN: PullResponse[ContactInfo] with ContactInfo LN has already received before

    SN -> LN: PullRequest with exactly the same bloom filter LN sent us previously
    LN -> SN: [...] no PullResponse

    SN -> LN: PullRequest with the empty bloom filter
    LN -> SN: PullResponse[...]

    Assert: Expect no response to our PullRequest that has the same bloom filter, since such a bloom filter implies we have the identical CRDS table.

### GOSSIP BLOOM-FILTERS C03-02

    Goal: The test ensures the node will reply with exact CRDS data that is specified as missing in the bloom filter.

    SN joins the cluster and sends a pull request with an empty bloom filter to LN.
    After SN receives the pull request, SN puts all received values in the bloom filter except one randomly selected value.
    Then SN sends again the pull request with the created bloom filter.
    LN should not send back any value found in the bloom filter but should send back the one value that is missing in the bloom filter.

    SN <> LN
    SN -> LN: PullRequest with an empty bloom filter
    LN -> SN: PullResponse
    LN -> SN: PullResponse
    LN -> SN: PullResponse
    ...
    SN collects lots of CRDS values and then randomly selects one value that won't be included in the bloom filter.
    SN -> LN: PullRequest with the bloom filter that doesn't contain one value
    LN -> SN: PullResponse with the missing CRDS value.

    Assert: Expect that SN will receive the value missing in the bloom filter and will not receive any value that is in the bloom filter.

### GOSSIP BLOOM-FILTERS C03-03

    Goal: The test ensures the node will send a created CRDS value that is missing in the bloom filter.
    Note: this test is similar to the C03-02 test, but it's using PushMessage to share the created CRDS value with LN.

    SN joins the cluster and sends a push message with ContactInfo value to LN.
    Next, SN sends a pull request with an empty bloom filter.
    In the pull response received from LN, SN should receive the ContactInfo value it previosuly sent to LN in the first push message.

    SN <> LN
    SN -> LN: PushMessage with the ContactInfo
    SN -> LN: PullRequest with an empty bloom filter
    LN -> SN: PullResponse - SN should receive the ContactInfo value it sent in the first PushMessage

    Assert: Expect that SN will receive the ContactInfo value sent in the first push message.

### GOSSIP BLOOM-FILTERS C03-04

    Goal: The test ensures the node will not send a created CRDS value that is contained in the bloom filter.

    SN joins the cluster and sends a push message with NodeInstance value to LN.
    Next, SN creates a bloom filter and inserts the created NodeInstance value.
    Then, SN sends a pull request with the created bloom filter.
    In the pull response, SN should not receive the NodeInstance value it previosuly sent in the first push message to LN.

    SN <> LN
    SN -> LN: PushMessage with the NodeInstance
    SN -> LN: PullRequest with the bloom filter containing the NodeInstance value
    LN -> SN: PullResponse (without NodeInstance)

    Assert: Expect that SN will not receive the NodeInstance value sent in the first push message.

### GOSSIP PRUNE C04-01

    Goal: The test ensures the node will send a prune message when it starts receiving too many push messages from another node.

    SN joins the cluster and it waits for push messages from other four LN.
    For every received push message, SN resends it to LN1.
    After some time SN should should receive the prune message from LN1.

    SN <> LN1 & LN2 & LN3 & LN4
    Then repeat:
        LN1 -> SN: PushMessage[CrdsValues1]
        // Only in this case send the message to LN2 instead of LN1:
        SN -> LN2: PushMessage[CrdsValues1]

        LN2 -> SN: PushMessage[CrdsValues2]
        SN -> LN1: PushMessage[CrdsValues2]

        LN3 -> SN: PushMessage[CrdsValues3]
        SN -> LN1: PushMessage[CrdsValues3]

        LN4 -> SN: PushMessage[CrdsValues4]
        SN -> LN1: PushMessage[CrdsValues4]

    Until:
        LN1 -> SN: PruneMessage

    Assert: Expect that SN will receive from LN1 a PruneMessage with the destination field in the PruneData struct set to the SN's public key.

### GOSSIP PRUNE C04-02

    Goal: The test ensures the node will stop sending specified push messages after it receives a prune message from another node.

    SN joins the cluster of four LNs and gathers push messages from all of them.
    After SN receives all push messages from all LNs coming from all origins (e.g. messages sent by LN1 originating
    from nodes LN1, LN2, LN3 and LN4, and so on) - then SN sends prune messages to all of the nodes:
    e.g. LN1 will be pruned for messages originating from nodes LN2, LN3 and LN4.
    After sending the prunes, SN should only receive push messages coming directly from the origins
    (e.g. message from LN1 originating from LN1).

    SN <> LN1 & LN2 & LN3 & LN4

    Wait until SN receives all of these:
        LN1 -> SN: PushMessage with CRDS values originating from all LNs
        LN2 -> SN: PushMessage with CRDS values originating from all LNs
        LN3 -> SN: PushMessage with CRDS values originating from all LNs
        LN4 -> SN: PushMessage with CRDS values originating from all LNs

    Then:
        SN -> LN1: PruneMessage for origins LN2, LN3, LN4
        SN -> LN2: PruneMessage for origins LN1, LN3, LN4
        SN -> LN3: PruneMessage for origins LN1, LN2, LN4
        SN -> LN4: PruneMessage for origins LN1, LN2, LN3

    Then expect:
        LN1 -> SN: PushMessage with CRDS values originating only from LN1
        LN2 -> SN: PushMessage with CRDS values originating only from LN2
        LN3 -> SN: PushMessage with CRDS values originating only from LN3
        LN4 -> SN: PushMessage with CRDS values originating only from LN4

    Assert: Expect pruned LNs will stop sending push messages with CRDS values originating from other LNs to SN.

### GOSSIP PRUNE C04-03

    Goal: The test ensures the node will stop sending specified push messages after it receives a prune message from another node.
    Note: This test is a variation of the C04-02 test, but in this test the pruned origin is another SN.

    SN1 and SN2 nodes join the cluster of four LNs.
    SN2 sends a push message with SN2's ContactInfo to LN1.
    SN1 should receive from LN1 a push message containing SN2's ContactInfo.
    Then, SN1 prunes SN2 by sending a prune message to LN1 - pruning all messages coming from SN2 origin.
    SN2 sends the push message to LN1 again, but this time SN1 should not receive any message
    originating from SN2 sent by LN1.

    SN1 <> LN1 & LN2 & LN3 & LN4
    SN2 <> LN1 & LN2 & LN3 & LN4
    SN2 -> LN1: PushMessage with CRDS value originating from SN2
    LN1 -> SN1: PushMessage with CRDS value originating from SN2
    SN1 -> LN1: PruneMessage pruning the SN2 origin
    SN2 -> LN1: PushMessage with CRDS value originating from SN2
    LN1 -> SN1: PushMessage with any CRDS value not originating from SN2

    Assert: Expect SN1 to not receive any message originating from SN2 sent by LN1 after sending the prune.

## Performance

The result of these tests can vary from machine to machine depending on their hardware capabilities.

### GOSSIP PING P01-01

    Goal: The test checks the node performance when increasing number of peers connect to it and send ping messages.

    The test is run in several iterations, where number of synthetic nodes is increased each iteration. 
    Synthetic nodes connect to a local node and send 1000 pings expecting 1000 pongs in response. 

    for X in (10, 20, 50, 100, ..., 500)
        for SN in (1, 2, ..., X): 
            SN <> LN
            SN -> LN: Ping (token-1)
            LN -> SN: Pong (correct token-1 hash)
            SN -> LN: Ping (token-2)
            LN -> SN: Pong (correct token-2 hash)
            ...
            SN -> LN: Ping (token-1000)
            LN -> SN: Pong (correct token-1000 hash)

    Output: A table with statistics:

| peers | requests | completion %  |  time (s) | requests/s  |
| ---- | ---- | ---- | ---- | ---- |
|  10 | 1000 | 100.00 | 2.55  |3918.69 |
|  20 | 1000 | 100.00 | 4.51 |4436.05 |
|  50 | 1000 | 100.00 | 10.31 | 4847.79 |
| 100 | 1000 | 100.00 | 20.45 | 4889.66 |
| 200 | 1000 | 100.00 | 40.18 | 4977.79 |
| 300 | 1000 | 100.00 | 59.37 |5053.00 |
| 500 | 1000 | 100.00 | 96.11  | 5202.33 |


### GOSSIP PING P01-02

    Goal: The test checks the node performance when 100 peers connect to it and send increasing number of ping messages.

    The test is run in several iterations, where number of sent ping messages is increased each iteration and number of connected peers remains at 100.
    Synthetic nodes connect to a local node and send X (10, 20, 50, 100, ...) pings expecting the same number of pongs in response. 

    for X in (10, 20, 50, 100, ..., 1000):
        for SN in (1, 2, ..., 100): 
            SN <> LN
            SN -> LN: Ping (token-1)
            LN -> SN: Pong (correct token-1 hash)
            SN -> LN: Ping (token-2)
            LN -> SN: Pong (correct token-2 hash)
            ...
            SN -> LN: Ping (token-X)
            LN -> SN: Pong (correct token-X hash)

    Output: A table with statistics:

|  peers  |  requests  |  completion %  |  time (s)  |  requests/s  |
|---------|------------|----------------|------------|--------------|
|     100 |         10 |         100.00 |       0.68 |      1463.55 |
|     100 |         20 |         100.00 |       0.98 |      2050.76 |
|     100 |         50 |         100.00 |       1.33 |      3758.13 |
|     100 |        100 |         100.00 |       2.20 |      4553.35 |
|     100 |        200 |          84.36 |       2.94 |      5741.83 |
|     100 |        300 |          77.30 |       4.01 |      5784.04 |
|     100 |        500 |          74.09 |       7.10 |      5216.01 |
|     100 |        800 |          70.34 |      13.19 |      4266.60 |
|     100 |       1000 |          75.87 |      16.54 |      4585.97 |

### GOSSIP JOIN-CLUSTER P02-01

    Goal: The test checks the node performance when increasing number of peers connect to it.

    The test is run in several iterations, where number of synthetic nodes is increased each iteration. 
    Synthetic nodes connect to a local node and expect to receive at least one push message and one pull request. 

    for X in (10, 20, 50, 100, ..., 500)
        for SN in (1, 2, ..., X): 
            SN <> LN
            LN -> SN: PushMessage
            LN -> SN: PullRequest

    Output: A table with statistics:

|  peers  |  requests  |  completion %  |  time (s)  |  requests/s  |
|---------|------------|----------------|------------|--------------|
|       5 |          1 |         100.00 |       7.50 |         0.67 |
|      10 |          1 |         100.00 |       7.32 |         1.37 |
|      20 |          1 |          70.00 |      52.10 |         0.27 |
|      50 |          1 |          30.00 |      55.22 |         0.27 |
|     100 |          1 |          15.00 |      60.42 |         0.25 |
|     200 |          1 |           8.00 |      70.79 |         0.23 |
|     300 |          1 |           5.67 |      81.02 |         0.21 |
|     500 |          1 |           3.80 |     101.67 |         0.19 |

### GOSSIP JOIN-CLUSTER P02-02

    Goal: The test checks the node performance when increasing number of peers connect to it.

    The test is run in several iterations, where the number of synthetic nodes increases each iteration. 
    Synthetic nodes connect to a local node, send a pull request and expect to receive a pull response.

    for X in (10, 20, 50, 100, 200, ..., 1000)
        for SN in (1, 2, ..., X): 
            SN <> LN
            SN -> LN: PullRequest
            LN -> SN: PullResponse

    Output: A table with statistics:

|  peers  |  requests  |  completion %  |  time (s)  |  requests/s  |
|---------|------------|----------------|------------|--------------|
|       1 |          1 |         100.00 |       0.11 |         9.33 |
|       2 |          1 |         100.00 |       0.11 |        18.68 |
|       5 |          1 |          20.00 |       5.11 |         0.20 |
|      10 |          1 |          10.00 |       5.11 |         0.20 |
|      20 |          1 |          20.00 |       5.11 |         0.78 |
|      50 |          1 |          22.00 |       5.12 |         2.15 |
|     100 |          1 |          16.00 |       5.13 |         3.12 |
|     200 |          1 |           8.50 |       5.15 |         3.30 |
|     500 |          1 |           3.20 |       5.44 |         2.94 |
|     800 |          1 |           2.50 |       5.59 |         3.58 |
|    1000 |          1 |           2.20 |       5.70 |         3.86 |

### GOSSIP PRUNE P04-01

    Goal: The test checks the node performance when increasing number of peers connect to it and spam it with push messages.

    The test is run in several iterations, where number of synthetic nodes is increased each iteration. 
    Synthetic nodes connect to a cluster consisting of 4 local nodes. Whenever a synthetic node receives
    a push message it sends it to the local node 1 or to local node 2 if the push message was comming from local node 1.
    This is repeated until a synthetic nodes receives the prune message.

    for X in (10, 20, 50, 100, 200, 300)
        for SN in (1, 2, ..., X): 
            SN <> LN1, LN2, LN3, LN4
            LN -> SN: PushMessage
            SN -> LN1 or LN2: PushMessage coming from LNx
            ...
            LNx -> SN: PruneMessage

    Note: LNx means any local node.
    
    Output: A table with statistics:

|  peers  |  requests  |  completion %  |  time (s)  |  requests/s  |
|---------|------------|----------------|------------|--------------|
|       5 |          1 |         100.00 |       9.77 |         0.51 |
|      10 |          1 |          90.00 |      53.68 |         0.17 |
|      20 |          1 |         100.00 |      43.20 |         0.46 |
|      50 |          1 |          58.00 |      72.26 |         0.40 |
|     100 |          1 |          36.00 |      93.82 |         0.38 |
|     200 |          1 |          23.50 |     138.34 |         0.34 |
|     300 |          1 |          19.67 |     185.27 |         0.32 |

## Resistance

### GOSSIP PING R01-01

    Goal: The test checks whether the node will ignore nodes that fail to respond correctly to pong messages.

    LN responds to a ping request from SN after receiving an invalid hash in the prior pong response.

    SN <> LN
    LN -> SN: ping (token-1)
    SN -> LN: pong (invalid token-1 hash)
    SN -> LN: ping (token-2)
    LN -> SN: pong (correct token-1 hash)

    Assert: LN sends the pong response to SN's ping request with the correct token-2 hash.
    Improvement: LN should ignore SN after SN sent the invalid ping response earlier.

### GOSSIP PING R01-02

    Goal: The test checks if the node will ignore other nodes that send unsolicited pong messages.

    LN responds to a ping request after receiving an unsolicited prior pong message from SN that never peformed the proper handshake.

    SN -> LN: pong (unsolicited token-1 hash)
    SN -> LN: ping (token-2)
    LN -> SN: pong (correct token-2 hash)

    Assert: LN sends the pong response to SN's ping request with the correct token-2 hash.
    Improvement: LN should ignore SN after SN sent an unsolicited pong message earlier.

### GOSSIP PING R01-03

    Goal: The test checks whether the node will respond to a ping that contains an invalid signature.

    SN sends a ping message that is not signed correctly to LN.
    LN ignores the ping and does not respond with a pong message.

    SN -> LN: ping (invalid signature)

    Assert: LN ignores the ping.

### GOSSIP PING R01-04

    Goal: The test checks whether the node will ignore nodes that send pongs with the wrong signature.

    SN sends a pull request to initiate the connection.
    LN ignores the pull request but sends a ping message request instead.
    SN responds with a pong message that contains an invalid signature and then sends another pull request.

    SN -> LN: pull request
    LN -> SN: ping
    SN -> LN: pong (with the invalid signature)
    SN -> LN: pull request

    Assert: LN ignores SN pull request messages since SN failed to respond with a correct pong message.

### GOSSIP JOIN-CLUSTER R02-01

    Goal: The test checks whether the node will accept any randomly generated NodeInstance.

    SN1 and SN2 join the cluster.
    SN1 sends a random NodeInstance belonging to no actual node and signs it with a randomly generated keypair to LN.
    LN retransmits the received NodeInstance to other nodes in the cluster via the PushMessage.

    SN1 <> LN
    SN2 <> LN
    SN1 -> LN: Push message with a random NodeInstance signed with a random keypair
    LN -> SN2: Push message with a random NodeInstance signed with a random keypair

    Assert: LN forwards NodeInstance to LN2 via the push message.
    Improvement: LN should ignore any unchecked NodeInstance that doesn't belong to an actual node in the cluster.

### GOSSIP JOIN-CLUSTER R02-02

    Goal: Test ensures the node correctly checks for open ports listed in the IP echo request message.

    LN receives an IP Echo Server request with a list of unreachable ports.
    LN tries to verify reachability for each port.
    Since ports are unreachable, LN should respond with a failed IP echo server response.

    TCP socket -> LN: IP echo server request with a list of unavailable ports
    LN -> TCP socket: a failed IP echo server response

    Assert: LN will reply with a failed IP echo server response.
    Note: A synthetic node is not used in this test.

### GOSSIP JOIN-CLUSTER R02-03

    Goal: The test ensures that the node will ignore pull requests from nodes that contain ContactInfo with an invalid signature.

    SN sends a pull request containing ContactInfo with the wrong signature to initiate the connection.
    LN ignores the pull request and the ContactInfo within the pull request.

    SN -> LN: pull request with the wrong signature

    Assert: LN ignores a pull request and the ContactInfo if the signature is not valid.

### GOSSIP JOIN-CLUSTER R02-04

    Goal: The test ensures that the node will ignore pull requests from nodes that contain ContactInfo with an invalid wallclock.

    SN sends a pull request with ContactInfo containing an invalid wallclock to initiate the connection.
    LN ignores the pull request and the ContactInfo within the pull request.

    SN -> LN: pull request with the wrong wallclock

    Assert: LN ignores a pull request and the ContactInfo if the wallclock is invalid.

### GOSSIP JOIN-CLUSTER R02-05

    Goal: The test ensures that nodes will not share ContactInfo of new nodes with the cluster that contain an expired wallclock.

    SN1 and SN2 join the cluster of LNs.
    SN2 uses the expired wallclock in its ContactInfo messages.
    SN1 never receives the SN2's ContactInfo from the cluster due to the expired wallclock.

    Note: in all of the below message, SN2 always shares its ContactInfo message with the expired walclock.
    SN1 <> LN1
    SN2 <> LN2

    Assert: A ContactInfo belonging to SN2 is never shared with SN1 through the cluster since it contains the expired wallclock.

### GOSSIP JOIN-CLUSTER R02-06

    Goal: The test ensures that nodes will reject push messages containing a ContactInfo with an invalid signature.

    Two SNs are started.
    Only SN1 joins the cluster, while SN2 idly waits for connections, unaware of any cluster.
    SN1 shares the ContactInfo with the wrong signature belonging to SN2 with the cluster.
    LN node will ignore ContactInfo with the wrong signature and SN2 will never join the cluster.

    SN1 <> LN
    SN2 is started idly in the background
    SN1 -> LN: PushMessage[ContactInfo(SN2) with the wrong signature]

    Assert: SN2 is never contacted by LN since LN received SN2's ContactInfo with an invalid signature.

### GOSSIP JOIN-CLUSTER R02-07-t1

    Goal: The test ensures that nodes will reject push messages containing a fuzzed Vote data.

    Two SNs and two local nodes are started. When SN1 receives a Vote push message from any local node,
    it fuzzes it (changes some of the data) and sends it to the other local node. SN2 should not receive the fuzzed message.

    SN1 & SN2 <> LN1 & LN2
    LN1 -> SN1: PushMessage with Vote data
    SN1 fuzzes the Vote data
    SN1 -> LN2: PushMessage with fuzzed Vote data

    Assert: SN2 never receives the fuzzed message.

### GOSSIP JOIN-CLUSTER R02-07-t2

    Goal: The test ensures that nodes will reject push messages containing a fuzzed ContactInfo data.
    Note: A different variant of R02-07-t1

    Two SNs and two local nodes are started. When SN1 receives a ContactInfo push message from any local node,
    it fuzzes it (changes some of the data) and sends it to the other local node. SN2 should not receive the fuzzed message.

    SN1 & SN2 <> LN1 & LN2
    LN1 -> SN1: PushMessage with ContactInfo data
    SN1 fuzzes the Vote data
    SN1 -> LN2: PushMessage with fuzzed ContactInfo data

    Assert: SN2 never receives the fuzzed message.

### GOSSIP JOIN-CLUSTER R02-07-t3

    Goal: The test ensures that nodes will reject push messages containing a fuzzed LowestSlot data.
    Note: A different variant of R02-07-t1

    Two SNs and two local nodes are started. When SN1 receives a LowestSlot push message from any local node,
    it fuzzes it (changes some of the data) and sends it to the other local node. SN2 should not receive the fuzzed message.

    SN1 & SN2 <> LN1 & LN2
    LN1 -> SN1: PushMessage with LowestSlot data
    SN1 fuzzes the Vote data
    SN1 -> LN2: PushMessage with fuzzed LowestSlot data

    Assert: SN2 never receives the fuzzed message.

### GOSSIP JOIN-CLUSTER R02-07-t4

    Goal: The test ensures that nodes will reject push messages containing a fuzzed EpochSlots data.
    Note: A different variant of R02-07-t1

    Two SNs and two local nodes are started. When SN1 receives a EpochSlots push message from any local node,
    it fuzzes it (changes some of the data) and sends it to the other local node. SN2 should not receive the fuzzed message.

    SN1 & SN2 <> LN1 & LN2
    LN1 -> SN1: PushMessage with EpochSlots data
    SN1 fuzzes the Vote data
    SN1 -> LN2: PushMessage with fuzzed EpochSlots data

    Assert: SN2 never receives the fuzzed message.

### GOSSIP JOIN-CLUSTER R02-07-t5

    Goal: The test ensures that nodes will reject push messages containing a fuzzed NodeInstance data.
    Note: A different variant of R02-07-t1

    Two SNs and two local nodes are started. When SN1 receives a NodeInstance push message from any local node,
    it fuzzes it (changes some of the data) and sends it to the other local node. SN2 should not receive the fuzzed message.

    SN1 & SN2 <> LN1 & LN2
    LN1 -> SN1: PushMessage with NodeInstance data
    SN1 fuzzes the Vote data
    SN1 -> LN2: PushMessage with fuzzed NodeInstance data

    Assert: SN2 never receives the fuzzed message.

### GOSSIP JOIN-CLUSTER R02-07-t6

    Goal: The test ensures that nodes will reject push messages containing a fuzzed SnapshotHashes data.
    Note: A different variant of R02-07-t1

    Two SNs and two local nodes are started. When SN1 receives a SnapshotHashes push message from any local node,
    it fuzzes it (changes some of the data) and sends it to the other local node. SN2 should not receive the fuzzed message.

    SN1 & SN2 <> LN1 & LN2
    LN1 -> SN1: PushMessage with SnapshotHashes data
    SN1 fuzzes the Vote data
    SN1 -> LN2: PushMessage with fuzzed SnapshotHashes data

    Assert: SN2 never receives the fuzzed message.

### GOSSIP JOIN-CLUSTER R02-08

    Goal: The test checks how the node will react in case of receiving some random bytes from newly connected synthetic node.

    A synthetic node joins the cluster and after initializing connection sends some random bytes to the local node.
    Local node ignores the message, but retains the connection and sends push messages and pull requests to the synthetic node.

    SN <> LN
    SN -> LN: random bytes
    LN: ignores the message and retains connection

    Assert: LN retains the connection with SN after receiving some random bytes.
    Improvement: LN should drop the connection with SN.

### GOSSIP JOIN-CLUSTER R02-09

    Goal: The test checks how the node will react in case of receiving some random bytes in a pull response from synthetic node.

    A synthetic node joins the cluster and whenever it receives a pull request it responds with random bytes. 
    Local node ignores the pull response, but retains the connection and sends pull requests to the synthetic node.

    SN <> LN
    LN -> SN: pull request
    SN -> LN: random bytes
    LN: ignores the pull response and retains connection

    Assert: LN retains the connection with SN after receiving some random bytes.
    Improvement: LN should drop the connection with SN.

### GOSSIP BLOOM FILTERS R03-01

    Goal: The test ensures that nodes will ignore pull requests with invalid bloom filters.

    A synthetic node joins the cluster and after initializing the connection it sends a pull request with invalid bloom filter. 
    Local node ignores the request and doesn't respond back.

    SN <> LN
    SN -> LN: pull request with invalid bloom filter
    LN: ignores the pull request and doesn't respond back

    Assert: LN ignores the pull request and doesn't respond back with pull response.

### GOSSIP PRUNE R04-01

    Goal: The goal of this test is to check whether the node who signs the prune message also has to be the node that sends that message.
    Note: This test is a simple variation of the test C04-02.

    SN1 joins the cluster of four LNs and gathers push messages from all of them.
    After SN1 receives all push messages from all LNs coming from all origins (e.g. messages sent by LN1 originating
    from nodes LN1, LN2, LN3 and LN4, and so on) - then SN1 decides to send a prune messages to all of the nodes:
    e.g. LN1 will be pruned for messages originating from nodes LN2, LN3 and LN4.
    But:
    Prune messages signed by SN1 are sent from a completely new SN2 that never joined the cluster.
    After sending the prunes from SN2, the original node SN1 should only receive
    push messages coming directly from the origins (e.g. message from LN1 originating from LN1).

    SN1 <> LN1 & LN2 & LN3 & LN4
    LNs -> SN1: PushMessage
    SN2 -> LNs: PruneMessage(crafted and signed by SN1) to all nodes
    LNs -> SN1: PushMessage - only messages coming directly from origins

    Assert: Expect pruned nodes stop sending push messages to SN1.
    Improvement: Don't let LNs accept prune message coming from another node (SN2) outside the cluster.

### GOSSIP PRUNE R04-02

    Goal: Check whether the node will reject prune messages signed and sent by the wrong node (malicious node).
    Note: This test is a simple variation of the test C04-02.

    SN1 and SN2 joins the cluster of four LNs.
    SN1 gathers push messages from all of LNs.
    After SN1 receives all push messages from all LNs coming from all origins (e.g. messages sent by LN1 originating
    from nodes LN1, LN2, LN3 and LN4, and so on) - then SN1 decides to send a prune messages to all of the nodes:
    e.g. LN1 will be pruned for messages originating from nodes LN2, LN3 and LN4.
    But:
    Prune messages crafted for SN1 are signed by SN2 sent from SN2.
    After sending malicious prunes from SN2, the original node SN1 should continue to receive push messages coming from all origins.

    SN1 <> LN1 & LN2 & LN3 & LN4
    SN2 <> LN1 & LN2 & LN3 & LN4
    LNs -> SN1: PushMessage
    SN2 -> LNs: PruneMessage(crafted for SN1, but signed by SN2) to all nodes
    LNs -> SN1: PushMessage - messages coming from all origins (above pruning failed)

    Assert: Expect that malicious prune messages sent from SN2 node will have no effect.

### GOSSIP PRUNE R04-03

    Goal: Check behavior of the nodes in the cluster after ignoring incoming prune messages.

    SN joins the cluster of four LNs.
    SN awaits for push messages and for each received push message,
    SN updates the origin and resends the push message back to LN that sent the push message.
    Eventually, SN will get pruned by all LNs, but then SN will continue to spam those LNs with push messages.

    SN <> LN
    LNx -> SN: PushMessage from node x (1, 2, 3, ...)
    SN -> LNx: PushMessage to the node that sent the message
    ...
    LNx -> SN: PruneMessage from all nodes
    LNx -> SN: PushMessage from all nodes
    SN -> LNx: PushMessage to all nodes, even the nodes that sent us the prune message

    Assert: Expect that the SN will always receive push messages no matter whether our node received the prune message or not.
    Improvement: Expect that LNs will start to ignore SN after the SN ignores their prune messages.

### GOSSIP PRUNE R04-04

    Goal: The test ensures the node will not stop sending specified push messages after it receives a prune message with an invalid signature.

    SN joins the cluster of four LNs and gathers push messages from all of them.
    After SN receives all push messages from all LNs coming from all origins, SN sends
    prune messages with invalid signature to all of the nodes.
    After sending such prunes, SN should continue to receive push messages from all origins.

    SN <> LN1 & LN2 & LN3 & LN4

    Wait until SN receives all of these:
        LN1 -> SN: PushMessage with CRDS values originating from all LNs
        LN2 -> SN: PushMessage with CRDS values originating from all LNs
        LN3 -> SN: PushMessage with CRDS values originating from all LNs
        LN4 -> SN: PushMessage with CRDS values originating from all LNs

    Then:
        SN -> LN1: PruneMessage with the invalid signature for origins LN2, LN3, LN4
        SN -> LN2: PruneMessage with the invalid signature for origins LN1, LN3, LN4
        SN -> LN3: PruneMessage with the invalid signature for origins LN1, LN2, LN4
        SN -> LN4: PruneMessage with the invalid signature for origins LN1, LN2, LN3

    Then expect that the above action had no effect:
        LN1 -> SN: PushMessage with CRDS values originating from all LNs
        LN2 -> SN: PushMessage with CRDS values originating from all LNs
        LN3 -> SN: PushMessage with CRDS values originating from all LNs
        LN4 -> SN: PushMessage with CRDS values originating from all LNs

    Assert: Expect that all LNs will continue sending push messages with CRDS values originating from other LNs to SN after receiving prune messages with an invalid signature.

### GOSSIP PRUNE R04-05

    Goal: The test ensures the node will not stop sending specified push messages after it receives a prune message with an invalid wallclock.

    SN joins the cluster of four LNs and gathers push messages from all of them.
    After SN receives all push messages from all LNs coming from all origins - then SN sends
    prune messages with the invalid wallclock to all of the nodes.
    After sending such prunes, SN should continue to receive push messages from all origins.

    SN <> LN1 & LN2 & LN3 & LN4

    Wait until SN receives all of these:
        LN1 -> SN: PushMessage with CRDS values originating from all LNs
        LN2 -> SN: PushMessage with CRDS values originating from all LNs
        LN3 -> SN: PushMessage with CRDS values originating from all LNs
        LN4 -> SN: PushMessage with CRDS values originating from all LNs

    Then:
        SN -> LN1: PruneMessage with the invalid wallclock for origins LN2, LN3, LN4
        SN -> LN2: PruneMessage with the invalid wallclock for origins LN1, LN3, LN4
        SN -> LN3: PruneMessage with the invalid wallclock for origins LN1, LN2, LN4
        SN -> LN4: PruneMessage with the invalid wallclock for origins LN1, LN2, LN3

    Then expect that the above action had no effect:
        LN1 -> SN: PushMessage with CRDS values originating from all LNs
        LN2 -> SN: PushMessage with CRDS values originating from all LNs
        LN3 -> SN: PushMessage with CRDS values originating from all LNs
        LN4 -> SN: PushMessage with CRDS values originating from all LNs

    Assert: Expect that all LNs will continue sending push messages with CRDS values originating from other LNs to SN.

### GOSSIP PRUNE R04-06

    Goal: The test ensures the node will not stop sending specified push messages after it receives a prune message with an invalid destination.

    SN joins the cluster of four LNs and gathers push messages from all of them.
    After SN receives all push messages from all LNs coming from all origins - then SN sends
    prune messages with the invalid destination to all of the nodes.
    After sending such prunes, SN should continue to receive push messages from all origins.

    SN <> LN1 & LN2 & LN3 & LN4

    Wait until SN receives all of these:
        LN1 -> SN: PushMessage with CRDS values originating from all LNs
        LN2 -> SN: PushMessage with CRDS values originating from all LNs
        LN3 -> SN: PushMessage with CRDS values originating from all LNs
        LN4 -> SN: PushMessage with CRDS values originating from all LNs

    Then:
        SN -> LN1: PruneMessage with the invalid destination (not LN1) for origins LN2, LN3, LN4
        SN -> LN2: PruneMessage with the invalid destination (not LN2) for origins LN1, LN3, LN4
        SN -> LN3: PruneMessage with the invalid destination (not LN3) for origins LN1, LN2, LN4
        SN -> LN4: PruneMessage with the invalid destination (not LN4) for origins LN1, LN2, LN3

    Then expect that the above action had no effect:
        LN1 -> SN: PushMessage with CRDS values originating from all LNs
        LN2 -> SN: PushMessage with CRDS values originating from all LNs
        LN3 -> SN: PushMessage with CRDS values originating from all LNs
        LN4 -> SN: PushMessage with CRDS values originating from all LNs

    Assert: Expect that all LNs will continue sending push messages with CRDS values originating from other LNs to SN.

[spec]: https://github.com/eigerco/solana-spec/blob/main/gossip-protocol-spec.md
