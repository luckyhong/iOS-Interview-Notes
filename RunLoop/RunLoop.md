### 1.`Runloop` 和线程的关系？

- 一个线程对应一个 `Runloop`
- 主线程的默认就有了 `Runloop`
- 子线程的 `Runloop` 以懒加载的形式创建
- `Runloop` 存储在一个全局的可变字典里，线程是 `key` ，`Runloop` 是 `value`

`Runloop` 和线程是密切相关的，因为 `Runloop` 是线程的事件循环机制。每个线程都有一个 `Runloop` 对象与之关联。

`Runloop` 负责处理线程中的事件，包括处理输入源（如点击事件、网络请求等）、定时器和触摸事件等。当线程中有事件发生时，`Runloop` 会接收到事件并将其分发给相应的处理代码。如果线程中没有事件发生，`Runloop` 会使线程进入休眠状态，以节省资源。

`Runloop` 的主要作用是管理线程中的事件循环，使线程能够高效地处理事件并保持响应性。在开发中，通常会使用 `NSRunLoop` 类来访问当前线程的 `Runloop` 对象，并使用 `CFRunLoop` 函数来管理 `Runloop`。

总之，`Runloop` 是线程的事件循环机制，负责处理线程中的事件，使线程能够高效地响应事件并保持活跃状态。

MJ简答：

- 每条线程都有唯一的一个与之对应的RunLoop对象
- RunLoop保存在一个全局的Dictionary里，线程作为key，RunLoop作为value
- 线程刚创建时并没有RunLoop对象，RunLoop会在第一次获取它时创建
- RunLoop会在线程结束时销毁
- 主线程的RunLoop已经自动获取（创建），子线程默认没有开启RunLoop

### 2.讲一下 `Runloop` 的 `Mode`?

- 一个RunLoop包含5个 `Mode`，常用的有三个，其中一个是占位用的
- `Mode` 里面有一个或者多个 `Source`、`timer`、 `Observer`
  - `Source`事件源，分 0 和 1；本质就是函数回调
  - `timer` 就是计时器
  - `Observer` 顾名思义观察者
- RunLoop启动时只能选择其中一个Mode，作为currentMode
- `Mode` 的切换（需要切换Mode，只能退出当前Loop，再重新选择一个Mode进入）
- 如果Mode里没有任何Source0/Source1/Timer/Observer，RunLoop会立马退出

`Runloop` 的 `Mode` 是一个字符串，用于表示 `Runloop` 处理事件的方式和优先级。每个 `Runloop` 都可以关联多个 `Mode`，并且可以在不同的 `Mode` 下处理不同类型的事件。

在 iOS 和 macOS 中，有多个预定义的 `Mode`，比如 `NSDefaultRunLoopMode`、`NSRunLoopCommonModes`、`UITrackingRunLoopMode` 等。其中，`NSDefaultRunLoopMode` 是默认的 `Mode`，用于处理大部分事件；`NSRunLoopCommonModes` 则是一个共享的 `Mode`，可以同时处理多个 `Mode` 中的事件；`UITrackingRunLoopMode` 是一个专门用于处理用户滑动操作的 `Mode`。

除了预定义的 `Mode`，开发者还可以自定义 `Mode`，以满足特定的需求。在自定义 `Mode` 时，需要指定该 `Mode` 下要处理的事件类型，并使用 `CFRunLoopModeRef` 或 `NSRunLoopMode` 类型来表示。

在 `Runloop` 中，事件会按照 `Mode` 的优先级进行处理。当某个 `Mode` 中有事件需要处理时，`Runloop` 会切换到该 `Mode` 并处理事件；如果该 `Mode` 中没有事件需要处理，`Runloop` 则会切换到下一个优先级更高的 `Mode`，直到所有 `Mode` 中的事件都被处理完毕。

总之，`Runloop` 的 `Mode` 是一个字符串，用于表示 `Runloop` 处理事件的方式和优先级。每个 `Runloop` 可以关联多个 `Mode`，并且可以在不同的 `Mode` 下处理不同类型的事件。开发者可以使用预定义的 `Mode`，也可以自定义 `Mode` 来满足特定的需求。

下面以 iOS 开发中的实际场景为例，说明如何使用 `Runloop` 的 `Mode`。

