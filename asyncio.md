---
layout: default
title: asyncio
permalink: /py/asyncio
parent: python
has_children: true
has_toc: true
---
<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

# async io包和async/await
At the heart of async IO are coroutines. A coroutine is a specialized version of a Python generator function. Let’s start with a baseline definition and then build off of it as you progress here: a coroutine is a function that can suspend its execution before reaching return, and it can indirectly pass control to another coroutine for some time.

```python
#!/usr/bin/env python3
# countasync.py

import asyncio

async def count():
    print("One")
    await asyncio.sleep(1)
    print("Two")

async def main():
    await asyncio.gather(count(), count(), count())

if __name__ == "__main__":
    import time
    s = time.perf_counter()
    asyncio.run(main())
    elapsed = time.perf_counter() - s
    print(f"{__file__} executed in {elapsed:0.2f} seconds.")
```
在执行脚本的时候注意和不使用async的区别。

输出的次序是async io 的核心。与每个count()调用交互的是一个时间循环或者叫做coordinator,在每个任务到达`await asyncio.sleep(1)`的时候，函数都会告诉事件循环然后把控制权交给它，“I’m going to be sleeping for 1 second. Go ahead and let something else meaningful be done in the meantime.”。

While using `time.sleep()` and `asyncio.sleep()` may seem banal, they are used as stand-ins for any time-intensive processes that involve wait time. (The most mundane thing you can wait on is a `sleep()` call that does basically nothing.) That is, `time.sleep()` can represent any time-consuming blocking function call, while `asyncio.sleep()` is used to stand in for a non-blocking call (but one that also takes some time to complete).

As you’ll see in the next section, the benefit of awaiting something, including `asyncio.sleep()`, is that the surrounding function can temporarily cede control to another function that’s more readily able to do something immediately. In contrast, `time.sleep()` or any other blocking call is incompatible with asynchronous Python code, because it will stop everything in its tracks for the duration of the sleep time.

## Async IO规则

- `async def`语法会引入原生的coroutine或者异步的生成器。也可以使用`async with`和`async for`表达式。

- `await`关键字会将函数控制权交给事件循环。如果python在`g()`的作用域内找到了一个`await f()`表达式，这时await就会告诉事件循环，"暂停g()的循环直到等待f()的结果返回，同时让其他的东西运行"。 

```python
async def g():
    # Pause here and come back to g() when f() is ready
    r = await f()
    return r
```

对于什么时候可以使用能不能使用async/await由一套严格的规则:
- 使用async def引入的是coroutine,可以使用await,return 或者yield,这都是可选的，可以声明`async def noop(): pass`:
    - 使用await或者return会创建一个coroutine函数，要调用coroutine函数必须await它来得到结果。
    - 在async def块中使用yield比较少见，这回创建一个异步的生成器，可以用async for来迭代。
    - 使用async def定义的任何东西都不能使用`yield from`,这会引发`SyntaxError`

- 在def函数外使用yield也会引发`SyntaxError`，在async def coroutine外使用await也是这样，只能在`coroutine`体里面使用`await`.

下面是一些例子
```python
async def f(x):
    y = await z(x)  # OK - `await` and `return` allowed in coroutines
    return y

async def g(x):
    yield x  # OK - this is an async generator

async def m(x):
    yield from gen(x)  # No - SyntaxError

def m(x):
    y = await z(x)  # Still no - SyntaxError (no `async def` here)
    return y
```
在使用`await f()`的时候，`f()`是一个awaitable的对象。一个`awaitable`的对象或者是1.coroutine或者是2.会返回iterator的定义了.__await__()方法的对象。大多数情况下只需要担心第一种情况。

这带给了我们一个更技术性的问题:旧的将函数标记为coroutine的方式是给普通的def函数使用`@asyncio.coroutine`修饰，结果是基于生成器的coroutine,在3.5版本之后引入了async/await这个方法就过时了。
这两种coroutine是等价的但是第一种是基于生成器的，第二种是原生的corotine：

```python
import asyncio

@asyncio.coroutine
def py34_coro():
    """Generator-based coroutine, older syntax"""
    yield from stuff()

async def py35_coro():
    """Native coroutine, modern syntax"""
    await stuff()
```

写程序的时候尽量使用原生的coroutine,3.10之后就会移除基于coroutine的生成器。

之后的内容会简单的接触基于生成器的coroutine进行解释。async/await被引入的原始是使得coroutine作为独立的特征存在来和正常的生成器函数进行曲分。

Here’s one example of how async IO cuts down on wait time: given a coroutine makerandom() that keeps producing random integers in the range [0, 10], until one of them exceeds a threshold, you want to let multiple calls of this coroutine not need to wait for each other to complete in succession. You can largely follow the patterns from the two scripts above, with slight changes:

