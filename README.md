# Ordered Parallel Processor [![Build Status](https://travis-ci.org/Ullink/ordered-parallel-processor.svg?branch=master)](https://travis-ci.org/Ullink/ordered-parallel-processor)
Ordered processing going parallel

Inspired by the article [Exploiting Data Parallelism in Ordered Data Streams](https://software.intel.com/en-us/articles/exploiting-data-parallelism-in-ordered-data-streams)
from the [Intel Guide for Developing Multithreaded Applications](https://software.intel.com/en-us/articles/intel-guide-for-developing-multithreaded-applications).

This implementation brings a lightweigth solution for unlocking code that it only synchronized because of ordered/sequential requirements.

## Use case

Multiple threads concurrently execute the following code.

```java
synchronized(this)
{
  A = read();
  B = process(A);
  write(B);
}
```

We need to have all operations in the same order to guarantee consistency between the read() and the write() ordering.

Not very efficient because only 1 thread can EncodeProcessing() at a time.
And Thread n+1 can't WriteToNetwork() while Thread n moved to WriteToDisk()

### Ordered Parallel

```java
OrderedScheduler scheduler = new OrderedScheduler()
OrderedPipe pipe1 = scheduler.createPipe();
OrderedPipe pipe2 = scheduler.createPipe();

public void execute()
{
  FooInput input;
  synchronized (this)
  {
    // ticket will "record" the ordering of read() calls, and use it to guarantee same write() ordering
    ticket = scheduler.getNextTicket();
    input = read();
  }
  
  try (ticket)
  {
    // this will be executed concurrently (obviously needs to be thread-safe)
    BarOutput output = process(input);
    
    // each pipe will be sequentialy processed (in the order of the ticket)
    // pipe.run() will return true if the task was executed by the current thread, and false if it will be executed by another thread
    pipe1.run(ticket, () => { write1(output); } );
    pipe2.run(ticket, () => { write2(output); } );
  }
}
```