假设我们有一个应用程序，其中包含一个长时间运行的网络请求。当用户在应用程序中进行其他操作时，我们希望仍然能够响应网络请求的回调，即使用户在其他 `ViewController` 中打开了模态视图或者切换了标签页等。

为了实现这个功能，我们可以在应用程序启动时，创建一个后台线程，并在该线程中启动一个 `Runloop`。在网络请求的回调中，我们将结果传递给主线程，并使用 `performSelectorOnMainThread:withObject:waitUntilDone:modes:` 方法来执行回调方法。其中，我们可以通过指定 `modes` 参数来指定回调方法在哪个 `Mode` 下执行。

下面是示例代码：

```objective-c
- (void)startBackgroundTask {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // 启动一个 Runloop
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    });
}

- (void)performNetworkRequest {
    // 发起网络请求
    [MyNetworkManager sendRequestWithCompletionHandler:^(id result) {
        // 在主线程执行回调
        [self performSelectorOnMainThread:@selector(handleNetworkResponse:)
                               withObject:result
                            waitUntilDone:NO
                                    modes:@[NSDefaultRunLoopMode, UITrackingRunLoopMode]];
    }];
}

- (void)handleNetworkResponse:(id)result {
    // 处理网络请求结果
}
```

在上面的代码中，我们在后台线程中启动了一个 `Runloop`，并在网络请求的回调中使用 `performSelectorOnMainThread:withObject:waitUntilDone:modes:` 方法来执行回调方法。由于我们指定了 `modes` 参数为 `@[NSDefaultRunLoopMode, UITrackingRunLoopMode]`，所以回调方法会在这两个 `Mode` 下执行。这样，即使用户在其他 `ViewController` 中打开了模态视图或者切换了标签页等，我们仍然能够响应网络请求的回调。

### 3.讲一下你在项目中是如何使用 `Runloop` 的 `Mode`?

在实际开发中，`Runloop` 的 `Mode` 可以用于多种场景，例如：

1. 在网络请求中使用 `Runloop` 的 `Mode` 来确保网络请求回调在正确的线程中执行，以避免线程竞争和死锁等问题。
2. 在 UI 线程中使用 `Runloop` 的 `Mode` 来监听定时器事件。在 iOS 开发中，通常使用 `NSTimer` 来创建定时器。如果不指定定时器的 `Mode`，则定时器事件将会在默认的 `NSDefaultRunLoopMode` 下执行，这可能会导致定时器事件在处理其他 UI 事件时被阻塞。为了避免这种情况，可以将定时器的 `Mode` 设置为 `NSRunLoopCommonModes`，以让定时器事件在多个 `Mode` 下执行。
3. 在 UI 线程中使用 `Runloop` 的 `Mode` 来监听输入源事件。例如，可以使用 `CFRunLoopSource` 来创建一个输入源，然后将输入源添加到 `Runloop` 中，并指定输入源事件监听的 `Mode`。这样，在指定的 `Mode` 下，当输入源事件发生时，`Runloop` 就会调用相应的回调方法。
4. 在多线程场景中使用 `Runloop` 的 `Mode` 来协调不同线程之间的操作。例如，可以在后台线程中启动一个 `Runloop`，并在主线程中使用 `performSelector:onThread:withObject:waitUntilDone:modes:` 方法来将任务发送到后台线程中执行。这样，在指定的 `Mode` 下，后台线程就会响应任务，并在执行完任务后再次进入休眠状态，等待下一个任务的到来。

总之，`Runloop` 的 `Mode` 在 iOS 项目中有多种用途，可以用于处理网络请求、定时器、输入源和多线程协调等场景。熟练掌握 `Runloop` 的 `Mode` 使用方法，可以帮助开发者更好地管理线程和事件循环，提高应用程序的性能和响应性。

### 4.讲一下 `Observer` ？

```objective-c
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```

`Runloop` 中的 `Observer` 是一种监听机制，开发者可以通过添加 `Observer` 来监听 `Runloop` 的状态变化并执行相应的操作。在 iOS 和 macOS 开发中，`Observer` 通常用于监控应用程序的状态，例如在应用程序进入休眠状态时释放资源、在应用程序恢复活动状态时重新加载数据等。

`Runloop` 中的 `Observer` 主要分为以下四类：

