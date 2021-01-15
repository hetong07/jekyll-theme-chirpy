---
title: Paxos and Raft
author: hetong07
date: 2020-05-25 19:42:00 -0700
categories: [Blogging, Technical]
tags: [paxos, raft, distributed system]
---

This is not a complete explanation of the Paxos and the Raft, but just some thoughts about some corner cases.

### Livelock and leader in the Paxos
The basic paxos algorith works well but has a **livelock** issue: two proposers 
may each grow the instance number to infinite but no one can proceed. To solve
this problem, the author proposes the notion of **distinctive** proposer, which
acts as the leader to initiate all the propose. But it does not present how to 
select such leader. According to the [3], the usually used leader election algorithm is Bully algorithm:
>  A node that starts an election sends its server ID to all of its peers. If it gets a response from any peer with a higher ID, it will not be the leader. If all responses have lower IDs than it becomes the leader. If a node receives an election message from a node with a lower-numbered ID, then the node starts its own election. Eventually, the node with the highest ID will be the winner.

which is much like the one used in the Paxos.

Since there is a leader, I suspect that unless there was a proposer failure, the proposal returned by acceptors (if any) will be the same.

Also, this issue can be solved via exponential backoff, but it will rise the latency.

### History confilct
The basic paxos works well if there is no network issue underlying. For example, assuming there are five servers/processes {A, B, C, D, E}, and there is a network partition and divides the servers into three group {A, B}, {C}, {D, E}. If {A ,B} is about to commit value "(1, Foo)" while {D, E} is about to commit "(2, Bar)", the final result will be determined by C:
- if C joins {A ,B}, "Foo" will be committed
- if C stands with {D, E}, "Bar" will be committed
At first glance, it seems no harm, but consider then the servers are all down, which history is the history? The servers diverge with two different history. (Refer to "Conflict Resolution Example" in [2] for detials).

Paxos seems has no solution to this, and according to [2], ths naive way to solve this is to let the application itself to poll history from peers to make up the inconsistency.

While on the contratry,  Raft solves this problem via:
- master is the ultimate arbiter of the history, and it can rewrite conflict history.
- **new** master is not allowed to commit old round values (I cannot remember clearly)

I think this is one of the case why Raft is more easy to understand than Paxos.

I guess this problem can be solve by the *catchup* mechanism described in [5], it is exactly achieved by the application layer.

### Multi-Paxos
The Multi-Paxos works much like the Raft.

### Some notions about Paxos
Each round of Paxos is like a transacion, *acceptors* will noly reply the proposal received in this round, this is where I struggled to understand at first.

### Raft implementation
I have implemented the mit 6.824 (2019) version of Raft in https://github.com/hetong07/mit6.824. The final lab is left not finished, but it should be very easy.


## Reference
1. [Paxos made simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
2. [Understanding Paxos](https://understandingpaxos.wordpress.com/)
3. [Understanding Paxos](https://www.cs.rutgers.edu/~pxk/417/notes/paxos.html)
4. [Raft](https://raft.github.io/raft.pdf)
5. [Paxos made live](https://dl.acm.org/doi/pdf/10.1145/1281100.1281103)
