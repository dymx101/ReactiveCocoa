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

**Command**由[RACCommand][]类表示，在某个action发生时触发，创建和订阅一个signal。这就让他很容易在用户和app交互时，执行side-effecting的工作。

通常由action触发command是UI驱动的，比如说点击了一个按钮。Commands也可以基于一个signal而被自动disable掉，而且这个disabled的状态可以由禁用同此command相关的任何控件来表现。

在OS X上，RAC为[NSButton][NSButton+RACCommandSupport]增加了一个`rac_command`属性，该属性会自动设置好这些行为。

### Connections

Connection由[RACMulticastConnection][]类表示，是一个可以被任意数量的subscriber共享的[subscription](#subscription)。

[Signals](#signals)默认的状态是_cold_，也就是他们每增加一个新的subscription的时候才开始工作。这种行为通常是我们所需要的，因为数据会为每个subscriber重新计算，但是这也可能带来问题，比如如果signal有side effects或者signal的工作很耗资源(例如发送网络请求)。

一个connect是通过 [RACSignal][RACSignal+Operations]上的 `-publish` 或 `-multicast:` 方法创建，确保只有一个基础subscription被创建，不管connection被订阅多少次。一旦connected, connection的signal据说就变_hot_了，基础subscription将一直保持活动直到所有对connection的subscriptions都被[处理掉(disposed)](#disposables)为止。

## Sequences

**Sequence**由 RACSequence类代表，是一个 _pull-driven_的[stream](#streams)。

Sequence是一种集合类型，同`NSArray`的目标相似。但是和数组不同的是，sequence中的值默认是lazily加载的(i.e. 仅当他们被需要时)，如果仅仅sequence的一部分被使用，就潜在的提升了性能。就像Cocoa的其他集合类型，sequences不可以是nil。

Sequences类似于 闭包的序列 ([Clojure's sequences][seq]) (即[lazy-seq][])，或者是[Haskell][]中的[List][]。

RAC 增加了一个 `-rac_sequence`方法到多数的Cocoa集合类型当中，允许他们被作为 [RACSequences][RACSequence] 使用。

## Disposables

**[RACDisposable][]** 类被用于取消和资源清理。

Disposables通常用来取消订阅一个[signal](#signals)。当一个[subscription](#subscription)被disposed时，相应的subscriber将不再从signal收到任何事件。而且，任何同subscription相关的工作(后台处理，网络请求等)将被取消，因为不再需要他们返回结果。

更多关于cancellation的信息，参见 RAC [Design Guidelines][]。

## Schedulers

**Scheduler**由[RACScheduler][]代表，是一系列[signals](#signals)用于进行工作和发送结果的执行队列。

Schedulers类似于GCD队列，不同的是，schedulers支持取消(通过 [disposables](#disposables))，并且总是顺序的执行。除了+immediateScheduler，schedulers不提供同步执行。这就有助于避免死锁，并鼓励使用 [signal operators][RACSignal+Operations]，代替blocking。

[RACScheduler][] 也在某种程度上类似于`NSOperationQueue`，但是schedulers不允许任务被重新排序或者互相依赖。

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
