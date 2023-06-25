# Structured Concurrency Deep-Dive

* Certain I/O- or CPU-intensive operations can be made to run faster by executing them concurrently. 
* Since Java 5 we can use `ExecutorService` to easily execute tasks concurrently.
* But Java’s current implementation of concurrency (including ExecutorService) is unstructured, which can make error handling and cancellation with multiple tasks a challenge. 
* When multiple tasks are started up asynchronously, we currently aren’t able to cancel the remaining tasks once the first task returns an error.
* Structured Concurrency is proposed as a preview JEP in Java 21 to improve the situation.

## Unstructured Code Example

```java
// TODO: easy-to-relate code example that uses ExecutorService to demonstrate unstructured concurrency
```

* Unstructured
  * Tasks run independently
  * No hierarchy, scope or other structure
  * So communication between tasks is difficult (errors, cancel requests)
* Leads to complex code

## Definition

**Structured concurrency** treats multiple tasks running in different threads as a single unit of work, thereby streamlining error handling and cancellation, improving reliability, and enhancing observability.

## History of the feature

* Incubating API - JEP 428.
* Re-incubated - JEP 437.
* Preview - JEP 453.

## Structured Code Example

```java
// TODO: apply structured concurrency to the earlier code example
```

## Configuration

* `--enable-preview`
* `--add-modules jdk.incubator.concurrent` or define a `module-info.java` with similar content.

## Task Scopes

...


## References

* ...

## From Hanno's Blog Post At Foojay.io

Java’s current implementation of concurrency is unstructured, which can make error handling and cancellation with multiple tasks a challenge. When multiple tasks are started up asynchronously, we currently aren’t able to cancel the remaining tasks once the first task returns an error.

Let’s illustrate this point with a code example from the JEP:

```java
Response handle() throws ExecutionException, InterruptedException {
    Future<String>  user = executor.submit(() -> findUser());
    Future<Integer> order = executor.submit(() -> fetchOrder());
    String theUser = user.get();   // Join findUser
    int theOrder = order.get();  // Join fetchOrder
    return new Response(theUser, theOrder);
}
```

When the user.get() call results in an error, there is no way for us to cancel the second task when we want to prevent getting a result that won’t be used anyway. Though when we would rewrite this code to use just a single thread, the situation would become a lot simpler:

```java
Response handle() throws IOException {
    String theUser = findUser();
    int theOrder = fetchOrder();
    return new Response(theUser, theOrder);
}
```

See? Here we would be able to prevent the second call once the first one has failed.

In general, multithreaded programming in Java would be easier, more reliable, and more observable if the parent-child relationships between tasks and their subtasks were expressed syntactically — just as for single-threaded code. The syntactic structure would delineate the lifetimes of subtasks and enable a runtime representation of the inter-thread hierarchy, enabling error propagation and cancellation as well as meaningful observation of the concurrent program.

Enter structured concurrency. We’ve now rewritten the code example to make use of the new StructuredTaskScope API:

```java
Response handle() throws ExecutionException, InterruptedException {
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
      Future<String>  user  = scope.fork(() -> findUser());
      Future<Integer> order = scope.fork(() -> fetchOrder());

      scope.join();           // Join both forks
      scope.throwIfFailed();  // ... and propagate errors

      // Here, both forks have succeeded, so compose their results
      return new Response(user.resultNow(), order.resultNow());
  }
}
```

In structured concurrency, subtasks work on behalf of a task. The task awaits the subtasks’ results and monitors them for failures. The StructuredTaskScope class allows developers to structure a task as a family of concurrent subtasks, and to coordinate them as a unit. Subtasks are executed in their own threads by forking them individually and then joining them as a unit and, possibly, cancelling them as a unit. The subtasks’ successful results or exceptions are aggregated and handled by the parent task.

In contrast to the original example, understanding the lifetimes of the threads involved here is easy. Under all conditions their lifetimes are confined to a lexical scope, namely the body of the try-with-resources statement. Furthermore, the use of StructuredTaskScope ensures a number of valuable properties:

Error handling with short-circuiting. If one of the subtasks fail, the other is cancelled if it has not yet completed. This is managed by the cancellation policy implemented by ShutdownOnFailure; other policies like ShutdownOnSuccess are also available.

Cancellation propagation. If the thread running handle() is interrupted before or during the call to join(), both forks are cancelled automatically when the thread exits the scope.

Clarity — The above code has a clear structure: Set up the subtasks, wait for them to either complete or be cancelled, and then decide whether to succeed (and process the results of the child tasks, which are already finished) or fail (and the subtasks are already finished, so there is nothing more to clean up).

By the way: it is by no means coincidental that structured concurrency is coming to Java at the same time as virtual threads. Modern Java programs will likely use an abundance of threads, and they need to be correctly and robustly coordinated. Structured concurrency can provide exactly that, while also enabling observability tools to display threads as they are understood by the developer.
