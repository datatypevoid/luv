# LibUV in Lua

The [luv][] project provides access to the multi-platform support library
[libuv][] in Lua code. It was primarily developed for the [luvit][] project as
the built-in `uv` module, but can be used in other Lua environments.

More information about the core libuv library can be found at the original
[libuv documentation page][].

### TCP Echo Server Example

Here is a small example showing a TCP echo server:

```lua
local uv = require("luv") -- "luv" when stand-alone, "uv" in luvi apps

local server = uv.new_tcp()
server:bind("127.0.0.1", 1337)
server:listen(128, function (err)
  assert(not err, err)
  local client = uv.new_tcp()
  server:accept(client)
  client:read_start(function (err, chunk)
    assert(not err, err)
    if chunk then
      client:write(chunk)
    else
      client:shutdown()
      client:close()
    end
  end)
end)
print("TCP server listening at 127.0.0.1 port 1337")
uv.run() -- an explicit run call is necessary outside of luvit
```

### Module Layout

The luv library contains a single Lua module referred to hereafter as `uv` for
simplicity. This module consists mostly of functions with names corresponding to
their original libuv versions. For example, the libuv function `uv_tcp_bind` has
a luv version at `uv.tcp_bind`. Currently, only one non-function field exists:
`uv.constants`, which is a table.

### Functions vs Methods

In addition to having simple functions, luv provides an optional method-style
API. For example, `uv.tcp_bind(server, host, port)` can alternatively be called
as `server:bind(host, port)`. Note that the first argument `server` becomes the
object and `tcp_` is removed from the function name. Method forms are
documented below where they exist.

## Contents

This documentation is mostly a retelling of the [libuv API documentation][]
within the context of luv's Lua API, with low-level implementation details and
unexposed C functions and types omitted.

- [Error handling][]
- [Version checking][]
- [`uv_loop_t`][] — Event loop
- [`uv_req_t`][] — Base request
- [`uv_handle_t`][] — Base handle
  - [`uv_timer_t`][] — Timer handle
  - [`uv_prepare_t`][] — Prepare handle
  - [`uv_check_t`][] — Check handle
  - [`uv_idle_t`][] — Idle handle
  - [`uv_async_t`][] — Async handle
  - [`uv_poll_t`][] — Poll handle
  - [`uv_signal_t`][] — Signal handle
  - [`uv_process_t`][] — Process handle
  - [`uv_stream_t`][] — Stream handle
    - [`uv_tcp_t`][] — TCP handle
    - [`uv_pipe_t`][] — Pipe handle
    - [`uv_tty_t`][] — TTY handle
  - [`uv_udp_t`][] — UDP handle
  - [`uv_fs_event_t`][] — FS Event handle
  - [`uv_fs_poll_t`][] — FS Poll handle
- [File system operations][]
- [Thread pool work scheduling][]
- [DNS utility functions][]
- [Threading and synchronization utilities][]
- [Miscellaneous utilities][]

## Error Handling

[Error handling]: #error-handling

In libuv, errors are negative numbered constants; however, these errors and the
functions used to handle them are not exposed to luv users. Instead, if an
internal error is encountered, the luv function will return to the caller an
assertable `nil, err, name` tuple where `err` is a string with the format
`{name}: {message}` where `{name}` is the error name provided internally by
`uv_err_name` and `{message}` is a human readable error message provided
internally by `uv_strerror`. For convenience, the `name` used to construct
`err` is provided as a third argument.

This tuple is referred to below as the `fail` pseudo-type.

When a function is called successfully, it will return either a value that is
relevant to the operation of the function, or the integer `0` to indicate
success, or sometimes nothing at all. These cases are documented below.

## Version Checking

[Version checking]: #version-checking

### `uv.version()`

Returns the libuv version packed into a single integer. 8 bits are used for each
component, with the patch number stored in the 8 least significant bits. E.g.
for libuv 1.2.3 this would be 0x010203.

**Returns:** `integer`

### `uv.version_string()`

Returns the libuv version number as a string. For non-release versions the
version suffix is included.

**Returns:** `string`

## `uv_loop_t` — Event loop

[`uv_loop_t`]: #uv_loop_t--event-loop

The event loop is the central part of libuv's functionality. It takes care of
polling for I/O and scheduling callbacks to be run based on different sources of
events.

In luv, there is an implicit uv loop for every Lua state that loads the library.
You can use this library in an multi-threaded environment as long as each thread
has it's own Lua state with its corresponding own uv loop. This loop is not
directly exposed to users in the Lua module.

### `uv.loop_close()`

Closes all internal loop resources. In normal execution, the loop will
automatically be closed when it is garbage collected by Lua, so it is not
necessary to explicitly call `loop_close()`. Call this function only after the
loop has finished executing and all open handles and requests have been closed,
or it will return `EBUSY`.

**Returns:** `0` or `fail`

### `uv.run([mode])`

> optional `mode` defaults to `"default"`

This function runs the event loop. It will act differently depending on the
specified mode:

  - `"default"`: Runs the event loop until there are no more active and
  referenced handles or requests. Returns `true` if `uv.stop()` was called and
  there are still active handles or requests. Returns `false` in all other
  cases.

  - `"once"`: Poll for I/O once. Note that this function blocks if there are no
  pending callbacks. Returns `false` when done (no active handles or requests
  left), or `true` if more callbacks are expected (meaning you should run the
  event loop again sometime in the future).

  - `"nowait"`: Poll for I/O once but don't block if there are no pending
  callbacks. Returns `false` if done (no active handles or requests left),
  or `true` if more callbacks are expected (meaning you should run the event
  loop again sometime in the future).

**Returns:** `boolean` or `fail`

**Note:** Luvit will implicitly call `uv.run()` after loading user code, but if
you use the luv bindings directly, you need to call this after registering
your initial set of event callbacks to start the event loop.

### `uv.loop_alive()`

Returns `true` if there are referenced active handles, active requests, or
closing handles in the loop; otherwise, `false`.

**Returns:** `boolean` or `fail`

### `uv.stop()`

Stop the event loop, causing `uv.run()` to end as soon as possible. This
will happen not sooner than the next loop iteration. If this function was called
before blocking for I/O, the loop won't block for I/O on this iteration.

**Returns:** Nothing.

### `uv.backend_fd()`

Get backend file descriptor. Only kqueue, epoll, and event ports are supported.

This can be used in conjunction with `uv.run("nowait")` to poll in one thread
and run the event loop's callbacks in another

**Returns:** `integer` or `nil`

**Note**: Embedding a kqueue fd in another kqueue pollset doesn't work on all
platforms. It's not an error to add the fd but it never generates events.

### `uv.backend_timeout()`

Get the poll timeout. The return value is in milliseconds, or -1 for no timeout.

**Returns:** `integer`

### `uv.now()`

Returns the current timestamp in milliseconds. The timestamp is cached at the
start of the event loop tick, see `uv.update_time()` for details and rationale.

The timestamp increases monotonically from some arbitrary point in time. Don't
make assumptions about the starting point, you will only get disappointed.

**Returns:** `integer`

**Note**: Use `uv.hrtime()` if you need sub-millisecond granularity.

### `uv.update_time()`

Update the event loop's concept of "now". Libuv caches the current time at the
start of the event loop tick in order to reduce the number of time-related
system calls.

You won't normally need to call this function unless you have callbacks that
block the event loop for longer periods of time, where "longer" is somewhat
subjective but probably on the order of a millisecond or more.

**Returns:** Nothing.

### `uv.walk(callback)`

Walk the list of handles: `callback` will be executed with each handle.

**Returns:** Nothing.

```lua
-- Example usage of uv.walk to close all handles that aren't already closing.
uv.walk(function (handle)
  if not handle:is_closing() then
    handle:close()
  end
end)
```

## `uv_req_t` — Base request

[`uv_req_t`]: #uv_req_t--request-handle

`uv_req_t` is the base type for all libuv request types.

### `uv.cancel(req)`

Cancel a pending request. Fails if the request is executing or has finished
executing. Only cancellation of `uv_fs_t`, `uv_getaddrinfo_t`,
`uv_getnameinfo_t` and `uv_work_t` requests is currently supported.

## `uv_handle_t` — Base handle

[`uv_handle_t`]: #uv_handle_t--base-handle

`uv_handle_t` is the base type for all libuv handle types. All API functions
defined here work with any handle type.

### `uv.is_active(handle)`

> method form `handle:is_active()`

