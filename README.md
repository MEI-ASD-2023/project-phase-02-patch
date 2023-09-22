# Project Phase 1

## Organisational details

- Groups of **three**
- **Deadline: 23:59:59 2023-10-18 (Quarta-feira)**
- Cluster access will be provided in due course for trialling your system: **you must pre-register your groups by sending an e-mail to the course professors ([nmp@fct.unl.pt](mailto:nmp@fct.unl.pt) and [a.davidson@fct.unl.pt](mailto:a.davidson@fct.unl.pt)) with the subject "*ASD 23/24 GROUP REGISTRATION*"**.
  - You will get a reply from the professor (eventually) providing your usernames and passwords to access the cluster.
  - If you cannot form a group of three, smaller groups may be created, but this is highly discouraged due to the workload.
  - Anyone who is struggling to find a group to join should email the course professors **ASAP**.

## Overall goals

### Implementation

1. Develop a distributed system (in Java, using [Babel](https://github.com/pfouto/babel-core), source code [provided](./src)) that implements a *reliable* broadcast algorithm (Lecture 1 and Lecture 2) using a gossip/epidemic protocol (Lecture 2) for deciding which peers to communicate with.
2. Add the *"anti-entropy"* optimisation (Lecture 2), to reduce the number of redundant messages that are sent throughout the system.
3. Implement the *[HyParView](https://asc.di.fct.unl.pt/~jleitao/pdf/dsn07-leitao.pdf)* protocol for allowing your reliable broadcast algorithm to request peers to communicate with more efficiently.

### Report

Write a report, using the [latex template](./latex) provided, that details the following information.

1. The design of your system
2. The methodology that you used to design it
3. An analysis of how your system deals with transient failures of processes
4. A performance analysis of your system, relative to the original system as well as the impact of the anti-entropy and HyParView optimisations.

### Constraints

- Your distributed system must contain 100+ processes
- A static membership of nodes that are already known can be assumed.

## Submission details

The code and report that you submit should be pushed to this repository. You will then submit the **link to your repository** and the **commit ID** corresponding to the version of the project that you would like to have evaluated. Details for submission will be provided closer to the deadline.

## Additional context

### Architectural overview

### Basic implementation

In the basic implementation, there will be an application (example will be provided) that will simply generate messages, and log which messages have been delivered by the system. This protocol will communicate with the Reliable Broadcast algorithm that you will implement, which will communicate directly with the wider network via a Gossip protocol.

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

### Anti-entropy optimisation

Your anti-entropy module should communicate with the network before any message is broadcast, to locate a gossip neighbour which has not already received the broadcast message. This *should* reduce the number of messages in the system, without contradicting the guarantees of the reliable broadcast.

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
|     // Send message to peer (based on anti-entropy) |----->
|                                                     |
|                      // Receive messages from peers |<-----
|_____________________________________________________|
         |                                ^
         |                                |
    Request_peer()                    Receive(peer)
         |                                |
         v                                |
______________________________________________________
|                 Anti-entropy algorithm              |
|                                                     |
|                // Communicate with network to learn |---->
|                // a neighbour process that has not  |
|                // received the message m yet        |<----
|_____________________________________________________|
```

### HyParView optimisation

We discuss how HyParView works to provide a more optimal overlay network for propagating messages, [see below](#hyparview) for more details.

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

The javadoc of the framework can be found here: <https://asc.di.fct.unl.pt/~jleitao/babel/>. A more detailed description can be found in the slides provided in the [additional docs](./docs/babel-slides.pdf)

The framework was specifically designed thinking about two complementary goals: $i)$ quick design and implementation of efficient distributed protocols; and $ii)$ teaching distributed algorithms in advanced courses. A significant effort was made to make it such that the code maps closely to protocols descriptions using (modern) pseudo-code. The goal is that you can focus on the key aspects of the protocols, their operation, and their correctness, and that you can easily implement and execute such protocols.

While this is the fourth year that Babel is being used in this course, and the current version has also been used to develop several research prototypes, the framework itself is still considered a prototype, and naturally some bugs can be found. Any problems that you encounter can be raised with the course professors.


### HyParView

[HyParView](https://asc.di.fct.unl.pt/~jleitao/pdf/dsn07-leitao.pdf), developed in 2007, is a fault-tolerant overlay network. HyParView is based on two distinct partial views of the system that are maintained for different purposes and using different mechanisms. A small partial-view (active view) that is used to ensure communication and cooperation between nodes, that is managed using a reactive strategy, where the contents of these views are only changed in reaction to an external event, such as a node failing or joining the system (or indirect consequences of these events), these views rely on TCP as an unreliable fault detector. A second and larger view (passive view) is used for fault-tolerance, as a source of quick replacements on the active view when this view is not complete. The fact that HyParView maintains a highly stable active view (and hence the overlay denoted by these views is also stable) offers some possibility to improve the communication pattern of nodes disseminating information.

