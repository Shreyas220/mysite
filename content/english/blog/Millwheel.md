---
title: "Fault tolerance in MillWheels"
meta_title: ""
description: "Understanding Fault Tolerance in Streaming Systems "
date: 2025-03-10T05:00:00Z
image: "/images/millwheel1.png"
categories: ["Streaming Systems", "Distributed Systems"]
author: "Shreyas Mishra"
tags: ["fault tolerance", "research paper"]
draft: false
---


## Introduction

Fault tolerance is super important for modern streaming systems since data is always in transit and failure can happen at any stage of the system. <br>
We can run into issues like lost messages, processing the same message twice, or keeping track of state properly. This can lead to unreliable results or even system downtime. <br>
For example, think about a system that monitors web searches to spot sudden trends. If a part of the system fails or some data gets dropped, it might miss a new trend or, even worse, trigger false alerts.


## MillWheel’s Fault Tolerance Mechanism


MillWheel addressed these challenges by providing framework level fault tolerance where any node or edge in the processing graph could fail without affecting the correctness of results, it does it  through a few core mechanisms:

- **Persistent State Storage:**
    
    State and metadata are continuously written to a durable store. This allows recovery to a consistent point.
    
- **Low-Latency Recovery:**
    
    Rapid restoration of state and in-flight messages minimizes downtime. Recovery mechanisms reload state and pending messages quickly.
    
- **Idempotent Processing:**
    
    Each message is associated with a unique ID. Before processing, the system checks whether this event has already been handled, ensuring that duplicate processing is avoided.
    
- **Distributed Consistency:**
    
    MillWheel coordinates state updates across distributed nodes, ensuring that the system remains consistent even in the face of failures.
    

let's dive a little deeper into the mechanisms

MillWheel provides at-least-once delivery, meaning every record is retried only after it has successfully processed and acknowledged. 

This ensures no data is lost but introduces the potential for duplicate processing, like what if a receiver might process a record, persist the new state, but then crash before sending the ACK. When the record is redelivered, the system reprocesses it, considering it a new record. 

To prevent duplicate processing, we assign a global unique ID to each record at production time. Since keeping all processed IDs in memory would be impractical for long running systems, MillWheel employs a two-tier approach: a bloom filter for fast inmemory checking, and a persistent backing store for definitive verification

Once the changes are committed, the ACKs are sent to the sender so that it stops retrying.

Only after all processing is complete and the changes are committed does the system send acknowledgments back to senders, signaling that they can stop retrying delivery. The system also intelligently manages these record IDs, keeping them only as long as necessary. Once the system confirms that no more retries will occur (typically after internal senders have completed their retry cycles), it cleans up these duplicate detection records to conserve resources.

Now let's understand how this commits and recovery from the state system works 

### Checkpoint commits

#### Two Types of State for a node

##### Hard State
    
    This is the data that’s permanently stored in a backing store (database or disk). 
    
##### Soft State

    This includes things kept in memory such as caches or temporary aggregates. 
    


MillWheel workers save their current state very often, sometimes every second or even after each record. These frequent checkpoints allow the system to recover quickly by loading the most recent saved state rather than rebuilding from scratch.
Importantly, MillWheel allows workers to access their saved state without pausing all processing. This concurrent checkpointing and processing enables the system to maintain high throughput even while ensuring fault tolerance.<br>
To maintain consistency across related pieces of state, MillWheel wraps state updates in atomic operations. This atomicity ensures that all parts of a key's state are updated together, maintaining invariants and preventing partial updates. For instance, if a financial application updates both an account balance and a transaction log, these updates must succeed or fail together to prevent inconsistencies.


### Dealing with “Zombie” Processes 

One of the most difficult challenges in distributed streaming systems involves managing "zombie" processes and preventing stale writes. When work migrates between machines, perhaps because a process failed or due to load rebalancing <br>there's a risk that an old or "zombie" process might later send updates that are no longer valid.

For example, you have a Node A working on a task, but then they get replaced by Node B because they were too slow or failed. 

MillWheel solves this through a tokenized write validation system. Each update includes a special token that verifies its validity. When a new process takes over responsibility for a key, it invalidates older tokens, ensuring that only the most current process can update that key. 

Now if Process A (the “zombie”) later tries to update the task with outdated information, the system will ignore it because Process B’s token has already taken over. The system automatically rejects these stale writes, preserving consistency.


## Lets try to understand this with an example

Imagine our streaming system has 4 nodes :
1. **Source** → Ingests events
2. **Enrichment** → Adds product details to each event
3. **Aggregation** → Computes on events
4. **Alerting** → Sends notifications for unusual patterns

Now, suppose the part that crunches numbers (the **Aggregation** stage) suddenly crashes. Here's what happened:

- The server was working on a batch of 100 transactions.
- It finished 60 transactions before crashing.
- It had saved its progress (a checkpoint) after 60 transactions but did not ACK the Enrichment node for that last 10 messages .
- That means 10 transactions were processed after the checkpoint but not ACK to the previous node, and 40 transactions were still in the middle of being processed when the crash occurred.

**How the system recovers:**

##### 1. Noticing Something’s Wrong:
   The system keeps an eye on all servers. When the aggregation server stops replying, the system marks it as failed and holds onto any messages that were still being processed.

##### 2. Restarting and Loading the Last Checkpoint:
   A new server instance starts up and loads the saved state from the checkpoint (which only goes up to 60 transactions).

##### 3. Replaying the Events:
   The system now replays the messages that were lost:
   - The new instance loads the state checkpoint containing data for the first 60 events
   - It will replay from the 51th message as did not get ack after that.
   - Which means it also replays the 10 events that were processed after the checkpoint (even though they were already processed).
   
   To avoid counting any transaction twice, every event has a unique ID. The system checks these IDs, if it sees an ID it’s already processed, it simply skips it.


##### conclusion

Even though the server crashed in the middle of processing, the system recovers by rolling back to the last checkpoint and reprocessing the missing events, all while ensuring that no transaction is counted twice.