1. `kCFRunLoopEntry`：当 `Runloop` 进入循环时触发，可以在此时执行一些初始化操作。
2. `kCFRunLoopBeforeTimers`：当 `Runloop` 即将处理定时器事件时触发。
3. `kCFRunLoopBeforeSources`：当 `Runloop` 即将处理输入源事件时触发。
4. `kCFRunLoopExit`：当 `Runloop` 退出循环时触发，可以在此时执行一些清理操作。

开发者可以通过 `CFRunLoopObserverCreate` 函数创建一个 `Observer`，并指定要监听的状态和相应的回调方法。在回调方法中，开发者可以执行需要的操作，例如释放资源、重新加载数据等。可以使用 `CFRunLoopAddObserver` 函数将 `Observer` 添加到 `Runloop` 中，以便监听 `Runloop` 的状态变化。

下面是一个简单的示例代码，演示了如何使用 `Observer` 监听 `Runloop` 的状态变化：

```objective-c
static void runLoopObserverCallback(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    switch (activity) {
        case kCFRunLoopEntry:
            NSLog(@"Runloop entered");
            break;
        case kCFRunLoopBeforeTimers:
            NSLog(@"Runloop about to process timers");
            break;
        case kCFRunLoopBeforeSources:
            NSLog(@"Runloop about to process sources");
            break;
        case kCFRunLoopExit:
            NSLog(@"Runloop exited");
            break;
        default:
            break;
    }
}

- (void)addRunLoopObserver {
    CFRunLoopObserverContext context = {0, (__bridge void *)(self), NULL, NULL, NULL};
    CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                                            kCFRunLoopAllActivities,
                                                            YES,
                                                            0,
                                                            &runLoopObserverCallback,
                                                            &context);
    CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kCFRunLoopDefaultMode);
    CFRelease(observer);
}
```

在上面的代码中，我们创建了一个 `Observer`，并指定要监听 `Runloop` 的所有状态变化。在回调方法中，我们根据状态变化的类型输出相应的日志。最后，我们使用 `CFRunLoopAddObserver` 函数将 `Observer` 添加到当前线程的 `Runloop` 中。

`Runloop` 中的 `Observer` 其实就是一种监听机制，可以用于监听 `Runloop` 的状态变化并执行相应的操作。通过添加 `Observer`，开发者可以在 `Runloop` 进入循环、处理定时器事件、处理输入源事件和退出循环等不同状态下执行自定义操作，从而更好地管理线程和事件循环。

### 5.讲一下 `Runloop` 的内部实现逻辑？

`Runloop` 是 iOS 和 macOS 中事件循环的实现机制，用于管理线程的事件处理和休眠状态。它的内部实现逻辑可以分为以下几个部分：

1. `Runloop` 的启动和退出：当 `Runloop` 启动时，它会进入一个循环中，并等待事件的到来。当没有事件需要处理时，`Runloop` 会进入休眠状态，直到有新的事件到来或者超时时间到达。当 `Runloop` 退出时，它会清理资源并返回。
2. `Runloop` 的模式切换：一个 `Runloop` 可以关联多个不同的 `Mode`，每个 `Mode` 可以处理不同类型的事件。在处理事件时，`Runloop` 会根据当前的 `Mode` 选择相应的事件进行处理。当 `Runloop` 在一个 `Mode` 下没有事件需要处理时，它会自动切换到下一个优先级更高的 `Mode`，直到所有 `Mode` 中的事件都被处理完毕。
3. `Runloop` 的事件处理：当 `Runloop` 接收到一个事件时，它会根据当前的 `Mode` 选择相应的事件处理器进行处理。事件处理器可以是由应用程序提供的回调方法，也可以是系统提供的默认处理器。在处理事件时，`Runloop` 会将事件发送给事件处理器，并等待事件处理器的返回结果。如果事件处理器返回一个非零值，`Runloop` 会暂停处理事件，并等待下一个事件的到来。
4. `Runloop` 的定时器：`Runloop` 可以使用定时器来执行一些周期性的任务。当定时器到期时，`Runloop` 会将定时器事件添加到事件队列中，并在相应的 `Mode` 下等待事件处理器的处理。在处理定时器事件时，`Runloop` 会先判断定时器是否已经过期，如果过期则将定时器事件添加到事件队列中，否则直接将事件发送给事件处理器。

