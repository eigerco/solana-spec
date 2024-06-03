# Introduction

The purpose of this index is to provide an overview of the testing approaches for the gossip spec. It is intended to evolve as the framework matures, leaving room for novel cases and extensions of existing cases, as called for by any protocol idiosyncrasies that may come to light during the development process.

Some test cases have been consolidated when similar behaviour is tested with differing messages. The final implementation of these cases will be subject to factors such as node setup and teardown details, test run time (and potentially runtime) constraints, readability and maintainability.

> [!Tip]
> **Naming conventions used in this document**
> - _Node_ - a validator running the gossip
> - _Synthetic node_ - a lightweight gossip node that is used only for testing gossip protocol while interacting with the node via tests.
> - _Cluster_ - a network of validators with a leader that produces blocks

## Usage

The tests can be run with `cargo test` once the test suite setup is properly configured and dependencies (node instance to be tested) are satisfied. See the [README](README.md) for details.

Tests are grouped into the following categories: conformance, performance, and resistance. Each test is named after the category it belongs to, in addition to what's being tested. For example, `c_ping_001_expect_pong` is a conformance test that tests the `ping` message by expecting a `pong` response from the node.

The full naming convention is: `(test-group-id)_(message-type)_(subtest-no)_(extra-test-desc)`.

# Types of Tests

## Conformance

The conformance tests aim to verify the node adheres to the network protocol. In addition, they include some naive error cases with malicious and fuzzing cases consigned to the resistance tests. These tests also aim to evaluate the proper behaviour of a node when receiving unsolicited response messages.

## Performance

The performance tests aim to verify the node maintains a healthy throughput under pressure. This is principally done through simulating load with synthetic peers and evaluating the node's responsiveness. Synthetic peers will need to be able to simulate the behaviour of a full node by implementing handshaking, message sending and receiving.

These tests are intended to verify the node remains healthy under "reasonable load". Additionally these tests will be pushed to the extreme for resistance testing with heavier loads.

## Resistance

The resistance tests are designed for the early detection and avoidance of weaknesses exploitable through malicious behaviour. They attempt to probe boundary conditions with comprehensive fuzz testing and extreme load testing. The nature of the peers in these cases will depend on how accurately they needs to simulate node behaviour. It will likely be a mixture of simple sockets for the simple cases and peers used in the performance tests for the more advanced.

# Test Index

The test index makes use of symbolic language in describing connection and message sending directions.

| Symbol | Meaning |
|--------------|-----------|
| `-> A` | A synthetic node sends a message `A` to Solana node |
| `<- B` | Solana node sends a message `B` to a synthetic node |
| `>> C` | A synthetic node broadcasts a message `C` to set of nodes in the cluster |
| `<< D` | Solana node broadcasts a message `D` to set of nodes in the cluster |
| `<- [no-reply]` | Solana node doesn't answer with a reply |
| `<>` | The synthetic node joined the cluster |
| `LNx` | Local node `x` |
| `SNx` | Synthetic node `x` |


## Conformance

### GOSSIP-CONFORMANCE-001

    The node correctly replies with a pong message to a ping request.

    -> ping (token)
    <- pong (correct token hash)

    Assert: the received token hash is correct.

### GOSSIP-CONFORMANCE-002

    The node correctly successfully exchanges ping/pong messages.

    <- ping (token-1)
    -> pong (correct token-1 hash)
    -> ping (token-2)
    <- pong (correct token-2 hash)

    Assert: the received token hash is correct.

### GOSSIP-CONFORMANCE-003

    The node correctly rejects ping request after receiving an invalid hash in the prior pong response.

    <> the node joins our cluster
    <- ...
    <- ping (token-1)
    -> pong (invalid token-1 hash)
    -> ping (token-2)
    <- [no-reply]

    Assert: the node’s ignores the ping message.

### GOSSIP-CONFORMANCE-004

    The node correctly rejects ping request after receiving an unsolicited prior pong message.

    -> pong (unsolicited token-1 hash)
    -> ping (token-2)
    <- [no-reply]

    Assert: the node’s ignores the ping message.

### GOSSIP-CONFORMANCE-005

    The synthetic node joins the cluster.

    -> PushMessage[NodeInstance]
    -> PushMessage[Version]
    -> Ping (token1)
    -> PullRequest[LegacyContactInfo]

    Expect messages:
    <- Pong (token1 hash)
    <- Ping (token2)
    <- PullRequest[...]
    <- PushMessage[...]

    Assert: all listed messages should be received.

### GOSSIP-CONFORMANCE-006

    The synthetic node represents the cluster and the node uses synthetic node's address as an entrypoint to join the cluster.

    <- PushMessage[NodeInstance]
    <- PushMessage[Version]
    <- Ping (token1)
    <- PullRequest[LegacyContactInfo]
    <- ...

    Assert: all listed messages should be received from the node that is joining our cluster.

### GOSSIP-CONFORMANCE-007

    Two synthetic node joins the cluster of two nodes.
    Each synthetic node joins the cluster using the separate node as an entrypoint node.

    <> LN1 with SN1
    <> LN2 with SN2
    SN2: LN2<- PushMessage[LegacyContactInfo(SN1)]

    Assert: The legacy contact info belonging to one synthetic node is shared with the second synthetic node through the cluster.

### GOSSIP-CONFORMANCE-example-for-push-message

    Start a local cluster of Solana nodes and test the push message mechanism in regards to size of the active set.

    Setup:
    - start a local node cluster of three nodes
    - start 10 synthetic nodes which join the cluster

    << push message

    Assert: At no point, no more than 9 nodes will receive a push message from the same node.
