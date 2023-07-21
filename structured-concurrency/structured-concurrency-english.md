# How To Do Structured Concurrency in Java 21

It was a Friday night and we'd gone out for some drinks with a few colleagues.
We ended up at one of those places where the drinks aren't listed on the menu, so when the waiter asked us what drink we wanted we just asked if he could list a few.
He was being very thorough, because he started listing all the options, and by the time he got to the 25th item some of us were like: "What was the first option again?" 🙈
We could tell he was trying to be patient, but his list recital had been for nothing and he couldn't completely hide his disappointment.
Concurrent programming with Java can sometimes be exactly like that.
When you configure a few threads to do work in parallel, some of the work you let them do could potentially be for nothing.
Java 21 will introduce 'structured concurrency' as a preview feature, allowing you to prevent unnecessary work like this.

## Unstructured Concurrency

So what 'unnecessary work' are we talking about here?? 
Well, it has everything to do with Java's current implementation of concurrency, which is _unstructured_.
Tasks run independently of each other, without any hierarchy, scope, or other structure, which means they cannot easily pass errors or cancellation intent to each other.
To illustrate this, let's look at a code example that could've easily taken place at the same restaurant where our patient waiter has been working his Friday nights:

```java
public class MultiWaiterRestaurant implements Restaurant {
    @Override
    public MultiCourseMeal announceMenu() throws ExecutionException, InterruptedException {
        Waiter grover = new Waiter("Grover");
        Waiter zoe = new Waiter("Zoe");
        Waiter rosita = new Waiter("Rosita");

        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            Future<Course> starter = executor.submit(() -> grover.announceCourse(CourseType.STARTER));
            Future<Course> main = executor.submit(() -> zoe.announceCourse(CourseType.MAIN));
            Future<Course> dessert = executor.submit(() -> rosita.announceCourse(CourseType.DESSERT));

            return new MultiCourseMeal(starter.get(), main.get(), dessert.get());
        }
    }
}
```

It seems like this restaurant also has multi-course meals on offer!
Then, in theory, the restaurant could choose to have three waiters announce today's courses - one for each course.
In principle this could work very well using multiple threads.
The different courses might be announced in the wrong order, but that's probably the worst thing that can happen.
Or is it?

Consider the fact that the `announceCourse(..)` method in the `Waiter` class could fail by throwing an `OutOfStockException` if one of the ingredients for the course is currently not in stock.
When that happens, the `announceMenu()` method can no longer construct and return a valid instance of `MultiCourseMeal`.

And now you can probably think of a few things that are wrong with this piece of code:

* If `zoe.announceCourse(CourseType.MAIN)` takes a long time to execute but `grover.announceCourse(CourseType.STARTER)` fails in the meantime, the `announceMenu(..)` method will unnecessarily wait for the main course announcement by blocking on `main.get()`, instead of cancelling it (which would be the sensible thing to do).
- If an exception happens in `zoe.announceCourse(CourseType.MAIN)`, `main.get()` will throw it, but `grover.announceCourse(CourseType.STARTER)` will continue to run in its own thread, resulting in thread leakage.
- If the thread executing `announceMenu(...)` is interrupted, the interruption will not propagate to the subtasks: all threads that run an `announceCourse(..)` invocation will leak, continuing to run even after `announceMenu()` has failed.

Ultimately the problem here is that our program is logically structured with task-subtask relationships, but these relationships exist only in the mind of the developer. 
We might all prefer structured code that reads like a sequential story, but this example simply doesn't meet that criterion.
And that makes it a classic example of Java's _unstructured concurrency_.

In contrast, the execution of single-threaded code _always_ enforces a hierarchy of tasks and subtasks.
Consider the single-threaded version of our restaurant example:

```java
public class SingleWaiterRestaurant implements Restaurant {
    @Override
    public MultiCourseMeal announceMenu() throws OutOfStockException {
        Waiter elmo = new Waiter("Elmo");

        Course starter = elmo.announceCourse(CourseType.STARTER);
        Course main = elmo.announceCourse(CourseType.MAIN);
        Course dessert = elmo.announceCourse(CourseType.DESSERT);

        return new MultiCourseMeal(starter, main, dessert);
    }
}
```

