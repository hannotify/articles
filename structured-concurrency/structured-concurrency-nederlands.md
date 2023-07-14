# Structured Concurrency Deep-Dive

In dit artikel leggen we je uit wat structured concurrency is en hoe je dit in Java 21 kunt toepassen. 

## Concepts
Concurrency heeft alles te maken met threads. Daarom eerst een quick refresher van de belangrijkste begrippen.

In een computer worden instructies (je programmacode) uitgevoerd door een proces. Het besturingssysteem (OS) wijst resources, zoals geheugen, toe aan het proces. Processen zijn onafhankelijk van elkaar en hebben ieder hun eigen geheugenadresruimte. We zeggen ook wel dat een proces een "eenheid van resources" is. 

Een proces bestaat uit één of meerdere threads. Een thread vorm een "eenheid van uitvoering": de kleinst mogelijke uitvoering van een reeks geprogrammeerde instructies. Meerdere threads binnen één proces delen vanalles, zoals resources, de uitvoerbare code, geheugenadresruimte en process state. Eén thread kan het process en alle andere threads laten crashen.

OPTIONEEL: De naam thread staat voor "thread of execution": het is een snelle afwisseling tussen uitvoering en wachten. Een thread heeft een CPU nodig om zijn werk te doen. Er kan maar één thread op een CPU actief zijn. De scheduler van het OS bepaalt wanneer een thread de CPU mag gebruiken. Als je horizonaal uittekent wanneer een thread actief is en wanneer niet, krijg je een stippellijn: "a group of filaments twisted together". 

OPTIONEEL: Vyssotsky kwam voor het eerst met deze naam in 1966. In de jaren 90 werden threads populairder. Applicaties wilden beter gebruik maken van de beschikbare resources en zo de indruk wekken dat er meerdere taken tegelijk werden uitgevoerd. Met de komst van meerdere CPU cores in de jaren 2000 is multithreading zich verder gaan ontwikkelen.

## Concurrency vs. parallelism
Omdat er maar één thread tegelijkertijd op een CPU mag, vindt er concurrentie plaats tussen meerdere threads, als zij allemaal de CPU nodig hebben. Deze strijd om de CPU noemen we **concurrency**. Het is de uitdaging om een zo efficiënt mogelijke verdeling te vinden "over time". De maat voor concurrency is **throughput**: het _aantal_ taken dat we kunnen verwerken per tijdseenheid.

**Parallelisme** gaat over het verdelen van één taak "over space": we delen de taak op in samenwerkende subtaken over meerdere CPU's. De maat voor parallelisme is **latency**: de _tijdsduur_ van een enkele taak. 

## Unstructured vs. structured
Wat bedoelen we nu met de begrippen structured en unstructured? Laten we eerst kijken naar het programmeren van sequentiële code. 

We kennen allemaal de term spaghetticode. Deze metafoor komt voort uit code die gebruik maakt van `goto` statements voor flow control: het onder voorwaarden springen naar andere regels. Het is een "one way jump". Wist je trouwens dat `goto` een keyword is in Java? Een flow diagram van zoiets lijkt op een bord niet te ontwarren spaghetti. Dit soort code is moeilijk te lezen, begrijpen en debuggen. Het is unstructured. 

Structured programming is daarentegen een paradigma dat gebruik maakt van control flow statements als `if/else` en `while/for` en daarnaast block structures en subroutines kent. De term werd bedacht door de Nederlander Dijkstra in 1968. Spaghetti verandert hierdoor in een hiërarchie van afgeschermde en samenhangende blokken code met ieder hun eigen _scope_. Blokken code (bijvoorbeeld functies) kunnen met elkaar communiceren via duidelijke entry en exit points en een "two way jump": na het aanroepen keer je terug.

## Unstructured concurrency
Ook concurrent programming kan gestructureerd of ongestructureerd in elkaar zitten. De huidige implementatie in Java is ongestructureerd. Taken lopen zelfstandig, los van elkaar, er is geen hiërarchie, scope of andere structuur en taken kunnen niet gemakkelijk fouten of annuleringen aan elkaar doorgeven. Een eenvoudig voorbeeld:
```java
Response handle(int userId, int orderId) 
	throws ExecutionException, InterruptedException {
  Future<User>  u = exec.submit(() -> service.find(userId));
  Future<Order> o = exec.submit(() -> service.fetch(orderId));

  var user = u.get();  // 1
  var order = o.get(); // 2
  
  return new Response(user, order);
}
```
Bij `1` en `2` kunnen `ExecutionException` en `InterruptedException` optreden.

