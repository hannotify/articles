
## SIDEBAR Structured Concurrency and Virtual Threads

[name]

* Why are they a good match?

## SIDEBAR: Concurrent Programming Refresher

OPTIONAL: The term "thread" stands for "thread of execution": it involves a fast alternation between execution and waiting. A thread needs a CPU to do its work. Only one thread can be active on a CPU at the same time. The OS scheduler determines when a thread can use the CPU. If you horizontally plot the active and inactive periods of a thread, you get a dotted line: "a group of filaments twisted together."

OPTIONAL: Vyssotsky first introduced this term in 1966. Threads gained popularity in the 1990s as applications sought to make better use of available resources and create the impression of executing multiple tasks simultaneously. With the advent of multiple CPU cores in the 2000s, multithreading further evolved.

## SIDEBAR: Unstructured vs. Structured

Let's discuss the terms unstructured and structured. Let's first look at programming sequential code.

We're probably all familiar with the term 'spaghetti code'. This metaphor arises from code that uses `goto` statements for flow control, which involves conditionally jumping to other lines. It's a "one-way jump." (Did you know that `goto` is a keyword in Java?) Such code is difficult to read, understand, and debug. It is unstructured.

On the other hand, structured programming is a paradigm that uses control flow statements such as `if/else` and `while/for`, along with block structures and subroutines. The term was coined by Dutch computer scientist Dijkstra in 1968. Spaghetti code transforms into a hierarchy of enclosed and cohesive code blocks, each with its own _scope_. Code blocks (e.g. 'functions') can communicate with each other through clear entry and exit points, allowing for a 'two-way jump': you return after invoking.