总之，`Runloop` 的内部实现逻辑涉及到**启动和退出**、**模式切换**、**事件处理**和**定时器**等多个方面。

### 6.`autoreleasePool` 在何时被释放？

`@autoreleasepool` 在 Objective-C 中用于管理内存的释放。它的生命周期取决于作用域，当程序执行到 `@autoreleasepool` 的结束花括号时，自动释放池中的对象将被释放。同时，自动释放池也会在特定的情况下自动释放，例如在 Run Loop 完成一次迭代后、使用 `NSThread` 创建新线程时、使用 GCD 执行块时等。eg：

1. 一个线程的主 `@autoreleasepool` 在这个线程的 Run Loop 中完成一次迭代之后，就会被释放。
2. 当使用 `NSThread` 的 `detachNewThreadSelector:toTarget:withObject:` 方法创建新线程时，新线程会使用自己的 `@autoreleasepool`，并在任务完成后自动释放。
3. 在使用 GCD（Grand Central Dispatch）时，当执行块返回时，GCD 会自动释放自动释放池中的对象。

### 7.解释一下 `事件响应` 的过程？

事件响应的过程可以分为以下几个步骤：

1. 事件产生：事件可以是用户触摸屏幕、按下按钮等。
2. 事件传递：当事件产生时，iOS 系统会将事件传递给最上层的视图对象，并沿着视图层次结构向下传递，直到找到第一个能够处理事件的对象为止。如果事件到达了视图层次结构的最底层仍然没有被处理，则事件会被丢弃。
3. 事件处理：当视图对象接收到事件时，它会首先检查自己是否能够处理事件。如果能够处理，则视图对象会调用自己的响应方法，例如 `touchesBegan:withEvent:` 方法来响应事件。如果不能处理，则视图对象会将事件传递给它的下一个响应者对象，直到事件被处理为止。
4. 响应链：视图对象之间形成了一个响应链，事件从最上层的视图对象开始向下传递，直到找到第一个能够处理事件的对象为止。响应链可以通过 `nextResponder` 属性进行连接，每个视图对象都可以通过该属性找到下一个响应者对象。
5. 事件响应结束：当事件被处理后，它会被标记为已处理，并从视图层次结构中移除。如果事件被传递到了视图层次结构的最底层仍然没有被处理，则事件会被丢弃。

总之，事件响应是指用户与应用程序交互时，应用程序接收到事件并做出响应的过程。在 iOS 中，事件会从最上层的视图对象开始向下传递，直到找到第一个能够处理事件的对象为止。每个视图对象都可以通过 `nextResponder` 属性找到下一个响应者对象，形成一个响应链。

### 8.解释一下 `手势识别` 的过程？

当 `_UIApplicationHandleEventQueue() `识别了一个手势时，其首先会调用 `Cancel` 将当前的 `touchesBegin/Move/End` 系列回调打断。随后系统将对应的 `UIGestureRecognizer` 标记为待处理。

苹果注册了一个 `Observer` 监测 `BeforeWaiting` (Loop即将进入休眠) 事件，这个 `Observer` 的回调函数是 `_UIGestureRecognizerUpdateObserver()`，其内部会获取所有刚被标记为待处理的 `GestureRecognizer`，并执行`GestureRecognizer` 的回调。

当有 `UIGestureRecognizer` 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

### 9.解释一下 `GCD` 在 `Runloop` 中的使用？

`GCD`由 子线程 返回到 主线程，只有在这种情况下才会触发 RunLoop。会触发 RunLoop 的 Source 1 事件。

### 10.解释一下 `NSTimer`？

`NSTimer` 其实就是 `CFRunLoopTimerRef`，他们之间是 `toll-free bridged` 的。一个 `NSTimer` 注册到 `RunLoop` 后，`RunLoop` 会为其重复的时间点注册好事件。例如 `10:00`, `10:10`, `10:20` 这几个时间点。`RunLoop` 为了节省资源，并不会在非常准确的时间点回调这个`Timer`。`Timer` 有个属性叫做 `Tolerance` (宽容度)，标示了当时间点到后，容许有多少最大误差。

