---
layout: default
title: ReadWrite Lock
parent: Interview Practice Problems
grand_parent: Multithreading
has_children: true
nav_order: 5
permalink: /multithreading/problems/readWrite_lock
---
<div align="center" markdown="1">
ReadWrite Lock / Java resources / Tutorial

{: .fs-8 .fw-400 }
</div>

## ReadWrite Lock

## Problem Statement
Imagine you have an application where you have multiple readers and multiple writers. You are asked to design a lock which lets multiple readers read at the same time, but only one writer write at a time.

## Solution

First of all let us define the APIs our class will expose. We'll need two for writer and two for reader. These are:
* acquireReadLock
* releaseReadLock
* acquireWriteLock
* releaseWriteLock

This problem becomes simple if you think about each case:
1. Before we allow a reader to enter the critical section, we need to make sure that there's no writer in progress. It is ok to have other readers in the critical section since they aren't making any modifications
2. Before we allow a writer to enter the critical section, we need to make sure that there's no reader or writer in the critical section.

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter23.png)

Note that all the methods are synchronized on the ReadWriteLock object itself.

Let's start with the reader use case. We can have multiple readers acquire the read lock and to keep track of all of them; we'll need a count. We increment this count whenever a reader acquires a read lock and decrement it whenever a reader releases it.

Releasing the read lock is easy but before we acquire the read lock, we need to be sure that no other writer is currently writing. Again, we'll need some variable to keep track of whether a writer is writing. Since only a single writer can write at a given point in time, we can just keep a boolean variable to denote if the write lock is acquired or not. Let's translate what we have discussed so far into code.

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter24.png)

Note how we are checking in a loop whether isWriteLocked is true and then calling wait(). Also pay attention to the fact that the methods are synchronized so only one reader will be able to decrement reader in releaseReadLock.

For the writer case, releasing the lock would be as simple as setting the isWriteLocked variable to false but don't forget to call notify() too since there might be readers waiting in the acquireReadLock() method.

Acquiring the write lock is a little tricky, we have to check two things whether any other writer has already set isWriteLocked to true and also if any reader has incremented the readers variable. If isWriteLocked equals false and no reader is writing then the writer should proceed forward.

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter25.png)

The astute reader will notice a bug in the code we have so far. Try finding it, before reading ahead !

For the impatient, note that in our acquireWriteLock() method we have a while loop which has readers != 0 condition. We should remember to call notify() whenever any snippet of code can change the value of the readers variable and make the loop condition in acquireWriteLock() false. If we don't call notify then any thread waiting in the loop will never be woken up.

Within the releaseReadLock() method, we should call notify() after decrementing readers to make sure that any blocked readers should be able to proceed forward.

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter26.png)

Note that with the given implementation, it is possible for a writer to starve and never get a chance to acquire the write lock since there could always be at least one reader which has the read lock acquired.

## Complete Code
The complete code with a test-case appears below. Run the code and examine the output messages. We start a reader and a writer thread initially. The writer blocks until the read lock is released. Also, we release the reader-lock through another reader thread.

A second writer thread is blocked forever since the first writer thread never releases the write-lock. The execution eventually times out.

{% highlight java %}
class Demonstration {

    public static void main(String args[]) throws Exception {

        final ReadWriteLock rwl = new ReadWriteLock();

        Thread t1 = new Thread(new Runnable() {

            @Override
            public void run() {
                try {

                    System.out.println("Attempting to acquire write lock in t1: " + System.currentTimeMillis());
                    rwl.acquireWriteLock();
                    System.out.println("write lock acquired t1: " + +System.currentTimeMillis());

                    // Simulates write lock being held indefinitely
                    for (; ; ) {
                        Thread.sleep(500);
                    }

                } catch (InterruptedException ie) {

                }
            }
        });

        Thread t2 = new Thread(new Runnable() {

            @Override
            public void run() {
                try {

                    System.out.println("Attempting to acquire write lock in t2: " + System.currentTimeMillis());
                    rwl.acquireWriteLock();
                    System.out.println("write lock acquired t2: " + System.currentTimeMillis());

                } catch (InterruptedException ie) {

                }
            }
        });

        Thread tReader1 = new Thread(new Runnable() {

            @Override
            public void run() {
                try {

                    rwl.acquireReadLock();
                    System.out.println("Read lock acquired: " + System.currentTimeMillis());

                } catch (InterruptedException ie) {

                }
            }
        });

        Thread tReader2 = new Thread(new Runnable() {

            @Override
            public void run() {
                System.out.println("Read lock about to release: " + System.currentTimeMillis());
                rwl.releaseReadLock();
                System.out.println("Read lock released: " + System.currentTimeMillis());
            }
        });

        tReader1.start();
        t1.start();
        Thread.sleep(3000);
        tReader2.start();
        Thread.sleep(1000);
        t2.start();
        tReader1.join();
        tReader2.join();
        t2.join();
    }
}

class ReadWriteLock {

    boolean isWriteLocked = false;
    int readers = 0;

    public synchronized void acquireReadLock() throws InterruptedException {

        while (isWriteLocked) {
            wait();
        }

        readers++;
    }

    public synchronized void releaseReadLock() {
        readers--;
        notify();
    }

    public synchronized void acquireWriteLock() throws InterruptedException {

        while (isWriteLocked || readers != 0) {
            wait();
        }

        isWriteLocked = true;
    }

    public synchronized void releaseWriteLock() {
        isWriteLocked = false;
        notify();
    }
}
{% endhighlight %}