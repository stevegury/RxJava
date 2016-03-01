## RxJava v2 Design

Terminology, principles, contracts, and other aspects of the design of RxJava v2.

### Terminology & Definitions

##### Interactive

Producer obeys consumer-driven flow control.
Consumer manages capacity by requesting data.

##### Reactive

Producer is in charge. Consumer has to do whatever it needs to keep up.

##### Hot

When used to refer to a data source (such as an `Observable`), it means it does not have side-effects when subscribed to.

For example, an `Observable` of mouse events. Subscribing to that `Observable` does not cause the mouse events, but starts receiving them.

(Note: Yes, there are *some* side-effects of adding a listener, but they are inconsequential as far as the 'hot' usage is concerned).

##### Cold

When used to refer to a data source (such as an `Observable`), it means it has side-effects when subscribed to.

For example, an `Observable` of data from a remote API (such as an RPC call). Each time that `Observable` is subscribed to causes a new network call to occur.

##### Reactive/Push

Producer is in charge. Consumer has to do whatever it needs to keep up.
(This is Observable)
e.g.:
- Observable without backpressure (RxJS, RxNet)
- i.e. v1.x Observable without backpressure or 2.x Observable
- Callbacks (the producer call the function at its convenience)
- IRQ, mouse events, io interrupts

##### Synchronous Interactive/Pull

Consumer is in charge. Producer has to do whatever it needs to keep up.
e.g.:
- Iterable (Sync)
- 2.x/1.x Observable (without concurrency, producer and consumer on the same thread)
- 2.x Flowable (without concurrency, producer and consumer on the same thread)

##### Async Pull (Async Interactive)

Consumer requests data when it wishes, and the data is then pushed when the producer wishes to. The Reactive Streams `Publisher` is an instance of "async pull", as is the 'AsyncEnumerable' in .Net.
(This is Flowable)
e.g.:
- Future, Promise (hot created)
- Single is the lazy variant of this
- 2.x Flowable
- 1.x Observable (with backpressure)
- AsyncEnumerable/AsyncIterable (Async)

There is an overhead for achieving this, that's why we also have 2.x Observable.

##### Flow Control

Flow control is any mitigation strategies that a consumer applies to reduce the flow of data.
e.g. (Controling the production of data, or dropping data, buffering...)

##### Eager

Containing object immediately start work when it is created.
e.g. A `Future` once created has work being performed and represents the eventual value of that work. It can not be deferred once created.

##### Lazy

Containing object does nothing until it is subscribed to or otherwise started.
e.g. `Observable.create` does not start any work until `Observable.subscribe` starts the work.


### RxJava & Related Types

##### Observable

Stream that supports async and synchronous push. It does not support interactive flow control (`request(n)`).

Usable for:

- sync or async
- push
- 0, 1, many or infinite items

Flow control support:

- buffering, sampling, throttling, windowing, dropping, etc
- temporal and count-based strategies

* Type Signature *

```
class Observable<T> {
  void subscribe(Observer<T> observer);

  interface Observer<T> {
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
    void onSubscribe(Disposable d);
  }
}
```

The rule for using this type signature is:

> onSubscribe onNext* (onError | onComplete)?


##### Flowable

Stream that supports async and synchronous push and pull. It supports interactive flow control (`request(n)`).

Usable for:

- pull sources
- push Observables with backpressure strategy (ie. `Observable.toFlowable(onBackpressureStrategy)`)
- sync or async
- 0, 1, many or infinite items

Flow control support:

- buffering, sampling, throttling, windowing, dropping, etc
- temporal and count-based strategies
- `request(n)` consumer demand signal
  - for pull-based sources, this allows batched "async pull"
  - for push-based sources, this allows backpressure signals to conditionally apply strategies (i.e. drop, first, buffer, sample, fail, etc)

You get a flowable from:

- Converting a Observable with a backpressure strategy
- Create from sync/async OnSubscribe API (which participate in backpressure semantics)

* Type Signature *

```
class Flowable<T> implements Flow.Publisher<T>, io.reactivestreams.Publisher<T> {
  void subscribe(Subscriber<T> subscriber);

  interface Subscriber<T> implements Flow.Subscriber<T>, io.reactivestreams.Subscriber<T> {
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
    void onSubscribe(Flowable.Subscription d);
  }

  interface Subscription implements Flow.Subscription, io.reactivestreams.Subscription {
    void cancel();
    void request(long n);
  }
}
```

