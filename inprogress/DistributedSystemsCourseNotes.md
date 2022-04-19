Course - UCSC (Univerity of California Santa Cruz) CSE138 (Distributed Systems)
***

* What is a distributed system?

A distributed system is a system where I can't get my work done because somewhere a computer has failed that I don't even have heard of. - Leslie Lamport

A distributed system is 
   * running on several nodes
   * connected by a network and
   * characterized by **partial failure** [some parts of the system broken, other ok], impossible for the software to know that something (or what) has failed. Usually independent failures are possible. In non distributed setting, either system is running or failed. There won't be partial failure. 

Node - software or service running on a computer

# Philosophies
## Cloud computing
Lots of computers, highly likely some computer might have failed.
Expect failures and work around it. So we keep operating in the event of failure

## High Performance computing
failure less likely, so restart computation potentially from a checkpoint

# Failures

Consider simple scenario of 2 machines exchanging messages -
* Packet drops at network
* Request is really slow to reach [either slow on network or on either machines]
* M2 crashes
* M2 is slow to respond
* Response slow on network
* Response got lost
* Corrupt transmission (Byzantine faults, like M2 is rogue, or someone else on the network corrupts or delays or introduces some other malicious behavior)

**If you send a request to another node and don't receive a response, it is impossible for you to know for sure why.**

**How do real systems deal with it**
With timeouts - Need to know a good value for timeout, so not a perfect solution

If we knew the network latency max between 2 nodes and processing time by a node, we can use this as an estimate for timeout. But we don't know any of this, so we need to guess.

Palvaro's definition: **Partial failures + unbounded latencies**

### Why build distributed systems when so difficult
* Higher performance with multiple computers
* More data than can fit on single computer
* Reliability (more copies of data)
* Throughput

# Chapter 2
* Times and clocks
* Lamport diagrams
* Network models
* Causality and happens-before
* State and events

## Clocks
Time is used in following ways -
1. Mark a point in time.
2. Define an interval of time or duration.

Computers have 2 types of (physical) clocks
* Time of day clocks
  * Can be synchronized between computers using NTP.
  * Can jump forwards or backwards.
  * Can't be used to measure durations
* Monotonic clocks
  * Only goes forwards
  * Good for measuring durations but not good for specific point in time. It's time is only meaningful on the specific computer

### Logical clock
order of events.

A -> B "A happened before B"
This implies A could have caused B and B couldn't have caused A.

This can be used in the system design to prevent users noticing B before A is noticed.

### Lamport diagrams
* If A and B are events in same process, then A -> B if time(A) < time(B)
* mesage send response
* Transitivity

## Network Model
### Synchronous Network
There exists some N where N is units of time such that no message takes longer than N to be delivered.

### Asynchronous Network
There exists no N where N is units of time such that no message takes longer than N to be delivered.

* Partially synchronous :- There exists some N such that no message takes longer than N but we don't know the value of N beforehand.

# Chapter 3 - partial order,  logical/lamport/vector clocks
**Partial Order Set** A set with a binary relation <= such that the relation is
* reflexive - for all elements a of set, a <= a
* antisymmetric - for all elements a, b of set, if a <= b and b <= a then a = b
* transitive - for all elements a, b, c of set, if a <= b and b <= c then a <= c

Example of POSET - all subsets of a set and a subset relation.
Events with Happens before relation is a partial order set but it doesn't have a reflexive relation.

In poset, there could be set members that are not related to each other. In contrast, total ordered set is a poset where every pair of members is related.

## Logical clocks
**Lamport Clock** - If A -> B, then LC(A) < LC(B). Lamport clocks are consistent with causality. But LC(A) < LC(B) doesn't imply A -> B.

LC Algorithm

1. Every process maintains a counter initialized to 0.
2. On every event, process increments the counter.
3. Process sends the counter along with the message.
4. On receiving a message, process sets its local counter to [1 + max(local counter, remote counter from message)].

### Vector clocks
**Vector Clock** - VC(A) < VC(B) if and only if A -> B. This means vector clocks are consistent with causality and characterizes causality.

Vector Clock Algorithm
1. Every process maintains a vector of integers initialized to 0. Length of the vector is same as the number of processes. Every process has one entry in this vector.
2. On each event, each process increments it's position in the vector clock.
3. On send, process increments it's position and sends the vector as part of the message.
4. On receive, process updates it's vector clock for each entry slot to max of it's value and value from the message. Then increment it's entry for receive event.

# Chapter 4 - Executions, properties of execuctions and delivery order
**Delivery Guarantees**
* FIFO Delivery
* Causal Delivery
* Totally Ordered Delivery

**Lamport Clock Property** - If A->B then LC(A) < LC(B), but LC(A) < LC(B) doesn't imply A->B

**Vector clock property** - A -> B if and only if VC(A) < VC(B). Otherwise A and B are concurrent.

