---
layout: post
title:  Producer-Consumer pattern
categories: [Thread,Concurrency, ProducerConsumer]
---

#### Blocking Queues:

  Blocking queues support the producer-consumer pattern. They provide blocking *put* and *take* methods as well as timed equivalents of poll and offer (insert).
  When consumers frequently wait until more work is available, it could indicate there is not much work to do after all or
  it could indicate that the ratio of producers to consumers needs to be adjusted.
  
  The blocking helps with avoiding out of memory issues with bounded blocking queues.
  Offer method does not block, instead returns a failure status if the item cannot be enqueued. This could help with taking steps to deal with overload, 
  reducing # of producer threads, or throttling producers.
  
  Types of blocking queues:
  * LinkedBlockingQueue - FIFO queue analogous to LinkedList
  * ArrayBlockingQueue - FIFO queue analogous to ArrayList
  * PriorityBlockingQueue - priority-ordered queue
  * SynchronousQueue - Not really a queue in that it maintains no storage space for queued elements. 
    It maintains a list of queued threads waiting to enqueue or dequeue an element. It performs a direct handoff of a task from producer to available consumer 
    and gives feedback about the state of the task to the producer. Put and take will block unless there is a thread waiting to participate.
    Generally only suitable when there are enough consumers to take the handoff. 
    
  Code excerpt for producer-consumer pattern (Pg. 92):
  ```
  public class FileCrawler implements Runnable {
    private final BlockingQueue<File> queue;
    private final File root;
    
    public FileCrawLer(BlockingQueue<File> queue, File root) { ... }
    
    public void run() {
      try {
        crawl(root); // this puts the crawled files in the root to the queue (queue.put(file))
      } catch(InterruptedException e) { ... }
    }
  }
  
  public class FileIndexer implements Runnable {
    private final BlockingQueue<File> queue;
    
    public FileIndexer(BlockingQueue<File> queue) { ... }
    
    public void run() {
      try {
        index(queue.take());
      } catch(InterruptedException e) { ... }
    }
  }
  
  public static void startProcessing(File[] roots) {
    final BlockingQueue<File> files;
    
    for(File root: roots) {
      new Thread(new FileCrawler(files, root)).start();
    }
    for(int i=0; i< NUM_CONSUMERS; i++) {
      new Thread(new FileIndexer(files)).start();
    }
  }
  ```
  
##### What do you do if the queue is interrupted?
  
*put* and *take* methods in BlockingQueue are blocking and hence interruptible.
There are 2 choices to handle an InterruptedException

1. Propagate the exception - Propagate to your caller - do not catch InterruptedExceotion or throw it again after performing cleanup. 
   If your code is part of a runnable, it cannot throw back an InterruptedException.
2. Restore the interrupt - catch and restore the interrupted status on the current thread (Thread.currentThread.interrupt()). 
   This can be done if your code is part of a runnable.
Only *swallow* an interrupt if your code is extending Thread and therefore you control all the code higher up on the call stack.
