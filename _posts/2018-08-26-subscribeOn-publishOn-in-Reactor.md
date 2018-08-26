---
layout: post
title: subscribeOn and publishOn operators in Reactor
tags: [projectreactor, reactivestreams]
---

[`Reactor`](https://projectreactor.io/) does not enforce a concurrency model, it leaves that to the developer. By default the execution happens in the thread of the caller to `subscribe()`. 
Let's consider the following method where we subscribe with 4 consumers to a Flux of Integers which print out the thread name while processing the emitted elements in the Flux.
 
```java
public static void createSubscribers(Flux<Integer> flux) {
    IntStream.range(1, 5).forEach(value ->
        flux.subscribe(integer -> System.out.println(value + " consumer processed "
                + integer + " using thread: " + Thread.currentThread().getName())));
}
```

Considering the following Flux:

```java
Flux<Integer> flux1 = Flux.range(0, 2);
```

the `createsSubscribers(flux1)` will print out the followings:

```bash
1 consumer processed 0 using thread: main
1 consumer processed 1 using thread: main
2 consumer processed 0 using thread: main
2 consumer processed 1 using thread: main
3 consumer processed 0 using thread: main
3 consumer processed 1 using thread: main
4 consumer processed 0 using thread: main
4 consumer processed 1 using thread: main
```

As we can we all the logs are on the `main` thread, which is the thread of the caller to `subscribe()` method.

In `Reactor` the execution context is determined by the used `Scheduler`, which is an interface and the `Schedulers` provides implementation with static methods.
 
Some operators replace the default `Scheduler`, for example the `Flux#delayElements(Duration)` uses the `Schedulers.parallel()` instance.

The following code 

```java
Flux<Integer> flux2 = Flux.range(0, 2).delayElements(Duration.ofMillis(1));
createsSubscribers(flux2);
```

will print out the followings:

```bash
3 consumer processed 0 using thread: parallel-3
2 consumer processed 0 using thread: parallel-2
4 consumer processed 0 using thread: parallel-4
1 consumer processed 0 using thread: parallel-1
3 consumer processed 1 using thread: parallel-7
4 consumer processed 1 using thread: parallel-5
1 consumer processed 1 using thread: parallel-6
2 consumer processed 1 using thread: parallel-8
```
 
The `Schedulers.parallel()` creates as many threads as there are CPUs but at least 4. 
 
There are two ways to change explicitly the execution context (Scheduler) in a reactive pipeline via the `publishOn` and `subscribeOn` methods.
 
Let's consider the following example

```
Flux<String> flux1 = Flux.just("foo", "bar");
Flux<String> flux2 = flux1.map(s -> s.toUpperCase());
flux2.subscribe(s -> System.out.println(s));
``` 

With the `map` operator we create another `Flux` (flux2) and when we subscribe to this flux2 instance, then flux2 implicitly subscribes to flux1 as well.
And when the flux1 starts to emit elements it will call flux2 which will call our `Subscriber`. We can notice that there is a subscription process and emitting process. 
In Reactor terminology the flux1 would be the `upstream` and flux2 the `downstream`. 

The `publishOn` influences the emitting process. It takes an emitted element from the upstream and replays it downstream while executing the callback on a worker from the associated `Scheduler`. This means it will also affect where the subsequent operators will execute. (until another `publishOn` is chained in).

The `subscribeOn` rather influences the subscription process. It doesn't matter where it is put in the reactive pipeline, it always affects the context of the source emission, and does not affect the behavior of subsequent calls to `publishOn`. If there are multiple `subscribeOn` in the chain the earliest `subscribeOn` call in the chain is actually taken into account. 

Considering the following example:

```java
Flux<Integer> flux3 = Flux.range(0, 2)
    // this is influenced by subscribeOn
    .doOnNext(s -> System.out.println(s + " before publishOn using thread: " + Thread.currentThread().getName()))
    .publishOn(Schedulers.elastic())
    // the rest is influenced by publishOn
    .doOnNext(s -> System.out.println(s + " after publishOn using thread: " + Thread.currentThread().getName()))
    .subscribeOn(Schedulers.single());
createsSubscribers(flux3);
```

it will print out the followings:

```bash
0 before publishOn using thread: single-1
1 before publishOn using thread: single-1
0 after publishOn using thread: elastic-2
1 consumer processed 0 using thread: elastic-2
1 after publishOn using thread: elastic-2
1 consumer processed 1 using thread: elastic-2
0 before publishOn using thread: single-1
1 before publishOn using thread: single-1
0 after publishOn using thread: elastic-3
2 consumer processed 0 using thread: elastic-3
1 after publishOn using thread: elastic-3
2 consumer processed 1 using thread: elastic-3
0 before publishOn using thread: single-1
1 before publishOn using thread: single-1
0 after publishOn using thread: elastic-4
3 consumer processed 0 using thread: elastic-4
1 after publishOn using thread: elastic-4
3 consumer processed 1 using thread: elastic-4
0 before publishOn using thread: single-1
1 before publishOn using thread: single-1
0 after publishOn using thread: elastic-5
4 consumer processed 0 using thread: elastic-5
1 after publishOn using thread: elastic-5
4 consumer processed 1 using thread: elastic-5
```

If we include `delayElements` operator after the second `doOnNext` operator we would change the `Scheduler` for our subscribers to `Schedulers#parallel()` instance. 

```java
Flux<Integer> flux4 = Flux.range(0, 2)
    // this is influenced by subscribeOn
    .doOnNext(s -> System.out.println(s + " before publishOn using thread: " + Thread.currentThread().getName()))
    .publishOn(Schedulers.elastic())
    // influenced by publishOn
    .doOnNext(s -> System.out.println(s + " after publishOn using thread: " + Thread.currentThread().getName()))
    .delayElements(Duration.ofMillis(1))
    // influcend by the Schedulers.parallel() caused by the delayElements operator
    .subscribeOn(Schedulers.single());
createsSubscribers(flux4);
```

And indeed in the logs we will see:

```bash
0 before publishOn using thread: single-1
1 before publishOn using thread: single-1
0 after publishOn using thread: elastic-3
0 before publishOn using thread: single-1
1 before publishOn using thread: single-1
0 after publishOn using thread: elastic-2
0 before publishOn using thread: single-1
1 before publishOn using thread: single-1
0 after publishOn using thread: elastic-4
0 before publishOn using thread: single-1
1 before publishOn using thread: single-1
0 after publishOn using thread: elastic-5
1 consumer processed 0 using thread: parallel-1
1 after publishOn using thread: parallel-1
2 consumer processed 0 using thread: parallel-2
1 after publishOn using thread: parallel-2
3 consumer processed 0 using thread: parallel-3
1 after publishOn using thread: parallel-3
4 consumer processed 0 using thread: parallel-4
1 after publishOn using thread: parallel-4
1 consumer processed 1 using thread: parallel-7
2 consumer processed 1 using thread: parallel-8
3 consumer processed 1 using thread: parallel-5
4 consumer processed 1 using thread: parallel-6
```