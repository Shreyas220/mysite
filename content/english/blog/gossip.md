---
title: "Gossip Protocol"
meta_title: ""
description: "Understanding Gossip protocol "
date: 2025-03-10T05:00:00Z
image: "/images/image.jpg"
categories: ["Distributed Systems"]
author: "Shreyas Mishra"
tags: ["protocol"]
draft: false
---

## Gossip

Introduction to gossip protocol

## Why do we need gossip protocol ?

1. **Centralized Coordination Overhead**
    - centralized coordinators creates bottlenecks and single points of failure. Leader failures paralyze the system until a new leader is elected

2. **Inefficient Failure Detection**
    - Heartbeat-based monitoring requires constant communication, consuming excessive bandwidth and overload the network but slow detection of node failures leads to stale data and service disruptions

3. **Unreliable Multicast Communication**
    - Multicast protocols struggle with packet loss, network splits, and scalability in large clusters. A multicast message to 1,000 nodes could overload the network and fail to reach all nodes

4. **Slow State Synchronization**
    - Manual or slow periodic reconciliation of node states can cause delays and inconsistencies when nodes operate with outdated data, risking decision-making errors (e.g., duplicate transactions)

# how gossip protocol works

- Nodes communicate with randomly selected peers.
- Each node operates with limited local knowledge of the system.
- Communication occurs at regular intervals.
- The size of transmitted data is limited per gossip round.
- The protocol assumes network paths may fail.
- Interactions are infrequent to reduce overhead.

# Types of gossip protocol

1. Anti Entropy 
    1. Nodes periodically compare their entire dataset with other nodes to identify and rectify inconsistencies
2. Rumor Mongering 
    1. sharing only the latest updates
    2.  might flood the network with frequent cycles
3. Aggregation Gossip Protocol
4. **Dissemination Protocol Variants**:
    1. **Event Dissemination Protocols**: Gossip periodically about events without triggering gossip directly.
    2. **Background Data Dissemination Protocols**: Continuously gossip about node-associated information, suitable for environments where propagation latency isn't critical[2](https://en.wikipedia.org/wiki/Gossip_protocol).

**epidemic based protocol** 

- SWIM
- Serf (Hashicorp built on top of swim)

![](https://highscalability.com/content/images/2024/02/1nu8hxn-__squarespace_cacheversion-1689515298461.gif)

# **Probabilistic Analysis of Gossip Protocols**
