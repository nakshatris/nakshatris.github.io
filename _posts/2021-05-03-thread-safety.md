---
layout: post
title:  Thread Safety
categories: [Thread,Concurrency]
---

##### Terms
State variable - Instance or static fields which stores the state of the object and can affect the object's externally visible behaviour. 
Shared state - A variable that can be accessed by multiple threads.
Mutable - variable's value that could change during its lifetime.
Invariant - It is a logical assertion that is held to always be true during a certain phase of execution. For example - a binary search tree has the following invariant: For any node n, every node in the left subtree of n has a value less than n's value, and every node in the right subtree of n has a value greater than n's value. We call that the BST invariant.
Reentrancy - If a thread tries to acquire a lock that it already holds, the request succeeds. This is due to reentrancy. It is implemented by associating with each lock an acquisition count and an owning thread. When the acquisition count becomes zero, the lock is released. An unheld lock can be then acquired by a different thread.

##### Three ways to ensure thread safety:
* Do not share state variables across threads
* Make the state variable immutable
* Use synchronization whenever accessing state variables

##### Things to note:

* Atomicity - Beware of the following operations prone to race conditions: 
  1. read-modify-write
      ```
      @NotThreadSafe
      public class Counter {
        private long count = 0;

        public long getCount() {...}
        public void update() {
         ...
         count++;   // read-modify-write opration: fetch the current value, add one to it, write the new value back.
        }
      }
      ```
      Possible solutions to ensure thread safety: Use AtomicLong, or use synchronized keyword on update

  2. check-then-act 
      ```
      @NotThreadSafe
      public class Singleton() {
        private static Singleton instance;

        private Singleton() {..}

        public Singleton getInstance() {
          if (instance == null) {         // check-then-act operation
            instance = new Singleton();
          }
          return instance;
        }
      }
      ```
      Possible solution: Add synchronized keyword to method or instance variable.
      
  3. Compound actions - related state variables A and B are atomic individually, but may not be atomic with respect to each other.
      ```
      // Invariant: child age < mom age, child age < dad age
      @NotThreadSafe
      public class FamilyAgeCounter {
        private AtomicLong childAge;
        private AtomicLong momAge;
        private AtomicLong dadAge;

        public long[] getAges() {...}
        public void afterXYears(int x) {
         ...
         childAge += x;
         momAge += x;
         dadAge += x;
         ...
        }
      }    
      ```
      Even though childAge(10), momAge(35)  and dadAge(35) are individually atomic, there is a window where thread A calls afterXYears(30) and childAge is updated(40), but momAge(35) and dadAge(35) are not updated, thread B calling getAges() during the window will see that the invariant does not hold.

      Solution: Update related state variables in a single atomic operation.
      ```
      // Invariant: child age < mom age, child age < dad age
      @ThreadSafe
      public class FamilyAgeCounter {
        private AtomicLong childAge;
        private AtomicLong momAge;
        private AtomicLong dadAge;

        public synchronized long[] getAges() {...}
        public void afterXYears(int x) {
         ...
         synchronized(this) {
           childAge += x;
           momAge += x;
           dadAge += x;
         }
         ...
        }
      }    
      ```
  4. fdfds

##### Common mistakes:

1. Getters do not need synchronization: It is a common mistake to assume that synchronization needs to be used only when writing to shared variables; this is simply not true.

For each mutable state variable that may be accessed by more than one thread, **all accesses** to that variable must be performed with the same lock held. In this case, we say that the variable is guarded by that lock.
      
