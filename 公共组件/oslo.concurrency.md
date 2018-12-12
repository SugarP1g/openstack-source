# oslo.concurrency

作者：Bill\_Xiang\_

原文：https://blog.csdn.net/Bill_Xiang_/article/details/78624752

oslo.concurrency 是一个为 OpenStack 其他项目提供用于管理线程的工具库，这样， OpenStack 其他项目可以直接调用 oslo.concurrency 库利用其锁机制安全的运行多线程和多进程应用，也可以运行外部进程。本文总结了 oslo.concurrency 中常用的工具类或方法及其对应的使用方法。

## 1. lockutils

lockutils 模块封装了 oslo 库的锁机制，其中，定义了读写锁、信号量以及同步装饰器方法等。本节分别介绍这些类和方法的实现与使用。

### 1.1 锁机制

lockutils 中的锁机制实质上是直接使用了 fasteners 的实现，所以本节直接介绍 fasteners 库中的读写锁和共享内存，即 ReaderWriteerLock 和 InterProcessLock 类。

#### 1.1.1 ReaderWriterLock 类

ReaderWriterLock 类实现了一个读写锁，读写锁实际是一种特殊的自旋锁，它把对共享资源的访问者划分成读者和写者，读者只对共享资源进行读访问，写者则需要对共享资源进行写操作。

fasteners 通过 ReadWriterLock 类实现了读锁和写锁，其在该类中定义了 READER 和 WRITER 标识分别表示申请的锁是读锁还是写锁。使用该类可以获得多个读锁，但只能存在一个写锁。在目前的版本中，该类不能实现从读写升级到写锁，且写锁在加锁时不能获取读锁；而以后可能会对这些问题进行优化。该类的主要方法如下：

- read_lock()：为当前线程申请一个读锁，其只有在没有为其他线程分配写锁时才能获取成功；如果另一个线程以及获取了写锁，调用该方法会返回 RuntimeError 异常。
- write_lock()：为当前线程申请一个写锁，其只有在没有为其他线程分配读锁或写锁时才能获取成功；一旦申请写锁成功，将会阻塞申请读锁的线程。
- is_reader()：判断当前线程是否申请了读锁。
- is_writer()：判断当前线程是否申请了写锁，或已申请但尚未获得写锁。
- owner()：判断当前线程是否申请了锁；如果获得了锁是读锁还是写锁。

ReaderWriterLock 类中包含 \_writer 属性表示占有写锁的线程， \_readers 属性表示占有读锁的线程集合， \_pending\_writers 属性表示正在等待分配写锁的线程的队列， \_current_thread 属性表示当前线程， \_cond 属性表示 threading.Condition 对象。上述的这些方法就是通过操作这几个属性实现读写锁的。

#### 1.2.1 InterProcessLock 类

InterProcessLock 类是一个在 POSIX 系统上工作的进程间锁机制的实现类。该类会通过当前操作系统确定其内部的实现机制，通过该类可以实现多进程应用中共享内存、进程间通信与同步以及加锁等操作。其主要实现了 tryLock() 方法实现加锁， unlock() 方法实现解锁，对于其具体实现，由于操作系统的不同其实现方法也不同，本人能力有限不再进行深入解析，有兴趣的同学可以自行研究。

### 1.2 信号量

lockutils 中也实现了信号量，准确来说是一个信号量垃圾收集的容器。这个集合在内部使用一个字典，这样当一个信号量不再被任何线程使用时，它将被垃圾收集器自动从这个容器中移除。其提供了一个 get(name) 方法，可以通过名称获取一个信号量。在具体使用时，可以直接 lockutils 中的 internal_lock(name, semaphores=None) 获取这个信号量容器。

### 1.3 同步装饰器

lockutils 中定义了两个同步装饰器方法：

- synchronized(name, lock_file_prefix=None, external=False, lock_path=None, semaphores=None, delay=0.01)
- synchronized_with_prefix(lock_file_prefix)

前者直接使用 @synchronized(name) 对装饰的方法加同步锁；而后者可以通过重新定义使用 @synchronized(name) 对装饰的方法加一个带有前缀的同步锁。

### 1.4 外部锁

lockutils 中也定义了两个方法分别用来获取和删除外部锁：

- external_lock(name, lock_file_prefix=None, lock_path=None)
- remove_external_lock_file(name, lock_file_prefix=None, lock_path=None, semaphores=None) 。

