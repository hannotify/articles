# Structured Concurrency Deep-Dive

## Current Implementation

* Unstructured
  * Tasks run independently
  * No hierarchy, scope or other structure
  * Communication between tasks is difficult (errors, cancel requests)
* Leads to complex code

## Definition

**Structured concurrency** treats multiple tasks running in different threads as a single unit of work, thereby streamlining error handling and cancellation, improving reliability, and enhancing observability.

## History of the feature

* Incubating API - JEP 428.
* Re-incubated - JEP 437.
* Preview - JEP 453.

## Configuration

* `--enable-preview`
* `--add-modules jdk.incubator.concurrent` or define a `module-info.java` with similar content.

## Task Scopes

...


## References

* ...