---
sidebar_position: 1
---

# What is an EarthNode?

The **EarthNodes** are defined as the containers of the core logic of the World Mobile Chain system. They function as the brain of the system that interconnects all the other types of nodes, and are composed of a number of software modules, communicating through the central module called The Internode API. The rest of the modules provide an authentication layer (Decentralised Identity (DID) module), a ledger layer (blockchain module), and a telecommunications layer (telecom module).

The DID module provides the interface to a digital ID solution. In the blockchain module, a distributed ledger records all the transactions that take place on the network. For economical and performance, but also for privacy purposes, some transaction data is segregated into the public ledger component, which is anonymised and links to the private component, that contains the full data of the transaction in an encrypted distributed ledger. The settlement of these transactions on the public ledger is done either in batches or as a single transaction depending on its priority, while the private ledger is stored in real-time.

The telecom module handles the communications functions of the network. EarthNodes can operate from anywhere in the world, however, the routing of traffic within the network has a weighting towards closer nodes to improve performance and quality of services. The EarthNode operator may select to run different elements of the EarthNode depending on the specification of their hardware.

The **Internode API** is responsible for the communication between all nodes and subsystems throughout the network. The API acts as the bridge and translator between the DID module, the telecom module and the blockchain. It also provides the communication between the EarthNodes, Air Nodes and Aether Nodes as well as interfacing with third party applications. The primary responsibilities of this module are:

- Processing user registration and authentication requests
- Financial transactions - including balance checking and making payments
- Handling service requests - requests for access to services
- Handling telecommunications events (call attempts, call started, call ended, sms sent, Internet data connection, etc.) and storing them into the blockchain
- Handling and storing Quality of Service (QoS) and other quality measurements in the blockchain
- Simplifying the complexity of the business logic to the rest of the system, calling the appropriate smart contracts and performing all the actions necessary for each one of the use cases
- Enforcing that rules and contracts set in the blockchain are followed by the telecom module

The **DID Module** is responsible for interfacing with the decentralised digital identity solution for creation and maintenance of user digital identities.The key responsibilities of this module are:

- Identity registration
- Credential management
- Authentication

The **Blockchain Module** provides security, immutability, transparency and privacy. Key responsibilities of the blockchain:

- Financial ledger - user account balances and record of transactions
- Reward mechanism - processing of rewards for the nodes to ensure automated payments once conditions of smart contracts are met
- Distributed secure storage - of user data and metadata through public blockchain and private data vault (e.g. user aliases and telephone numbers, KYC, call logs, etc.)

The **Telecommunications Module** is a key module within the overall architecture, responsible for the following processes:

- Call signalling - signalling layer of the calls enabling call setup and teardown
- Media routing - media layer of the calls for voice and video communication
- Message routing - messaging layer for transmission of P2P and SMS messaging
- Service management - processing service requests
- QoS monitoring - analysis and real time recording of the network quality including Mean Opinion Score, jitter, packet loss, etc.
- Self healing network - analysis and algorithms for operating network and updating of distributed routing tables
- Distributed hash tables for nodes - node address tables required for routing
