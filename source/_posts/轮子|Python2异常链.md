---
title: 轮子|Python2异常链
tags:
  - Python
  - 轮子
  - 原创
reward: true
date: 2017-12-03 16:35:06
---

介绍一个自己造的轮子，Python2异常链。

<!--more-->

# 需求

>习惯了用java撸码，虽说胶水代码多，但能比较好的用代码表达思路；而Python则简洁到了简陋的地步——各种鸡肋的语法糖，各种不完善的机制。比如错误处理。

**Python2没有异常链，让问题排查变得非常困难**：

```python
# coding=utf-8
import sys


class UnexpectedError(StandardError):
    pass


def divide(division, divided):
    if division == 0:
        raise ValueError("illegal input: %s, %s" % (division, divided))
    ans = division / divided
    return ans


a = 0
b = 0
try:
    print divide(a, b)
except ValueError as e:
    # m_counter.inc("err", 1)
    raise UnexpectedError("illegal input: %s, %s" % (a, b))
except ZeroDivisionError as e:
    # m_counter.inc("err", 1)
    raise UnexpectedError("divide by zero")
except StandardError as e:
    # m_counter.inc("err", 1)
    raise UnexpectedError("other error...")
```

打印异常如下：

```
Traceback (most recent call last):
  File "/Users/monkeysayhi/PycharmProjects/Wheel/utils/tmp/tmp.py", line 22, in <module>
    raise UnexpectedError("illegal input: %s, %s" % (a, b))
__main__.UnexpectedError: illegal input: 0, 0
```

不考虑代码风格，是标准的Python2异常处理方式：**分别捕获异常，再统一成一个异常，只有msg不同，重新抛出**。这种写法又丑又冗余，顶多可以改成这样：

```python
try:
    print divide(a, b)
except StandardError as e:
    # m_counter.inc("err", 1)
    raise UnexpectedError(e.message)
```

即便如此，也无法解决一个最严重的问题：**明明是11行抛出的异常，但打印出来的异常栈却只能追踪到22行重新抛出异常的`raise`语句**。重点在于没有记录cause，使我们追踪到22行之后，不知道为什么会抛出cause，也就无法定位到实际发生问题的代码。

## 异常链

最理想的方式，还是在异常栈中打印异常链：

```python
try:
    print divide(a, b)
except StandardError as cause:
    # m_counter.inc("err", 1)
    raise UnexpectedError("some msg", cause)
```

就像Java的异常栈，区分“要抛出的异常UnexpectedError和引起该异常的原因cause”：

```
java.lang.RuntimeException: level 2 exception
	at com.msh.demo.exceptionStack.Test.fun2(Test.java:17)
	at com.msh.demo.exceptionStack.Test.main(Test.java:24)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:147)
Caused by: java.io.IOException: level 1 exception
	at com.msh.demo.exceptionStack.Test.fun1(Test.java:10)
	at com.msh.demo.exceptionStack.Test.fun2(Test.java:15)
	... 6 more
```

上述异常栈表示，RuntimeException由IOException导致；1行与9行下是各异常的调用路径trace。不熟悉Java异常栈的可参考[你真的会阅读Java的异常信息吗？](/2017/10/02/你真的会阅读Java的异常信息吗？/)。

# 轮子

## 调研让我们拒绝重复造轮子

Python3已经支持了异常链，通过from关键字即可记录cause。

Python2 future包提供的所谓异常链`raise_from`我是完全没明白到哪里打印了cause:

```python
from future.utils import raise_from


class DatabaseError(Exception):
    pass


class FileDatabase:
    def __init__(self, filename):
        try:
            self.file = open(filename)
        except IOError as exc:
            raise_from(DatabaseError('failed to open'), exc)


# test
fd = FileDatabase('non_existent_file.txt')
```

那么，11行抛出的IOError呢？？？似乎仅仅多了一句无效信息（future包里的`raise e`）。

```
Traceback (most recent call last):
  File "/Users/mobkeysayhi/PycharmProjects/Wheel/utils/tmp/tmp.py", line 17, in <module>
    fd = FileDatabase('non_existent_file.txt')
  File "/Users/mobkeysayhi/PycharmProjects/Wheel/utils/tmp/tmp.py", line 13, in __init__
    raise_from(DatabaseError('failed to open'), exc)
  File "/Library/Python/2.7/site-packages/future/utils/__init__.py", line 454, in raise_from
    raise e
__main__.DatabaseError: failed to open
```

有知道正确姿势的求点破。

## 没找到重复轮子真是极好的

非常简单：

```python
import traceback


class TracedError(BaseException):
    def __init__(self, msg="", cause=None):
        trace_msg = msg
        if cause is not None:
            _spfile = SimpleFile()
            traceback.print_exc(file=_spfile)
            _cause_tm = _spfile.read()
            trace_msg += "\n" \
                         + "\nCaused by:\n\n" \
                         + _cause_tm
        super(TracedError, self).__init__(trace_msg)


class ErrorWrapper(TracedError):
    def __init__(self, cause):
        super(ErrorWrapper, self).__init__("Just wrapping cause", cause)


class SimpleFile(object):
    def __init__(self, ):
        super(SimpleFile, self).__init__()
        self.buffer = ""

    def write(self, str):
        self.buffer += str

    def read(self):
        return self.buffer
```

**目前只支持单线程模型**，github上有doc和测试用例，[戳我戳我](https://github.com/monkeysayhi/utils/blob/master/common/traced_error.py)。

一个测试输出如下：

```
Traceback (most recent call last):
  File "/Users/monkeysayhi/PycharmProjects/Wheel/utils/exception_chain/traced_error.py", line 68, in <module>
    __test()
  File "/Users/monkeysayhi/PycharmProjects/Wheel/utils/exception_chain/traced_error.py", line 64, in __test
    raise MyError("test MyError", e)
__main__.MyError: test MyError

Caused by:

Traceback (most recent call last):
  File "/Users/monkeysayhi/PycharmProjects/Wheel/utils/exception_chain/traced_error.py", line 62, in __test
    zero_division()
  File "/Users/monkeysayhi/PycharmProjects/Wheel/utils/exception_chain/traced_error.py", line 58, in zero_division
    a = 1 / 0
ZeroDivisionError: integer division or modulo by zero
```

另外，为方便处理后重新抛出某些异常，还提供了ErrorWrapper，仅接收一个cause作为参数。用法如下：

```python
for pid in pids:
    # process might have died before getting to this line
    # so wrap to avoid OSError: no such process
    try:
        os.kill(pid, signal.SIGKILL)
    except OSError as os_e:
        if os.path.isdir("/proc/%d" % int(pid)):
            logging.warn("Timeout but fail to kill process, still exist: %d, " % int(pid))
            raise ErrorWrapper(os_e)
        logging.debug("Timeout but no need to kill process, already no such process: %d" % int(pid))
```

---

>参考：
>
>* [Cheat Sheet: Writing Python 2-3 compatible code](http://python-future.org/compatible_idioms.html)
>