**Vector Clock Algorithm**
1. Every process is initialized with a vector of 0s with vector length same as the number of processes.
2. Anytime an event occurs in a proecss, the vector clock is updated by incrementing the counter at the process index.
3. Whenever a message is received, take pointwise max of incoming vector clock and local clock (after incrementing it for the current process receive)

**Causal History event** - All events that happened before event.

**Non comparable** - Concurrent event or causally independent.

**Protocol** - Set of rules process use to communicate with each other.
* **Valid Run of protocol** - If the protocol has not been violated **yet**.

### FIFO Delivery
 If a process sends a message m2 after message m1, then any process delivering both messages delivers m1 first.

**Receive and Delivery** - Receive is the event that process received the message over the network. Delivery is the event when the process actually handles the event. Delivery order might be different than the receive order (say if a sequence number is used to order delivery even though they are received out of order). Sending a message is that a process does, receiving a message is something that happens to a process. Delivery is again something a process can do to a message that it receives.

FIFO delivery is a part of TCP. Without TCP, we would have to use sequence number on each message and delay delivering a message until all messages with sequence number below it has been delivered.

FIFO delivery makes sense only in the context of a specific pair of processes exchanging messages. For example, if there are 3 processes P1, P2 and P3, FIFO ordering doesn't make sense between a message m1 from P1 to P2 and another message m2 from P3 to P2.

Vacuous FIFO delivery property by not delivering any messages. The FIFO ordering property will not be violated by dropping all messages.

### Causal Delivery
If m1's send `happens before` m2's send, then message m1's delivery must happen beffore m2's delivery. This can be implemented by the delivery of the event based on the vector clocks associated with the message.

### Total Order Delivery
If a process delivers m1 and then m2, then call processes delivering m1 and m2 must deliver m1 and then m2.

**Violation of Total Order Delivery**

|Time| Process 1| Process2 | Process 3| Process 4 |
|---|----|----|----|----|
|0|Send m1 to process 2 and 3|||Send m2 to process 2 and 3|
|1||Receive m1 from process 1|Receive m2 from process 3|
|2||Receive m2 from process 3|Receive m1 from process 1||

In this example, process 2 delivers m1 and then m2. Process 3 delivers m2 and then m1. This violates total order delivery. This can be a problem say if process 2 and process 3 are KV store replicas. If both m1 and m2 update the same keys by sending messages to replicas, because of violating total order, the replicas go out of sync. Process 2 ends up storing the value from m2 since it received that later where as process 3 ends up storing the value from m1 since it received m1 later.

# Chapter 5

## Recap of delivery properties
* **FIFO Delivery** - If a process sends m1 before m2, then all processes delivering m1 and m2, deliver m1 before delivering m2.
* **Causal Delivery** - If m1's send `happens before` m2's send, then m1 is delivered before m2 is delivered.
* **TO delivery** - If a process delivers m1 before m2, then all processes deliver m1 before m2.

### How are these delivery properties related
Executions that satisfy **Causal Delivery** also satisfy **FIFO Delivery** since send order of the message define a causal dependency. But **TO Delivery** property is not related to the causal order of the messages but it can be any order. Only requirement is that all processes deliver in the same order. Hence ti's possible that in a system all executions might have **TO delievery** property but not have **FIFO** or **Causal**.

## Implementing Causal broadcast (with vector clocks)
- Unicast messages - Point to Point - 1 sender and 1 receiver
- Multicast messages - 1 sender and many receivers.
  - Broadcast messages - 1 sender and *everyone* (sender included?) receives.

- Causal delivery vs causal broadcast
  - Causal delivery is the property of executions that we care about.
  - Causal broadcast is an algorithm that we use to ensure causal delivery.


### Vector Clocks
Twist - Don't count messsage receive as events.

- When a process sends a message, it increments its own position in it's vector clock. It includes the updated vector clock with the message.
- When a process delivers a message, it updates it's vector clock to the pointwise maximum of it's local vector clock and received vector clock on the message.

Given that we are only implementing the delivery for broadcasts, looking at the vector clocks help us decide whether to delivery the message right away or wait. For example, if our vector clock is [0, 0, 0], and we receive a message from a process with vector clock [1, 0, 0], we know that there's only one event that we are not aware of since the difference is 1. That event is the send of this message. 

But say if we receive a message from a process with vector clock [1, 1, 0], we know that there are 2 events that we are not aware of. One of them is this message, but there's another event that must be delivered before this. Hence we wait instead of delivering it immediately.

Note: this assumption is only valid if all messages are broadcast. Otherwise, it's possible that the missing event was not intended for us and we can't wait.

### Deliverability Condition
A message m is deliverable at a process p if:

* VC(m)[k] = VC(p)[k] + 1, k is the index of the sender process.
* VC(m)[k] = VC(p)[k], k is index of all processes except sender process.

Since we want to use just a single increment of vector clock to deliver a message, we don't want to assign events for delivery or receives.

Given this ordering, every processes is going to deliver the message in same order if all messages are causally ordered. If all messages are causally ordered, this provides total order.

Do we achieve total order with this? No. If a process P receives messages from 2 processes A and B, they won't be comparable and any order of delivery is consistent with causal delivery.


