# Structured Concurrency in Java 21

It was a Friday night and we'd gone out for some drinks with a few colleagues.
We ended up at one of those places where the drinks aren't listed on the menu, so when the waiter asked us what drink we wanted we just asked him if he could list a few.
He was being very thorough, because he ended up listing all the options, and by the time he got to the 25th item some of us were like: "What was the first option again?" 🙈
We could tell he was trying to be patient with us, but his list recital had been for nothing and he couldn't completely hide his disappointment.
Did you know that concurrent programming with Java can sometimes be exactly like that?
Because if you configure a few threads to do work in parallel, potentially some of the work you let them do will be for nothing.
Java 21 will introduce 'structured concurrency' as a preview feature, allowing you to prevent this unnecessary work.

## Unstructured Concurrency

So what 'unnecessary work' are we even talking about here? 
Well, it has everything to do with Java's current implementation of concurrency, which is _unstructured_.
Tasks run independently of each other, without any hierarchy, scope, or other structure, which means they cannot easily pass errors or cancellation attempts to each other.
To illustrate this, let's look at a code example that's in the same domain as the article's introduction:

```java
    public MultiCourseMeal announceMenu() throws ExecutionException, InterruptedException, OutOfStockException {
        Waiter grover = new Waiter("Grover");
        Waiter zoe = new Waiter("Zoe");
        Waiter rosita = new Waiter("Rosita");

        Future<Course> starter;
        Future<Course> main;
        Future<Course> dessert;

        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            starter = executor.submit(() -> grover.announceCourse(CourseType.STARTER));
            main = executor.submit(() -> zoe.announceCourse(CourseType.MAIN));
            dessert = executor.submit(() -> rosita.announceCourse(CourseType.DESSERT));
        }

        return new MultiCourseMeal(starter.get(), main.get(), dessert.get());
    }
```

So suppose the place where we confused our waiter earlier also has multi-course meals on offer.
Then, in theory, the restaurant could choose to have three waiters announce today's courses.
In principle this could work very well using multiple threads.
The different courses might be announced in the wrong order, but that's the worst thing that can happen.
Or is it?

Consider the fact that the `announceCourse(..)` method could fail by throwing an `OutOfStockException` if one of the ingredients for the course is out of stock.
When that occurs, the `announceMenu()` method can no longer construct and return a valid instance of `MultiCourseMeal`.

And now you can probably think of a few things that are wrong with this piece of code:

* If `zoe.announceCourse(CourseType.MAIN)` takes a long time to execute but `grover.announceCourse(CourseType.STARTER)` fails in the meantime, the `announceMenu(..)` method will unnecessarily wait for the main course announcement by blocking on `main.get()` instead of cancelling it.
- If `zoe.announceCourse(CourseType.MAIN)` throws an exception, `main.get()` will throw an exception, but `grover.announceCourse(CourseType.STARTER)` will continue to run in its own thread, resulting in thread leakage.
- If the thread executing `announceMenu(...)` is interrupted, the interruption will not propagate to the subtasks: all threads that run the `announceCourse(..)` invocations will leak, continuing to run even after `announceMenu()` has failed.

Ultimately the problem here is that our program is logically structured with task-subtask relationships, but these relationships exist only in the developer's mind. 
We might prefer structured code that reads like a sequential story, but this example simply doesn't meet that criterion.
And that makes it a classic example of Java's _unstructured concurrency_.

In contrast, the execution of single-threaded code always enforces a hierarchy of tasks and subtasks.
Consider the single-threaded version of the `announceMenu()` method:

```java
public MultiCourseMeal announceMenu() throws OutOfStockException {
    Waiter elmo = new Waiter("Elmo");

    Course starter = elmo.announceCourse(CourseType.STARTER);
    Course main = elmo.announceCourse(CourseType.MAIN);
    Course dessert = elmo.announceCourse(CourseType.DESSERT);

    return new MultiCourseMeal(starter, main, dessert);
}
```

Here, we don't have any of the problems we had before.
The subtasks (announcing the courses) will be executed in the right order, and if one subtask fails the remaining one(s) won't even be started.
And because all work runs in the same thread, there is no risk of thread leakage.

So from these two examples it is evident that concurrent programming would be a lot easier and more intuitive if it would be able to enforce the hierarchy of tasks and subtasks, just like single-threaded code can.
This is where structured concurrency comes in!

## Structured Concurrency

