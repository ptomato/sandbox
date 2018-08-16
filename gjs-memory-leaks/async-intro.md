# Async, Promises & Threads in GJS


While JavaScript engines use threading behind the scenes, JavaScript programs use a single-threaded event loop. This means that a long, synchronous operation can block the event loop from proceeding until it is completed. This is especially noticeable in applications that involve user input/output and in the case of a Gnome Shell extension can lock up the whole desktop.

Since GJS is JavaScript bindings for the Gnome API we have tools not available in standard JavaScript that we can leverage like [GTask][gtask], but we also lack some tools like [Web Workers][mdn-webworkers].

## Table of Contents

1. [Basic Usage](#basic-usage)
2. [GTask API](#gtask-api)
2. [Spawning Processes](#spawning-processes)
2. [Event Loop and Sources](#basic-usage)
2. [Signals](#signals)


## Basic Usage

Hopefully you're somewhat familiar with [Promises][mdn-promises] and [async functions][mdn-async], but we'll run through a quick example and cover a few GJS specifics.

```js
const GLib = imports.gi.GLib;

// In GJS Promises are scheduled in GLib's event loop. You could also use
// Gtk.init()/Gtk.main() or if you're writing a Shell extension or a
// GApplication/GtkApplication there will be one running already.
let loop = GLib.MainLoop.new(null, false);
    
// This is a simple Promise-returning function that waits one second before
// resolving. We'll use it in a try-catch later, but if we didn't it would
// require a catch clause, either directly attached or on the invocation:
//
//     return new Promise().catch(e => logError(e));
//     myPromiseFunc().catch(logError);
//
// logError() is a handy global function in GJS that takes an Error() and
// prints a simple backtrace.
function myPromiseFunc() {
    return new Promise((resolve, reject) => {
        GLib.usleep(1000000);
        resolve();
    });
}

// This is a simple async function we'll use to invoke a few myPromiseFunc()'s
// in a loop. async functions always return a Promise object, implicitly.
async function myAsyncFunction(name) {
    try {
        for (let i = 0; i < 2; i++) {
            // By using `await` to resolve myPromiseFunc() before scheduling the
            // next we allow other operations to be scheduled in between
            await myPromiseFunc();
            log(name);
        }
    } catch (e) {
        logError(e);
    }
}

// Invoke two of our function
myAsyncFunction('test1');

// async functions implicitly return a Promise so we can attach a then() clause
myAsyncFunction('test2').then(result => {
    log('finished');
    loop.quit();
});
log('started');

// Run the loop
loop.run();
```

Expected output:

```sh
$ time gjs async-intro.js
Gjs-Message: 17:53:27.814: JS LOG: started
Gjs-Message: 17:53:27.814: JS LOG: test1
Gjs-Message: 17:53:28.815: JS LOG: test2
Gjs-Message: 17:53:29.815: JS LOG: test1
Gjs-Message: 17:53:29.816: JS LOG: test2
Gjs-Message: 17:53:29.816: JS LOG: finished

real	0m4.053s
user	0m0.172s
sys	0m0.007s
```

Notice the script takes 4 seconds since once a Promise has started executing it is still synchronous. Also pay attention to how `log('started');` is scheduled *before* the first invocation of `myPromiseFunc()` due to the use of `await`.

Basic Promise behaviour and scheduling is important to understand to truly leverage asynchronous code in GJS, so you should revisit Mozilla's guide to [Using Promises][mdn-promises] and [async functions][mdn-async] if anything seems confusing.

## GTask API

[GTask][gtask] is a Gnome API commonly used to implement asynchronous functions that often run in dedicated threads, can be prioritized in GLib's loop and cancelled. 

Keep in mind that using async/await you can regain synchronous behaviour, so there are very few situations where you can't use GTask async functions to replace their synchronous versions!

Since GTask functions use a familiar `foo_async(callback)` pattern, it's really quite simple to wrap one in a Promise and use it in an async function. Let's start with a common task of reading the contents of a file.

```js
const Gio = imports.gi.Gio;
const GLib = imports.gi.GLib;

let loop = GLib.MainLoop.new(null, false);

function loadContents(file, cancellable=null) {
    // We use an explicit Promise, instead of an async function, because we need
    // resolve() & reject() to break out of the AsyncResult callback...
    return new Promise((resolve, reject) => {
        file.load_contents_async(cancellable, (source_object, res) => {
            // ...and a try-catch to propagate errors through the Promise chain
            try {
                res = source_object.load_contents_finish(res);
                
                // AsyncResult functions return an 'ok' boolean, but we ignore
                // them since an error will be thrown anyways if it's %false
                let [ok, contents, etag_out] = res;
                
                resolve(contents);
            } catch (e) {
                reject(e);
            }
        });
    });
}

// We'll use loadContents() in an async function, but you could also use 'await'
// in place of 'return' and include the Promise in this function
async function loadFile(path, cancellable=null) {
    try {
        let file = Gio.File.new_for_path(path);
        let contents = await loadContents(file, cancellable);
            
        return contents;
    } catch (e) {
        logError(e);
    }
}

// We can use a Gio.Cancellable object to allow the function to be cancelled,
// like with a 'Cancel' button in a dialog, or just leave it %null.
let cancellable = new Gio.Cancellable();

loadFile('/proc/cpuinfo', cancellable).then(contents => {
    log(contents);
    loop.quit();
});

// To cancel the operation we invoke cancel() on the cancellable object, which
// will throw the "Gio.IOErrorEnum: Operation was cancelled" in loadContents()
cancellable.cancel();

loop.run();
```

Using async functions you can leverage try-catch-finally in combination with `then()` and `catch()` to construct flexible error handling routines to suit your use-case. As of GJS-1.54 `finally()` is also available for Promises.

So let's remove the cancellable and re-use `loadContents()` and `loadFile()` to test how threaded Promises play out. If you don't have suitable files handy, you can create an empty 10MB and 10KB file to compare:

```sh
$ dd if=/dev/zero of=10mb.txt count=10240 bs=1024
$ dd if=/dev/zero of=10kb.txt count=10 bs=1024
```

```js
let start = Date.now();

loadFile('10mb.txt', null).then(contents => {
    log(`10mb finished, ${Date.now() - start}ms elapsed`);
});

loadFile('10kb.txt', null).then(contents => {
    log(`10kb finished, ${Date.now() - start}ms elapsed`);
});
```

Expected output:

```sh
$ gjs async-file.js
Gjs-Message: 18:41:55.122: JS LOG: 10kb finished, 2ms elapsed
Gjs-Message: 18:41:55.149: JS LOG: 10mb finished, 36ms elapsed
```

Longer operations still take longer, but they don't block shorter operations. In fact, because `loadFile()` is a Promise and we didn't using `await`, the two operations are essentially running in parallel. Try switching the order of the functions or reading the 10mb file twice:

```sh
$ gjs async-file.js 
Gjs-Message: 18:50:04.043: JS LOG: 10mb finished, 40 ms elapsed
Gjs-Message: 18:50:04.043: JS LOG: 10mb finished, 40 ms elapsed
```

## Spawning Processes

`Gio.Subprocess` is similar to `subprocess.py` in Python, allowing you to spawn and communicate with applications asynchronously using the GTask API. It is preferred over GLib's lower-level functions since it automatically reaps child processes avoiding zombie processes and prevents dangling file descriptors. This is especially important in GJS because of how "out" parameters are handled.

Consider the following snippet using `GLib.spawn_async_with_pipes()`. In other languages we would pass "in" `null` as a function argument for pipes like `stdin` that we don't plan on using, preventing them from being opened. In GJS all three pipes are opened implicitly and must be explicitly closed, or we may eventually get a *"Too many open files"* error.

```js
let [ok, pid, stdin, stdout, stderr] = GLib.spawn_async_with_pipes(
    null,                   // working directory
    ['ls' '-la'],           // argv
    null,                   // envp
    GLib.SpawnFlags.NONE,   // flags
    null                    // child_setup function
);
```

Let's do a simple exercise of using `Gio.Subprocess` to execute `ls -a` in the current directory then log the output manually.

```js
const Gio = imports.gi.Gio;
const GLib = imports.gi.GLib;

let loop = GLib.MainLoop.new(null, false);

async function execCommand(argv, cancellable=null) {
    try {
        // There is also a reusable Gio.SubprocessLauncher class available
        let proc = new Gio.Subprocess({
            argv: argv,
            // There are also other types of flags for merging stdout/stderr,
            // redirecting to /dev/null or inheriting the parent's pipes
            flags: Gio.SubprocessFlags.STDOUT_PIPE
        });
        
        // Classes that implement GInitable must be initialized before use, but
        // an alternative in this case is to use Gio.Subprocess.new(argv, flags)
        //
        // If the class implements GAsyncInitable then Class.new_async() could
        // also be used and awaited in a Promise.
        proc.init(null);

        let stdout = await new Promise((resolve, reject) => {
            // communicate_utf8() returns a string, communicate() returns a
            // a GLib.Bytes and there are "headless" functions available as well
            proc.communicate_utf8_async(null, cancellable, (proc, res) => {
                let ok, stdout, stderr;

                try {
                    [ok, stdout, stderr] = proc.communicate_utf8_finish(res);
                    resolve(stdout);
                } catch (e) {
                    reject(e);
                }
            });
        });

        return stdout;
    } catch (e) {
        logError(e);
    }
}

execCommand(['ls', '-a']).then(stdout => {
    stdout.split('\n').map(line => log(line));
    loop.quit();
});

loop.run();
```

Expected output:

```sh
$ gjs async-proc.js
Gjs-Message: 16:57:26.784: JS LOG: .
Gjs-Message: 16:57:26.784: JS LOG: ..
Gjs-Message: 16:57:26.784: JS LOG: async-file.js
Gjs-Message: 16:57:26.784: JS LOG: async-intro.js
Gjs-Message: 16:57:26.784: JS LOG: async-proc.js
Gjs-Message: 16:57:26.784: JS LOG: async-signal.js
Gjs-Message: 16:57:26.784: JS LOG: async-source.js
Gjs-Message: 16:57:26.784: JS LOG: 
```

## GLib Main Event Loop

[GLib's event loop][glib-mainloop] also gives us some other options like priority scheduling and condition based callbacks. The main benefit over the Promise API for general usage is the granular priority of the main event loop. For example, `GLib.PRIORITY_DEFAULT` is `0` while `GLib.PRIORITY_LOW` is `300`, giving a lot of room for custom priorities.

JavaScript's async functions are usually not suitable as a source callback since a `GSource` will persist based on  a `true` (`GLib.SOURCE_CONTINUE`) or `false` (`GLib.SOURCE_REMOVE`) return value.

There are a few general usage functions like `GLib.timeout_add()` and `GLib.idle_add()` you can use to schedule callbacks, but a more real-world example might be reading from a stream when it has bytes to read.

```js
const GLib = imports.gi.GLib;

let loop = GLib.MainLoop.new(null, false);

// You can use custom priorities by passing an integer.
GLib.timeout_add_seconds(299, 1, () => {
    log('low priority');
    return GLib.SOURCE_REMOVE;
});

GLib.timeout_add_seconds(GLib.PRIORITY_HIGH, 1, () => {
    log('high priority');
    return GLib.SOURCE_REMOVE;
});

GLib.timeout_add_seconds(GLib.PRIORITY_DEFAULT, 1, () => {
    log('default priority');
    return GLib.SOURCE_REMOVE;
});

// Since this is an input stream the condition is always GLib.IOCondition.IN,
// but in some cases you can specify the condition that triggers the callback
let stdin = new Gio.UnixInputStream({ fd: 0 });
let source = stdin.create_source(null);
source.set_callback(() => {
    try {
        let input = stdin.read_bytes(4096, null).toArray().toString();
        log(input.slice(0, -1));

        // return %true or GLib.SOURCE_CONTINUE to wait for more data
        return GLib.SOURCE_CONTINUE;
    } catch (e) {
        logError(e);
        
        // return %false or GLib.SOURCE_REMOVE to destroy the source
        return GLib.SOURCE_REMOVE;
    }
});
source.attach(null);

loop.run();
```

Expected output:

```sh
$ gjs async-priority.js
Gjs-Message: 18:54:11.232: JS LOG: high priority
Gjs-Message: 18:54:11.232: JS LOG: default priority
Gjs-Message: 18:54:11.232: JS LOG: low priority
type in terminal and press enter
Gjs-Message: 18:54:15.337: JS LOG: type in terminal and press enter
```

## GSignals

GSignals are used quite a bit in Gnome, but what does that have to do with async functions and Promises? Since async functions *immediately* return a Promise object they can be used as callbacks for signals that would normally block until the callback finished its operation.

Although many signals have void or irrelevant return values, some like `GtkWidget::delete-event` are propagated to other objects depending on the return value (usually boolean). As of at least gjs-1.52, a Promise returned by a signal callback will be coerced to `true`, but this behaviour may change in future.

Take for example `Gio.SocketService::incoming` whose documentation says:

> ...the handler must immediately return, or else it will block additional incoming connections from being serviced.

But also says:

> [Return] TRUE to stop other handlers from being called


```js
const Gio = imports.gi.Gio;
const GLib = imports.gi.GLib;

let loop = GLib.MainLoop.new(null, false);

let service = new Gio.SocketService();

// Arrow-functions can be async functions, too
service.connect('incoming', async (service, connection, source_object) => {
    try {
        connection.close(null);
    } catch (e) {
        logError(e);
    }
});

loop.run();
```

On the topic of GSignals and performance, it's worth noting that a global lock is acquired for [signal handlers lookup][gsignal-lock]. If you are subclassing a GObject it can be more performant to override the default handler using vfuncs, especially with signals like `GtkWidget::draw` which can be called frequently:

```js
var MyWidget = GObject.registerClass({
    GTypeName: 'MyWidget'
}, class MyWidget extends Gtk.DrawingArea {
    vfunc_draw(cr) {
        cr.$dispose();
        return false;
    }
});
```

[gsignal-lock]: https://gitlab.gnome.org/GNOME/glib/blob/master/gobject/gsignal.c#L3179
[gtask]: https://developer.gnome.org/gio/stable/GTask.html
[glib-mainloop]: https://developer.gnome.org/glib/stable/glib-The-Main-Event-Loop.html
[mdn-webworkers]: https://developer.mozilla.org/docs/Web/API/Web_Workers_API
[mdn-promises]: https://developer.mozilla.org/docs/Web/JavaScript/Guide/Using_promises
[mdn-async]: https://developer.mozilla.org/docs/Web/JavaScript/Reference/Statements/async_function