如果某个时间点被错过了，例如执行了一个很长的任务，则那个时间点的回调也会跳过去，不会延后执行。就比如等公交，如果 10:10 时我忙着玩手机错过了那个点的公交，那我只能等 10:20 这一趟了。

`CADisplayLink` 是一个和屏幕刷新率一致的定时器（但实际实现原理更复杂，和 NSTimer 并不一样，其内部实际是操作了一个 `Source`）。如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和 `NSTimer` 相似），造成界面卡顿的感觉。在快速滑动 `TableView` 时，即使一帧的卡顿也会让用户有所察觉。`Facebook` 开源的 `AsyncDisplayLink` 就是为了解决界面卡顿的问题，其内部也用到了 `RunLoop`。

NSTimer 循环引用示例：

在 iOS 中，`NSTimer` 可能会导致循环引用的问题。当一个 `NSTimer` 对象被加入到 RunLoop 中时，会对其 target 对象进行强引用，如果 target 对象同时也强引用了 `NSTimer` 对象，就会导致循环引用的问题。循环引用会导致内存泄漏，使得程序占用的内存越来越多，最终导致程序崩溃。

以下是一个 `NSTimer` 循环引用的示例：

```objective-c
@interface MyViewController : UIViewController

@property(nonatomic, strong) NSTimer *myTimer;

@end

@implementation MyViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.myTimer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(timerDidFire) userInfo:nil repeats:YES];
}

- (void)timerDidFire {
    NSLog(@"Timer did fire");
}

@end
```

在上面的代码中，`MyViewController` 对象会强引用 `NSTimer` 对象，而 `NSTimer` 对象又会强引用 `MyViewController` 对象，从而造成循环引用。当 `MyViewController` 对象被释放时，由于 `NSTimer` 对象对其进行了强引用，导致 `MyViewController` 对象无法被释放，从而造成内存泄漏。

为了避免循环引用的问题，可以使用弱引用来解决。例如，在上面的代码中，可以将 `MyViewController` 对象声明为弱引用，如下所示：

```objective-c
@interface MyViewController : UIViewController

@property(nonatomic, weak) NSTimer *myTimer;

@end

@implementation MyViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.myTimer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(timerDidFire) userInfo:nil repeats:YES];
}

- (void)timerDidFire {
    NSLog(@"Timer did fire");
}

@end
```

在上面的代码中，将 `MyViewController` 对象声明为弱引用，避免了循环引用的问题。当 `MyViewController` 对象被释放时，由于 `NSTimer` 对象只是对其进行了弱引用，所以不会造成内存泄漏的问题。

需要注意的是，由于 `NSTimer` 是基于 Run Loop 实现的，所以在使用 `myTimer` 对象时，必须保证 Run Loop 一直处于运行状态，否则 `myTimer` 对象将不会触发。如果在子线程中使用 `myTimer` 对象，还需要手动创建 Run Loop 并将其添加到子线程的线程中。

### 11.`AFNetworking` 中如何运用 `Runloop`?

`AFURLConnectionOperation` 这个类是基于 `NSURLConnection` 构建的，其希望能在后台线程接收 `Delegate` 回调。为此 `AFNetworking` 单独创建了一个线程，并在这个线程中启动了一个 `RunLoop`：

```objective-c
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}

+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}
```

`RunLoop` 启动前内部必须要有至少一个 `Timer`/`Observer`/`Source`，所以 `AFNetworking` 在 `[runLoop run]` 之前先创建了一个新的 `NSMachPort` 添加进去了。通常情况下，调用者需要持有这个 `NSMachPort (mach_port)` 并在外部线程通过这个 `port` 发送消息到 `loop` 内；但此处添加 `port` 只是为了让 `RunLoop` 不至于退出，并没有用于实际的发送消息。

```objective-c
- (void)start {
    [self.lock lock];
    if ([self isCancelled]) {
        [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    } else if ([self isReady]) {
        self.state = AFOperationExecutingState;
        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    [self.lock unlock];
}
```

当需要这个后台线程执行任务时，`AFNetworking` 通过调用 `[NSObject performSelector:onThread:..]` 将这个任务扔到了后台线程的 `RunLoop` 中。

在 AFNetworking 中，Runloop 被广泛用于异步网络请求和响应处理。具体来说，当使用 AFNetworking 进行网络请求时，AFNetworking 会将请求和响应操作放入 Runloop 中执行，以保证网络请求能够异步执行，不会阻塞主线程。