Here, we don't have _any_ of the problems we had before.
Our waiter Elmo will announce the courses in exactly the right order, and if one subtask fails the remaining one(s) won't even be started.
And because all work runs in the same thread, there is no risk of thread leakage.

So from these two examples it is evident that concurrent programming would be a lot easier and more intuitive if it would be able to enforce the hierarchy of tasks and subtasks, just like single-threaded code can.
This is where structured concurrency comes in!

## Structured Concurrency

The term _structured concurrency_ originates from the 1960s with the fork-join model, but the concept was formulated by Sústrik in 2016 for goroutines [1]. 
Independently, Elizarov came up with the same concept for Kotlin's coroutines [2].
So the feature has been prominent in both Go and Kotlin, and will soon also make its introduction in Java!

In a stuctured concurrency approach, threads have a clear hierarchy, their own scope, and clear entry and exit points. 
Just like with function calls, a tree of threads is created with parent-child relationships.
Moreover, a scope continues until all child threads have completed.
Structured concurrency yields a strict nesting of the lifetimes of operations in a way that mirrors their syntactic nesting in the code.
This streamlined error and cancellation propagation ultimately leads to improved reliability and observability in concurrent code.

## Shutdown on Failure

Let's now take a look at a structured, concurrent version of our menu announcement code in Java:

```java
public class StructuredConcurrencyRestaurant implements Restaurant {
    @Override
    public MultiCourseMeal announceMenu() throws ExecutionException, InterruptedException {
        Waiter grover = new Waiter("Grover");
        Waiter zoe = new Waiter("Zoe");
        Waiter rosita = new Waiter("Rosita");

        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            Supplier<Course> starter = scope.fork(() -> grover.announceCourse(CourseType.STARTER));
            Supplier<Course> main = scope.fork(() -> zoe.announceCourse(CourseType.MAIN));
            Supplier<Course> dessert = scope.fork(() -> rosita.announceCourse(CourseType.DESSERT));

            scope.join(); // 1
            scope.throwIfFailed(); // 2

            return new MultiCourseMeal(starter.get(), main.get(), dessert.get()); // 3
        }
    }
}
```
The scope's purpose is to keep the threads together.
At `1`, we wait (`join`) until _all_ threads are done with their work. 
If one of the threads is interrupted, an `InterruptedException` is thrown here. 
At `2`, an `ExecutionException` can be thrown if an exception occurs in one of the threads. 
Once we reach `3`, we can be sure everything has gone well, and we can retrieve and process the results.

Actually, the main difference with the code we had before is the fact that we create threads (`fork`) within a new `scope`. 
Now we can be certain that the lifetimes of the three threads are confined to this scope, which coincides with the body of the try-with-resources statement.

Furthermore, we've gained _short-circuiting behaviour_. 
When one of the `announceCourse(..)` subtasks fails, the others are cancelled if they have not completed yet.
This behaviour is managed by the `ShutdownOnFailure` policy.
We've also gained _cancellation propagation_.
When the thread that runs `announceMenu()` is interrupted before or during the call to `scope.join()`, all subtasks are cancelled automatically when the thread exits the scope.

## Shutdown on Success

So a _shutdown-on-failure_ policy cancels any remaining tasks in the scope if one of the tasks has failed.
A _shutdown-on-success_ policy is also available: it cancels any remaining tasks in the scope if one of the tasks has succeeded.
It can be used to avoid any unneccesary work when a succesful result has already been achieved.
Which would actually be a perfect way to solve the problems that our patient waiter from the article introduction was experiencing!

Let's see what a shutdown-on-success implementation would look like in that case:

