---
layout: post
title:  Kafka delivery semantics
categories: [Kafka, DS]
---

**Concepts**
- Consumer group: Set of consumers which cooperate to consume from some topics. The partitions of all topics are divided amongst the consumers of the group. 
  Hence, a consumer can be assigned to multiple partitions, however, messages in a partition are consumed by a single assigned consumer.
- Group rebalance - when consumers join/leave, the partitions are re-assigned known as rebalancing the group. The group management uses a grouping protocol 
  built into Kafka itself, and not zookeeper. One of the brokers is made th coordinator for group management. The leader of the topic *__consumer_offsets* topic.
  

**Kafka consumer delivery semantics**

