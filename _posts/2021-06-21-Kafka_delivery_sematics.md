---
layout: post
title:  Kafka delivery semantics
categories: [Kafka, DS]
---

### Concepts
- **Broker**: Kafka is run as a cluster of one or more servers that can span multiple datacenters or cloud regions. Some of these servers form the storage layer, 
  called the brokers. The other servers run Kafka connect to continuously import/export data.
- **Consumer group**: Set of consumers which cooperate to consume from some topics. The partitions of all topics are divided amongst the consumers of the group. 
  Hence, a consumer can be assigned to multiple partitions, however, messages in a partition are consumed by a single assigned consumer. 
  There can be multiple consumers from different consumer groups reading from the same partition. This means, a unique message can be consumed by only one 
  consumer per group.
- **Group rebalance**: When consumers join/leave, the partitions are re-assigned known as rebalancing the group. The group management uses a grouping protocol 
  built into Kafka itself, and not zookeeper. One of the brokers is made the coordinator for group management. The leader of the topic *__consumer_offsets* topic
  is chosen as the coordinator.
- **Offset management**: When a new group is created, the group uses the offset reset policy *auto.offset.reset* to determine the offset position. Typycally, it 
  is either latest offset or earliest offset. Correct offset management is crucial because it affects **delivery semantics**.
  
### Kafka producer delivery semantics

**At-least once**: Each message is published into Kafka at least once, which means there might be duplicates. 
When the producer receives an ACK from the Kafka broker, it means that the broker has received the message. This can be done by setting ACK value > 0. If the topic han been configured to have replication, then setting ACK = 1 will only wait for an acknowledgement from one of the replicas. This can still lead to data loss once its published if the message is not successfully replicated (leader broker goes down before replication, the other brokers will not get the data). 
To ensure at-least once delivery, set the *acks* = all and the min.insync.replicas to ensure broker sends ACK only once the min insync replicas have received the message. If there is an error or ACK times out, the message producer retries which may cause duplicate publish.
      
**At-most once**: A message is delivered maximum only once. No duplicates possible, but a message may be missed.
This is equivalent to set-and-forget technique, where the producer sends the message and does not wait for the ACK. If the ACK timesout or fails, there is no retry attempt made, hence the message is not delivered to the consumer. 

**Exactly-once**: Each message is delivered once and only once. This is now possible with kafka version >= 0.11.x. The producer send operation can be made idempotent by setting *enable.idempotence* configuration to true. If set per partition, the retries config will default to Integer. MAX_VALUE and the acks config will default to all. This allows for no duplicates, no data loss, and in-order semantics.

#####How it works 
When a producer retries on error, the same message — which is still sent by the producer multiple times — will only be written to the Kafka log on the broker once. This is because each message will now hold a sequence number which will be used for de-duplication. This sequence number is persisted to replicated log, even if the leader broker crashes, the log is not lost.
  
### Kafka consumer delivery semantics

**At-least once**: Every message is processed, but duplicate processing is possible. This is the default configuration.
By default, a consumer is configured for auto commit. Auto commit works as cron job which periodically commits the offset on a regualr interval set using *auto.commit.offset.ms* (default - 5s). In such cases, if the consumer crashes, after rebalance, the new consumer picks messages from the last committed offset, which could be as old as the auto commit interval, causing duplication.

Remedies to reduce duplicates:
- The commit interval can be shortened.
- To avoid auto commit, set *enable.auto.commit* to false and use synchronous commits
  - Cons: This can lead to performance setback because consumer is blocked until offset is commited.
- Increase the amount of data fetched to improve synchronous commit performance
  - Cons: This also increases amount of duplication in worst case failures.
- Use async commits
  - Pros: 
    - consumer is not blocked until offset is committed
  - Cons:
    - Async commits have no retry. Sync commits retry by default until they succeed, or until an unrecoverable error has occured. Why? To maintain ordering. 
      A later async commit could potentially be overriden by the previous async commit eventually succeeding after retry. A retry of an old commit can cause 
      duplication.
      Remedy: You can use the onSuccess/onFailure calling to retry the commit, if needed, but you need to deal with the ordering problem.
    - If the last async commit fails before a rebalance/shut down, it will cause duplicates.
      Remedy: Hook into close/rebalance events and perform sync commits. For rebalance, there are 2 steps - partition revoke and partion re-assignment. The 
      synchronous offset commit can be performed before partition revokement, so that the subsequent consumer assignd can start from the committed offset.
      
**At-most once**: A message is processed maximum only once. No duplicates possible, but a message may be missed.
This can be achieved by synchronously committing the offset before processing the message. In case commit fails, the mssage is not processed. If the commit succeeds, the message is processed and the message at the next offset is subsequently consumed. An alternative method is to use async commit and unread the message if commit fails.

**Exactly-once**: Each message is consumed one and only once. This semantic is not supported by Kafka consumers. However, this can be achieved in data systems by extending the at-least once semantic with a primary key to allow for de-duplication.

