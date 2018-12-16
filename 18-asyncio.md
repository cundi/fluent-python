# CHAPTER 18 Concurrency with asyncio

`Concurrency is about dealing with lots of things at once.
Parallelism is about doing lots of things at once.
Not the same, but related.
One is about structure, one is about execution.
Concurrency provides a way to structure a solution to solve a problem that may (but not
necessarily) be parallelizable. [1]
— Rob Pike
Co-inventor of the Go language`

【注释１】

Professor Imre Simon [2] liked to say there are two major sins in science: using different
words to mean the same thing and using one word to mean different things. If you do
any research on concurrent or parallel programming you will find different definitions
for “concurrency” and “parallelism.” I will adopt the informal definitions by Rob Pike,
quoted above.

【注释２】

For real parallelism, you must have multiple cores. A modern laptop has four CPU cores
but is routinely running more than 100 processes at any given time under normal, casual
use. So, in practice, most processing happens concurrently and not in parallel. The
computer is constantly dealing with 100+ processes, making sure each has an oppor‐
tunity to make progress, even if the CPU itself can’t do more than four things at once.
Ten years ago we used machines that were also able to handle 100 processes concurrently,
but on a single core. That’s why Rob Pike titled that talk “Concurrency Is Not Parallelism
(It’s Better).”

就真并行来说，你必须使用多核心。

This chapter introduces asyncio , a package that implements concurrency with corou‐
tines driven by an event loop. It’s one of the largest and most ambitious libraries ever
added to Python. Guido van Rossum developed asyncio outside of the Python repos‐
itory and gave the project a code name of “Tulip”—so you’ll see references to that flower
when researching this topic online. For example, the main discussion group is still called
python-tulip.

Tulip was renamed to asyncio when it was added to the standard library in Python 3.4.
It’s also compatible with Python 3.3—you can find it on PyPI under the new official
name. Because it uses yield from expressions extensively, asyncio is incompatible with
older versions of Python.

>The Trollius project—also named after a flower—is a backport of asyncio to Python 2.6 and newer,
> replacing yield from with
> yield and clever callables named From and Return . A yield from ... expression becomes yield From(...) ; and when a coroutine needs to return a result, you write raise Return(result) instead ofreturn result . Trollius is led by Victor Stinner, who is also an asyncio core developer, and who kindly agreed to review this chapter as this book was going into production.

In this chapter we’ll see:
• A comparison between a simple threaded program and the asyncio equivalent,
showing the relationship between threads and asynchronous tasks
• How the asyncio.Future class differs from concurrent.futures.Future
• Asynchronous versions of the flag download examples from Chapter 17
• How asynchronous programming manages high concurrency in network applica‐
tions, without using threads or processes
• How coroutines are a major improvement over callbacks for asynchronous pro‐
gramming
• How to avoid blocking the event loop by offloading blocking operations to a thread
pool
• Writing asyncio servers, and how to rethink web applications for high concurrency
• Why asyncio is poised to have a big impact in the Python ecosystem
Let’s get started with the simple example contrasting threading and asyncio .

## Thread Versus Coroutine: A Comparison 线程与协程的比较

During a discussion about threads and the GIL, Michele Simionato posted a simple but
fun example using multiprocessing to display an animated spinner made with the
ASCII characters "|/-\" on the console while some long computation is running.
I adapted Simionato’s example to use a thread with the Threading module and then a
coroutine with asyncio , so you can see the two examples side by side and understand
how to code concurrent behavior without threads.

在讨论线程和ＧＩＬ时，

The output shown in Examples 18-1 and 18-2 is animated, so you really should run the
scripts to see what happens. If you’re in the subway (or somewhere else without a WiFi
connection), take a look at Figure 18-1 and imagine the \ bar before the word “thinking”
is spinning.

![images]()

Figure 18-1. The scripts spinner_thread.py and spinner_asyncio.py produce similar
output: the repr of a spinner object and the text Answer: 42. In the screenshot, spin‐
ner_asyncio.py is still running, and the spinner message \ thinking! is shown; when the
script ends, that line will be replaced by the Answer: 42.