The term _structured concurrency_ originates from the 1960s with the fork-join model (TODO: citation), but the concept was formulated by Sústrik in 2016 for goroutines (TODO: citation). 
Independently, Elizarov came up with the same concept for Kotlin's coroutines (TODO: citation). 
In this approach, threads have a clear hierarchy, their own scope, and entry and exit points. 
Similar to function calls, it creates a tree of threads with parent-child relationships. 
A scope continues until all child threads have completed. 
This streamlined error and cancellation propagation leads to improved reliability and observability. 
There is a strict nesting of the lifetimes of operations in a way that mirrors their syntactic nesting in the code.

Let's look at a structured, concurrent version of our menu announcement code:

TODO

```java
Response handle(int userId, int orderId) 
	throws ExecutionException, InterruptedException {
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<User>  u = scope.fork(() -> service.find(userId));
    Future<Order> o = scope.fork(() -> service.fetch(orderId));

    scope.join();          // 1
    scope.throwIfFailed(); // 2
    
    return new Response(u.resultNow(), o.resultNow()); // 3
  }
}
```

The main difference from the previous code is that here we create the threads (`fork`) within a new `scope`. This scope "keeps the threads together." At `1`, we wait (`join`) until _both_ threads are ready. If one of the threads is interrupted, it throws an `InterruptedException` here. At `2`, an `ExecutionException` can be thrown if an exception occurs in one or both of the threads. We have chosen to propagate this error(s). This is optional but considered good practice. Once we reach `3`, everything has gone well, and we can retrieve and process the results.

### How to Use
Structured concurrency has been available as an incubating feature since Java 19 but has been promoted to a preview feature in Java 21. This means that in Java 21, you only need to use the compiler option `--enable-preview` to start working with it, and it is guaranteed that this feature, possibly in a slightly updated form, will be included in a later release.

### Structured Concurrency and Virtual Threads

[name]

* Why are they a good match?

### Structured Code Example

[name]

* Same code example, but using `StructuredTaskScope`

###





## SIDEBAR: Concurrent Programming Refresher

Concurrency is closely related to threads, so let's start with a quick refresher on the key concepts.

In a computer, instructions (your program code) are executed by a process. The operating system assigns resources, such as memory, to the process. Processes are independent of each other and have their own memory address space. We also refer to a process as a "unit of resources."

A process consists of one or more threads. A thread represents the "unit of execution": the smallest possible execution of a sequence of programmed instructions. Multiple threads within a single process share various things, such as resources, executable code, memory address space, and process state. One thread can cause the process and all other threads to crash.

OPTIONAL: The term "thread" stands for "thread of execution": it involves a fast alternation between execution and waiting. A thread needs a CPU to do its work. Only one thread can be active on a CPU at the same time. The OS scheduler determines when a thread can use the CPU. If you horizontally plot the active and inactive periods of a thread, you get a dotted line: "a group of filaments twisted together."

OPTIONAL: Vyssotsky first introduced this term in 1966. Threads gained popularity in the 1990s as applications sought to make better use of available resources and create the impression of executing multiple tasks simultaneously. With the advent of multiple CPU cores in the 2000s, multithreading further evolved.

## SIDEBAR: Concurrency vs. Parallelism

Since only a single thread can run on a CPU core at a time, there is competition among multiple threads if they all need it. This competition for the CPU is called **concurrency**. The challenge is to find the most efficient distribution over time. The measure of concurrency is **throughput**: the _number_ of tasks we can process per unit of time.

**Parallelism**, on the other hand, involves dividing a single task "over space": we split the task into collaborating subtasks across multiple CPUs. The measure of parallelism is **latency**: the _duration_ of an individual task.

## SIDEBAR: Unstructured vs. Structured

Let's discuss the terms unstructured and structured. Let's first look at programming sequential code.

We're probably all familiar with the term 'spaghetti code'. This metaphor arises from code that uses `goto` statements for flow control, which involves conditionally jumping to other lines. It's a "one-way jump." (Did you know that `goto` is a keyword in Java?) Such code is difficult to read, understand, and debug. It is unstructured.

On the other hand, structured programming is a paradigm that uses control flow statements such as `if/else` and `while/for`, along with block structures and subroutines. The term was coined by Dutch computer scientist Dijkstra in 1968. Spaghetti code transforms into a hierarchy of enclosed and cohesive code blocks, each with its own _scope_. Code blocks (e.g. 'functions') can communicate with each other through clear entry and exit points, allowing for a 'two-way jump': you return after invoking.