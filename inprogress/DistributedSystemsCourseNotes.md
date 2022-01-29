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