Let’s review the spinner_thread.py script first (Example 18-1).

Example 18-1. spinner_thread.py: animating a text spinner with a thread

```python
import threading, itertools, time, sys


class Signal: # 1
    go = True


def spin(msg, signal): # 2
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'): # 3
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status)) # 4
        time.sleep(.1)
        if not signal.go: # 5
            break
    write(' ' * len(status) + '\x08' * len(status)) # 6


def slow_function():
    # pretend waiting a long time for I/O
    time.sleep(3)
    return 42


def supervisor():
    signal = Signal()
    spinner = threading.Thread(target=spin, args=('thinking!', signal))
    print('spinner object:', spinner)
    spinner.start()
    result = slow_function()
    signal.go = False
    spinner.join()
    return result


def main():
    result = supervisor() # 15
    print('Answer:', result)


if __name__ == '__main__':
    main()
```

1. This class defines a simple mutable object with a `go` attribute we’ll use to control the thread from outside.

2. This function will run in a separate thread. The `signal` argument is an instance of the `Signal` class just defined.

3. This is actually an infinite loop because `itertools.cycle` produces items cycling from the given sequence forever.

4. The trick to do text-mode animation: move the cursor back with backspace characters ( `\x08` ).

5. If the `go` attribute is no longer `True` , exit the loop.

6. Clear the status line by overwriting with spaces and moving the cursor back to the beginning.

7. Imagine this is some costly computation.

8. Calling ``sleep` will block the main thread, but crucially, the GIL will be released so the secondary thread will proceed.

9. This function sets up the secondary thread, displays the thread object, runs the slow computation, and kills the thread.

10. Display the secondary thread object. The output looks like `<Thread(Thread-1, initial)>` .

11. Start the secondary thread.

12. Run `slow_function`; this blocks the main thread. Meanwhile, the spinner is animated by the secondary thread.

13. Change the state of the `signal`; this will terminate the `for` loop inside the `spin` function.

14. Wait until the `spinner` thread finishes.

15. Run the `supervisor` function.

Note that, by design, there is no API for terminating a thread in Python. You must send
it a message to shut down. Here I used the signal.go attribute: when the main thread
sets it to false, the spinner thread will eventually notice and exit cleanly.

Now let’s see how the same behavior can be achieved with an @asyncio.coroutine
instead of a thread.

>### Note
>As noted in the “Chapter Summary” on page 498 (Chapter 16),
>asyncio uses a stricter definition of “coroutine.” A coroutine
>suitable for use with the asyncio API must use yield from and
>not yield in its body. Also, an asyncio coroutine should be driv‐
>en by a caller invoking it through yield from or by passing the
>coroutine to one of the asyncio functions such as asyncio.async(...) 
>and others covered in this chapter. Finally, the
>@asyncio.coroutine decorator should be applied to coroutines,
>as shown in the examples.

Take a look at Example 18-2.

`Example 18-2. spinner_asyncio.py: animating a text spinner with a coroutine`

```python
import asyncio
import itertools
import sys

@asyncio.coroutine #1
def spin(msg): #2
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))
        try:
            yield from asyncio.sleep(.1) #3
        except asyncio.CancelledError: #4
            break
    write(' ' * len(status) + '\x08' * len(status))

@asyncio.coroutine
def slow_function():
    # pretend waiting a long time for I/O
    yield from asyncio.sleep(3)
    return 42

@asyncio.coroutine
def supervisor():
    spinner = asyncio.async(spin('thinking!'))
    print('spinner object:', spinner)
    result = yield from slow_function()
    spinner.cancel()
    return result

def main():
    loop = asyncio.get_event_loop()
    result = loop.run_until_complete(supervisor())
    loop.close()
    print('Answer:', result)
