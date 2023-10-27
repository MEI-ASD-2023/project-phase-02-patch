# Project Phase 2 <!-- omit in toc -->

## Project generation link <!-- omit in toc -->

Use the following link to generate this project repository for your group

**<https://classroom.github.com/a/QSCr0f1Y>**

## Table of contents <!-- omit in toc -->

- [Organisational details](#organisational-details)
- [Overall goals](#overall-goals)
  - [Implementation](#implementation)
  - [Report](#report)
- [Architectural overview](#architectural-overview)
  - [State Machine Replication](#state-machine-replication)
  - [Agreement Protocol](#agreement-protocol)
  - [Application](#application)
  - [Clients (YCSB)](#clients-ycsb)
- [Evaluation Criteria](#evaluation-criteria)
- [Submission details](#submission-details)
- [Additional context](#additional-context)
  - [Architectural overview](#architectural-overview-1)
    - [Basic implementation](#basic-implementation)
    - [Anti-entropy optimisation](#anti-entropy-optimisation)
    - [HyParView optimisation](#hyparview-optimisation)
  - [Programming environment (Babel)](#programming-environment-babel)
  - [HyParView](#hyparview)
  - [Cluster](#cluster)

## Organisational details

- **Same groups** as defined in phase 1 of the project.
- **Deadline: 23:59:59 2023-11-29 (Quarta-feira)**
- Cluster access is provided for trialling your system.

## Overall goals

### Implementation

The project focuses on the study and implementation of a distributed system for managing the state of a Hash map. Your system should implement **both** [Paxos](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)- **and** [Multi-Paxos](https://dada.cs.washington.edu/research/tr/2009/09/UW-CSE-09-09-02.PDF)-based state machine replication (SMR) protocols. To this end, the goals/steps of this first phase of the project are defined as follows:

1. Implement Paxos-based state machine replication.
2. Implement MultiPaxos-based state machine replication.
3. Develop a distributed system (in Java, using [Babel](https://github.com/pfouto/babel-core), source code [provided](./src)) that implements a distributed Hash Map interface, that is compatible with both SMR protocols.
4. Conduct experimental analysis using 3 nodes, and using [YCSB-based](https://github.com/brianfrankcooper/YCSB) clients to compare the system using both SMR protocols, comparing throughput and latency, as observed by clients.

The base source code for your project should be developed in the `src/asd-project2-base` folder provided in the base [source code](./src/asd-project2-base/) folder.

### Report

Write a report, using the [latex template](./latex) provided, that details the following information.

1. The design and implementation of your system
2. A performance analysis of your system, comparing both SMR approaches.

The report must have a maximum of 8 pages, including figures and tables, but excluding bibliography.

## Architectural overview

![](docs/system-design.png)

The diagram above details the mandatory structure for your distributed system, including the **interfaces** that must be handled. The Java project contained in the `src/` folder already provides the Babel events that are used to instantiate these interfaces.

Every protocol that you will implement should have an equivalent interface to communicate with the other local protocols (this is instantiated by Requests and Indications). This should allow you to swap one agreement protocol implementation for another while your process still executes correctly. Notice that the messages exchanged by each protocol that you develop with the (equivalent) protocol on other processes is not fixed. You should model these messages following the specification of the protocol that you are implementing. Notice that if you careful in your implementation the State Machine Replication protocol could be fully agnostic to the underlying agreement protocol (either Paxos or Multi-Paxos).

### State Machine Replication

This protocol is responsible for receiving the (client) operations from the application, and replicate them through the agreement protocol (i.e., Paxos or Multi-Paxos). It is the state machine that keeps track of which operation has been decided for each position of the sequence of commands of the state machine and notifies the application to execute these operations (in the appropriate order).

The protocol is also responsible for managing the communication channel (i.e., open TCP connections), manage the membership of the system, which includes requesting to be added to the replica set when a new replica joins the system, and inform the agreement protocol of modifications in the membership. This implies that the state machine replication protocol is responsible also for detecting failures of replicas, attempting to re-establish connections with them a (configurable) number of times, and when suspecting that a replica having failed, issue an operation for the state machine to remove that replica.

When joining a replica set, it is necessary to copy the current state of the system when the new replica is added. This process, usually named *state transfer* is handled by the state machine replication protocol, that can request a copy of the current application state to the application, and upon receiving it, can send that state (and the information concerning the current position of the sequence of commands in which the replica was added) to the newly joining replica. To avoid all existing replicas to do this, one way you can address this is by having the replica that received the request of the new replica to be added to the system replying to the new replica with this information (notice that the State Machine Replication protocol needs to exchange messages through the network with other processes).

Your implementation of this protocol should be, as much as possible, independent of the underlying agreement protocol, although you might need to pass an additional parameter to the state machine to inform of the protocol being used for agreement. This happens because when using Multi-Paxos, there is a notion of a leader (that the state machine needs to be notified about by the Multi-Paxos protocol), whereas when using Paxos this notion does not exist. In Multi-Paxos, since only the leader proposes commands to be decided, other replicas that receive requests from clients must send these requests to the current leader, again at the state machine replication protocol level. This implies that when the leader changes, requests received by clients that have not been ordered yet by the agreement protocol, must be resubmitted to the new leader. When using Paxos, whenever a new instance of the protocol is initialized (i.e., for each of the positions of the sequence of operations managed by the state machine) it might be required that the state machine protocol provides to the protocol the membership of the system (whereas in Paxos, a copy of the membership of the system can be maintained and managed at the level of the Multi-Paxos protocol).

### Agreement Protocol

You will implement the two different variants of agreement protocol in this project: Paxos and Multi-Paxos. In your implementation avoid optimisations such as running multiple instances at the same time (as this requires a significant engineering effort to implement correctly).

Paxos does not have a leader, and every process will propose commands to be decided across the different instances. You can decide what to do when a Paxos protocol receives a message for an instance that it has not yet started (i.e., where it has not received a proposal): you can reply following the protocol specification, or you can store the message locally and wait for the local state machine protocol to propose a value for that instance, and only after that process the message. Notice that you need to be aware of membership changes to be able to compute how many processes are a majority. The best suggestion for implementing Paxos, since you will need multiple instances, is to store the state of the Paxos algorithm within a Java class, and store multiple instances of this class in a map, where the key is the instance number. As such when you receive a message for a particular instance, you can fetch the current state of that instance in your map, and process the message by only modifying the state of Paxos for that instance. All Paxos messages have to be tagged with the instance for which they refer (instances can be a monotonic integer).

Multi-Paxos implementations should follow the specification presented in the Lectures. Notice however that in this case you will be required to answer to instances without having received a local initial proposal. This is because when using Multi-Paxos, only the leader proposes commands. In this protocol you also have to be careful to track the membership of the system, since you must know which replicas are part of the system in each instance to be able to compute the majority.

Notice that in both protocols, a process should not participate in instances that occur when the replica is not part of the system (i.e., before being added to the system or after being removed from the system).

### Application

The application layer is going to be provided to you. It will have the code necessary to support all interactions to clients and it will use the prescribed interface to interact with the state machine replication layer.

The application can also expose its internal state (in a serialized form) and install a copy of state gathered from another replica. This is relevant to allow a new replica of the application to be added to the system while the system is operating.

### Clients (YCSB)

To inject load in the system (i.e., execute operations) we are going to use the popular YCSB system. We will provide a driver that allows for YCSB to interact with the replicated application described above. Note that YCSB might send operations to any of the active replicas. Also YCSB executed multiple ``client threads'' where each one emulated an individual client. This is important because in the experiments we will want to study the differences between the latency (measured in milli-seconds) and throughput (measured in operations per second) as the total number of clients in the system increases. YCSB already computes and outputs both of these metrics for you.

You might need to run more than one instance of YCSB such that you can distribute the load of clients across multiple machines (a single physical machine has limits on the number of client threads it can execute concurrently without the clients becoming bottlenecked). The evaluation should saturate the servers and not the clients, as that would yield incorrect experimental results. If you rely on multiple instances of YCSB to run your experiments, you will need to (manually) combine the results outputted by each instance. To that end you should compute the average of the latency observed on each instance, and you should compute the sum of the throughput reported by each YCSB instance.

## Evaluation Criteria

The project delivery includes both the code and a written report that should have the format of a short paper. The report must contain clear and readable pseudo-code for each of the implemented protocols, alongside a description of the intuition of these protocols. A correctness argument for protocol that was devised or adapted by students will be positively considered in grading the project. The written report should also provide information about all experimental work conducted by the students to evaluate their solution in practice (i.e., description of experiments, setup, parameters) as well as the results and a discussion of those results.

- The project will be evaluated by the correctness of the implemented solutions, its efficiency, and the quality of the implementations (in terms of code readability).
- The quality and clearness of the report of the project will have an impact the final grade. Students with a poorly written report run the risk of penalisation, based on the evaluation of the correctness the solutions employed.

This phase of the project will be graded in a scale from 1 to 20 with the following considerations:

- Groups that only implement a Reliable Broadcast algorithm (using an epidemic/Gossip model) and that experimentally evaluate those protocols with a single set of experimental parameters, will at most have a grade of $12$.
- Groups that implement both optimisations (Anti-Entropy and HyParView) and experimentally evaluate them can get up to $4$ additional points.
- Groups that consider *dynamic peer memberships*, and additionally conduct experimental evaluation of all implementations and optimisations using a combination of two different payload sizes for content stored and retrieved from the system and two different rates of requests issued by their test application, can get $4$ additional points.

## Submission details

The code and report that you submit should be pushed to this repository. You will then submit the **link to your repository** and the **commit ID** corresponding to the version of the project that you would like to have evaluated. Details for submission will be provided closer to the deadline.

## Additional context

### Architectural overview

#### Basic implementation

In the basic implementation, there will be an application (see the base code in the [src](./src/asd-project1-base/)) that will simply generate messages, and log which messages have been delivered by the system. This protocol will communicate with the Reliable Broadcast algorithm that you will implement, which will communicate directly with the wider network via a Gossip protocol.

The base code floods the entire system with messages, which is inefficient. You will first have to modify it to select a subset (`t`) of other processes to interact with.

```
______________________________________________________
|                                                     |
|                                                     |
|                     Application                     |
|                                                     |
|_____________________________________________________|
          |                               ^
          |                               |
      broadcast(m)                     deliver(m)
          |                               |
          v                               |
______________________________________________________
|                  Reliable Broadcast                 |
|                                                     |
|               // Send messages to gossip neighbours |----->
|                                                     |
|          // Receive messages from gossip neighbours |<-----
|_____________________________________________________|
```

#### Anti-entropy optimisation

Your anti-entropy module should communicate with the network before any message is send to other node, to know which message the remote peer is missing.
This *should* reduce the number of messages in the system, without contradicting the guarantees of the reliable broadcast.

```
______________________________________________________
|                                                     |
|                                                     |
|                     Application                     |
|                                                     |
|_____________________________________________________|
          |                               ^
          |                               |
      broadcast(m)                     deliver(m)
          |                               |
          v                               |
______________________________________________________        
|      Reliable Broadcast with Anti-entropy           |
|                                                     |
|    // Send messages to peer (based on anti-entropy) |----->
|                                                     |
|                      // Receive messages from peers |<-----
|_____________________________________________________|
         |                                ^
         |                                |
    Request_peer()                    Receive(peer)
         |                                |
         v                                |
______________________________________________________
|                 Membership algorithm                |
|                                                     |
|                // Communicate with network to learn |---->
|                // a neighbour process               |
|                                                     |<----
|_____________________________________________________|
```

#### HyParView optimisation

We discuss [below](#hyparview) how HyParView works to provide a more optimal overlay network for propagating messages. The advantage of using HyParView is that the membership algorithm selects peers to ensure a more efficient propagation of messages.

```
______________________________________________________
|                                                     |
|                                                     |
|                     Application                     |
|                                                     |
|_____________________________________________________|
          |                               ^
          |                               |
      broadcast(m)                     deliver(m)
          |                               |
          v                               |
______________________________________________________
|     Reliable Broadcast with Anti-Entropy            |
|                                                     |
|    // Send messages to peers learned from HyParView |----->
|                                                     |
|                      // Receive messages from peers |<-----
|_____________________________________________________|
         |                                ^
         |                                |
  Request_peers()                    Receive(peers)
         |                                |
         v                                |
______________________________________________________
|                                                     |
|                                                     |
|                     HyParView                       |
|                                                     |
|_____________________________________________________|
```

### Programming environment (Babel)

The students will develop their project using the Java language (version 11 minimum). Development will be conducted using a framework developed in the context of the NOVA LINCS laboratory written by Pedro Fouto, Pedro Ákos Costa, João Leitão, known as **Babel**.

The framework uses to the [Netty framework](https://netty.io) to support inter-process communication through sockets (although it was designed to hide this from the programmer). The framework will be discussed in the labs. An example application will be provided that is responsible for generating and logging messages in the system.

The javadoc of the framework can be found here: <https://asc.di.fct.unl.pt/~jleitao/babel/>. A more detailed description can be found in the slides provided in the [additional docs](./docs/babel-slides.pdf). Example code for running a broadcast algorithm is provided in the base [source code](./src/asd-project1-base/) folder.

The framework was specifically designed thinking about two complementary goals: $i)$ quick design and implementation of efficient distributed protocols; and $ii)$ teaching distributed algorithms in advanced courses. A significant effort was made to make it such that the code maps closely to protocols descriptions using (modern) pseudo-code. The goal is that you can focus on the key aspects of the protocols, their operation, and their correctness, and that you can easily implement and execute such protocols.

While this is the fourth year that Babel is being used in this course, and the current version has also been used to develop several research prototypes, the framework itself is still considered a prototype, and naturally some bugs can be found. Any problems that you encounter can be raised with the course professors.

### HyParView

[HyParView](https://asc.di.fct.unl.pt/~jleitao/pdf/dsn07-leitao.pdf), developed in 2007, is a fault-tolerant overlay network. HyParView is based on two distinct partial views of the system that are maintained for different purposes and using different mechanisms. A small partial-view (active view) that is used to ensure communication and cooperation between nodes, that is managed using a reactive strategy, where the contents of these views are only changed in reaction to an external event, such as a node failing or joining the system (or indirect consequences of these events). These views rely on TCP as an unreliable fault detector. A second and larger view (passive view) is used for fault-tolerance, as a source of quick replacements on the active view when this view is not complete. The fact that HyParView maintains a highly stable active view (and hence the overlay denoted by these views is also stable) offers some possibility to improve the communication pattern of nodes disseminating information.

### Cluster

The technical specification of the cluster as well as the documentation on how to use it (including changing your group password and making reservations) is online at: <https://cluster.di.fct.unl.pt>. You should read the documentation carefully. Once you have received your credentials you should be able to use it freely.
