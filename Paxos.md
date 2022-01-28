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
- Proposer - A server that can propose values. These proposed values become candidates from which a value will be chosen. These are the **active** servers in the system that drive the consensus by communicating with other nodes.
- Acceptor - A server that decides to accept a value. These are voting nodes whose count matters in which value gets chosen. These nodes are **passive** in the sense that their action is based only on local knowledge.
Note: there's no preference for accepting a specific value here. The voter's decision is to ensure that algorithm converges and eventually terminates and is key part of the distributed algorithm. This is important since each node acts independently, the only thing that they ensure is, they follow some `Rules` while making decision and the objective is if all nodes follow the rules, then the cluster of nodes will end up with an agreement.
- Learners - A client (or server) that doesn't participate in the decision making process but is interested in the decision.
- Persistence - It's important to note that whenever a node responds to a message, it commits to the response since other decisions are taken based on this response. For example, if a node has responded saying that a value was accepted, it must ensure that a new response (potentially after restart) doesn't claim that no value was accepted. Typically this is ensured by storing some state in pesistent state (disk) that is not lost on restart. So to vote on a proposal, before responding the node must store it's vote so that it continues to act according to it's previous response.

### Solution
Here we'll build up to the actual paxos algorithm by building on some trivial steps and solving problems associated with each step.

#### Designated Chooser
Let's assume there's one node that decides on the chosen value. It accepts the first value. This solves the consensus part where nodes agree on a value. But if this node crashes, the system is stuck and can't make progress.

To solve this, we need multiple acceptors. This helps progress in 2 ways: If a node crashes, other nodes can continue to find the chosen value (since for a majority we don't need all nodes to be available). If a node crashes after a value was chosen, we can find this value from other acceptors.

#### Multiple acceptors

**Accept First Value** - In this approach, each acceptor would accept a value if it has not accepted any value before. If already a value has been accepted by this acceptor, it doesn't accept any new value.

Let's see when this breaks down - If there's only a single value to be proposed, this solution works since each acceptor is going to accept the same value. But what happens when there are multiple proposals? It's possible that different acceptors accept different proposals and no majority is reached. Now the algorithm is stuck since these acceptors won't accept any more values and no value can ever be chosen. This violates the liveness property.

**Accept every value** - In this approach, acceptors accept every value that has been proposed. This ensures that algorithm doesn't get stuck but this ends up choosing more than one value. Let's say there are 3 nodes A, B and C. Say value X was proposed at the same time as Y was proposed. A and B accept X. Then B and C receive Y. Now both X and Y have majority which violates safety property. The issue with this approach is, the algorithm continues to propose values even after a value was chosen.

**2 phase accept** - Once a value is chosen, new values shouldn't be proposed. To ensure this, there is another phase in the algorithm where it checks if a value has already been chosen. Before proposing a value, a proposer checks if acceptors have already chosen a value. This is done by asking acceptors for the values they have accepted until now. If there's a majority that accepted a value, then proposer proposes the same value instead of a new value.

The problem with this approach is synchronization. For 2 concurrent proposers, both can find that no value has been accepted yet and end up proposing a new value and with the current protocol, both get a majority.

To fix this, we introduce a serialization of proposals. This part of the algorithm ensures that we can globally order each proposal even when they are running concurrently. Then acceptors that receive multiple proposals make a consistent decision based on the order. 

To get a globaly consistent order, each proposal is given an id using counter and node id. Proposal Id: "Largest counter seen so far + 1". "Node Id>". This ensures that when 2 proposal ids are compared, there is always a smaller id. The only constraint is, proposal ids can't be reused and counter shouldn't go back. To ensure this proposers need to either persistently store the used counter values or query from the acceptors for highest proposal id seen so far. This ensures that proposal number is at least as high as the chosen propsoal if a value was chosen. If no value was chosen, it need not be highest based on the query.

Here's the actual paxos algorithm.

| Proposer | Acceptor |
|----------|-----------|
| 1. Choose a proposal number higher than any proposals seen and send a prepare request with the proposal| |
| | 2a. Acceptor has not responded to any proposal higher than this. Acceptor marks no to every proposal lower than this, persists it's response and responds with [yes, highest proposal accepted and value for the proposal]
| | 2b. Acceptor has responded to a proposal higher than this, hence responds with [no, highest proposal responded and value for the proposal]
|3a. If there's a majority with "yes" response and there's an accepted value by any acceptor, propose that value from highest proposal id. ||
|3b. If there's a majority with yes and there's no accepted value, propose the original intended value.||
|3c. If there's no majority with "yes", restart with a higher proposal id. ||
|| 4. HandleAccept: If an acceptor has not responded to a proposal higher than this, persists and returns [accept, value]. Otherwise returns[reject, highest accepted value]
|5. At this point if majority has accepted a proposal, that proposal is "chosen". But it may not be known yet that a proposal is chosen.||
|6. If a proposer gets an accept from majority from acceptors, the proposer knows that a proposal is chosen and can broadcast chosen value||
|||

Liveness of this protocol is guaranteed if a single proposer is able to run the algorithm with a majority acceptors responding. If there are 2 proposers that are running, they can cause rejection of each other by choosing higher and higher proposal numbers and can never terminate. To ensure this, once a proposer detects a higher proposal, it pauses and retries after some period. This pause period can be increased by backing off exponentially so that eventually there's no contention.

Safety of this proposal is guaranteed by the fact that once a proposal achieves majority (by the fact of majority acceptors persisting it as accepted), the algorithm ensures the same value is chosen. To see why this is the case, consider a new proposer that might be in different steps at the instant a proposal gets majority -

Step 1. Here the proposer has no knowledge that a value was chosen, so it will propose a new value.
Step 3. Here the proposer will learn that it was accepted by some acceptor. It must be the value with highest proposal id. This is proven inductively since any proposer right after the majority must follow the same algorithm and propose the same value.
Step 5. If a proposer has reached this step, it must have received a yes from majority in prepare and this implies the value was chosen after acceptors responded yes to this proposal. If this is the case, the chosen value must have higher proposal id than the current proposal. Hence proposer's accept request will be rejected by a majority.