```

1. Coroutines intended for use with asyncio should be decorated with @asyncio.coroutine . This not mandatory, but is highly advisable. See explanation following this listing.

2. Here we don’t need the signal argument that was used to shut down the thread in the spin function of Example 18-1.

3. Use yield from asyncio.sleep(.1) instead of just time.sleep(.1) , to sleep without blocking the event loop.

4. If asyncio.CancelledError is raised after spin wakes up, it’s because cancellation was requested, so exit the loop.

5. `slow_function` is now a coroutine, and uses yield from to let the event loop proceed while this coroutine pretends to do I/O by sleeping.

6. The yield from asyncio.sleep(3) expression handles the control flow to the main loop, which will resume this coroutine after the sleep delay.

7. `supervisor` is now a coroutine as well, so it can drive `slow_function` with `yield from`.

8. `asyncio.async(...)` schedules the `spin` coroutine to run, wrapping it in a `Task` object, which is returned immediately.

9. Display the `Task` object. The output looks like `<Task pending coro=<spin() running at spinner_asyncio.py:12>>` .

10. Drive the `slow_function()` . When that is done, get the returned value. Meanwhile, the event loop will continue running because `slow_function` ultimately uses `yield from asyncio.sleep(3)` to hand control back to the main loop.

11. A `Task` object can be cancelled;this raises `asyncio.CancelledError` at the `yield` line where the coroutine is currently suspended. The coroutine may catch the exception and delay or even refuse to cancel.

12. Get a reference to the event loop.

13. Drive the `supervisor` coroutine to completion; the return value of the coroutine is the return value of this call.

>### Cautious
>Never use `time.sleep(...)` in `asyncio` coroutines unless you want
>to block the main thread, therefore freezing the event loop and
>probably the whole application as well. If a coroutine needs to
>spend some time doing nothing, it should `yield from asyncio.sleep(DELAY)` .

The use of the `@asyncio.coroutine` decorator is not mandatory, but highly recom‐
mended: it makes the coroutines stand out among regular functions, and helps with
debugging by issuing a warning when a coroutine is garbage collected without being
yielded from—which means some operation was left unfinished and is likely a bug. This
is not a `priming decorator`.

Note that the line count of `spinner_thread.py` and `spinner_asyncio.py` is nearly the same.
The `supervisor` functions are the heart of these examples. Let’s compare them in detail.
Example 18-3 lists only the `supervisor` from the `Threading` example.

`Example 18-3. spinner_thread.py: the threaded supervisor function`

```python
def supervisor():
    signal = Signal()
    spinner = threading.Thread(target=spin,
    args=('thinking!', signal))
    print('spinner object:', spinner)
    spinner.start()
    result = slow_function()
    signal.go = False
    spinner.join()
    return result
```

For comparison, Example 18-4 shows the supervisor coroutine.

`Example 18-4. spinner_asyncio.py: the asynchronous supervisor coroutine`

```python
@asyncio.coroutine
def supervisor():
    spinner = asyncio.async(spin('thinking!'))
    print('spinner object:', spinner)
    result = yield from slow_function()
    spinner.cancel()
    return result
```

Here is a summary of the main differences to note between the two supervisor im‐
plementations:

- An `asyncio.Task` is roughly the equivalent of a `threading.Thread`. Victor Stinner, special technical reviewer for this chapter, points out that “a Task is like a green thread in libraries that implement cooperative multitasking, such as `gevent`.”

- A `Task` drives a coroutine, and a Thread invokes a callable.

- You don’t instantiate `Task` objects yourself, you get them by passing a coroutine to` asyncio.async(...)` or `loop.create_task(...)`.

• When you get a Task object, it is already scheduled to run (e.g., by asyn
cio.async ); a Thread instance must be explicitly told to run by calling its start
method.
• In the threaded supervisor , the slow_function is a plain function and is directly
invoked by the thread. In the asyncio supervisor , slow_function is a coroutine
driven by yield from .
• There’s no API to terminate a thread from the outside, because a thread could be
interrupted at any point, leaving the system in an invalid state. For tasks, there is