## Chandy Lamport snapshot algorithm
Snapshots of distributed systems - How to take snapshot of each process such that they are not inconsistent. For example, if we asked every process to take a snapshot at the same instant, they would be consistent with each other. But it's not possible to synchronize clocks - can't use time of day clocks.

Required Property - If A `happened before` B and B is in the snapshot, A must be in the snapshot.

### Terminology

* **Channel** - Connection from one process to another. They are one way. For 2 process P1 and P2, we need a channel from P1->P2 and another for P2->P1.

In flight messages are said to be **in** the channel. If a message has been sent from P1 and not received (or delivered?) at P2, it's considered to be in channel C12. Once delivered, the channel is empty.

The channels are FIFO.

# Chapter 6
**Distributed Snapshot** - If we take snapshot of each process state such that it has event B in some process, then all events that `happened before` B are also in the snapshot.

This is a consistent snapshot. This consisetnt is not same as consistency term used in other contexts.

Chandy Lamport algorithm was published before vector clocks were mainstream (either not published or not popular). Hence this algorithm doesn't use vector clocks. But since then there have been variations that use vector clocks.
## Chandy Lamport snapshot algorithm
Recording a snapshot is kicked of by a process called `initiator` process. One or more processes can initiate a snapshot.

- An initiator process
  - records it's own state
  - Sends a `marker message` out on all it's outgoing channels.
  - Starts recording all messages it gets on all its incoming channels.
- When process Pi receives a marker message on channel Cki
  - If it is the first marker Pi has seen:
    - Pi records its state
    - Pi will mark the channel Cki as empty
    - Pi will send the marker on all its outgoing channels
    - Start recording messages on all incoming channels except Cki
  - Otherwise (Pi has already seen a marker message)
    - Pi will stop recording messages on channel Cki


The marker message is sent on the same channel as the regular messages. So a receipt of marker message guarantees that the receiving processes has seen all causally preceeding messages before it sees the marker. Also, sending process sends the marker message after saving the state. The events not included in snapshot happen after the send event of marker message. So all the messages that are before the marker message correspond to events before the state was saved.

This is guaranteed due to the channels having FIFO behavior.

With N processes, total N(N-1) marker messages are sent.

## Limitation/Assumptions/Properties of Chandy Lamport snapshot algorithm

- Assumptions
  - Channels have FIFO behavior. This is a requirement for Chandy Lamport snapshot algorithm to work.
  - No need to pausing of application messages in Chandy-Lamport.
  - CL algorithm assumes messages are not lost or corrupted or duplicated.
  - Also assumes processes don't crash.

- Properties
  - Snapshot it takes are consistent.
  - Guaranteed to terminate given assumptions of messages not lost etc.
  - Works fine with more than one initiator of snapshot.

## Centralized vs decentralized algorithms
A `decentralized` algorithm is that can have multiple instances.

examples: Chandy-Lamport, Paxos

A `centralized` algorithm must be initiated by exactly one process.

## Snapshots uses
- Checkpoints - An algorithm doesn't have to start from scratch.
- Deadlock detection - Deadlock is a property of a system that once it's true, it remains true. So if a snapshot has a deadlock, currently the system is still deadlocked.
- Detection of any `stable property`: A property that is once true it remains true.

# Chapter 7
## Chandy Lamport wrapup
**Channels** - We assume every process has a channel to every other process.
The process needs to be strongly connected. This means process can reach to all processes but may be indirectly. With this we can simulate every process being connected to every other process.

* Snapshot
  * Process State - Events that occurred
  * Channel State - Messages recorded as part of recording channels.
## Safety and Liveness

All properties are either safety or liveness or a combination.

Delivery guarantees fall into safety properties. Safety properties say that something bad won't happen.

Liveness properties say good thing will happen.
eg. eventual delivery of messages.

Usually liveness properties are specified with "eventually".

- Safety Properties
  - Say a "bad" thing won't happen.
  - Can be violated in a finite execution.

- Liveness Properties
  - say that a "good thing" will eventually happen.
  - cannot be violated in a finite execution (eg. If delivery didn't happen, it's just that the property is not satisfied yet. Not that it has been violated.)

## Reliable Delivery
Let P1 be a process that sends a message `m` to process P2. If neither P1 nor P2 crashes, P2 eventually delivers the message.

## Classifying faults and fault models
Fault models tell us which kinds of faults can occur.

- **Omission Fault** - Message getting lost (A process fails to send or receive a single message).
- **Timing fault** - Message slow - a process responds too late or too early.
- **Crash fault** - Process fails by halting (stops sending/receiving messages). As far as external observer is concernted, it's crashed if it's not sending or receivng messages, even though process might be up.
- **Byzantine fault** - A process behaves in an arbitrary or unpredictable ways or malicious ways.

Crash faults are a special case of omission faults where all the messages froma a given process are lost. Hence any protocol that can tolerate omission faults can tolerate crash faults.

## Two General Problem
