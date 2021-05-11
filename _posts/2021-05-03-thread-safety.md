---
layout: post
title:  Thread Safety
categories: [Thread,Concurrency]
---

##### Terms
**State variable** - Instance or static fields which stores the state of the object and can affect the object's externally visible behaviour. 
**Shared state** - A variable that can be accessed by multiple threads.
**Mutable** - variable's value that could change during its lifetime.
**Invariant** - It is a logical assertion that is held to always be true during a certain phase of execution. For example - a binary search tree has the following invariant: For any node n, every node in the left subtree of n has a value less than n's value, and every node in the right subtree of n has a value greater than n's value. We call that the BST invariant.
**Reentrancy** - If a thread tries to acquire a lock that it already holds, the request succeeds. This is due to reentrancy. It is implemented by associating with each lock an acquisition count and an owning thread. When the acquisition count becomes zero, the lock is released. An unheld lock can be then acquired by a different thread.
**Volatile** - Alternative, weaker form of synchronization to ensure that updates to a variable are propagated predictably to other threads.

##### Three ways to ensure thread safety:
* Do not share state variables across threads OR
* Make the state variable immutable OR
* Use synchronization whenever accessing state variables

##### Thread safety points to note:

* Atomicity - Beware of the following operations prone to race conditions: 
```
  @NotThreadSafe
  public class NoVisibility {
    private static boolean read;
    private static int number;

    private static class ReaderThread extends Thread {
      public void run() {
        while(!ready)
           Thread.yield();
        System.out.println(number);
      }
    }
    public static void main(String[] args) {
      new ReaderThread().start();
      number = 42;
      read = true;
    }
  }    
  ```
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
        private AtomicInteger childAge;
        private AtomicInteger momAge;
        private AtomicInteger dadAge;

        public Integer[] getAges() {...}
        public void afterXYears(int x) {
         ...
         childAge.addAndGet(x);
         momAge.addAndGet(x);
         dadAge.addAndGet(x);
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
        private long childAge;
        private long momAge;
        private long dadAge;

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
      **OR**
      ```
      // Make it immutable
      @ThreadSafe
      public class FamilyAge {
        private final int childAge;
        private final int momAge;
        private final int dadAge;
        
        public FamilyAge(int childAge, int dadAge, int momAge) {
          this.childAge = childAge;
          this.dadAge = dadAge;
          this.momAge = momAge;
        }
        
        public FamilyAge afterXYears(int x) {
          int cAge = this.childAge + x;
          int dAge = this.dadAge + x;
          int mAge = this.momAge + x;
          return new FamilyAge(cAge, dAge, mAge);
        }
      }    
      ```
* Visibility - *synchronized* keyword is not only about atomicity or demarcating *critical sections*, it also provides *memory visibility*. Without    synchronization, change in variable state may not be visible to other threads and moreover, can also lead to a phenomemon called *reordering*. Example: This program may loop forever since the value of read may never be visible. It could also yield 0 if value of read is visible, but not of number, due to reordering. Potential solution is to use the synchronized keyword around shared data (recommended) or using volatile variables. *Locking can guarantee both visibility and atomicity; volative variables can only guarantee visibility.*
* Nonatomic 64 bit operations - JVM is permitted to treat 64-bit read or write as 2 separate 32-bit writes. This can lead to a shared long value to have stale high 32 bits and latest low 32 bits. Solution: Declare them volatile, AtomicLong or guarded by lock.

##### Thread safety techniques:

* Immutable objects are always thread safe.
  An object is immutable if:
   * Its state cannot be mofified after construction;
   * All its fields are final; (its possible without this, but very very hard); and 
   * It is properly constructed (the *this* reference does not escape during contruction)
  Immutable objects are hard to create however:
  For example, the below does not satisfy the first condition above: 
  ```
  class UnsafeStates {
    private final String[] states = new String[] { "AK", "AL", "ON" };
    public String[] getStates() {
        return states;
    }
  }
  
  public static void main(String[] args) {
    UnsafeStates unsafeStates = new UnsafeStates();
    String[] states = unsafeStates.getStates();
    states[0] = "CA";
    for (String state : unsafeStates.getStates()) {
        System.out.print(state + " ");  // will print CA AL ON
    }
  }
  ```
  To make the above class immutable:
  ```
  class SafeStates {
    private final String[] states = new String[] { "AK", "AL", "ON" };
    public String[] getStates() {
        return Arrays.copyOf(states, states.length);
    }
  }
  ```
  To make the above class safely mutable
  ```
  class SafeMutableStates {
    private final Set<String> states = new HashSet<>();
    public synchronized void addState(String s) {
        states.add(s);
    }
    public synchronized boolean containsState(String s) {
        return states.contains(s);
    }
  }
  ```
* Mutable objects must be safely published i.e both the reference to the object and the object's state must be visible to other threads at the same time
  * Initialize an object reference from static initializer *public static Holder holder = new Holder(42);*
  * storing a reference to it into a volatile field or atomicReference 
    ```
    public AtomicReference<Holder> holderRef; 
    public void initialize() {
      Holder holder = new Holder(42);
      holderRef = new AtomicReference<Holder>(holder);
    }
    ```
  * storing a reference to it into a final field of a properly constructed object *public final Holder der = new Holder(42);*
  * storing a reference to it into a field that is guarded by a lock
* Using ThreadLocal : provides get and set accessor methods that maintain a separate copy of the value for each thread that uses it, so a get returns the most recent value passed to set *from the currently running thread*. ThreadLocal<T> can be imagined to be similar to Map<Thread, T>
* Create final fields unless they need to be mutable. However, please note, as described in point 1, declaring a field *final* may not guarantee its immutability.
* Java monitor pattern (using the object's own intrinsic lock to guard mutable state encapsulated by the object). Example - SafeMutableStates class. You could also use a private lock. Private lock disables client from acquiring the lock, allowing for more safety.
  ```
  public class SafeMutableStatesWithPrivateLock {
    private final Object lock = new Object();
    private final Set<String> states = new HashSet<>();
    public void addState(String s) {
      synchronized(lock) { ...
      }
    }
    public boolean containsState(String s) {
      synchronized(lock) { ...
        return states.contains(s);
      }
    }
  }
  ```

##### Common mistakes:

**Mistake 1**: Getters do not need synchronization
It is a common mistake to assume that synchronization needs to be used only when writing to shared variables; this is simply not true.

For each mutable state variable that may be accessed by more than one thread, **all accesses** to that variable must be performed with the same lock held. In this case, we say that the variable is guarded by that lock.

**Mistake 2**: As long as each mutable state is guarded by a lock, the code is thread safe
For every invariant that involves more than 1 variable, **all** the variables involved in that invariant must be guarded by the **same** lock.

**Mistake 3**: Concurrent collections do not need additional synchronization
```
if(!vector.contains(element))
  vector.add(element)
```
Even though vector provides thread safe methods, a race condition such as above (put-if-absent), which contains an atomicity violation bug. Some fixes include the below:

[Source credits link](http://dig.cs.illinois.edu/papers/checkThenAct.pdf)
<img width="459" alt="Fixes for race conditions on concurrent collections" src="https://user-images.githubusercontent.com/44378362/117681504-2a631980-b180-11eb-80dc-091378410832.png">