AFNetworking 通过使用 `AFURLConnectionOperation` 类来实现网络请求和响应操作。`AFURLConnectionOperation` 类继承自 `NSOperation` 类，它将网络请求和响应操作封装为一个操作对象，并将其加入到操作队列中执行。在操作执行过程中，AFNetworking 会使用 Runloop 来处理网络请求和响应事件，以便及时响应网络事件，不会阻塞主线程。

具体来说，AFNetworking 在 `AFURLConnectionOperation` 的执行过程中，会将 `NSURLConnection` 对象加入到 Runloop 中，并使用 `CFRunLoopRun` 函数来启动 Runloop，等待网络请求和响应事件的发生。当网络事件发生时，Runloop 会立即响应事件，并调用相应的回调函数，以便处理网络请求和响应操作。在网络请求和响应操作完成后，AFNetworking 会使用 `CFRunLoopStop` 函数停止 Runloop，以便终止网络请求和响应操作。

需要注意的是，由于 Runloop 是基于线程的，因此在使用 AFNetworking 时，必须保证在主线程中使用 Runloop。如果在子线程中使用 Runloop，可能会导致线程阻塞，从而影响程序的性能和响应速度。因此，AFNetworking 会自动将网络请求和响应操作放入主线程的 Runloop 中执行，以保证网络请求的顺利进行。

简记：AFNetworking 中广泛使用 Runloop 来实现异步网络请求和响应处理。它将网络请求和响应操作封装为一个操作对象，并将其加入到操作队列中执行。在操作执行过程中，AFNetworking 会使用 Runloop 来处理网络请求和响应事件，以保证网络请求能够异步执行，不会阻塞主线程。

### 12.`PerformSelector` 的实现原理？

当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。

### 13.利用 `runloop` 解释一下页面的渲染的过程？

当我们调用 `[UIView setNeedsDisplay]` 时，这时会调用当前 `View.layer` 的 `[view.layer setNeedsDisplay]`方法。

这等于给当前的 `layer` 打上了一个脏标记，而此时并没有直接进行绘制工作。而是会到当前的 `Runloop` 即将休眠，也就是 `beforeWaiting` 时才会进行绘制工作。

紧接着会调用 `[CALayer display]`，进入到真正绘制的工作。`CALayer` 层会判断自己的 `delegate` 有没有实现异步绘制的代理方法 `displayer:`，这个代理方法是异步绘制的入口，如果没有实现这个方法，那么会继续进行系统绘制的流程，然后绘制结束。

`CALayer` 内部会创建一个 `Backing Store`，用来获取图形上下文。接下来会判断这个 `layer` 是否有 delegate。

如果有的话，会调用 `[layer.delegate drawLayer:inContext:]`，并且会返回给我们 `[UIView DrawRect:]` 的回调，让我们在系统绘制的基础之上再做一些事情。

如果没有 `delegate`，那么会调用 `[CALayer drawInContext:]`。

以上两个分支，最终 `CALayer` 都会将位图提交到 `Backing Store`，最后提交给 `GPU`。

至此绘制的过程结束。

要注意的是，页面渲染过程中的每个步骤都需要保证主线程一直处于运行状态，以便及时响应用户的操作和事件。在这个过程中，Runloop 扮演了重要的角色，它可以保证主线程一直处于运行状态，同时也可以处理用户的操作和事件，以便及时响应用户的需求。

### 14.如何使用 `Runloop` 实现一个常驻线程？这种线程一般有什么作用？

一般做法是向 `Runloop` 中放一个 `port`。

使用 Runloop 实现一个常驻线程的过程如下：

1. 创建一个新的线程，在这个线程中启动 Runloop。
2. 在 Runloop 中添加输入源或定时器等任务，在这些任务的回调函数中执行相应的操作。
3. 在需要执行任务的时候，向这个线程发送消息，并在线程中执行相应的任务。
4. 当不需要这个线程的时候，停止 Runloop 并退出线程。

常驻线程的作用主要有两个方面：