Er is een aantal (verborgen) problemen met deze code:
- If `find(..)` takes a long time to execute, but `fetch(..)` fails in the meantime, `handle(..)` will wait unnecessarily for `find(..)` by blocking on `u.get()`, rather than cancelling it.
- If `find(..)` throws an exception, then `u.get()` will throw an exception, but `fetch(..)` will continue to run in its own thread, leading to thread leakage.
- If the thread executing `handle(..)` is interrupted, the interruption will not propagate to the subtasks: both `find(..)` and `fetch(..)` threads will leak.

Als programmeur hebben we het liefst gestructureerde code dat leest als een sequentieel verhaaltje, want dat reflecteert de structuur van een taak. Dat is in bovenstaande voorbeeld niet zo. Structured Concurrency lost deze problemen op.

## Structured concurrency
De term structured concurrency heeft zijn roots in de 1960's bij het fork-join model, maar het concept is geformuleerd door Sústrik in 2016 voor goroutines. Onafhankelijk daarvan bedacht Elizarov hetzelfde voor Kotlin's coroutines. Hierbij hebben threads een duidelijke hiërarchie, een eigen scope, daardoor entry en exit points. Er ontstaat, net als bij functieaanroepen, een tree van threads met parent-childrelaties. Een scope eindigt pas als alle child threads beëindigd zijn. Het doorgeven van fouten en annulering wordt zo gestroomdlijnd. Dit leidt tot verbeterde betrouwbaarheid en observeerbaarheid. Er is a strict nesting of the lifetimes of operations in a way that mirrors their syntactic nesting in the code.

Hetzelfde voorbeeld als hierboven, maar dan structured:
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

Het belangrijkste verschil met de vorige code is dat we hier de threads creëren (`fork`) in een nieuwe `scope`. Deze scope "houdt de threads bij elkaar". Bij `1` wordt er gewacht (`join`) totdat _beide_ threads klaar zijn. Als een van de threads onderbroken werd, treedt hier de `InterruptedException` op. Bij `2` kan een `ExecutionException` gegooid worden als er in een of beide threads een exception optrad. We hebben ervoor gekozen om deze fout(en) te propageren. Dit is optioneel, maar wel zo netjes. Als we bij `3` zijn aangekomen, is alles goed gegaan en kunnen we de resultaten ophalen en verwerken. 

### How to use 
Structured concurrency is sinds Java 19 beschikbaar als incubating feature, maar is voor Java 21 gepromoveerd naar preview feature. Dat betekent dat je in Java 21 slechts de compiler optie `--enable-preview` hoeft te gebruiken om ermee aan de slag te gaan en dat het zeker is dat deze feature, eventueel in iets bijgewerkte vorm, in een latere release terecht komt.

### Structured Concurrency and Virtual Threads

[naam]

* Why are they a good match?

### Structured Code example

[naam]

* same code example, but using `StructuredTaskScope`

### Benefits of StructuredTaskScope

[naam]

BRAM: Deels hierboven al uitgelegd.

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

## Wrap-up

[naam]

## References
1. [https://en.wikipedia.org/wiki/Thread_(computing)](https://en.wikipedia.org/wiki/Thread_(computing))
1. [https://en.wikipedia.org/wiki/Structured_programming](https://en.wikipedia.org/wiki/Structured_programming)
1. [https://en.wikipedia.org/wiki/Structured_concurrency](https://en.wikipedia.org/wiki/Structured_concurrency)
1. [https://auroratide.com/posts/understanding-kotlin-coroutines](https://auroratide.com/posts/understanding-kotlin-coroutines)

## Bios

### Bram
Bram Janssens is trainer/consultant bij Info Support. Hij houdt ervan om zijn kennis met veel enthousiasme over te dragen. Plezier in het werk vindt hij het belangrijkst!

### Hanno

Hanno Embregts is software-architect, conferentiespreker en trainer bij Info Support. Hij houdt ervan als zijn werk afwisselend is; daarom combineert hij Java-softwareontwikkeling met spreken op conferenties en het geven van trainingen bij het Kenniscentrum van Info Support.