```python
#!/usr/bin/env python3
# rand.py

import asyncio
import random

# ANSI colors
c = (
    "\033[0m",   # End of color
    "\033[36m",  # Cyan
    "\033[91m",  # Red
    "\033[35m",  # Magenta
)

async def makerandom(idx: int, threshold: int = 6) -> int:
    print(c[idx + 1] + f"Initiated makerandom({idx}).")
    i = random.randint(0, 10)
    while i <= threshold:
        print(c[idx + 1] + f"makerandom({idx}) == {i} too low; retrying.")
        await asyncio.sleep(idx + 1)
        i = random.randint(0, 10)
    print(c[idx + 1] + f"---> Finished: makerandom({idx}) == {i}" + c[0])
    return i

async def main():
    res = await asyncio.gather(*(makerandom(i, 10 - i - 1) for i in range(3)))
    return res

if __name__ == "__main__":
    random.seed(444)
    r1, r2, r3 = asyncio.run(main())
    print()
    print(f"r1: {r1}, r2: {r2}, r3: {r3}")
```

这个程序使用一个主的coroutine,makerandom()然后对于三个不同的输入同时运行。大多数程序都会包含小的模块化的corouine,然后用一个封装函数将每个coroutine封装在一起。main() is then used to gather tasks (futures) by mapping the central coroutine across some iterable or pool.

在这个小例子中，任务池是range(3),尽管产生随机整数使用asuncio可能不是非常好的选择，asyncio.sleep()使用解决io-bound问题。

