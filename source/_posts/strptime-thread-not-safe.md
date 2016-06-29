---
title: 说出来你们可能不信，strptime竟然是非线程安全的
date: 2016-03-12 22:12:22
category: python
tags:
- strptime
- 线程
---

### 事情是这样的

今天在做实验的时候，用到了这样的一个函数：

{% codeblock lang:python %}
from datetime import datetime
import threading


def filter_updatetime(duration):
    def _filter_updatetime(news):
        if 'updateTime' in news:
            updateTime = datetime.strptime(news['updateTime'], '%Y/%m/%d %X')
            now  = datetime.now()
            return abs((now-updateTime).total_seconds())<duration
        else:
            logging.debug('%s has no updateTime'%news['title'])
            return False
    return _filter_updatetime
{% endcodeblock %}

是在多线程里用的，但执行的时候偶尔会遇到：
```
AttributeError: 'module' object has no attribute '_strptime' 
```
这样的错误，我一脸日了狗的表情，因为我曾多次遇到类似的这个错误，多是因为我比较粗心，忘记了```from datetime import datetime```而是直接用了```import datetime```，然而这次我并没有犯这种低级的错误，然后我发现这次说找不到的属性是```_strptime```而不是```strptime```，我心说大事不好！

我一开始没有想到是线程的问题，但查的时候发现为数不多的结果里都说是在多线程的时候才会出现这个问题，就又加了多线程来查，最后发现是因为线程安全的问题。

### 问题重现

- **Windows**: Python 2.7.8 |Anaconda 2.1.0 (64-bit) | (default, Jul 2 2014, 15:12:11) [MSC v.1500 64 bit (AMD64)]
- **OS X**: Python 2.7.6 (default, Sep  9 2014, 15:04:36) 

都会出现这个问题，使用下面简单的代码就可以重现：

{% codeblock lang:python issue7980.py %}
from datetime import datetime
import threading


def run():
    for _ in range(100):
        threading.Thread(target = datetime.strptime, args=('2016/03/12 10:59:07', '%Y/%m/%d %X')).start()


if __name__ == '__main__':
    run()
{% endcodeblock %}

错误信息（OS X）：

{% codeblock lang:shell %}
Exception in thread Thread-2:
Traceback (most recent call last):
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/threading.py", line 810, in __bootstrap_inner
    self.run()
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/threading.py", line 763, in run
    self.__target(*self.__args, **self.__kwargs)
AttributeError: _strptime
{% endcodeblock %}

### 问题出在

[issue7980](http://bugs.python.org/issue7980)中对这个问题是这么解释的：
> Thread safety: The use of strptime is thread safe, but with one important caveat.  The first use of strptime is not thread safe because the first use will import _strptime.  That import is not thread safe and may throw AttributeError or ImportError.  To avoid this issue, import _strptime explicitly before starting threads, or call strptime once before starting threads.

实际上strptime本身是线程安全的，但是需要注意的是，第一次用这个函数的时候不是线程安全的，因为第一次使用这个函数的时候，会import _strptime，这个导入的过程不是线程安全的，在这个过程可能会抛出异常或者导入错误。
如果想要避免这个问题，那就**在线程启动之前，调用一次这个函数**，使导入过程不发生在子线程中就好



