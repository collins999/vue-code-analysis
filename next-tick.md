```
/*  */

var isUsingMicroTask = false;

var callbacks = [];
var pending = false;

function flushCallbacks() {
    pending = false;
    // 因为，会出现这么一种情况：nextTick 的回调函数中还使用 nextTick。
    // 如果 flushCallbacks 不做特殊处理，直接循环执行回调函数，会导致里面nextTick 中的回调函数会进入回调队列。
    var copies = callbacks.slice(0);
    callbacks.length = 0;
    for (var i = 0; i < copies.length; i++) {
        copies[i]();
    }
}

// 这里我们有使用微任务的异步延迟包装器。
// 在2.5中，我们使用（宏）任务（与微任务结合）。
// 但是，当状态在重新绘制之前更改时，它有一些微妙的问题
//（例如6813，在过渡中）。
// 另外，在事件处理程序中使用（宏）任务会导致一些奇怪的行为
// 这是无法避免的（例如7109、7153、7546、7834和8109）。
// 所以我们现在在任何地方都使用微任务。
// 这种权衡的一个主要缺点是
// 如果微任务的优先级太高，并且可能会在两者之间触发
// 顺序事件（例如有解决方法的#4521、#6690）
// 甚至在同一事件的冒泡之间（#6566）。
var timerFunc;

// The nextTick behavior leverages the microtask queue, which can be accessed
// via either native Promise.then or MutationObserver.
// MutationObserver has wider support, however it is seriously bugged in
// UIWebView in iOS >= 9.3.3 when triggered in touch event handlers. It
// completely stops working after triggering a few times... so, if native
// Promise is available, we will use it:
/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
    var p = Promise.resolve();
    timerFunc = function() {
        p.then(flushCallbacks);
        // In problematic UIWebViews, Promise.then doesn't completely break, but
        // it can get stuck in a weird state where callbacks are pushed into the
        // microtask queue but the queue isn't being flushed, until the browser
        // needs to do some other work, e.g. handle a timer. Therefore we can
        // "force" the microtask queue to be flushed by adding an empty timer.
        if (isIOS) { setTimeout(noop); }
    };
    isUsingMicroTask = true;
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
        isNative(MutationObserver) ||
        // PhantomJS and iOS 7.x
        MutationObserver.toString() === '[object MutationObserverConstructor]'
    )) {
    // Use MutationObserver where native Promise is not available,
    // e.g. PhantomJS, iOS7, Android 4.4
    // (#6466 MutationObserver is unreliable in IE11)
    var counter = 1;
    var observer = new MutationObserver(flushCallbacks);
    var textNode = document.createTextNode(String(counter));
    observer.observe(textNode, {
        characterData: true
    });
    timerFunc = function() {
        counter = (counter + 1) % 2;
        textNode.data = String(counter);
    };
    isUsingMicroTask = true;
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
    // HTML5中规定setTimeout的最小时间延迟是4ms，也就是说理想环境下异步回调最快也是4ms才能触发。
    // Vue使用这么多函数来模拟异步任务，其目的只有一个，就是让回调异步且尽早调用。
    // 而 MessageChannel 和 setImmediate 的延迟明显是小于 setTimeout的
    timerFunc = function() {
        setImmediate(flushCallbacks);
    };
} else {
    // Fallback to setTimeout.
    timerFunc = function() {
        setTimeout(flushCallbacks, 0);
    };
}

function nextTick(cb, ctx) {
    var _resolve;
    callbacks.push(function() {
        if (cb) {
            try {
                cb.call(ctx);
            } catch (e) {
                handleError(e, ctx, 'nextTick');
            }
        } else if (_resolve) {
            _resolve(ctx);
        }
    });
    // nextTick源码中使用了一个异步锁的概念，即接收第一个回调函数时，先关上锁，执行异步方法。
    // 此时，浏览器处于等待执行完同步代码就执行异步代码的情况。
    if (!pending) {
        pending = true;
        timerFunc();
    }
    // $flow-disable-line
    if (!cb && typeof Promise !== 'undefined') {
        return new Promise(function(resolve) {
            _resolve = resolve;
        })
    }
}

/*  */


```
# 解析
在 Vue 2.4 之前都是使用的 microtasks，但是 microtasks 的优先级过高，在某些情况下可能会出
现比事件冒泡更快的情况，但如果都使用 macrotasks 又可能会出现渲染的性能问题。所以在新版本中，
会默认使用 microtasks，但在特殊情况下会使用 macrotasks，比如 v-on。

# 这里有两个问题需要注意：

#### 如何保证只在接收第一个回调函数时执行异步方法？

nextTick源码中使用了一个异步锁的概念，即接收第一个回调函数时，先关上锁，执行异步方法。此时，浏览器处于等待执行完同步代码就执行异步代码的情况。

#### 执行 flushCallbacks 函数时为什么需要备份回调函数队列？执行的也是备份的回调函数队列？

因为，会出现这么一种情况：nextTick 的回调函数中还使用 nextTick。如果 flushCallbacks 不做特殊处理，直接循环执行回调函数，会导致里面nextTick 中的回调函数会进入回调队列。