![](https://files.realpython.com/media/asyncio-rand.dffdd83b4256.gif)

# asyncio 设计模式

Async IO comes with its own set of possible script designs, which you’ll get introduced to in this section.


## Chaining Coroutines

A key feature of coroutines is that they can be chained together. (Remember, a coroutine object is awaitable, so another coroutine can await it.) This allows you to break programs into smaller, manageable, recyclable coroutines:

```python
#!/usr/bin/env python3
# chained.py

import asyncio
import random
import time

async def part1(n: int) -> str:
    i = random.randint(0, 10)
    print(f"part1({n}) sleeping for {i} seconds.")
    await asyncio.sleep(i)
    result = f"result{n}-1"
    print(f"Returning part1({n}) == {result}.")
    return result

async def part2(n: int, arg: str) -> str:
    i = random.randint(0, 10)
    print(f"part2{n, arg} sleeping for {i} seconds.")
    await asyncio.sleep(i)
    result = f"result{n}-2 derived from {arg}"
    print(f"Returning part2{n, arg} == {result}.")
    return result

async def chain(n: int) -> None:
    start = time.perf_counter()
    p1 = await part1(n)
    p2 = await part2(n, p1)
    end = time.perf_counter() - start
    print(f"-->Chained result{n} => {p2} (took {end:0.2f} seconds).")

async def main(*args):
    await asyncio.gather(*(chain(n) for n in args))

if __name__ == "__main__":
    import sys
    random.seed(444)
    args = [1, 2, 3] if len(sys.argv) == 1 else map(int, sys.argv[1:])
    start = time.perf_counter()
    asyncio.run(main(*args))
    end = time.perf_counter() - start
    print(f"Program finished in {end:0.2f} seconds.")
```

Pay careful attention to the output, where part1() sleeps for a variable amount of time, and part2() begins working with the results as they become available:


In this setup, the runtime of `main()` will be equal to the maximum runtime of the tasks that it gathers together and schedules.

## Using a Queue
asyncio包提供了一个`queue`类，这个类和queue模块的类相似。目前为止的例子暂不需要queue结构，在chain.py中每个任务由一个coroutine集合构成，互相之间进行await,在一个链中互相传递单一的输出。

有一个其他的结构可以和asyncio一起工作:很多的生产者互相之间没有关联，向一个队列中添加items，每个生产者可能给队列添加多个项，可能在任意的没声明的事件。一组消费者从队列中拉取items，不等待其他的信号。

这种设计中没有任何单独的消费者和生产者之间的chain.消费这不知道生产者的数量甚至提前不知道队列中将要添加的所有Item的数量。

生产者或者消费者向队列中放或者从队列中提取item的时候可能会花费额外的事件。队列充当了生产者和消费者通信的桥梁。

{: .note }
尽管为了线程安全队列通常用在单线程程序中，对于asyncio我们不需要担心线程安全。

同步版本的程序看上取相当弱，一组阻塞的生产者序列式的给队列添加item,每次一个生产生。所有生产者完成之后，队列才能继续被处理，一次一个消费者进行处理。这种设计方法有相当大的延迟。items需要存在在队列中而不是立刻进行处理。

对于异步的版本，如下代码，问题是需要有个信号告诉消费者产品完成了，否则`await q.get()`将会永远挂起，因为队列处于完全被处理装袋，但是消费者不知道产品已经完成了。

```python
#!/usr/bin/env python3
# asyncq.py

import asyncio
import itertools as it
import os
import random
import time

async def makeitem(size: int = 5) -> str:
    return os.urandom(size).hex()

async def randsleep(caller=None) -> None:
    i = random.randint(0, 10)
    if caller:
        print(f"{caller} sleeping for {i} seconds.")
    await asyncio.sleep(i)

async def produce(name: int, q: asyncio.Queue) -> None:
    n = random.randint(0, 10)
    for _ in it.repeat(None, n):  # Synchronous loop for each single producer
        await randsleep(caller=f"Producer {name}")
        i = await makeitem()
        t = time.perf_counter()
        await q.put((i, t))
        print(f"Producer {name} added <{i}> to queue.")

async def consume(name: int, q: asyncio.Queue) -> None:
    while True:
        await randsleep(caller=f"Consumer {name}")
        i, t = await q.get()
        now = time.perf_counter()
        print(f"Consumer {name} got element <{i}>"
              f" in {now-t:0.5f} seconds.")
        q.task_done()

async def main(nprod: int, ncon: int):
    q = asyncio.Queue()
    producers = [asyncio.create_task(produce(n, q)) for n in range(nprod)]
    consumers = [asyncio.create_task(consume(n, q)) for n in range(ncon)]
    await asyncio.gather(*producers)
    await q.join()  # Implicitly awaits consumers, too
    for c in consumers:
        c.cancel()

if __name__ == "__main__":
    import argparse
    random.seed(444)
    parser = argparse.ArgumentParser()
    parser.add_argument("-p", "--nprod", type=int, default=5)
    parser.add_argument("-c", "--ncon", type=int, default=10)
    ns = parser.parse_args()
    start = time.perf_counter()
    asyncio.run(main(**ns.__dict__))
    elapsed = time.perf_counter() - start
    print(f"Program completed in {elapsed:0.5f} seconds.")
```

The first few coroutines are helper functions that return a random string, a fractional-second performance counter, and a random integer. A producer puts anywhere from 1 to 5 items into the queue. Each item is a tuple of (i, t) where i is a random string and t is the time at which the producer attempts to put the tuple into the queue.

When a consumer pulls an item out, it simply calculates the elapsed time that the item sat in the queue using the timestamp that the item was put in with.

Keep in mind that asyncio.sleep() is used to mimic some other, more complex coroutine that would eat up time and block all other execution if it were a regular blocking function.

Here is a test run with two producers and five consumers:

```
$ python3 asyncq.py -p 2 -c 5
Producer 0 sleeping for 3 seconds.
Producer 1 sleeping for 3 seconds.
Consumer 0 sleeping for 4 seconds.
Consumer 1 sleeping for 3 seconds.
Consumer 2 sleeping for 3 seconds.
Consumer 3 sleeping for 5 seconds.
Consumer 4 sleeping for 4 seconds.
Producer 0 added <377b1e8f82> to queue.
Producer 0 sleeping for 5 seconds.
Producer 1 added <413b8802f8> to queue.
Consumer 1 got element <377b1e8f82> in 0.00013 seconds.
Consumer 1 sleeping for 3 seconds.
Consumer 2 got element <413b8802f8> in 0.00009 seconds.
Consumer 2 sleeping for 4 seconds.
Producer 0 added <06c055b3ab> to queue.
Producer 0 sleeping for 1 seconds.
Consumer 0 got element <06c055b3ab> in 0.00021 seconds.
Consumer 0 sleeping for 4 seconds.
Producer 0 added <17a8613276> to queue.
Consumer 4 got element <17a8613276> in 0.00022 seconds.
Consumer 4 sleeping for 5 seconds.
Program completed in 9.00954 seconds.
```

In this case, the items process in fractions of a second. A delay can be due to two reasons:

Standard, largely unavoidable overhead
Situations where all consumers are sleeping when an item appears in the queue
With regards to the second reason, luckily, it is perfectly normal to scale to hundreds or thousands of consumers. You should have no problem with python3 asyncq.py -p 5 -c 100. The point here is that, theoretically, you could have different users on different systems controlling the management of producers and consumers, with the queue serving as the central throughput.

So far, you’ve been thrown right into the fire and seen three related examples of asyncio calling coroutines defined with async and await. If you’re not completely following or just want to get deeper into the mechanics of how modern coroutines came to be in Python, you’ll start from square one with the next section.




