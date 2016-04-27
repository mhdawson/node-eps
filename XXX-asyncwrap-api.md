| Title  | AsyncWrap API |
|--------|---------------|
| Author | @trevnorris   |
| Status | DRAFT         |
| Date   | 2016-04-05    |

## Description

Since its initial introduction with the `AsyncListener` API, `AsyncWrap` has
slowly evolved to ensure a generalized API that would serve as a solid base for
module authors who wished to add events to the life cycle of the event loop.
The API presented below aims to accomplish this.

## API

### Overview

```js
// Standard way of requiring a module. Snake case follows core module
// convention.
const async_wrap = require('async_wrap');

// List of asynchronous providers (e.g. CRYPTO or TCPWRAP). Can be used for
// trace filtering of events.
const providers = async_wrap.Providers;

// init() is called during object construction. The object handle being
// constructed can be accessed as "this". Because init() is called during
// construction it is not safe to blindly inspect the constructed object at the
// moment. Though we are currently investigating how to make this safe.
function init(id, type, parentId) { }

// pre() is called just before the handle's callback is called. It can be
// called 0-N times for handles (e.g. TCPWRAP), and should be called exactly 1
// time for requests (e.g. FSREQWRAP).
function pre(id) { }

// post() is called just after the handle's callback has finished, and will be
// called regardless whether the handle's callback threw. If the handle's
// callback did throw then hasThrown will be true.
function post(id, hasThrown) { }

// destroy() is called when an AsyncWrap instance is destroyed. In cases like
// HTTPPARSER where the resource is reused, or timers where the handle is only
// a JS object, destroy() will be triggered manually soon after post() has
// completed. This is the only callback that does not receive the context as
// "this". Because it is called while the associated class is being destructed
// it is unsafe to operate on the object.
function destroy(id) { }

// Add a set of hooks to be called during the lifetime of asynchronous events.
// createHook() returns an instance of the AsyncHook constructor that can
// control how those hooks are used. For example, enabling or disabling their
// execution.
asyncHook = async_wrap.createHook({ init, pre, post, destroy });

// Allow callbacks of this AsyncHook instance to fire. This is not an implicit
// action after running createHook(), and must be explicitly run to being
// executing callbacks.
asyncHook.enable();

// Disable listening for new asynchronous events. Though this will not prevent
// callbacks from firing on asynchronous chains that have already run within
// the scope of an enabled AsyncHook instance.
asyncHook.disable();

// Enable hooks for the current synchronous execution scope. This will ensure
// the hooks are not in effect in case of multiple returns, or if an exception
// is thrown.
asyncHook.scope();
```

### `async_wrap`

The object returned from `require('async_wrap')`.

#### `async_wrap.createHook(hooks)`

* `hooks` {Object}
* Return: {AsyncHook}

Create a new `AsyncHook` instance that controls a given set of hooks. The four
optional hooks are `init`/`pre`/`post`/`destroy`.

#### `async_wrap.Providers`

List of all providers that may trigger the `init` callback.

### Constructor: `AsyncHook`

The `AsyncHook` constructor returns an instance that contains information about
the callbacks that are to fire during specific asynchronous events in the
lifetime of the event loop. The focal point of these calls centers around the
lifetime of `AsyncWrap`. These callbacks will also be called to emulate the
lifetime of handles and requests that do not fit this model. For example,
`HTTPPARSER` instances are recycled to improve performance. So the `destroy()`
callback would be called manually after a connection is done using it, just
before it's placed back into the unused resource pool.

All callbacks are optional. So if only resource cleanup needs to be tracked
then only the `destroy()` callback needs to be passed.

**Error Handling**: If any callback throws the application will print the
stack trace and exit. The exit path does follow that of any uncaught exception,
so any `'exit'` callbacks will fire. Unless the application is run with
`--abort-on-uncaught-exception`. In which case a stack trace will be printed
and the application will exit, leaving a core file.

The reason for this behavior is that these callbacks are running at potentially
volatile points in an object's lifetime. For example during class construction
and destruction. Because of this, it is deemed necessary to bring down the
process quickly as to prevent an unintentional abort in the future. This is
subject to change in the future if a comprehensive analysis is performed to
ensure an exception can follow the normal control flow without unintentional
side effects.

#### `asyncHook.enable()`

Enable the callbacks for a given `AsyncHook` instance. Once a callback fires on
an asynchronous event they will continue to fire for all nested asynchronous
events. Even after the instance has been disabled.

Callbacks are not implicitly enabled after an instance is created. Reason for
this is to not make any assumptions about the user's use case. Since
constructing the `asyncHook` during startup but not using it until later is a
perfectly reasonable use case. This API is meant to err on the side of
requiring explicit instructions from the user.

#### `asyncHook.disable()`

Disable the callbacks for a given `AsyncHook` instance. Doing this will prevent
the `init()`, etc., calls from firing for any new roots of asynchronous call
stacks, but will not prevent existing asynchronous call stacks that have
already been captured by the `AsyncHook` instance from continuing to fire.

While not part of the immediate development plan, it should be possible in the
future to allow selective tracking of asynchronous call stacks. The following
example demonstrates this:

