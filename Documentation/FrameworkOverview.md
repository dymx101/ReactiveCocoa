# Framework Overview

此文档是对于在ReactiveCoca框架中的不同组件的高级描述，此文档同时也尝试解释他们(不同组件)之间如何协同工作以及各自的责任。
此文档可以被当做学习新模块和发现更详细文档的开端。

需要寻找例子 或者 RAC的用法，请参见 [README][] 或者 [Design Guidelines][]。


## Streams

一个**stream**, 由 [RACStream][] 抽象类表示，是任何对象值得一个序列。

这些值有可能立刻可用，也有可能将来可用，但是，必须以连续的(Sequantially)方式获取。不可以在第一个值未处理之前，获取第二个值。

Stream是细胞单元([monads][])，这就允许复杂的操作基于一些基本的原函数(比如 `-bind:`)。[RACStream][]也实现了类似[Haskell][]中的[Monoid][]和[MonadZip][]类型。

[RACStream][]本身并不是很有用。大多数streams被用作[signals](#signals)和[sequances](#sequences)。

## Signals

**Signal**，由[RACSignal][]类来表示，是一个_push-driven_的[stream](#streams)。

Signal一般来说代表会在将来发送的数据，如果工作完成或者得到了数据，这些值会在signal上面发送，从而将他们推到订阅者手里。为了访问singal的数据，用户必须订阅[subscribe](#subscription)signal。

Signal发送三种类型的事件给订阅者：

  * **Next**事件从stream中提供一个新值。[RACStream][]方法只操作这类事件。和Cocoa的集合类型不同，一个signal包含`nil`值是完全合法的。
  * **Error**事件表示在signal完成之前发生了错误。这个事件可能包含`NSError`对象表示是什么错误。Errors必须特殊的处理 - 他们不被包含在stream发送的值当中。
  * **Completed**事件表示signal成功完成，不会再有值被添加到stream里面来。Complition必须特殊的处理 - 他不被包含在stream的值当中。

signal的生命周期中，会包含任意数量的`next`事件，跟随一个`error`或者`completed`事件(二者之一)。

### Subscription

**Subscriber** 是任何等待或者能够等待[signal](#signals)发出事件的对象。在RAC中，一个subscruber由任何遵守 [RACSubscriber][] 协议的对象表示。

**Subscription** 是在任何对于 [-subscribeNext:error:completed:][RACSingal] 方法的调用，或者对某一个相应的方便方法 (convenience methods)的调用之时创建。技术上讲，大部分[RACStream][]和[RACSignal][RACSignal+Operations]也会创建subscription，但是这些过渡的subscription一般是实现细节。

Subscriptions [保留他们的signals][Memory Management], 并在signals完成或出错时自动处理它们。Subscriptions也可以被[手动处理](#disposables)。

### Subjects

**Subject** 由[RACSubject][]类表示，是一个可以被手动控制的[signal](#signals)。

Subject可以被当做一个"可变"的signal, 就好像 `NSMutableArray` 对于 `NSArray`一样。它们在将非RAC代码连接到signal世界的过程中超级有用。

比如说，相比于在block回调中处理程序逻辑，block可以简单的向一个共享的subject发送事件。subject可以被当做一个[RACSignal][]返回， 从而隐藏了回调的实现细节。

一些subjects也提供额外的行为。特别是， [RACReplaySubject][] 可以被用于缓冲未来[subscribers](#subscription)的事件，比如当一个网络请求在相应的处理准备好之前就完成的情况。

### Commands

A **command**, represented by the [RACCommand][] class, creates and subscribes
to a signal in response to some action. This makes it easy to perform
side-effecting work as the user interacts with the app.

Usually the action triggering a command is UI-driven, like when a button is
clicked. Commands can also be automatically disabled based on a signal, and this
disabled state can be represented in a UI by disabling any controls associated
with the command.

On OS X, RAC adds a `rac_command` property to
[NSButton][NSButton+RACCommandSupport] for setting up these behaviors
automatically.

### Connections

A **connection**, represented by the [RACMulticastConnection][] class, is
a [subscription](#subscription) that is shared between any number of
subscribers.

[Signals](#signals) are _cold_ by default, meaning that they start doing work
_each_ time a new subscription is added. This behavior is usually desirable,
because it means that data will be freshly recalculated for each subscriber, but
it can be problematic if the signal has side effects or the work is expensive
(for example, sending a network request).

A connection is created through the `-publish` or `-multicast:` methods on
[RACSignal][RACSignal+Operations], and ensures that only one underlying
subscription is created, no matter how many times the connection is subscribed
to. Once connected, the connection's signal is said to be _hot_, and the
underlying subscription will remain active until _all_ subscriptions to the
connection are [disposed](#disposables).

## Sequences

A **sequence**, represented by the [RACSequence][] class, is a _pull-driven_
[stream](#streams).

Sequences are a kind of collection, similar in purpose to `NSArray`. Unlike
an array, the values in a sequence are evaluated _lazily_ (i.e., only when they
are needed) by default, potentially improving performance if only part of
a sequence is used. Just like Cocoa collections, sequences cannot contain `nil`.

Sequences are similar to [Clojure's sequences][seq] ([lazy-seq][] in particular), or
the [List][] type in [Haskell][].

RAC adds a `-rac_sequence` method to most of Cocoa's collection classes,
allowing them to be used as [RACSequences][RACSequence] instead.

## Disposables

The **[RACDisposable][]** class is used for cancellation and resource cleanup.

Disposables are most commonly used to unsubscribe from a [signal](#signals).
When a [subscription](#subscription) is disposed, the corresponding subscriber
will not receive _any_ further events from the signal. Additionally, any work
associated with the subscription (background processing, network requests, etc.)
will be cancelled, since the results are no longer needed.

For more information about cancellation, see the RAC [Design Guidelines][].

## Schedulers

A **scheduler**, represented by the [RACScheduler][] class, is a serial
execution queue for [signals](#signals) to perform work or deliver their results upon.

Schedulers are similar to Grand Central Dispatch queues, but schedulers support
cancellation (via [disposables](#disposables)), and always execute serially.
With the exception of the [+immediateScheduler][RACScheduler], schedulers do not
offer synchronous execution. This helps avoid deadlocks, and encourages the use
of [signal operators][RACSignal+Operations] instead of blocking work.

[RACScheduler][] is also somewhat similar to `NSOperationQueue`, but schedulers
do not allow tasks to be reordered or depend on one another.

## Value types

RAC offers a few miscellaneous classes for conveniently representing values in
a [stream](#streams):

 * **[RACTuple][]** is a small, constant-sized collection that can contain
   `nil` (represented by `RACTupleNil`). It is generally used to represent
   the combined values of multiple streams.
 * **[RACUnit][]** is a singleton "empty" value. It is used as a value in
   a stream for those times when more meaningful data doesn't exist.
 * **[RACEvent][]** represents any [signal event](#signals) as a single value.
   It is primarily used by the `-materialize` method of
   [RACSignal][RACSignal+Operations].

## Asynchronous Backtraces

Because RAC-based code often involves asynchronous work and queue-hopping, the
framework supports [capturing asynchronous backtraces][RACBacktrace] to make debugging
easier.

On OS X, backtraces can be automatically captured from any code, including
system libraries.

On iOS, only queue hops from within RAC and your project will be captured (but
the information is still valuable).

[Design Guidelines]: DesignGuidelines.md
[Haskell]: http://www.haskell.org
[lazy-seq]: http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/lazy-seq
[List]: http://www.haskell.org/ghc/docs/latest/html/libraries/base-4.6.0.1/Data-List.html
[Memory Management]: MemoryManagement.md
[monads]: http://en.wikipedia.org/wiki/Monad_(functional_programming)
[Monoid]: http://www.haskell.org/ghc/docs/latest/html/libraries/base-4.6.0.1/Data-Monoid.html#t:Monoid
[MonadZip]: http://www.haskell.org/ghc/docs/latest/html/libraries/base-4.6.0.1/Control-Monad-Zip.html#t:MonadZip
[NSButton+RACCommandSupport]: ../ReactiveCocoaFramework/ReactiveCocoa/NSButton+RACCommandSupport.h
[RACBacktrace]: ../ReactiveCocoaFramework/ReactiveCocoa/RACBacktrace.h
[RACCommand]: ../ReactiveCocoaFramework/ReactiveCocoa/RACCommand.h
[RACDisposable]: ../ReactiveCocoaFramework/ReactiveCocoa/RACDisposable.h
[RACEvent]: ../ReactiveCocoaFramework/ReactiveCocoa/RACEvent.h
[RACMulticastConnection]: ../ReactiveCocoaFramework/ReactiveCocoa/RACMulticastConnection.h
[RACReplaySubject]: ../ReactiveCocoaFramework/ReactiveCocoa/RACReplaySubject.h
[RACScheduler]: ../ReactiveCocoaFramework/ReactiveCocoa/RACScheduler.h
[RACSequence]: ../ReactiveCocoaFramework/ReactiveCocoa/RACSequence.h
[RACSignal]: ../ReactiveCocoaFramework/ReactiveCocoa/RACSignal.h
[RACSignal+Operations]: ../ReactiveCocoaFramework/ReactiveCocoa/RACSignal+Operations.h
[RACStream]: ../ReactiveCocoaFramework/ReactiveCocoa/RACStream.h
[RACSubject]: ../ReactiveCocoaFramework/ReactiveCocoa/RACSubject.h
[RACSubscriber]: ../ReactiveCocoaFramework/ReactiveCocoa/RACSubscriber.h
[RACTuple]: ../ReactiveCocoaFramework/ReactiveCocoa/RACTuple.h
[RACUnit]: ../ReactiveCocoaFramework/ReactiveCocoa/RACUnit.h
[README]: ../README.md
[seq]: http://clojure.org/sequences