Returns `true` if the handle is active, `false` if it's inactive. What "active”
means depends on the type of handle:

  - A [`uv_async_t`][] handle is always active and cannot be deactivated, except
  by closing it with `uv.close()`.

  - A [`uv_pipe_t`][], [`uv_tcp_t`][], [`uv_udp_t`][], etc. handle - basically
  any handle that deals with I/O - is active when it is doing something that
  involves I/O, like reading, writing, connecting, accepting new connections,
  etc.

  - A [`uv_check_t`][], [`uv_idle_t`][], [`uv_timer_t`][], etc. handle is active
  when it has been started with a call to `uv.check_start()`, `uv.idle_start()`,
  `uv.timer_start()` etc. until it has been stopped with a call to its
  respective stop function.

**Returns:** `boolean` or `fail`

### `uv.is_closing(handle)`

> method form `handle:is_closing()`

Returns `true` if the handle is closing or closed, `false` otherwise.

**Returns:** `boolean` or `fail`

**Note**: This function should only be used between the initialization of the
handle and the arrival of the close callback.

### `uv.close(handle, [callback])`

> method form `handle:close([callback])`

Request handle to be closed. `callback` will be called asynchronously after this
call. This MUST be called on each handle before memory is released.

Handles that wrap file descriptors are closed immediately but `callback` will
still be deferred to the next iteration of the event loop. It gives you a chance
to free up any resources associated with the handle.

In-progress requests, like `uv_connect_t` or `uv_write_t`, are cancelled and
have their callbacks called asynchronously with `ECANCELED`.

**Returns:** Nothing.

### `uv.ref(handle)`

> method form `handle:ref()`

Reference the given handle. References are idempotent, that is, if a handle is
already referenced calling this function again will have no effect.

**Returns:** Nothing.

See [Reference counting][].

### `uv.unref(handle)`

> method form `handle:unref()`

Un-reference the given handle. References are idempotent, that is, if a handle
is not referenced calling this function again will have no effect.

**Returns:** Nothing.

See [Reference counting][].

### `uv.has_ref(handle)`

> method form `handle:has_ref()`

Returns `true` if the handle referenced, `false` if not.

**Returns:** `boolean` or `fail`

See [Reference counting][].

### `uv.send_buffer_size(handle, [size])`

> method form `handle:send_buffer_size([size])`

Gets or sets the size of the send buffer that the operating system uses for the
socket.

If `size` is omitted, it will return the current send buffer size, otherwise it
will use `size` to set the new send buffer size.

This function works for TCP, pipe and UDP handles on Unix and for TCP and UDP
handles on Windows.

**Returns:**
- `integer` or `fail` (if `size` is `nil` or `0`)
- `0` or `fail` (if `size` is not `nil` and not `0`)

**Note**: Linux will set double the size and return double the size of the
original set value.

### `uv.recv_buffer_size(handle, [size])`

> method form `handle:recv_buffer_size([size])`

Gets or sets the size of the receive buffer that the operating system uses for
the socket.

If `size` is omitted, it will return the current receive buffer size, otherwise
it will use `size` to set the new receive buffer size.

This function works for TCP, pipe and UDP handles on Unix and for TCP and UDP
handles on Windows.

**Returns:**
- `integer` or `fail` (if `size` is `nil` or `0`)
- `0` or `fail` (if `size` is not `nil` and not `0`)

**Note**: Linux will set double the size and return double the size of the
original set value.

### `uv.fileno(handle)`

> method form `handle:fileno()`

Gets the platform dependent file descriptor equivalent.

The following handles are supported: TCP, pipes, TTY, UDP and poll. Passing any
other handle type will fail with `EINVAL`.

If a handle doesn't have an attached file descriptor yet or the handle itself
has been closed, this function will return `EBADF`.

**Returns:** `integer` or `fail`

**Warning**: Be very careful when using this function. libuv assumes it's in
control of the file descriptor so any change to it may lead to malfunction.

## Reference counting

[reference counting]: #reference-counting

The libuv event loop (if run in the default mode) will run until there are no
active and referenced handles left. The user can force the loop to exit early by
unreferencing handles which are active, for example by calling `uv.unref()`
after calling `uv.timer_start()`.

A handle can be referenced or unreferenced, the refcounting scheme doesn't use a
counter, so both operations are idempotent.

All handles are referenced when active by default, see `uv.is_active()` for a
more detailed explanation on what being active involves.

## `uv_timer_t` — Timer handle

[`uv_timer_t`]: #uv_timer_t--timer-handle

> [`uv_handle_t`][] functions also apply.

Timer handles are used to schedule callbacks to be called in the future.

### `uv.new_timer()`

Creates and initializes a new `uv_timer_t`. Returns the Lua userdata wrapping
it.

**Returns:** `uv_timer_t userdata` or `fail`

```lua
-- Creating a simple setTimeout wrapper
local function setTimeout(timeout, callback)
  local timer = uv.new_timer()
  timer:start(timeout, 0, function ()
    timer:stop()
    timer:close()
    callback()
  end)
  return timer
end

-- Creating a simple setInterval wrapper
local function setInterval(interval, callback)
  local timer = uv.new_timer()
  timer:start(interval, interval, function ()
    timer:stop()
    timer:close()
    callback()
  end)
  return timer
end

-- And clearInterval
local function clearInterval(timer)
  timer:stop()
  timer:close()
end
```

### `uv.timer_start(timer, timeout, repeat, callback)`

> method form `timer:start(timeout, repeat, callback)`

Start the timer. `timeout` and `repeat` are in milliseconds.

If `timeout` is zero, the callback fires on the next event loop iteration. If
`repeat` is non-zero, the callback fires first after `timeout` milliseconds and
then repeatedly after `repeat` milliseconds.

**Returns:** `0` or `fail`

### `uv.timer_stop(timer)`

> method form `timer:stop()`

Stop the timer, the callback will not be called anymore.

**Returns:** `0` or `fail`

### `uv.timer_again(timer)`

> method form `timer:again()`

Stop the timer, and if it is repeating restart it using the repeat value as the
timeout. If the timer has never been started before it raises `EINVAL`.

**Returns:** `0` or `fail`

### `uv.timer_set_repeat(timer, repeat)`

> method form `timer:set_repeat(repeat)`

Set the repeat interval value in milliseconds. The timer will be scheduled to
run on the given interval, regardless of the callback execution duration, and
will follow normal timer semantics in the case of a time-slice overrun.

For example, if a 50 ms repeating timer first runs for 17 ms, it will be
scheduled to run again 33 ms later. If other tasks consume more than the 33 ms
following the first timer callback, then the callback will run as soon as
possible.

**Returns:** Nothing.

### `uv.timer_get_repeat(timer)`

> method form `timer:get_repeat()`

Get the timer repeat value.

**Returns:** `integer`

## `uv_prepare_t` — Prepare handle

[`uv_prepare_t`]: #uv_prepare_t--prepare-handle

> [`uv_handle_t`][] functions also apply.

Prepare handles will run the given callback once per loop iteration, right
before polling for I/O.

```lua
local prepare = uv.new_prepare()
prepare:start(function()
  print("Before I/O polling")
end)
```

### `uv.new_prepare()`

Creates and initializes a new `uv_prepare_t`. Returns the Lua userdata wrapping
it.

**Returns:** `uv_prepare_t userdata` or `fail`

### `uv.prepare_start(prepare, callback)`

> method form `prepare:start(callback)`

Start the handle with the given callback.

**Returns:** `0` or `fail`

### `uv.prepare_stop(prepare)`

> method form `prepare:stop()`

Stop the handle, the callback will no longer be called.

**Returns:** `0` or `fail`

## `uv_check_t` — Check handle

[`uv_check_t`]: #uv_check_t--check-handle

> [`uv_handle_t`][] functions also apply.

Check handles will run the given callback once per loop iteration, right after
polling for I/O.

```lua
local check = uv.new_check()
check:start(function()
  print("After I/O polling")
end)
```

### `uv.new_check()`

Creates and initializes a new `uv_check_t`. Returns the Lua userdata wrapping
it.

**Returns:** `uv_check_t userdata` or `fail`

### `uv.check_start(check, callback)`

> method form `check:start(callback)`

Start the handle with the given callback.

**Returns:** `0` or `fail`

### `uv.check_stop(check)`

> method form `check:stop()`

Stop the handle, the callback will no longer be called.

**Returns:** `0` or `fail`

## `uv_idle_t` — Idle handle

[`uv_idle_t`]: #uv_idle_t--idle-handle

> [`uv_handle_t`][] functions also apply.

Idle handles will run the given callback once per loop iteration, right before
the [`uv_prepare_t`][] handles.

**Note**: The notable difference with prepare handles is that when there are
active idle handles, the loop will perform a zero timeout poll instead of
blocking for I/O.

**Warning**: Despite the name, idle handles will get their callbacks called on
every loop iteration, not when the loop is actually "idle".

