# Paxos

Paxos is a consensus algorithm which is used to agree on a value in a distributed system. Consider a scenario where there are 3 servers on a network and clients connect to these servers. The only API is `SetVarX()` and `GetVarX()`. `SetVarX()` is used to set a value for X and `GetVarX()` is used to get the value of X.

Here are some conditions that must be satisfied:
1. Reading X from any server results in same value.
1. If a value is set, all servers must set the same value.

This should be satisfied even when different clients connect to different servers, servers and clients can crash/restart at any time etc, the connectivity between clients and servers can be lost arbitrarily.

Initially, the 1st condition is trivially satisfied. Since there's no value set for X, reading from any server will return an empty value. Then the clients try to set a value, say client A sets A, client B sets B and client C sets C on 3 different servers. Atmost one of this is successful and eventually the set value must be either empty (all failed) or one of A, B or C. This is the consensus problem.

## Consensus in the context of replicated log

Replicated state machines receive same commands in the same order under following conditions

* Majority - System makes process as long as majority of servers are up (respond to messages from each other)
* Failure Model
    * Fail stop (non byzantine) - Servers fail by failing to receive messages and responding to messages, examples - crashes, disconnect from network etc.
    * Delayed/lost/out of order messages - Message can take arbitrarily long to be delivered, 

## Paxos - Basic and Multiple

* **Basic Paxos** - Basic paxos algorithm is a single decree algorithm that allows agreement on single value. This is the example from the introduction. This is an extremely important result from distributed systems theory perspective, it has been discussed a lot in the literature and formally proved. But it is not a very useful practically. For example, it doesn't specify a behavior when the set of servers need to be changed (say from 3 to 5.For a practical system, we are interested in agreement on a sequence of values. Then this can be used to record a consistent sequence of commands across multiple servers so that all servers end up in same state. This sequence of values is called a `Replicated Log`.

* **Multi Paxos** - Multi paxos is an extension of basic paxos that addresses practical concerns. This extension handles the needs for a practical replicated state machine like agreeing on sequence of values, efficiency, recovery, configuration changes etc.

## Basic Paxos

Objective is to agree on a single value with following properties -
(Agreed value is also called as "chosen")

### Safety
* Only one value is chosen.
* The chosen value must be one of the proposed values (This is not really a safety property but a non trivial property, this excludes always chosing an empty value)
* Once a value is chosen, it doesn't change.

### Liveness
* Some value from the proposed is chosen eventually. This ensures that the algorithm terminates.
* Eventually servers learn about the chosen value.

### Terminology
- Chosen - This is the value that is agreed upon. This is the algorithm's outcome.
- Accepted - This is a potential candidate for the value to be chosen. When a majority of servers have accepted on value, that is considered chosen. But it's possible that no one knows that it's the chosen value since this information is distributed. Some server has to collect accepted values from all servers and find if there's a majority. This is also similar to voting on a value.
- Proposer - A server that can propose values. These proposed values become candidates from which a value will be chosen.
- Acceptor - A server that decides to accept a value. These are voting nodes whose count matters in which value gets chosen. Note: there's no preference for accepting a specific value here. The voter's decision is to ensure that algorithm converges and eventually terminates and is key part of the distributed algorithm. This is important since each node acts independently, the only thing that they ensure is, they follow some `Rules` while making decision and the objective is if all nodes follow the rules, then the cluster of nodes will end up with an agreement.
- Learners - A client (or server) that doesn't participate in the decision making process but is interested in the decision.