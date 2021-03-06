---
layout: default
title: Superman Problem
parent: Interview Practice Problems
grand_parent: Multithreading
has_children: true
nav_order: 11
permalink: /multithreading/problems/superman
---
<div align="center" markdown="1">
Superman Problem / Java resources / Tutorial

{: .fs-8 .fw-400 }
</div>

## Superman Problem

## Problem Statement
You are designing a library of superheroes for a video game that your fellow developers will consume. Your library should always create a single instance of any of the superheroes and return the same instance to all the requesting consumers.

Say, you start with the class Superman. Your task is to make sure that other developers using your class can never instantiate multiple copies of superman. After all, there is only one superman!

## Solution
You probably guessed we are going to use the singleton pattern to solve this problem. The singleton pattern sounds very naive and simple but when it comes to implementing it correctly in Java, it's no cakewalk.

First let us understand what the pattern is. A singleton pattern allows only a single object/instance of a class to ever exist during an application run.

There are two requirements to make a class adhere to the singleton pattern:
* Declaring the constructor of a class private. When you declare the Superman class's constructor private then the constructor isn't visible outside the class or in its subclasses. Only the instance and static methods of the Superman class are able to access the constructor and create instances of the Superman class.
* The second trick is to create a public static method usually named getInstance() to return the only instance. We create a private static object of the class Superman and return it via the getInstance() method. We can control when to instantiate the lone static private instance. Here's what the code looks like:

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter49.png)

Here's what your interviewer will tell you when you write this code:
* What if the no one likes Superman and instead creates Batman in the game. You just created Superman and he kept waiting without ever being called upon to save the world. It's a waste of Superman's time and also the memory and other resources he'll consume.
* What if the creation of Superman is a very resource-intensive effort after all he's coming from planet Krypton. We would really like to only create Superman once we need him. Or in programming-speak, we want to lazily initialize Superman

The next version is what most candidates would write and is incorrect.

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter50.png)

As any reader of this course should realize by now (if I have done a good job of teaching) that the getInstance() method would fail miserably in a multi-threaded scenario. A thread can context switch out just before it initializes the Superman, causing later threads to also fall into the if clause and end up creating multiple superman objects.

The naive way to fix this issue is to use our good friend synchronized and either add synchronized to the signature of the getInstance() method or add a synchronized block within the method body. Thee mutual exclusion ensures that only one thread gets to initialize the object.

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter51.png)

The con of the above solution is that every invocation of the getInstance() method causes the invoking thread to synchronize, which is prohibitively more expensive in terms of performance than non-synchronized snippets of code. Can we synchronize only when initializing the singleton instance and not at other times? The answer is yes and leads us to an implementation known as double checked locking. The idea is that we do two checks for superman == null in a nested fashion. The first check is without synchronization and the second with. Once a singleton instance has been initialized, all future invocations of the getInstance() method don't pass the first null check and return the instance without getting involved in synchronization. Effectively, threads only synchronize when the singleton instance has not yet been initialized.

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter52.png)

The above solution seems almost correct. In fact, it'll appear correct unless you understand how the intricacies of Java's memory model and compiler optimizations can affect thread behaviors. The memory model defines what state a thread may see when it reads a memory location modified by other threads. The above solution needs one last missing piece but before we add that consider the below scenario:
1. Thread A comes along and gets to the second if check and allocates memory for the superman object but doesn't complete construction of the object and gets switched out. The Java memory model doesn't ensure that the constructor completes before the reference to the new object is assigned to an instance. It is possible that the variable superman is non-null but the object it points to, is still being initialized in the constructor by another thread.
2. Thread B wants to use the superman object and since the memory is already allocated for the object it fails the first if check and returns a semi-constructed superman object. Attempt to use a partially created object results in a crash or undefined behavior.

To fix the above issue, we mark our superman static object as volatile. The happens-before semantics of volatile guarantee that the faulty scenario of threads A and B never happens.

By now you'll probably appreciate how hard it is to get the singleton pattern right in a multithreaded scenario. Also, note that the discussed solution works with Java 1.5 or above. volatile's behavior in Java 1.4 and earlier is different and the double checked locking pattern is broken when run on those versions of Java.

Last but not the least, double-checked locking (DCL) is an antipattern and its utility has dwindled over time as the JVM startup and uncontended synchronization speeds have improved.

The next lesson explains alternate Singleton implementations in Java.

## Complete Code

The complete code appears below:

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter53.png)

Implementing a thread-safe Singleton class is a popular interview question with the double checked locking as the most debated implementation. For completeness, we present below all the various evolutions of the Singleton pattern in Java.
* The easiest way to create a singleton is to mark the constructor of the class private and create a private static instance of the class that is initialized inline. The instance is returned through a public getter method. The drawback of this approach is if the singleton object is never used then we have spent resources creating and retaining the object in memory. The static member is initialized when the class is loaded. Additionally, the singleton instance can be expensive to create and we may want to delay creating the object till it is actually required.

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter54.png)

The above approach is known as Eager Initialization because the singleton is initialized irrespective of whether it is used or not. Also, note that we don't need any explicit thread synchronization because it is provided for free by the JVM when it loads the Superman class.

{% highlight java %}
class Demonstration {
    public static void main( String args[] ) {
        Superman superman = Superman.getInstance();
        superman.fly();
    }
}

class Superman {
    private static Superman superman = new Superman();

    private Superman() {
    }

    public static Superman getInstance() {
        return superman;
    }

    public void fly() {
      System.out.println("I am flyyyyinggggg ...");
    }
}
{% endhighlight %}

* The next approach is to lazily create the singleton object. Lazy intialization means delaying creating a resource till the time of its first use. This saves precious resources if the singleton object is never used or is expensive to create. First, let's see how the pattern will be implemented in a single threaded environment.

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter55.png)

With the above approach we are able to introduce lazy initialization, however, the class isn't thread-safe. Also, we needlessly check if the instance is null every time we invoke getInstance() method.

To make the above code thread safe we synchronize the getInstance() method and get a thread-safe class.

![inter](https://raw.githubusercontent.com/JavaLvivDev/prog-resources/master/resources/inter/inter56.png)

Note that the method is synchronized on the class object. The problem with the above approach is we are serializing access for threads even after the singleton object is safely initialized the first time. This slows down performance unnecessarily. The next evolution is to move the lock inside of the method.

{% highlight java %}
class Demonstration {
    public static void main( String args[] ) {
        Superman superman = Superman.getInstance();
        superman.fly();
        
    }
}

class Superman {
    private static Superman superman;

    private Superman() {
    }

    public synchronized static Superman getInstance() {

        if (superman == null) {
            superman = new Superman();
        }

        return superman;
    }

    public void fly() {
      System.out.println("I am flyyyyinggggg ...");
    }    
}
{% endhighlight %}

* The next evolution is the double checked locking that we have already discussed in the previous lesson.

* Another implementation of the singleton pattern is the holder or Bill Pugh's singleton. The idea is to create a private nested static class that holds the static instance. The nested class Helper isn't loaded when the outer class Superman is loaded. The inner static class Helper is loaded only when the method getInstance() is invoked. This saves us from eagerly initializing the singleton instance.

{% highlight java %}
class Demonstration {
    public static void main( String args[] ) {
        Superman superman = Superman.getInstance();
        superman.fly();
    }
}

class Superman {

    private Superman() {
    }

    private static class Holder {
        private static final Superman superman = new Superman();
    }

    public static Superman getInstance() {
        return Holder.superman;
    }

    public void fly() {
        System.out.println("I am flyyyyinggggg ...");
    }    
}

{% endhighlight %}