```lua
local idle = uv.new_idle()
idle:start(function()
  print("Before I/O polling, no blocking")
end)
```

### `uv.new_idle()`

Creates and initializes a new `uv_idle_t`. Returns the Lua userdata wrapping
it.

**Returns:** `uv_idle_t userdata` or `fail`

### `uv.idle_start(idle, callback)`

> method form `idle:start(callback)`

Start the handle with the given callback.

**Returns:** `0` or `fail`

### `uv.idle_stop(check)`

> method form `idle:stop()`

Stop the handle, the callback will no longer be called.

**Returns:** `0` or `fail`

## `uv_async_t` — Async handle

[`uv_async_t`]: #uv_async_t--async-handle

> [`uv_handle_t`][] functions also apply.

Async handles allow the user to "wakeup" the event loop and get a callback
called from another thread.

```lua
local async
async = uv.new_async(function()
  print("async operation ran")
  async:close()
end)

async:send()
```

### `uv.new_async([callback])`

Creates and initializes a new `uv_async_t`. Returns the Lua userdata wrapping
it. A `nil` callback is allowed.

**Returns:** `uv_async_t userdata` or `fail`

**Note**: Unlike other handle initialization functions, this immediately starts
the handle.

### `uv.async_send(async)`

> method form `async:send()`

Wakeup the event loop and call the async handle's callback.

**Returns:** `0` or `fail`

**Note**: It's safe to call this function from any thread. The callback will be
called on the loop thread.

**Warning**: libuv will coalesce calls to `uv.async_send(async)`, that is, not
every call to it will yield an execution of the callback. For example: if
`uv.async_send()` is called 5 times in a row before the callback is called, the
callback will only be called once. If `uv.async_send()` is called again after
the callback was called, it will be called again.

## `uv_poll_t` — Poll handle

[`uv_poll_t`]: #uv_poll_t--poll-handle

> [`uv_handle_t`][] functions also apply.

