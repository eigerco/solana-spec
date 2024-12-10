# Solana Gossip Protocol

This repository contains documentation and implementation details related to the Gossip Protocol used by Solana nodes for inter-node communication.

## Overview

Solana nodes communicate with each other using a custom gossip protocol to maintain data consistency across the network.
This protocol allows nodes to share information, such as the current state of their replicated data stores, by sending various types of messages.
The protocol plays a crucial role in ensuring that all nodes in the cluster stay synchronized.

## Repository Contents

- [gossip-protocol-spec.md](./gossip-protocol-spec.md): This document provides a detailed specification of the gossip protocol, including the different types of messages that Solana nodes exchange.

- [implementation-details.md](./implementation-details.md): This document outlines the specifics of how the gossip protocol is implemented in Solana, including the modes in which nodes can operate and the networking details of gossip communication.

- [spec-test-suite.md](./spec-test-suite.md): This is a copy of a test suite document from another private repository. The repository contains implementation for all listed tests and will eventually be made public.

## Contributing

Contributions to this repository are welcome. If you find any issues or have suggestions for improvement, please feel free to submit a pull request or open an issue.

## About [Eiger](https://www.eiger.co)

We are engineers. We contribute to various ecosystems by building low level implementations and core components. We are working on several Solana related projects and we are happy to contribute to the ecosystem.

Contact us at hello@eiger.co
Follow us on [X/Twitter](https://x.com/eiger_co)