```js
const async_wrap = require('async_wrap');
const net = require('net');

// Pretend init, pre, post are all defined
const asyncHook = async_wrap.createHook({ init, pre, post });
asyncHook.enable();

net.createServer((c) => {
  // Only want to follow connections that match IP range.
  if (ipRangeRegExp.test(c.address().address))
    asyncHook.disable();
}).listen(PORT);
```

At which point no further calls to the hooks on that instance will be made for
that asynchronous branch.

#### `asyncHook.scope()`

Enable capture of asynchronous events until the current synchronous code
execution has completed. This is basically a small amount of sugar to prevent
accidentally forgetting to `disable()` a set of hooks in a function with
multiple returns. Also in case an error is thrown and caught by a domain or
`uncaughtException`.

### Hook Callbacks

Key events in the lifetime of asynchronous events have been categorized into
four areas. On instantiation, before/after the callback is called and when the
instance is destructed. For cases where resources are reused, instantiation and
destructor calls are emulated.

#### `init(id, type, parentId)`

* `id` {Number}
* `type` {String}
* `parentId` {Number}

Called when a class is constructed that has the possibility to trigger an
asynchronous event. This does mean the instance _will_ trigger a
`pre()`/`post()` event. Only that the possibility exists.

The `this` of each call is that of the object being constructed. While core
attempts to make accessing properties of these handles safe at construction, it
is not guaranteed (especially if third-party modules are involved) that
everything can be safely accessed. So it is not recommended to blindly
iterate over the object's properties. Despite this, it is important to give
access of the handle to the user so they can utilize tracking methods of their
choosing. For example:

```js
const handleMap = new Map();

function HandleInst(inst) {
  // Emulate performance.now()
  const t = process.hrtime();
  this.time = t[0] * 1e3 + t[1] / 1e6;
  this.inst = inst;
}

function init(id) {
  handleMap.set(id, new HandleInst(this));
}
```

Every instance is assigned a unique id. Taking advantage of the fraction space
in the 64-bit IEEE 754-2008 that ECMAScript defines as the **Number** type, the
number of id's that can be assigned are `2^53 - 1` (also defined as
`Number.MAX_SAFE_INTEGER`). At this size node can assign a new id every 100
nanoseconds and not run out for over 28 years. Because of this circumstance it
is not deemed necessary to find an alternative approach.

The id of the parent in the async stack trace is also passed. This can be used
to create call graphs, trace resource usage/leakage, etc.

#### `pre(id)`

* `id` {Number}

Called just before the return callback is called after completing an
asynchronous request. Or called on handles with events such as receiving a new
connection. For requests, such as `fs.open()`, this should be called exactly
once. For handles, such as a TCP server, this may be called 0-N times. Useful
for recording metrics or setting up state.

#### `post(id, didThrow)`

* `id` {Number}
* `didThrow` {Boolean}

Called immediately after the return callback is completed. If the callback
threw but was caught by a domain or `uncaughtException` `didThrow` will be set
to `true`. If the callback threw but was not caught then the process will exit
immediately without calling `post()`.

#### `destroy(id)`

* `id` {Number}

Called either when the class destructor is run, or if the resource is marked as
free. The destructor will usually run when explicitly called (the case for
handles) or when a request has completed. In at least one case this will be
triggered by GC. In the case of shared or cached resources, such as
`HTTPParser`, `destroy()` will be manually called when the TCP connection no
longer needs it.

Because the callback is called during deconstruction of the class instance it
is not safe to access the JS handle during `destroy()` execution. Reason this
callback is important is to allow resource cleanup. While APIs like `WeakMap()`
are great, they are currently not iterable and not as performant. Taking an
above example, here is how `destroy` can be used for resource cleanup:

```js
const handleMap = new Map();

function HandleInst(inst) {
  // Emulate performance.now()
  const t = process.hrtime();
  this.time = t[0] * 1e3 + t[1] / 1e6;
  this.inst = inst;
}

function init(id) {
  handleMap.set(id, new HandleInst(this));
}

function destroy(id) {
  handleMap.delete(id);
}
```


## API Exceptions

### net Client connection Event

Technically the `pre()`/`post()` events of the `'connection'` event for
`net.Server` would place the server as the active id. Problem is that this is
not intuitive in how the asynchronous chain would propagate. So instead make
the client the active id for the duration of the `'connection'` callback.

### Reused Resources

Resources like `HTTPParser` are reused throughout the lifetime of the
application. This means node will have to synthesize the `init()` and
`destroy()` calls. Also the id on the class instance will need to be changed
every time the resource is acquired for use.

For shared resources like `TimerWrap` this is not necessary since there is a
unique JS handle that will contain the unique id necessary for the calls.


## Notes

### `process.nextTick()` and Promises

Execution of the `nextTickQueue` and `MicrotaskQueue` are slightly special
asynchronous cases. Because they execute in the same `MakeCallback()` as the
asynchronous callback they will ultimately end up with the same parent as the
originating call.

In the case of `nextTick()`, many calls are meant to call or queue other tasks.
Which would easily end up causing async wrap to incur more of a cost than
calling each callback in the `nextTickQueue`. Though not calling the hooks
would lead to loss of stack information.

Node currently doesn't have sufficient API to notify calls to Promise
callbacks. In order to do so node would have to override the native
implementation.