Poll handles are used to watch file descriptors for readability and writability,
similar to the purpose of [poll(2)](http://linux.die.net/man/2/poll).

The purpose of poll handles is to enable integrating external libraries that
rely on the event loop to signal it about the socket status changes, like c-ares
or libssh2. Using `uv_poll_t` for any other purpose is not recommended;
`uv_tcp_t`, `uv_udp_t`, etc. provide an implementation that is faster and more
scalable than what can be achieved with `uv_poll_t`, especially on Windows.

It is possible that poll handles occasionally signal that a file descriptor is
readable or writable even when it isn't. The user should therefore always be
prepared to handle EAGAIN or equivalent when it attempts to read from or write
to the fd.

It is not okay to have multiple active poll handles for the same socket, this
can cause libuv to busyloop or otherwise malfunction.

The user should not close a file descriptor while it is being polled by an
active poll handle. This can cause the handle to report an error, but it might
also start polling another socket. However the fd can be safely closed
immediately after a call to `uv.poll_stop()` or `uv.close()`.

**Note** On windows only sockets can be polled with poll handles. On Unix any
file descriptor that would be accepted by poll(2) can be used.

### `uv.new_poll(fd)`

Initialize the handle using a file descriptor.

The file descriptor is set to non-blocking mode.

**Returns:** `uv_poll_t userdata` or `fail`

### `uv.new_socket_poll(fd)`

Initialize the handle using a socket descriptor. On Unix this is identical to
`uv.new_poll()`. On windows it takes a SOCKET handle.

The socket is set to non-blocking mode.

**Returns:** `uv_poll_t userdata` or `fail`

### `uv.poll_start(poll, events, callback)`

> method form `poll:start(events, callback)`

> optional `events` defaults to `"rw"`

Starts polling the file descriptor. `events` are: `"r"`, `"w"`, `"rw"`, `"d"`,
`"rd"`, `"wd"`, `"rwd"`, `"p"`, `"rp"`, `"wp"`, `"rwp"`, `"dp"`, `"rdp"`,
`"wdp"`, or `"rwdp"` where `r` is `READABLE`, `w` is `WRITABLE`, `d` is
`DISCONNECT`, and `p` is `PRIORITIZED`. As soon as an event is detected
the callback will be called with status set to 0, and the detected events set on
the events field.

The user should not close the socket while the handle is active. If the user
does that anyway, the callback may be called reporting an error status, but this
is not guaranteed.

**Returns:** `0` or `fail`

**Note** Calling `uv.poll_start()` on a handle that is already active is fine.
Doing so will update the events mask that is being watched for.

### `uv.poll_stop(poll)`

> method form `poll:stop()`

Stop polling the file descriptor, the callback will no longer be called.

**Returns:** `0` or `fail`

## `uv_signal_t` — Signal handle

[`uv_signal_t`]: #uv_signal_t--signal-handle

> [`uv_handle_t`][] functions also apply.

Signal handles implement Unix style signal handling on a per-event loop bases.

**Windows Notes:**

Reception of some signals is emulated on Windows:
  - SIGINT is normally delivered when the user presses CTRL+C. However, like on
  Unix, it is not generated when terminal raw mode is enabled.
  - SIGBREAK is delivered when the user pressed CTRL + BREAK.
  - SIGHUP is generated when the user closes the console window. On SIGHUP the
  program is given approximately 10 seconds to perform cleanup. After that
  Windows will unconditionally terminate it.
  - SIGWINCH is raised whenever libuv detects that the console has been resized.
  SIGWINCH is emulated by libuv when the program uses a uv_tty_t handle to write
  to the console. SIGWINCH may not always be delivered in a timely manner; libuv
  will only detect size changes when the cursor is being moved. When a readable
  [`uv_tty_t`][] handle is used in raw mode, resizing the console buffer will
  also trigger a SIGWINCH signal.
  - Watchers for other signals can be successfully created, but these signals
  are never received. These signals are: SIGILL, SIGABRT, SIGFPE, SIGSEGV,
  SIGTERM and SIGKILL.
  - Calls to raise() or abort() to programmatically raise a signal are not
  detected by libuv; these will not trigger a signal watcher.

**Unix Notes:**

  - SIGKILL and SIGSTOP are impossible to catch.
  - Handling SIGBUS, SIGFPE, SIGILL or SIGSEGV via libuv results into undefined
  behavior.
  - SIGABRT will not be caught by libuv if generated by abort(), e.g. through
  assert().
  - On Linux SIGRT0 and SIGRT1 (signals 32 and 33) are used by the NPTL pthreads
  library to manage threads. Installing watchers for those signals will lead to
  unpredictable behavior and is strongly discouraged. Future versions of libuv
  may simply reject them.

```lua
-- Create a new signal handler
local signal = uv.new_signal()
-- Define a handler function
uv.signal_start(signal, "sigint", function(signal)
  print("got " .. signal .. ", shutting down")
  os.exit(1)
end)
```

### `uv.new_signal()`

Creates and initializes a new `uv_signal_t`. Returns the Lua userdata wrapping
it.

**Returns:** `uv_signal_t userdata` or `fail`

### `uv.signal_start(signal, signum, callback)`

> method form `signal:start(signum, callback)`

Start the handle with the given callback, watching for the given signal.

**Returns:** `0` or `fail`

### `uv.signal_start_oneshot()`

> method form `signal:start_oneshot(signum, callback)`

Same functionality as `uv.signal_start()` but the signal handler is reset the moment the signal is received.

**Returns:** `0` or `fail`

### `uv.signal_stop(signal)`

> method form `signal:stop()`

Stop the handle, the callback will no longer be called.

**Returns:** `0` or `fail`

## `uv_process_t` — Process handle

[`uv_process_t`]: #uv_process_t--process-handle

> [`uv_handle_t`][] functions also apply.

Process handles will spawn a new process and allow the user to control it and
establish communication channels with it using streams.

### `uv.disable_stdio_inheritance()`

Disables inheritance for file descriptors / handles that this process inherited
from its parent. The effect is that child processes spawned by this process
don't accidentally inherit these handles.

It is recommended to call this function as early in your program as possible,
before the inherited file descriptors can be closed or duplicated.

**Returns:** Nothing.

**Note:** This function works on a best-effort basis: there is no guarantee that
libuv can discover all file descriptors that were inherited. In general it does
a better job on Windows than it does on Unix.

### `uv.spawn(file, options, onexit)`

Initializes the process handle and starts the process. If the process is
successfully spawned, this function will return the handle and pid of the child
process.

Possible reasons for failing to spawn would include (but not be limited to) the
file to execute not existing, not having permissions to use the setuid or setgid
specified, or not having enough memory to allocate for the new process.

```lua
local stdout = uv.new_pipe(false)
local stderr = uv.new_pipe(false)
local stdin = uv.new_pipe(false)

local handle, pid

local function onexit(code, signal)
  p("exit", {code=code,signal=signal})
end

local function onclose()
  p("close")
end

local function onread(err, chunk)
  assert(not err, err)
  if (chunk) then
    p("data", {data=chunk})
  else
    p("end")
  end
end

local function onshutdown()
  uv.close(handle, onclose)
end

handle, pid = uv.spawn("cat", {
  stdio = {stdin, stdout, stderr}
}, onexit)

p{
  handle=handle,
  pid=pid
}

uv.read_start(stdout, onread)
uv.read_start(stderr, onread)
uv.write(stdin, "Hello World")
uv.shutdown(stdin, onshutdown)
```

The `options` table accepts the following fields:

  - `options.args` - Command line arguments as a list of string. The first
  string should be the path to the program. On Windows this uses CreateProcess
  which concatenates the arguments into a string this can cause some strange
  errors. (See `options.verbatim` below for Windows.)
  - `options.stdio` - Set the file descriptors that will be made available to
  the child process. The convention is that the first entries are stdin, stdout,
  and stderr. (**Note** On Windows file descriptors after the third are
  available to the child process only if the child processes uses the MSVCRT
  runtime.)
  - `options.env` - Set environment variables for the new process.
  - `options.cwd` - Set current working directory for the subprocess.
  - `options.uid` - Set the child process' user id.
  - `options.gid` - Set the child process' group id.
  - `options.verbatim` - If true, do not wrap any arguments in quotes, or
  perform any other escaping, when converting the argument list into a command
  line string. This option is only meaningful on Windows systems. On Unix it is
  silently ignored.
  - `options.detached` - If true, spawn the child process in a detached state -
  this will make it a process group leader, and will effectively enable the
  child to keep running after the parent exits. Note that the child process
  will still keep the parent's event loop alive unless the parent process calls
  `uv.unref()` on the child's process handle.
  - `options.hide` - If true, hide the subprocess console window that would
  normally be created. This option is only meaningful on Windows systems. On
  Unix it is silently ignored.

The `options.stdio` entries can take many shapes.

  - If they are numbers, then the child process inherits that same zero-indexed
  fd from the parent process.
  - If `uv_stream_t` handles are passed in, those are used as a read-write pipe
  or inherited stream depending if the stream has a valid fd.
  - Including `nil` placeholders means to ignore that fd in the child.

When the child process exits, the `onexit` callback will be called with exit
code and signal.

**Returns:** `integer`, `integer`

### `uv.process_kill(process, sigmun)`

> method form `process:kill(sigmun)`

Sends the specified signal to the given process handle. Check the documentation
on `uv_signal_t` for signal support, specially on Windows.

**Returns:** `0` or `fail`

### `uv.kill(pid, sigmun)`

Sends the specified signal to the given PID. Check the documentation on
`uv_signal_t` for signal support, specially on Windows.

**Returns:** `0` or `fail`

## `uv_stream_t` — Stream handle

[`uv_stream_t`]: #uv_stream_t--stream-handle

> [`uv_handle_t`][] functions also apply.

Stream handles provide an abstraction of a duplex communication channel.
[`uv_stream_t`][] is an abstract type, libuv provides 3 stream implementations
in the form of [`uv_tcp_t`][], [`uv_pipe_t`][] and [`uv_tty_t`][].

### `uv.shutdown(stream, [callback])`

> method form `stream:shutdown([callback])`

Shutdown the outgoing (write) side of a duplex stream. It waits for pending
write requests to complete. The callback is called after shutdown is complete.

**Returns:** `uv_shutdown_t userdata` or `fail`

### `uv.listen(stream, backlog, callback)`

> method form `stream:listen(backlog, callback)`

Start listening for incoming connections. `backlog` indicates the number of
connections the kernel might queue, same as `listen(2)`. When a new incoming
connection is received the callback is called.

**Returns:** `0` or `fail`

### `uv.accept(stream, client_stream)`

> method form `stream:accept(client_stream)`

This call is used in conjunction with `uv.listen()` to accept incoming
connections. Call this function after receiving a callback to accept the
connection.

When the connection callback is called it is guaranteed that this function
will complete successfully the first time. If you attempt to use it more than
once, it may fail. It is suggested to only call this function once per
connection call.

**Returns:** `0` or `fail`

```lua
server:listen(128, function (err)
  local client = uv.new_tcp()
  server:accept(client)
end)
```

### `uv.read_start(stream, callback)`

> method form `stream:read_start(callback)`

Callback is of the form `(err, data)`.

Read data from an incoming stream. The callback will be made several times until
there is no more data to read or `uv.read_stop()` is called. When we've reached
EOF, `data` will be `nil`.

**Returns:** `0` or `fail`

```lua
stream:read_start(function (err, chunk)
  if err then
    -- handle read error
  elseif chunk then
    -- handle data
  else
    -- handle disconnect
  end
end)
```

### `uv.read_stop(stream)`

> method form `stream:read_stop()`

Stop reading data from the stream. The read callback will no longer be called.

This function is idempotent and may be safely called on a stopped stream.

**Returns:** `0` or `fail`

### `uv.write(stream, data, [callback])`

> method form `stream:write(data, [callback])`

Write data to stream.

`data` can either be a Lua string or a table of strings. If a table is passed
in, the C backend will use writev to send all strings in a single system call.

The optional `callback` is for knowing when the write is complete.

**Returns:** `uv_write_t userdata` or `fail`

### `uv.write2(stream, data, send_handle, [callback])`

> method form `stream:write2(data, send_handle, [callback])`

Extended write function for sending handles over a pipe. The pipe must be
initialized with `ipc` option `true`.

**Returns:** `uv_write_t userdata` or `fail`

**Note:** `send_handle` must be a TCP socket or pipe, which is a server or a
connection (listening or connected state). Bound sockets or pipes will be
assumed to be servers.

### `uv.try_write(stream, data)`

> method form `stream:try_write(data)`

Same as `uv.write()`, but won't queue a write request if it can't be completed
immediately.

Will return number of bytes written (can be less than the supplied buffer size).

**Returns:** `integer` or `fail`

### `uv.is_readable(stream)`

> method form `stream:is_readable()`

Returns `true` if the stream is readable, `false` otherwise.

**Returns:** `boolean`

### `uv.is_writable(stream)`

> method form `stream:is_writable()`

Returns `true` if the stream is writable, `false` otherwise.

**Returns:** `boolean`

### `uv.stream_set_blocking(stream, blocking)`

> method form `stream:set_blocking(blocking)`

Enable or disable blocking mode for a stream.

When blocking mode is enabled all writes complete synchronously. The interface
remains unchanged otherwise, e.g. completion or failure of the operation will
still be reported through a callback which is made asynchronously.

**Returns:** `0` or `fail`

**Warning**: Relying too much on this API is not recommended. It is likely to
change significantly in the future. Currently this only works on Windows and
only for `uv_pipe_t` handles. Also libuv currently makes no ordering guarantee
when the blocking mode is changed after write requests have already been
submitted. Therefore it is recommended to set the blocking mode immediately
after opening or creating the stream.

### `uv.stream_get_write_queue_size()`

> method form `stream:get_write_queue_size()`

Returns the stream's write queue size.

**Returns:** `integer`

## `uv_tcp_t` — TCP handle

[`uv_tcp_t`]: #uv_tcp_t--tcp-handle

> [`uv_handle_t`][] and [`uv_stream_t`][] functions also apply.

TCP handles are used to represent both TCP streams and servers.

### `uv.new_tcp([flags])`

Creates and initializes a new `uv_tcp_t`. Returns the Lua userdata wrapping it.
Flags may be a family string: `"unix"`, `"inet"`, `"inet6"`, `"ipx"`,
`"netlink"`, `"x25"`, `"ax25"`, `"atmpvc"`, `"appletalk"`, or `"packet"`.

**Returns:** `uv_tcp_t userdata` or `fail`

### `uv.tcp_open(tcp, sock)`

> method form `tcp:open(sock)`

Open an existing file descriptor or SOCKET as a TCP handle.

**Returns:** `0` or `fail`

**Note:** The passed file descriptor or SOCKET is not checked for its type, but it's required that it represents a valid stream socket.

### `uv.tcp_nodelay(tcp, enable)`

> method form `tcp:nodelay(enable)`

Enable / disable Nagle's algorithm.

**Returns:** `0` or `fail`

### `uv.tcp_keepalive(tcp, enable, [delay])`

> method form `tcp:keepalive(enable, [delay])`

Enable / disable TCP keep-alive. `delay` is the initial delay in seconds,
ignored when enable is `false`.

**Returns:** `0` or `fail`

### `uv.tcp_simultaneous_accepts(tcp, enable)`

> method form `tcp:simultaneous_accepts(enable)`

Enable / disable simultaneous asynchronous accept requests that are queued by
the operating system when listening for new TCP connections.

This setting is used to tune a TCP server for the desired performance. Having
simultaneous accepts can significantly improve the rate of accepting connections
(which is why it is enabled by default) but may lead to uneven load distribution
in multi-process setups.

**Returns:** `0` or `fail`

### `uv.tcp_bind(tcp, host, port, [flags])`

> method form `tcp:bind(host, port, [flags])`

Bind the handle to an host and port. `host` should be an IP address and
not a domain name. Any `flags` are set with a table with field `ipv6only`
equal to `true` or `false`.

When the port is already taken, you can expect to see an `EADDRINUSE` error
from either `uv.tcp_bind()`, `uv.listen()` or `uv.tcp_connect()`. That is, a
successful call to this function does not guarantee that the call to `uv.listen()`
or `uv.tcp_connect()` will succeed as well.

Use a port of `0` to let the OS assign an ephemeral port.  You can look it up
later using `uv.tcp_getsockname()`.

**Returns:** `0` or `fail`

### `uv.tcp_getpeername(tcp)`

> method form `tcp:getpeername()`

Get the current address to which the handle is bound. Returns an address table
with fields `ip`, `family`, and `port`.

**Returns:** `table` or `fail`
- `ip` : `string`
- `family` : `string`
- `port` : `integer`

### `uv.tcp_getsockname(tcp)`

> method form `tcp:getsockname()`

Get the address of the peer connected to the handle. Returns an address table
with fields `ip`, `family`, and `port`.

**Returns:** `table` or `fail`
- `ip` : `string`
- `family` : `string`
- `port` : `integer`

### `uv.tcp_connect(tcp, host, port, callback)`

> method form `tcp:connect(host, port, callback)`

Establish an IPv4 or IPv6 TCP connection.

The callback is made when the connection has been established or when a
connection error happened.

**Returns:** `uv_connect_t userdata` or `fail`

```lua
local client = uv.new_tcp()
client:connect("127.0.0.1", 8080, function (err)
  -- check error and carry on.
end)
```

### `uv.tcp_write_queue_size(tcp)`

> method form `tcp:write_queue_size()`

**Deprecated:** Please use `uv.stream_get_write_queue_size()` instead.

### `uv.tcp_close_reset()`

> method form `tcp:close_reset()`

Resets a TCP connection by sending a RST packet. This is accomplished by setting
the SO_LINGER socket option with a linger interval of zero and then calling
`uv.close()`. Due to some platform inconsistencies, mixing of `uv.shutdown()`
and `uv.tcp_close_reset()` calls is not allowed.

**Returns:** `0` or `fail`

## `uv_pipe_t` — Pipe handle

[`uv_pipe_t`]: #uv_pipe_t--pipe-handle

> [`uv_handle_t`][] and [`uv_stream_t`][] functions also apply.

Pipe handles provide an abstraction over local domain sockets on Unix and named pipes on Windows.

```lua
local pipe = uv.new_pipe(false)

pipe:bind('/tmp/sock.test')

pipe:listen(128, function()
  local client = uv.new_pipe(false)
  pipe:accept(client)
  client:write("hello!\n")
  client:close()
end)
```

### `uv.new_pipe(ipc)`

Creates and initializes a new `uv_pipe_t`. Returns the Lua userdata wrapping
it. The `ipc` argument is a boolean to indicate if this pipe will be used for
handle passing between processes.

**Returns:** `uv_pipe_t userdata` or `fail`

### `uv.pipe_open(file)`

Open an existing file descriptor or [`uv_handle_t`][] as a pipe.

**Returns:** `0` or `fail`

**Note**: The file descriptor is set to non-blocking mode.

### `uv.pipe_bind(pipe, name)`

> method form `pipe:bind(name)`

Bind the pipe to a file path (Unix) or a name (Windows).

**Returns:** `0` or `fail`

**Note**: Paths on Unix get truncated to sizeof(sockaddr_un.sun_path) bytes,
typically between 92 and 108 bytes.

### `uv.pipe_connect(pipe, name, [callback])`

> method form `pipe:connect(name, [callback])`

Connect to the Unix domain socket or the named pipe.

**Returns:** `uv_connect_t userdata` or `fail`

**Note**: Paths on Unix get truncated to sizeof(sockaddr_un.sun_path) bytes,
typically between 92 and 108 bytes.

### `uv.pipe_getsockname(pipe)`

> method form `pipe:getsockname()`

Get the name of the Unix domain socket or the named pipe.

**Returns:** `string` or `fail`

### `uv.pipe_getpeername(pipe)`

> method form `pipe:getpeername()`

Get the name of the Unix domain socket or the named pipe to which the handle is
connected.

**Returns:** `string` or `fail`

### `uv.pipe_pending_instances(pipe, count)`

> method form `pipe:pending_instances(count)`

Set the number of pending pipe instance handles when the pipe server is waiting
for connections.

**Returns:** Nothing.

**Note**: This setting applies to Windows only.

### `uv.pipe_pending_count(pipe)`

> method form `pipe:pending_count()`

Returns the pending pipe count for the named pipe.

**Returns:** `integer`

### `uv.pipe_pending_type(pipe)`

> method form `pipe:pending_type()`

Used to receive handles over IPC pipes.

First - call `uv.pipe_pending_count()`, if it's > 0 then initialize a handle of
the given type, returned by `uv.pipe_pending_type()` and call
`uv.accept(pipe, handle)`.

**Returns:** `string`

## `uv_tty_t` — TTY handle

[`uv_tty_t`]: #uv_tty_t--tty-handle

> [`uv_handle_t`][] and [`uv_stream_t`][] functions also apply.

TTY handles represent a stream for the console.

```lua
-- Simple echo program
local stdin = uv.new_tty(0, true)
local stdout = uv.new_tty(1, false)

stdin:read_start(function (err, data)
  assert(not err, err)
  if data then
    stdout:write(data)
  else
    stdin:close()
    stdout:close()
  end
end)
```

### `uv.new_tty(fd, readable)`

Initialize a new TTY stream with the given file descriptor. Usually the file
descriptor will be:

 - 0 - stdin
 - 1 - stdout
 - 2 - stderr

On Unix this function will determine the path of the fd of the terminal using
ttyname_r(3), open it, and use it if the passed file descriptor refers to a TTY.
This lets libuv put the tty in non-blocking mode without affecting other
processes that share the tty.

This function is not thread safe on systems that don’t support ioctl TIOCGPTN or TIOCPTYGNAME, for instance OpenBSD and Solaris.

**Returns:** `uv_tty_t userdata` or `fail`

**Note:** If reopening the TTY fails, libuv falls back to blocking writes.

### `uv.tty_set_mode(tty, mode)`

> method form `tty:set_mode(mode)`

Set the TTY using the specified terminal mode.

Parameter `mode` is a C enum with the following values:

  - 0 - UV_TTY_MODE_NORMAL: Initial/normal terminal mode
  - 1 - UV_TTY_MODE_RAW: Raw input mode (On Windows, ENABLE_WINDOW_INPUT is
  also enabled)
  - 2 - UV_TTY_MODE_IO: Binary-safe I/O mode for IPC (Unix-only)

**Returns:** `0` or `fail`

### `uv.tty_reset_mode()`

To be called when the program exits. Resets TTY settings to default values for
the next process to take over.

This function is async signal-safe on Unix platforms but can fail with error
code `EBUSY` if you call it when execution is inside `uv.tty_set_mode()`.

**Returns:** `0` or `fail`

### `uv.tty_get_winsize(tty)`

> method form `tty:get_winsize()`

Gets the current Window width and height.

**Returns:** `integer, integer` or `fail`

## `uv_udp_t` — UDP handle

[`uv_udp_t`]: #uv_udp_t--udp-handle

> [`uv_handle_t`][] functions also apply.

UDP handles encapsulate UDP communication for both clients and servers.

### `uv.new_udp([flags])`

Creates and initializes a new `uv_udp_t`. Returns the Lua userdata wrapping
it. The actual socket is created lazily. Flags may be a family string:
`"unix"`, `"inet"`, `"inet6"`, `"ipx"`, `"netlink"`, `"x25"`, `"ax25"`,
`"atmpvc"`, `"appletalk"`, or `"packet"`.

**Returns:** `uv_udp_t userdata` or `fail`

### `uv.udp_get_send_queue_size()`

> method form `udp:get_send_queue_size()`

Returns the handle's send queue size.

**Returns:** `integer`

### `uv.udp_get_send_queue_count()`

> method form `udp:get_send_count_size()`

Returns the handle's send queue count.

**Returns:** `integer`

### `uv.udp_open(udp, fd)`

> method form `udp:open(fd)`

Opens an existing file descriptor or Windows SOCKET as a UDP handle.

Unix only: The only requirement of the sock argument is that it follows the
datagram contract (works in unconnected mode, supports sendmsg()/recvmsg(),
etc). In other words, other datagram-type sockets like raw sockets or netlink
sockets can also be passed to this function.

The file descriptor is set to non-blocking mode.

Note: The passed file descriptor or SOCKET is not checked for its type, but
it's required that it represents a valid datagram socket.

**Returns:** `0` or `fail`

### `uv.udp_bind(udp, host, port, [flags])`

> method form `udp:bind(host, port, [flags])`

Bind the UDP handle to an IP address and port. Any `flags` are set with a table
with fields `reuseaddr` or `ipv6only` equal to `true` or `false`.

**Returns:** `0` or `fail`

### `uv.udp_getsockname(udp)`

> method form `udp:getsockname()`

Get the local IP and port of the UDP handle. Returns an address table with
fields `ip`, `family`, and `port`.

**Returns:** `table` or `fail`
- `ip` : `string`
- `family` : `string`
- `port` : `integer`

### `uv.udp_getpeername(udp)`

> method form `udp:getpeername()`

Get the remote IP and port of the UDP handle on connected UDP handles. Returns
an address table with fields `ip`, `family`, and `port`.

**Returns:** `table` or `fail`
- `ip` : `string`
- `family` : `string`
- `port` : `integer`

### `uv.udp_set_membership(udp, multicast_addr, interface_addr, membership)`

> method form `udp:set_membership(multicast_addr, interface_addr, membership)`

Set membership for a multicast address.

  - `multicast_addr` is multicast address to set membership for.
  - `interface_addr` is interface address.
  - `membership` can be the string `"leave"` or `"join"`.

**Returns:** `0` or `fail`

### `uv.udp_set_multicast_loop(udp, on)`

> method form `udp:set_multicast_loop(on)`

Set IP multicast loop flag. Makes multicast packets loop back to local
sockets.

`on` is a boolean.

**Returns:** `0` or `fail`

### `uv.udp_set_multicast_ttl(udp, tty)`

> method form `udp:set_multicast_ttl(tty)`

Set the multicast ttl.

`ttl` is an integer 1 through 255.

**Returns:** `0` or `fail`

### `uv.udp_set_multicast_interface(udp, interface_addr)`

> method form `udp:set_multicast_interface(interface_addr)`

Set the multicast interface to send or receive data on.

**Returns:** `0` or `fail`

### `uv.udp_set_broadcast(udp, on)`

> method form `udp:set_broadcast(on)`

Set broadcast on or off.

**Returns:** `0` or `fail`

### `uv.udp_set_ttl(udp, ttl)`

> method form `udp:set_ttl(ttl)`

Set the time to live.

`ttl` is an integer 1 through 255.

**Returns:** `0` or `fail`

### `uv.udp_send(udp, data, host, port, callback)`

> method form `udp:send(data, host, port, callback)`

Send data over the UDP socket. If the socket has not previously been bound
with `uv.udp_bind()` it will be bound to `0.0.0.0` (the "all interfaces" IPv4
address) and a random port number.

**Returns:** `uv_udp_send_t userdata` or `fail`

### `uv.udp_try_send(udp, data, host, port)`

> method form `udp:try_send(data, host, port)`

Same as `uv.udp_send()`, but won't queue a send request if it can't be
completed immediately.

**Returns:** `integer` or `fail`

### `uv.udp_recv_start(udp, callback)`

> method form `udp:recv_start(callback)`

Prepare for receiving data. If the socket has not previously been bound with
`uv.udp_bind()` it is bound to `0.0.0.0` (the "all interfaces" IPv4 address)
and a random port number.

**Returns:** `0` or `fail`

### `uv.udp_recv_stop(udp)`

> method form `udp:recv_stop()`

Stop listening for incoming datagrams.

**Returns:** `0` or `fail`

### `uv.udp_connect(udp, host, port)`

> method form `udp:connect(host, port)`

Associate the UDP handle to a remote address and port, so every message sent by
this handle is automatically sent to that destination. Calling this function
with a NULL addr disconnects the handle. Trying to call `uv.udp_connect()` on an
already connected handle will result in an `EISCONN` error. Trying to disconnect
a handle that is not connected will return an `ENOTCONN` error.

**Returns:** `0` or `fail`

## `uv_fs_event_t` — FS Event handle

[`uv_fs_event_t`]: #uv_fs_event_t--fs-event-handle

> [`uv_handle_t`][] functions also apply.

FS Event handles allow the user to monitor a given path for changes, for
example, if the file was renamed or there was a generic change in it. This
handle uses the best backend for the job on each platform.

### `uv.new_fs_event()`

Creates and initializes a new `uv_fs_event_t`. Returns the Lua userdata wrapping
it.

**Returns:** `uv_fs_event_t userdata` or `fail`

### `uv.fs_event_start(fs_event, path, flags, callback)`

> method form `fs_event:start(path, flags, callback)`

Start the handle with the given callback, which will watch the specified path
for changes. Any `flags` are set with a table with fields `watch_entry`, `stat`,
or `recursive` equal to `true` or `false`.

**Returns:** `0` or `fail`

### `uv.fs_event_stop()`

> method form `fs_event:stop()`

Stop the handle, the callback will no longer be called.

**Returns:** `0` or `fail`

### `uv.fs_event_getpath()`

> method form `fs_event:getpath()`

Get the path being monitored by the handle.

**Returns:** `string` or `fail`

## `uv_fs_poll_t` — FS Poll handle

[`uv_fs_poll_t`]: #uv_fs_poll_t--fs-poll-handle

> [`uv_handle_t`][] functions also apply.

FS Poll handles allow the user to monitor a given path for changes. Unlike
`uv_fs_event_t`, fs poll handles use `stat` to detect when a file has changed so
they can work on file systems where fs event handles can't.

### `uv.new_fs_poll()`

Creates and initializes a new `uv_fs_poll_t`. Returns the Lua userdata wrapping
it.

**Returns:** `uv_fs_poll_t userdata` or `fail`

### `uv.fs_poll_start(fs_poll, path, interval)`

> method form `fs_poll:start(path, interval)`

Check the file at `path` for changes every `interval` milliseconds.

**Note:** For maximum portability, use multi-second intervals. Sub-second
intervals will not detect all changes on many file systems.

**Returns:** `0` or `fail`

### `uv.fs_poll_stop()`

> method form `fs_poll:stop()`

Stop the handle, the callback will no longer be called.

**Returns:** `0` or `fail`

### `uv.fs_poll_getpath()`

> method form `fs_poll:getpath()`

Get the path being monitored by the handle.

**Returns:** `string` or `fail`

## File system operations

[File system operations]: #file-system-operations

Libuv provides a wide variety of cross-platform sync and async file system
operations.

All file descriptors `fd` are integers.

If a `callback` is passed to functions that accept one, the request `uv_fs_t` is
returned to the original caller and is asynchronously completed with results
passed to the callback. If omitted, the request is completed synchronously with
values returned to the original caller.

All `mode`s are integers indicating access permissions in decimal format. To use
an octal format, use a `tonumber`. For example: `tonumber("0666", 8)`.

### `uv.fs_close(fd, [callback])`

Equivalent to `close(2)`.

**Returns:** `boolean`

### `uv.fs_open(path, flags, mode, [callback])`

Equivalent to `open(2)`. Access `flags` may be an integer or one of: `"r"`,
`"rs"`, `"sr"`, `"r+"`, `"rs+"`, `"sr+"`, `"w"`, `"wx"`, `"xw"`, `"w+"`,
`"wx+"`, `"xw+"`, `"a"`, `"ax"`, `"xa"`, `"a+"`, `"ax+"`, or "`xa+`".

**Returns:** `integer`

**Note:** On Windows, libuv uses `CreateFileW` and thus the file is always
opened in binary mode. Because of this, the `O_BINARY` and `O_TEXT` flags are
not supported.

### `uv.fs_read(fd, size, offset, [callback])`

Equivalent to `preadv(2)`. Returns any data. An empty string indicates EOF.

**Returns:** `string`

### `uv.fs_unlink(path, [callback])`

Equivalent to `unlink(2)`.

**Returns:** `boolean`

### `uv.fs_write(fd, data, offset, [callback])`

Equivalent to `pwritev(2)`. Returns the number of bytes written.

**Returns:** `integer`

### `uv.fs_mkdir(path, mode, [callback])`

Equivalent to `mkdir(2)`.

**Returns:** `boolean`

### `uv.fs_mkdtemp(template, [callback])`

Equivalent to `mkdtemp(3)`.

**Returns:** `string`

### `uv.fs_rmdir(path, [callback])`

Equivalent to `rmdir(2)`.

**Returns:** `boolean`

### `uv.fs_scandir(path, [callback])`

Equivalent to `scandir(3)`, with a slightly different API. Returns a handle that
the user can pass to `uv.fs_scandir_next()`.

**Returns:** `uv_fs_t userdata`

### `uv.fs_scandir_next(fs)`

Called on a `uv_fs_t` returned by `uv.fs_scandir()` to get the next directory
entry data as a `name, type` pair. When there are no more entries, `nil` is
returned.

**Note:** This function only has a synchronous version. See `uv.fs_opendir` and
its related functions for an asynchronous version.

**Returns:** `string, string` or `nil` or `fail`

### `uv.fs_stat(path, [callback])`

Equivalent to `stat(2)`.

**Returns:** `table`
- `dev` : `integer`
- `mode` : `integer`
- `nlink` : `integer`
- `uid` : `integer`
- `gid` : `integer`
- `rdev` : `integer`
- `ino` : `integer`
- `size` : `integer`
- `blksize` : `integer`
- `blocks` : `integer`
- `flags` : `integer`
- `gen` : `integer`
- `atime` : `table`
  - `sec` : `integer`
  - `nsec` : `integer`
- `mtime` : `table`
  - `sec` : `integer`
  - `nsec` : `integer`
- `ctime` : `table`
  - `sec` : `integer`
  - `nsec` : `integer`
- `birthtime` : `table`
  - `sec` : `integer`
  - `nsec` : `integer`
- `type` : `string`

### `uv.fs_fstat(fd, [callback])`

Equivalent to `fstat(2)`.

**Returns:** `table` (see `uv.fs_stat` information)

### `uv.fs_lstat(path, [callback])`

Equivalent to `lstat(2)`.

**Returns:** `table` (see `uv.fs_stat` information)

### `uv.fs_rename(path, new_path, [callback])`

Equivalent to `rename(2)`.

**Returns:** `boolean`

### `uv.fs_fsync(fd, [callback])`

Equivalent to `fsync(2)`.

**Returns:** `boolean`

### `uv.fs_fdatasync(fd, [callback])`

Equivalent to `fdatasync(2)`.

**Returns:** `boolean`

### `uv.fs_ftruncate(fd, offset, [callback])`

Equivalent to `ftruncate(2)`.

**Returns:** `boolean`

### `uv.fs_sendfile(out_fd, in_fd, in_offset, size)`

Limited equivalent to `sendfile(2)`. Returns the number of bytes written.

**Returns:** `integer`

### `uv.fs_access(path, mode, [callback])`

Equivalent to `access(2)` on Unix. Windows uses `GetFileAttributesW()`. Access
`mode` can be an integer or a string containing `""R"` or `"W"` or `"X"`.
Returns `true` or `false` indicating access permission.

**Returns:** `boolean`

### `uv.fs_chmod(path, mode, [callback])`

Equivalent to `chmod(2)`.

**Returns:** `boolean`

### `uv.fs_fchmod(fd, mode, [callback])`

Equivalent to `fchmod(2)`.

**Returns:** `boolean`

### `uv.fs_utime(path, atime, mtime, [callback])`

Equivalent to `utime(2)`.

**Returns:** `boolean`

### `uv.fs_futime(fd, atime, mtime, [callback])`

Equivalent to `futime(2)`.

**Returns:** `boolean`

### `uv.fs_link(path, new_path, [callback])`

Equivalent to `link(2)`.

**Returns:** `boolean`

### `uv.fs_symlink(path, new_path, flags, [callback])`

Equivalent to `symlink(2)`.

**Returns:** `boolean`

### `uv.fs_readlink(path, [callback])`

Equivalent to `readlink(2)`.

**Returns:** `string`

### `uv.fs_realpath(path, [callback])`

Equivalent to `realpath(3)`.

**Returns:** `string`

### `uv.fs_chown(path, uid, gid, [callback])`

Equivalent to `chown(2)`.

**Returns:** `boolean`

### `uv.fs_fchown(fd, uid, gid, [callback])`

Equivalent to `fchown(2)`.

**Returns:** `boolean`

### `uv.fs_copyfile(path, new_path, flags, [callback])`

Copies a file from path to new_path.

**Returns:** `boolean`

### `uv.fs_opendir(path, [callback], [entries])`

Opens path as a directory stream. Returns a handle that the user can pass to
`uv.fs_readdir()`. The `entries` parameter defines the maximum number of entries
that should be returned by each call to `uv.fs_readdir()`.

**Returns:** `uv_dir_t userdata`

### `uv.fs_readdir(dir, [callback])`

Iterates over the directory stream `uv_dir_t` returned by a successful
`uv.fs_opendir()` call. A table of data tables is returned where the number
of entries `n` is equal to or less than the `entries` parameter used in
the associated `uv.fs_opendir()` call.

**Returns:** `table`
- `[1, 2, 3, ..., n]` : `table`
  - `name` : `string`
  - `type` : `string`

### `uv.fs_closedir(dir, [callback])`

Closes a directory stream returned by a successful `uv.fs_opendir()` call.

**Returns:** `boolean`

### `uv.fs_statfs(path, [callback])`

Equivalent to `statfs(2)`.

**Returns:** `table`
- `type` : `integer`
- `bsize` : `integer`
- `blocks` : `integer`
- `bfree` : `integer`
- `bavail` : `integer`
- `files` : `integer`
- `ffree` : `integer`

## Thread pool work scheduling

[Thread pool work scheduling]: #thread-pool-work-scheduling

Libuv provides a threadpool which can be used to run user code and get notified
in the loop thread. This threadpool is internally used to run all file system
operations, as well as `getaddrinfo` and `getnameinfo` requests.

```lua
local function work_callback(a, b)
  return a + b
end

local function after_work_callback(c)
  print("The result is: " .. c)
end

local work = uv.new_work(work_callback, after_work_callback)

work:queue(1, 2)

-- output: "The result is: 3"
```

### `uv.new_work(work_callback, after_work_callback)`

Creates and initializes a new `luv_work_ctx_t` (not `uv_work_t`). Returns the
Lua userdata wrapping it.

**Returns:** `luv_work_ctx_t userdata`

### `uv.queue_work(work_ctx, ...)`

> method form `work_ctx:queue(...)`

Queues a work request which will run `work_callback` in a new Lua state in a
thread from the threadpool with any additional arguments from `...`. Values
returned from `work_callback` are passed to `after_work_callback`, which is
called in the main loop thread.

**Returns:** `boolean` or `fail`

## DNS utility functions

[DNS utility functions]: #dns-utility-functions

All functions take an optional `callback`. If the callback is omitted, the
request is completed synchronously with values returned to the original caller.
If a callback is provided, the request `uv_fs_t` is returned to the original
caller and is asynchronously completed with results passed to the callback.

### `uv.getaddrinfo(host, port, [hints, [callback]])`

Equivalent to `getaddrinfo(3)`. Either `host` or `port` may be `nil` but not
both. The `hints` table may have fields: `socktype`, `protocol`, `addrconfig`,
`v4mapped`, `all`, `numberichost`, `passive`, `numericserv`, or `canonname`.

Returns a table of address tables where address tables may have fields
`addr`, `family`, `port`, `socktype`, `protocol`, `canonname`.

**Returns:** `table` or `fail`
- `[1, 2, 3, ..., n]` : `table`
  - `addr` : `string`
  - `family` : `string`
  - `port` : `integer`
  - `socktype` : `string`
  - `protocol` : `string`
  - `canonname` : `string`

### `uv.getnameinfo(address, [callback])`

Equivalent to `getnameinfo(3)`. The `address` table may have fields: `ip`,
`port`, and `family`.

**Returns:** `string, string` or `fail`

## Threading and synchronization utilities

[Threading and synchronization utilities]: #threading-and-synchronization-utilities

Libuv provides cross-platform implementations for multiple threading an
 synchronization primitives. The API largely follows the pthreads API.

### `uv.new_thread([options], entry, ...)`

Creates and initializes a `luv_thread_t` (not `uv_thread_t`). Returns the Lua
userdata wrapping it and asynchronously executes `entry`, which can be either
a Lua function or a Lua function dumped to a string. Additional arguments `...`
are passed to the `entry` function and an optional `options` table may be
provided. Currently accepted `option` fields are `stack_size`.

**Returns:** `luv_thread_t userdata` or `fail`

### `uv.thread_equal()`

> method form `thread:equal(other_thread)`

Returns a boolean indicating whether two threads are the same. This function is
equivalent to the `__eq` metamethod.

**Returns:** `boolean`

### `uv.thread_self()`

Returns the handle for the thread in which this is called.

**Returns:** `luv_thread_t`

### `uv.thread_join(thread)`

> method form `thread:join()`

Waits for the `thread` to finish executing its entry function.

**Returns:** `boolean` or `fail`

### `uv.sleep(msec)`

Pauses the thread in which this is called for a number of milliseconds.

**Returns:** Nothing.

**Note:** This is not a libuv function.

## Miscellaneous utilities

[Miscellaneous utilities]: #miscellaneous-utilities

### `uv.exepath()`

Returns the executable path.

**Returns:** `string` or `fail`

### `uv.cwd()`

Returns the current working directory.

**Returns:** `string` or `fail`

### `uv.chdir(cwd)`

Sets the current working directory with the string `cwd`.

**Returns:** `0` or `fail`

### `uv.get_process_title()`

Returns the title of the current process.

**Returns:** `string` or `fail`

### `uv.set_process_title(title)`

Sets the title of the current process with the string `title`.

**Returns:** `0` or `fail`

### `uv.get_total_memory()`

Returns the current total system memory in bytes.

**Returns:** `number`

### `uv.get_free_memory()`

Returns the current free system memory in bytes.

**Returns:** `number`

### `uv.get_constrained_memory()`

Gets the amount of memory available to the process in bytes based on limits
imposed by the OS. If there is no such constraint, or the constraint is unknown,
0 is returned. Note that it is not unusual for this value to be less than or
greater than the total system memory.

**Returns:** `number`

### `uv.resident_set_memory()`

Returns the resident set size (RSS) for the current process.

**Returns:** `integer` or `fail`

### `uv.getrusage()`

Returns the resource usage.

**Returns:** `table` or `fail`
- `utime` : `table` (user CPU time used)
  - `sec` : `integer`
  - `usec` : `integer`
- `stime` : `table` (system CPU time used)
  - `sec` : `integer`
  - `usec` : `integer`
- `maxrss` : `integer` (maximum resident set size)
- `ixrss` : `integer` (integral shared memory size)
- `idrss` : `integer` (integral unshared data size)
- `isrss` : `integer` (integral unshared stack size)
- `minflt` : `integer` (page reclaims (soft page faults))
- `majflt` : `integer` (page faults (hard page faults))
- `nswap` : `integer` (swaps)
- `inblock` : `integer` (block input operations)
- `oublock` : `integer` (block output operations)
- `msgsnd` : `integer` (IPC messages sent)
- `msgrcv` : `integer` (IPC messages received)
- `nsignals` : `integer` (signals received)
- `nvcsw` : `integer` (voluntary context switches)
- `nivcsw` : `integer` (involuntary context switches)

### `uv.cpu_info()`

Returns information about the CPU(s) on the system as a table of tables for each
CPU found.

**Returns:** `table` or `fail`
- `[1, 2, 3, ..., n]` : `table`
  - `model` : `string`
  - `speed` : `number`
  - `times` : `table`
    - `user` : `number`
    - `nice` : `number`
    - `sys` : `number`
    - `idle` : `number`
    - `irq` : `number`

### `uv.getpid()`

**Deprecated:** Please use `uv.os_getpid()` instead.

### `uv.getuid()`

Returns the user ID of the process.

**Returns:** `integer`

**Note:** This is not a libuv function and is not supported on Windows.

### `uv.getgid()`

Returns the group ID of the process.

**Returns:** `integer`

**Note:** This is not a libuv function and is not supported on Windows.

### `uv.setuid(id)`

Sets the user ID of the process with the integer `id`.

**Returns:** Nothing.

**Note:** This is not a libuv function and is not supported on Windows.

### `uv.setgid(id)`

Sets the group ID of the process with the integer `id`.

**Returns:** Nothing.

**Note:** This is not a libuv function and is not supported on Windows.

### `uv.hrtime()`

Returns a current high-resolution time in nanoseconds as a number. This is
relative to an arbitrary time in the past. It is not related to the time of day
and therefore not subject to clock drift. The primary use is for measuring
time between intervals.

**Returns:** `number`

### `uv.uptime()`

Returns the current system uptime in seconds.

**Returns:** `number` or `fail`

### `uv.print_all_handles()`

Prints all handles associated with the main loop to stderr. The format is
`[flags] handle-type handle-address`. Flags are `R` for referenced, `A` for
active and `I` for internal.

**Returns:** Nothing.

**Note:** This is not available on Windows.

**Warning:** This function is meant for ad hoc debugging, there are no API/ABI
stability guarantees.

### `uv.print_active_handles()`

The same as `uv.print_all_handles()` except only active handles are printed.

**Returns:** Nothing.

**Note:** This is not available on Windows.

**Warning:** This function is meant for ad hoc debugging, there are no API/ABI
stability guarantees.

### `uv.guess_handle(fd)`

Used to detect what type of stream should be used with a given file
descriptor `fd`. Usually this will be used during initialization to guess the
type of the stdio streams.

**Returns:** `string`

### `uv.gettimeofday()`

Cross-platform implementation of `gettimeofday(2)`. Returns the seconds and
microseconds of a unix time as a pair.

**Returns:** `integer, integer` or `fail`

### `uv.interface_addresses()`

Returns address information about the network interfaces on the system in a
table. Each table key is the name of the interface while each associated value
is an array of address information where fields are `ip`, `family`, `netmask`,
`internal`, and `mac`.

**Returns:** `table`
- `[name(s)]` : `table`
  - `ip` : `string`
  - `family` : `string`
  - `netmask` : `string`
  - `internal` : `boolean`
  - `mac` : `string`

### `uv.if_indextoname(ifindex)`

IPv6-capable implementation of `if_indextoname(3)`.

**Returns:** `string` or `fail`

### `uv.if_indextoiid(ifindex)`

Retrieves a network interface identifier suitable for use in an IPv6 scoped
address. On Windows, returns the numeric `ifindex` as a string. On all other
platforms, `uv.if_indextoname()` is used.

**Returns:** `string` or `fail`

### `uv.loadavg()`

Returns the load average as a triad. Not supported on Windows.

**Returns:** `number, number, number`

### `uv.os_uname()`

Returns system information.

**Returns:** `table`
- `sysname` : `string`
- `release` : `string`
- `version` : `string`
- `machine` : `string`

### `uv.os_gethostname()`

Returns the hostname.

**Returns:** `string`

### `uv.os_getenv(name, [size])`

Returns the environment variable specified by `name` as string. The internal
buffer size can be set by defining `size`. If omitted, `LUAL_BUFFERSIZE` is
used. If the environment variable exceeds the storage available in the internal
buffer, `ENOBUFS` is returned. If no matching environment variable exists,
`ENOENT` is returned.

**Returns:** `string` or `fail`

**Warning:** This function is not thread safe.

### `uv.os_setenv(name, value)`

Sets the environmental variable specified by `name` with the string `value`.

**Returns:** `boolean` or `fail`

**Warning:** This function is not thread safe.

### `uv.os_unsetenv()`

**Returns:** `boolean` or `fail`

**Warning:** This function is not thread safe.

### `uv.os_environ()`

Returns all environmental variables as a dynamic table of names associated with
their corresponding values.

**Returns:** `table`

**Warning:** This function is not thread safe.

### `uv.os_homedir()`

**Returns:** `string` or `fail`

**Warning:** This function is not thread safe.

### `uv.os_tmpdir()`

**Returns:** `string` or `fail`

**Warning:** This function is not thread safe.

### `uv.os_get_passwd()`

Returns password file information.

**Returns:** `table`
- `username` : `string`
- `uid` : `integer`
- `gid` : `integer`
- `shell` : `string`
- `homedir` : `string`

### `uv.os_getpid()`

Returns the current process ID.

**Returns:** `number`

### `uv.os_getppid()`

Returns the parent process ID.

**Returns:** `number`

### `uv.os_getpriority(pid)`

Returns the scheduling priority of the process specified by `pid`.

**Returns:** `number` or `fail`

### `uv.os_setpriority(pid, priority)`

Sets the scheduling priority of the process specified by `pid`. The `priority`
range is between -20 (high priority) and 19 (low priority).

**Returns:** `boolean` or `fail`

[luv]: https://github.com/luvit/luv
[luvit]: https://github.com/luvit/luvit
[libuv]: https://github.com/libuv/libuv
[libuv documentation page]: http://docs.libuv.org/
[libuv API documentation]: http://docs.libuv.org/en/v1.x/api.html