1. 执行长时间运行的任务：有些任务需要长时间运行，例如网络请求、数据处理、文件下载等，如果在主线程中执行，会导致 UI 卡顿或无响应。使用常驻线程可以将这些长时间运行的任务放入另一个线程中执行，避免影响主线程的性能和响应速度。
2. 实现后台任务：有些任务需要在应用进入后台或锁屏状态下继续执行，例如音乐播放、定位服务等，这时候就需要使用常驻线程来保证任务的持续执行。

使用常驻线程需要谨慎，因为它会一直占用系统资源，导致系统性能下降。因此，在使用常驻线程时，需要考虑线程的优先级和资源占用情况，以避免影响系统的正常运行。同时，也需要注意线程安全和内存管理等问题，以保证线程的稳定性和可靠性。

### 15.为什么 `NSTimer` 有时候不好使？

因为创建的 `NSTimer` 默认是被加入到了 `defaultMode`，所以当 `Runloop` 的 `Mode` 变化时，当前的 `NSTimer` 就不会工作了。

### 16.`PerformSelector:afterDelay:`这个方法在子线程中是否起作用？为什么？怎么解决？

不起作用，子线程默认没有 `Runloop`，也就没有 `Timer`。解决的办法是可以使用 `GCD` 来实现：`Dispatch_after`

### 17.什么是异步绘制？

异步绘制，就是可以在子线程把需要绘制的图形，提前在子线程处理好。将准备好的图像数据直接返给主线程使用，这样可以降低主线程的压力。

异步绘制指的是在后台线程中进行视图的绘制操作，以避免在主线程中进行视图绘制而导致的卡顿和延迟现象。异步绘制通常用于绘制复杂的视图或者在主线程上进行其他重要任务的同时，保证界面的流畅性和响应速度。

在 iOS 7 之后，Apple 引入了异步绘制机制，即 `drawViewHierarchyInRect:afterScreenUpdates:` 方法。这个方法可以在后台线程中进行视图的绘制操作，然后再将绘制结果绘制到屏幕上，从而避免在主线程中进行视图绘制而导致的卡顿和延迟现象。

具体来说，异步绘制的过程如下：

1. 在后台线程中调用 `drawViewHierarchyInRect:afterScreenUpdates:` 方法，执行视图的绘制操作。
2. 将绘制结果保存到离屏缓存中。
3. 在主线程中调用 `setNeedsDisplay` 方法，将离屏缓存中的绘制结果绘制到屏幕上。

使用异步绘制可以有效地提高界面的流畅性和响应速度，尤其是在绘制复杂视图或者进行其他重要任务时，可以避免因视图绘制而导致的卡顿和延迟现象，从而提高用户体验。要注意异步绘制需要在适当的时机调用，以避免影响界面的正常显示和操作。同时，也需要注意线程安全和内存管理等问题，以保证程序的稳定性和可靠性。

### 18.如何检测 `App` 运行过程中是否卡顿？

在 App 的运行过程中，如果出现卡顿现象，会给用户带来不良的使用体验，因此检测 App 运行过程中是否卡顿是非常有必要的。下面介绍一些常用的方法来检测 App 运行过程中是否卡顿：

1. FPS 监测：FPS（Frames Per Second）是用来衡量 App 动画流畅度的指标，通常在每秒钟渲染的帧数超过 60 帧时，我们认为动画比较流畅。可以通过在 App 中添加 FPS 监测工具来检测 App 的 FPS 值，从而了解 App 的动画流畅度。常用的 FPS 监测工具有 YYFPSLabel、FBMemoryProfiler 等。
2. 卡顿检测：卡顿检测可以通过设置一个阈值来检测 App 是否出现卡顿现象。通常我们认为当 App 运行过程中，某个任务执行时间超过 100 毫秒时，就会出现卡顿现象。可以在主线程中添加一个计时器，定时检测主线程是否在规定时间内完成任务，如果未完成，则认为出现了卡顿现象。常用的卡顿检测工具有 FBRetainCycleDetector、fishhook 等。
3. Crash 日志检测：Crash 日志可以记录 App 在运行过程中发生的异常情况，包括卡顿、闪退等问题。通过检测 Crash 日志，可以了解 App 的异常情况，从而找到卡顿的原因。可以使用 Xcode 或者第三方工具来查看 Crash 日志，如 PLCrashReporter 等。

检测 App 运行过程中是否卡顿需要在实际使用场景中进行测试，以便发现问题并进行优化。

