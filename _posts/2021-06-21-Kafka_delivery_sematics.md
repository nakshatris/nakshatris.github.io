---
layout: post
title:  Kafka delivery semantics
categories: [Kafka, DS]
---

###Concepts
- Consumer group: Set of consumers which cooperate to consume from some topics. The partitions of all topics are divided amongst the consumers of the group. 
  Hence, a consumer can be assigned to multiple partitions, however, messages in a partition are consumed by a single assigned consumer.
- Group rebalance: When consumers join/leave, the partitions are re-assigned known as rebalancing the group. The group management uses a grouping protocol 
  built into Kafka itself, and not zookeeper. One of the brokers is made th coordinator for group management. The leader of the topic *__consumer_offsets* topic is 
  chosen as th coordinator.
- Offset management: When a new group is created, the group uses the offset reset policy *auto.offset.reset* to determine the offset position. Typycally, it is 
  either latest offset or earliest offset. Correct offset management is crucial because it affects **delivery semantics**.
  
###Kafka consumer delivery semantics

**At-least once delivery**: Kafka guarantees no message will be missed, but duplicates are possible. 
By default, a consumer is configured for auto commit. Auto commit works as cron job which periodically commits the offset on a regualr interval set using *auto.commit.offset.ms*. In such cases, if the consumer crashes, after rebalance, the new consumer picks messages from the last committed offset, which could be as old as the auto commit interval, causing duplication.

Remedies:
To reduce duplicates, the interval can be shortened.
To avoid auto commit, set *enable.auto.commit* to false and use synchronous commits
  - Cons: This can lead to performance setback because consumer is blocked until offset is commited.
Increase the amount of data fetched to improve syncronous commit performance
  - Cons: This increases amount of duplication in worst case failures.