For Java 9+, we want to use the [multi-release jar file support](http://openjdk.java.net/jeps/238).

The rule for using this type signature is:

> onSubscribe onNext* (onError | onComplete)?


##### Single

Lazy representation of a single response (lazy equivalent of Future/Promise)

Usable for:

- pull sources
- sync or async
- 1 item

Flow control:

- Non-existant

* Type Signature *

```
class Single<T> {
  void subscribe(Single.Subscriber<T> subscriber);

  interface Subscriber<T> {
    void onSuccess(T t);
    void onError(Throwable t);
    void onSubscribe(Disposable d);
  }
}
```

> onSubscribe (onError | onSuccess)?


##### Completable

Lazy representation of a unit of work that can complete or fail

Semantic equivalent of `Observable.empty().doOnSubscribe()`.
Usable in scenarios often represented with types such as `Single<Void>` or `Observable<Void>`.

Usable for:

- sync or async
- 0 item

* Type Signature *

```
class Completable {
  void subscribe(Completable.Subscriber subscriber);

  interface Subscriber {
    void onComplete();
    void onError(Throwable t);
    void onSubscribe(Disposable d);
  }
}
```

> onSubscribe (onError | onComplete)?


##### Observer

Reactive consumer of events. (without consumer-driven flow control)
Manage unsubscription

##### Publisher

Interactive producer of events (with flow control).

[Reactive Streams producer](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.0/README.md#1-publisher-code) of data

##### Subscriber

Interactive consumer of events. (with consumer-driven flow control)
Manage unsubscription

[Reactive Streams consumer](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.0/README.md#2-subscriber-code) of data.

##### Subscription

[Reactive Streams state](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.0/README.md#3-subscription-code) of subscription supporting flow control and cancellation.

##### Processor

[Reactive Streams operator](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.0/README.md#4processor-code) for defining behavior between `Publisher` and `Subscriber`. It must obey the contracts of `Publisher` and `Subscriber`, meaning it is sequential, serialized, and must obey `request(n)` flow control.

##### Subject

A "hot", push-based data source that allows a producer to emit events to it and consumers to subscribe to events in a multicast manner. It is "hot" because consumers subscribing to it does not cause side-effects, or affect the data flow in any way. It is push-based and reactive because the producer is fully in charge.

A subject is used to decouple unsubscription. Termination is fully in the control of the producer. onError and onComplete are still terminal events.
Subject are stateful and retain their terminal state (for replaying to all/future subscribers).

Relation to Reactive Streams

- It can not implement Reactive Streams `Publisher` unless it is created with a default consumer-driven flow control strategy.
- It can not implement `Processor` since a `Processor` must compose `request(n)` which can not be done with multicasting or push.

e.g. `Subject.toFlowable(strategy).subscribe(subcriber1)`


##### Disposable

A type representing work that can be cancelled or disposed.

e.g. A Scheduler returns a Disposable that you use for disposing of the Scheduler.

##### Operator

An operators follow a specific lifecycle (union of the contract producer/consumer).

- It must propagate the `subscribe` event upstream (to the producer)
- It must obey the RxJava contract (serialize all events)
- If it has resources to cleanup it is responsible to watching (onError, onComplete, cancel/dispose) and do the necessary cleanup
- It must propagate the cancel/dispose upstream

Operator for Flowable, in the addition of the previous 4:

- It must propagate/negotiate the `request-n` event.


### Creation

```
Flowable.create(SyncGenerator generator)

Flowable.create(AsyncGenerator generator)

Observable<T>.create(OnSubscribe<Observer<T>> onSubscribe)

Single<T>.create(OnSubscribe<Single.Subscriber<T>> onSubscribe)

Completable<T>.create(OnSubscribe<Completable.Subscriber<T>> onSubscribe)
```

### Terminal behavior

A producer can terminate a stream by emitting `onComplete` or `onError`. A consumer can terminate a stream by calling `cancel`.

Any resource cleanup of the source or operators must account for any of these three termination events. In other words, if an operator needs cleanup, then it should register the cleanup callback with `cancel`, `onError` and `onComplete`.

The final `subscribe` will *not* invoke `cancel` after receiving an `onComplete` or `onError`.
