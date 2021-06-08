---
layout: post
title:  Distributed Systems
categories: [DS, Udemy]
---

#### Terms
* Process - An instance of an application/program (.jar, .exe, etc) created in the memory (RAM) by the Operating System. 
  It includes the memory and resources needed for the program to operate, such as registers, counters, stack, heap and instructure (code)
  Register - Data holding palces part of CPU
  Counter - Instruction pointer, keeping track of where the computer is in the program sequence
  Stack - Data structure storing info on the active subroutines in the computer program, also known as scratch space for the process
  Heap - Dynamically allocated memory for the process.
  
* Thread - Unit of execution within a process. Process could be single threaded (process and thread are one and the same, and there is only one thing happening).
  Multi threaded processes have more than 1 thread working at the same time (well, almost same time). 
![Screen Shot 2021-05-17 at 11 46 47 AM](https://user-images.githubusercontent.com/44378362/118518126-d323f280-b705-11eb-8919-283174783941.png)

* Distributed system - System of several processes, working on different machines, communicating through a network, 
  sharing a state or are working together to achieve a common goal

##### How to distribute work amongst different machines:
* Master - Workers architecture
  * Elect a leader - Prone to SPOF (single point of failure)
  * Automated leader election - agreement mechanism, service registry and discovery for nodes to know about each other and failure detection mechanism.

##### Apache Zookeeper
HA and high reliability coordination service used by Kafka, Hadoop, etc).
Runs in a cluster of an odd number of nodes (>= 3), using redundancy to allow failures

A distributed system can use a zookeeper cluster for coordination and distributed management instead of handling it themselves. Every node in a ZooKeeper tree is refered to as a znode which maintains the stat structure.
Two types of znodes:
* Persistent - Persists between session
* Ephemeral - Deleted once session ends.
The zNodes, are added with a SEQUENCE|EPHEMERAL flag. The node with the smallest appended sequence number becomes the leader. 

A client (the process that uses zookeeper) interacts with the zookeeper service through znodes, and can set watches. Any event (connection/nodecreation/deletion etc) triggers a notification to the client watching the znode. A watch event is a one-time trigger, sent to the client that sets the watch, which occurs when the data for which the watch was set changes.  

##### Herd Effect: This is one of the potential design issues with leader election.
1. Large number of nodes wait for an event, in this case about the leader failing.
2. When the event occurs, all nodes are notified and they all wake up, even though only one of them finally succeeds

Example - If a zookeeper cluster were to use the above design, herd effect might potentially cause severe performance issues. 