其需要指定锁文件的前缀、锁文件路径以及锁的名称，通过这些属性， lockutils 可以通过 \_get\_lock\_path(name, lock_file_prefix, lock_path=None) 方法获取锁的位置，并根据锁文件创建和删除一个外部锁。

## 2 processutils

processutils 模块定义了一系列系统级的工具类和辅助函数。本小节将介绍 processutils 库的相关类或方法。

### 2.1 进程或线程异常类

processutils 模块中定义了多个进程或线程异常类，如参数不合法异常 InvalidArgumentError 、不知名参数异常 UnknownArgumentError 、进程执行异常 ProcessExecutionError 、日志记录异常 LogErrors 等。

### 2.2 资源限制类

ProcessLimits 类封装了一个进程对资源的限制，这些限制主要包括以下几个方面：

- address_space：进程地址空间限制。
- core_file_size：core 文件大小限制。
- cpu_time：CPU 执行当前进程时间限制。
- data_size：数据大小限制。
- file_size：文件大小限制。
- memory_locked：加锁的内存大小限制。
- number_files：打开的文件最大数量限制。
- number_processes：进程的最大数量限制。
- resident_set_size：最大驻留集（RSS）大小限制。
- stack_size：栈大小限制。

### 2.3 进程执行方法

processutils 模块中定义了执行进程的方法等，主要方法包括以下几个：

- execute() 方法：该方法通过启动一个子进程提取并执行一个命令。主要参数有：
  - cmd：待执行的命令；
  - cwd：设置当前目录；
  - process_input：发送到打开的进程；
  - env_variables：进程环境变量；
  - check_exit_code：代表退出进程的 int、bool 或 list 值，默认为0，只有产生异常才会设置为其他值；
  - delay_on_retry：重试延迟时间，如果设置为 True ，表示马上进行重试操作；
  - attempts：cmd 重试次数；
  - run_as_root：该值如果设置为 True ，则为 cmd 命令加上 root_helper 指定的前缀；
  - root_helper：为命令指定的前缀；
  - shell：表示是否使用 shell 执行这个命令；
  - loglevel：执行命令记录日志的等级；
  - log_errors：监听错误日志，是一个 LogErrors 对象；
  - binary该值：如果为 True ，则返回 Unicode 编码的 stdout 后 stderr ；
  - prlimit：表示一个 ProcessLimits 对象，用于限制执行该 cmd 的命令的资源用量。
- trycmd() 方法：execute() 的一个装饰器，使用这个装饰器可以更加容易的处理错误和异常。返回一个包含命令输出 strdout 或 stderr 字符串的元组。如果err不为空，则表示执行命令出现异常或错误。
- ssh_execute()：通过 ssh 执行命令。
- get_worker_count()：获取默认的 worker 数量，返回 CPU 的数量；如果无法确定则返回1.
## 3 watchdog

watchdog 模块实现了一个看门狗程序，定义了一个 watch() 方法。如果操作执行时间超过阈值，则记录日志；此时可能发生了死锁或是一个耗时操作。其包含四个参数：

- logger：一个记录日志的对象。
- action：描述将执行的操作。
- level：表示记录日志的等级，默认为 debug 。
- after：发送消息之前的持续时间，默认为5s。

## 4 使用方法

上文介绍了 oslo.concurrency 的各个模块的实现，接下来将详细介绍如何使用这些模块更好的管理 OpenStack 项目的线程或进程。

```python
from oslo_concurrency import lockutils
 
@synchronized('mylock')
def foo(self, *args):
    ...
 
@synchronized('mylock')
def bar(self, *args):
    ...
```

为一个方法添加 @syschronized 装饰器，可以保证统一时刻只有一个线程执行这个方法；但是，同时可以有两个方法共享这个锁，此时统一时刻要么只能执行 foo 方法，要么只能执行 bar 方法。

```python
(in nova/utils.py)
from oslo_concurrency import lockutils
 
synchronized = lockutils.synchronized_with_prefix('nova-')
 
 
(in nova/foo.py)
from nova import utils
 
@utils.synchronized('mylock')
def bar(self, *args):
    ...
```

如果需要设置一个带有前缀的同步锁，可以使用如上的方式进行设置。

```python

        FORMAT = '%(asctime)-15s %(message)s'
        logging.basicConfig(format=FORMAT)
        LOG = logging.getLogger('mylogger')
 
        with watchdog.watch(LOG, "subprocess call", logging.ERROR):
            subprocess.call("sleep 10", shell=True)
            print "done"
```

如果设置一个看门狗，则可以使用 with 语法调用 watchdog.watch() 方法。