### 19.`Runloop` 在实际开发中的应用

- 控制线程生命周期（线程保活）
- 解决NSTimer在滑动时停止工作的问题
- 监控应用卡顿
- 性能优化

### 20.程序中添加每3秒响应一次的NSTimer，当拖动tableview时timer可能无法响应要怎么解决？

当拖动 `UITableView` 时，由于主线程忙于处理滚动事件，可能会导致添加的每 3 秒响应一次的 `NSTimer` 无法及时响应。为了解决这个问题，可以考虑以下两种方法：

- 使用 RunLoop 模式：可以将 `NSTimer` 添加到 `NSRunLoop` 中，并将其运行模式设置为 `NSRunLoopCommonModes`，这样 `NSTimer` 就可以在主线程的多个模式下响应。具体实现如下：

```objective-c
NSTimer *timer = [NSTimer timerWithTimeInterval:3.0 target:self selector:@selector(handleTimer:) userInfo:nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

- 暂停 Timer：在 `UITableView` 开始滚动时，可以暂停 `NSTimer`，等到滚动结束后再恢复 `NSTimer` 的运行。具体实现如下：

```objective-c
- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView {
    [self.timer setFireDate:[NSDate distantFuture]];
}

- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate {
    [self.timer setFireDate:[NSDate dateWithTimeIntervalSinceNow:3.0]];
}
```

### 21.`NSTimer`  与 `Runloop`  的关系？

`NSTimer` 是基于 `RunLoop` 实现的计时器，因此在使用 `NSTimer` 时需要了解 `RunLoop` 的相关知识。

`RunLoop` 是 iOS 中用于处理事件循环的机制，它可以监听各种输入源（如触摸事件、网络事件等），并在有事件发生时触发相应的回调函数。`NSTimer` 就是一种输入源，它可以在指定的时间间隔内重复触发回调函数。

使用 `NSTimer` 时，需要将其添加到 `RunLoop` 的运行循环中，并指定相应的运行模式。如果没有指定运行模式，则默认使用 `NSDefaultRunLoopMode`，这意味着 `NSTimer` 只能在当前 RunLoop 的默认模式下运行，当 RunLoop 进入其他模式时，`NSTimer` 就会停止运行。因此，在使用 `NSTimer` 时，需要将其运行模式设置为 `NSRunLoopCommonModes` 或者自定义模式，这样 `NSTimer` 就可以在多个运行模式下运行。

另外，由于 `NSTimer` 是基于 `RunLoop` 实现的，因此在使用 `NSTimer` 时需要注意一些细节问题，例如循环引用、定时器精度问题、主线程卡顿问题等。为了避免这些问题，可以考虑使用 GCD 的定时器或者 CADisplayLink 等替代方案。

### 22.`Runloop`  是怎么响应用户操作的， 具体流程是什么样的？

`RunLoop` 是 iOS 中用于处理事件循环的机制，主要用于监听各种输入源（如触摸事件、网络事件等），并在有事件发生时触发相应的回调函数。当用户在 iOS 设备上进行操作时，`RunLoop` 会响应用户的操作，具体流程如下：

1. 用户进行操作，例如点击屏幕、滚动列表等。
2. 操作会产生事件，例如触摸事件、滚动事件等。
3. 事件被添加到系统的事件队列中。
4. `RunLoop` 开始监听事件队列，当有事件发生时，就会触发相应的回调函数。
5. 回调函数被执行，处理相应的事件。
6. `RunLoop` 继续监听事件队列，等待下一个事件的发生。

在 iOS 中，主线程的 `RunLoop` 通常会运行在 `NSDefaultRunLoopMode` 模式下，这意味着主线程只能处理默认模式下的事件，当其他模式下的事件发生时，主线程的 `RunLoop` 就会进入休眠状态。为了让主线程的 `RunLoop` 能够处理其他模式下的事件，可以使用 `NSRunLoopCommonModes` 或者自定义模式。

需要注意的是，`RunLoop` 的运行循环是一个无限循环，只有在有事件发生时才会触发回调函数，因此需要避免在主线程中执行过多的耗时操作，以免阻塞主线程的运行循环，导致用户界面卡顿或者失去响应。同时，也需要注意线程安全和内存管理等问题，以保证程序的稳定性和可靠性。