```java
public record DrinkOrder(Guest guest, Drink drink) {}

public class StructuredConcurrencyBar implements Bar {
    @Override
    public DrinkOrder determineDrinkOrder(Guest guest) throws InterruptedException, ExecutionException {
        Waiter zoe = new Waiter("Zoe");
        Waiter elmo = new Waiter("Elmo");

        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<DrinkOrder>()) {
            scope.fork(() -> zoe.getDrinkOrder(guest, BEER, WINE, JUICE));
            scope.fork(() -> elmo.getDrinkOrder(guest, COFFEE, TEA, COCKTAIL, DISTILLED));

            return scope.join().result(); // 1
        }
    }
}
```

In this example the waiter is responsible for getting a valid `DrinkOrder` object based on the preferences of the guest and the current supply of drinks at the bar.
After the method `Water.getDrinkOrder(Guest guest, DrinkCategory... categories)` has been called, the waiter starts to list all available drinks in the supplied drink categories.
Once a guest hears something they like, they respond and the waiter creates a drink order.
As soon as our waitress Zoe has found a matching drink for her guest, the `getDrinkOrder(..)` method returns a `DrinkOrder` object and the scope will shut down. 
This means that any unfinished subtasks (such as the one in which Elmo is still listing different kinds of tea) will be cancelled.
The `result()` method at `1` will either return a valid `DrinkOrder` object, or throw an `ExecutionException` if one of the subtasks has failed.

### Custom Shutdown Policies

So two shutdown policies are provided out-of-the-box, but it's also possible to create your own by extending the class `StructuredTaskScope` and its protected `handleComplete(..)` method.
That will allow you to have full control over when the scope will shut down and what results will be collected.

## How to Use and Further Reading

Structured concurrency has been available as an incubating feature since Java 19, but has been promoted to a preview feature in Java 21, which means the feature is now almost complete.
This means that in order to use the feature in Java 21, you need to pass the compiler option `--enable-preview` to be able to work with it.
Also note that the feature might be tweaked some more in additional preview statuses based on any feedback developers might have, so be ready to apply certain changes when you upgrade to later versions of Java.
If you wish to learn even more about structured concurrency, JEP 453 [3] is a very interesting read and comes with some more details that we couldn't fit into this article.

## Wrap-up

We think structured concurrency is an exciting new feature that helps you optimise your concurrent code, while also making the structure of subtasks a lot clearer compared to previous versions of Java.
It proves yet again that Java is evolving at a rapid pace, and is on course to remain a modern and relevant programming language in the next few years.

## SIDEBAR: Concurrency Refresher

Let's refresh our memories on a few concurrency-related concepts.
In a computer, instructions (your program code) are executed by a process. 
The operating system assigns resources, such as memory, to the process. 
Processes are independent of each other and have their own memory address space. 
We also refer to a process as a "unit of resources."

A process consists of one or more threads. 
A thread represents the "unit of execution": the smallest possible execution of a sequence of programmed instructions. 
Multiple threads within a single process share various things, such as resources, executable code, memory address space, and process state. 
One thread can cause the process and all other threads to crash.

Since only a single thread can run on a CPU core at a time, there is competition among multiple threads for usage of the CPU core.
This is what we call _concurrency_. 
The challenge is to find the most efficient distribution over time. 
The measure of concurrency is called _throughput_: the number of tasks we can process per unit of time.

_Parallelism_, on the other hand, involves dividing a single task over space: we split the task into collaborating subtasks across multiple CPU cores. 
The measure of parallelism is _latency_: the duration of an individual task.

## References

1. [https://en.wikipedia.org/wiki/Structured_concurrency](https://en.wikipedia.org/wiki/Structured_concurrency)
2. [https://auroratide.com/posts/understanding-kotlin-coroutines](https://auroratide.com/posts/understanding-kotlin-coroutines)
3. [https://openjdk.org/jeps/453](https://openjdk.org/jeps/453)

## Bios

### Bram Janssens

Bram Janssens is a trainer & consultant at Info Support.
He likes to share knowledge in an enthousiastic way.
Above all he values having fun while at work.

### Hanno Embregts

Hanno Embregts is a software architect, international conference speaker & trainer at Info Support.
He likes his work best when it's fast-paced and versatile, which is why he juggles Java development, software architecture, public speaking and teaching courses at Info Support’s Knowledge Centre. 
