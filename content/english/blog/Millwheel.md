---
title: "Fault tolerance in MillWheels"
meta_title: ""
description: "Understanding Fault Tolerance in Streaming Systems "
date: 2025-03-10T05:00:00Z
image: "/images/millwheel.jpg"
categories: ["Streaming Systems", "Distributed Systems"]
author: "Shreyas Mishra"
tags: ["fault tolerance", "research paper"]
draft: false
---

🚧 Under Construction 🚧


## Introduction

Streaming systems operate under challenging conditions where data flows continuously, errors can occur at any stage, and any downtime may lead to lost or inconsistent information. This makes fault tolerance not just a feature but a necessity for ensuring that systems quickly recover from failures while maintaining state consistency and processing guarantee

Fault tolerance is a critical requirement for modern streaming systems where data flows continuously and errors can occur at any point in the processing pipeline. Traditional streaming frameworks often struggle with challenges such as message loss, duplicate processing, and state inconsistencies—problems that can lead to unreliable results or significant downtime. 

For example, imagine a real-time system that monitors web queries for sudden trends. 

If a node fails or an edge in the processing graph drops data, the system might miss an emerging trend or, worse, generate false alerts.

## MillWheel’s Fault Tolerance Mechanism


MillWheel addressed these challenges by providing framework-level fault tolerance where any node or edge in the processing graph could fail without affecting the correctness of results, it does it  through a few core mechanisms:

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

This ensures no data is lost but introduces the potential for duplicate processing, like what if a receiver might process a record, persist the new state, but then crash before sending the ACK. When the record is re-delivered, the system re-processes it, considering it a new record. 

To prevent duplicate processing, we assign a global unique ID to each record at production time. Since keeping all processed IDs in memory would be impractical for long-running systems, MillWheel employs a two-tier approach: a bloom filter for fast in-memory checking, and a persistent backing store for definitive verification

Once the changes are committed, the ACKs are sent to the sender so that it stops retrying.

Only after all processing is complete and the changes are committed does the system send acknowledgments back to senders, signaling that they can stop retrying delivery. The system also intelligently manages these record IDs, keeping them only as long as necessary. Once the system confirms that no more retries will occur (typically after internal senders have completed their retry cycles), it cleans up these duplicate detection records to conserve resources.

Now let's understand how this commits and recovery from the state system works 

### Commits

#### Two Types of State for a node

##### Hard State
    
    This is the data that’s permanently stored in a backing store (database or disk). 
    
##### Soft State

    This includes things kept in memory such as caches or temporary aggregates. 
    


To enable rapid recovery from failures, MillWheel workers save their current state extremely frequently—sometimes after every second or even after processing each individual record. These frequent checkpoints allow the system to recover quickly by loading the most recent saved state rather than rebuilding from scratch.
Importantly, MillWheel allows workers to access their saved state without pausing all processing. This concurrent checkpointing and processing enables the system to maintain high throughput even while ensuring fault tolerance.
To maintain consistency across related pieces of state, MillWheel wraps state updates in atomic operations. This atomicity ensures that all parts of a key's state are updated together, maintaining invariants and preventing partial updates. For instance, if a financial application updates both an account balance and a transaction log, these updates must succeed or fail together to prevent inconsistencies.

### Dealing with “Zombie” Processes and Stale Writes

One of the most difficult challenges in distributed streaming systems involves managing "zombie" processes and preventing stale writes. When work migrates between machines—perhaps because a process failed or due to load rebalancing—there's a risk that an old or "zombie" process might later send updates that are no longer valid.

For example, you have a Node A working on a task, but then they get replaced by Node B because they were too slow or failed. 

MillWheel solves this through a tokenized write validation system. Each update includes a special token that verifies its validity. When a new process takes over responsibility for a key, it invalidates older tokens, ensuring that only the most current process can update that key. 

Now if Process A (the “zombie”) later tries to update the task with outdated information, the system will ignore it because Process B’s token has already taken over. The system automatically rejects these stale writes, preserving consistency.

## Lets try to understand this with an example

Imagine we have a simple pipeline analyzing e-commerce transactions:

1. **Source** → Ingests events
2. **Enrichment** → Adds product details to each event
3. **Aggregation** → Computes on events
4. **Alerting** → Sends notifications for unusual patterns

Let's say the **Aggregation** computation fails in the middle of processing:

### Failure Scenario

1. A server running the Aggregation computation crashes while processing a batch of 100 events
2. The computation had processed 60 events and updated its state accordingly
3. 40 events were in-flight at the time of failure
4. The last state checkpoint happened after processing 50 events

### Recovery Process in Detail

#### 1. Failure Detection

The MillWheel system constantly monitors the health of computation instances through periodic heartbeats. When a server fails to respond:

- The system marks that computation instance as failed
- In-flight messages to the failed computation are buffered

#### 2. State Recovery

When a new instance is started:

For our Aggregation example:

- The new instance loads the state checkpoint containing data for the first 50 events
- The state includes per-category revenue counters and any other metadata
- All timers (like "emit daily totals at midnight") are restored
- The system identifies all messages that were sent to the failed instance but not acknowledged as processed

#### 3. Message Replay and Deduplication

This is where MillWheel's exactly-once guarantee really shines:

In our example:

- The source systems (Enrichment in this case) will replay messages that weren't acknowledged
- This includes all 40 unprocessed events, but might also include some of the 10 events processed after the last checkpoint
- Each message in MillWheel has a unique ID
- The system keeps a persistent log of processed message IDs as part of its state
- When a message arrives, it first checks if its ID is in the "already processed" log
- If found, the message is a duplicate and is silently dropped
- If not found, the message is processed and its ID is added to the log

#### 4. Exactly-Once Guarantee Implementation

The exactly-once guarantee relies on three mechanisms working together:

1. **Strong Delivery**: Messages are persistently stored until acknowledged
2. **Determinism**: Given the same inputs and state, a computation produces the same outputs
3. **Idempotent Updates**: Applying an update multiple times has the same effect as applying it once

For our Aggregation computation:

#### 5. Maintaining Consistency During Recovery

For each key/value pair in the system's state:

1. The persistent storage maintains the latest committed value
2. Each computation keeps track of the keys it has modified since the last checkpoint
3. When a checkpoint occurs, only modified keys are written to persistent storage
4. This makes checkpointing efficient while ensuring consistency

For our example:

- Only the category counters that changed during processing of those 10 events (between events 50-60) need to be recovered
- The system reconstructs this by replaying the events from 51-100