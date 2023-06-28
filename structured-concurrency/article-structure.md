# Structured Concurrency Deep-Dive

[introduction]

## Unstructured Concurrency

[naam]

* Unstructured code example

## Synchronous Code Reflects Task Structure

[naam]

## Enter Structured Concurrency

[naam]

* Definition

> Structured concurrency is an approach to concurrent programming that preserves the natural relationship between tasks and subtasks, which leads to more readable, maintainable, and reliable concurrent code. (https://openjdk.org/jeps/453)

* 'subtasks work on behalf of a task'
* Some theory
  * 'subtasks work on behalf of a task'
  * its power comes from two ideas:
    * (1) well-defined entry and exit points for the flow of execution through a block of code, and
    * (2) a strict nesting of the lifetimes of operations in a way that mirrors their syntactic nesting in the code.
* History of the feature (previous JEPs)
* Kotlin or Go reference?

### Structured Concurrency and Virtual Threads

[naam]

* Why are they a good match?

### Structured Code example

[naam]

* same code example, but using `StructuredTaskScope`

### Benefits of StructuredTaskScope

[naam]

* error handling with short-circuiting
* cancellation propagation
* clarity
* observability

### Executor Interface Remains Unstructured

[naam]

* To avoid confusion
* StructuredTaskScope.fork() deliberately returns a Supplier (not a Future) to indicate that its intended use is different

## Shutdown Policies

[naam]

* `ShutdownOnFailure`
* `ShutdownOnSuccess`

### Custom Shutdown Policies

(optioneel, als er nog woorden over zijn)

## How to use

[naam]

* --enable-preview

## Wrap-up

[naam]

## References

[1]
[2] 

## Bios

### Bram

[Bram]

### Hanno

[Hanno]

