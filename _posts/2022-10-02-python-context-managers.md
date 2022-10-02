## Introduction

Context Managers are a way to help the developers reduce the burden of writing repetitive code that is critical or if forgotten could cause unexpected exceptions, memory leaks, and overloading of external resources.

For example, every time someone opens a file, it is expected to be closed explicitly to ensure the data written to the file is flushed correctly and the same is saved. Below is the example of proper file opening and closing, that is expected from the developers.

```python
test_file = open("test_file.txt", mode="w")
test_file.write("This is a test file!")
test_file.close()
```

If the `close` is not explicitly called, 

- There is a possibility of data loss in case of an application crash, although Python (CPython) will try to clean up the files which are not in use by properly closing them (garbage collection)
- There is a limit to the number of files that a process(application) can keep open by the Operating Systems - if crossed that limit, the application cannot open new files
- Data may not be saved into the file until the `close` is called, which will provide incomplete data to the users who try to get the latest content from the file
- Users may not be able to delete the unclosed file on some systems

In the above example, it’s normal that the developers might miss calling the `close` method when writing a big application and it is also repetitive to keep writing the `close` every time a file is opened. 

In the case of file handling, it’s straightforward. There could be some complicated cases like when using threads, developers might need to acquire locks before doing some operation, then release the lock and might need to do additional cleanups. There are many such scenarios that force developers to write a custom prep/cleanup function that needs to be called before/after doing something.

```python
import threading

lock = threading.Lock()
lock.acquire()
try:
    # do something
except Exception:
    # handle exception
finally:
    # do some resource cleanup
    lock.release()
```

```python
try:
    # do something
finally:
    update_the_external_resource()
    close_or_clean_up_external_resources()
```

Python introduced the context managers via `with` statements to ease things up and make the code more consistent. The file handling example that we’ve seen above can be rewritten with the context managers, which will handle the file closing/cleanup for us.

```python
with open("test_file.txt", mode="w") as test_file:
    test_file.write("This is a test file!")
```

### How does this work?

Whenever we use the `with`, the method that we use along the with should be having the magic methods `__enter__` and `__exit__`. The `open` that we’ve used has `__enter__` and `__exit__` already written in the standard library. 

So, when we use the `open` with `with` statement, it executes the `__enter__` method, and whatever the enter method returns will be assigned to the test_file (which is called target). Then once the code within `with` the block is executed, the `__exit__` method will be invoked which will close the file correctly. 

Below is the equivalent of how Python will execute the above `with` block.

```python
file_manager = open("test_file.txt", mode="w")
file_enter = type(file_manager).__enter__
file_exit = type(file_manager).__exit__

try:
    test_file = file_enter(file_manager)
    test_file.write("This is a test file!")
except:
    # handle exception
finally:
    file_exit(file_manager)
```

 

### Advanced: How does this work?

[8. Compound statements — Python 3.10.7 documentation](https://docs.python.org/3/reference/compound_stmts.html#the-with-statement)

## Wring your own custom context managers

### Class based

Let's say we want to write a worker that pulls something from a queue and does some work. Before doing the work, the worker needs to pull the task from the queue and then let the queue know that it’s working on the task. Once completed, should send a message to the queue saying the task is completed or an error message saying the task failed and why.

```python
class Worker:
    # init code __init__

    def __enter__(self):
        task = get_task_from_queue()
        self.task = task
        return self
    def __exit__(self, exception, value, traceback):
        self.send_completed_message()

with Worker() as worker:
    worker.do_task()
     
```

### Function based

The same worker context manager can be implemented as a function using `contextmanager` decorator from `contextlib`.

```python
from contextlib import contextmanager

@contextmanager
def managed_resource(*args, **kwargs):
    worker = Worker()
    worker.task = get_task_from_queue()
    try:
        yield worker
    finally:
        worker.send_completed_message()
```

## How is this different from Decorators

We can see some kind of similarity between decorator and context manager in Python. Both of them let us do something before and after executing a piece of code.

The only difference is that the decorators are for block-level codes like functions or classes, whereas the context managers are for anonymous blocks of code that can be used anywhere.

## gotchas, limitations, best practices, bad practices to avoid

## categorize this into a design pattern or data type or programming paradigm or something like that

## some reference to actual usage in some software or open source code

- [PEP 343 – The “with” Statement | peps.python.org](https://peps.python.org/pep-0343/)
- [PEP 340 – Anonymous Block Statements | peps.python.org](https://peps.python.org/pep-0340/)
- [8. Compound statements — Python 3.10.7 documentation](https://docs.python.org/3/reference/compound_stmts.html#the-with-statement)
- [contextlib — Utilities for with-statement contexts — Python 3.10.7 documentation](https://docs.python.org/3/library/contextlib.html)
