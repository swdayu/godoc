Go编程语言文档翻译和整理

1. `语言规范 <go-language-spec.rst>`_
2. `类型系统 <go-type-system.rst>`_
3. `并发编程 <go-concurrency.rst>`_

参考链接

* `The Go Programming Language Specification <https://go.dev/ref/spec>`_
* `The Go Standard Library <https://pkg.go.dev/std>`_
* `The Go Package Sources <https://go.dev/src/>`_
* `The Go Doc Comments <https://go.dev/doc/comment>`_
* `The Go Command Documentation <https://go.dev/doc/cmd>`_
* `The Go Memory Model <https://go.dev/ref/mem>`_
* `Effective Go <https://go.dev/doc/effective_go>`_
* `Frequently Asked Questions (FAQ) <https://go.dev/doc/faq>`_
* `Language Design in the Service of Software Enginerring <https://go.dev/talks/2012/splash.article>`_
* `Go Data Structures: Interfaces <https://research.swtch.com/interfaces>`_
* `Concurrency in Go <https://go.dev/learn/#featured-books>`_

命名规范

* 包名应该简短清晰，用小写的单个词表示，不要用下划线和混合大小写
* 因为每个使用包的人都要输写这个包名，简洁性很重要
* 不要怕名字冲突，包名仅仅是默认名称，导入包时可以选择一个本地使用的别名
* 包名应该是包路径的基本名称，例如 src 目录下的包 "hash/crc32" 包名是 crc32
* 有了包名的隔离，包中的名字也就可以很简洁，例如 bufio.Reader io.Reader
* 好的包结构可以帮助取得好名字，例如 ring.Ring 是包唯一的类型，其构造函数可取名 ring.New
* 长名字不会自动增加可读性，有用的文档注释比特别长的名字来得更有价值
* 例如 once.Do(setup) 读起来很好，once.DoOrWaitUntilDone(setup) 并不更具可读性
* Getter 函数直接使用对象名，例如 Owner（导出函数名）对应未导出的 owner
* 接口名使用 er 等后缀，例如 Reader Writer Formatter CloseNotifier

同步机制

* 不大于机器字长的并发读写，保证读到的值是一致性的，不会是这个协程写一半那个协程写一半的坏值

  * 即读到的其他协程写的值一定是协程写之前的某个值或当前写的值
  * 但不保证协程写操作之前的代码执行完毕（可能乱序执行）
  * 也不保证协程写的值会被当前协程观察到（缺乏同步事件）

* 程序初始化

  * 程序初始化在单个协程中执行，但初始化代码可能创建其他并发协程
  * 同步点：开始执行包p的初始化代码之前 => p导入的包初始化代码都执行完毕
  * 同步点：main函数执行之前 => 所有初始化代码都执行完毕

* 协程启动

  * 同步点：新协程执行之前 => 以下代码都执行完毕
  * go语句之前的语句都执行完毕（保证不会乱序且结果已成功反应）
  * go语句声明的新协程创建成功

* 通道

  * 同步点：读通道返回之前 => 对应的写通道之前的代码以及写操作本身执行完毕
  * 同步点：读到通道关闭前 => 对应的通道关闭之前的代码以及关闭本身执行完毕
  * 同步点：从写通道一方看，写非缓冲通道完毕后，对应的读通道以及它之前的代码执行完毕
  * 同步点：对于容量为C的缓冲通道，k+C次的写通道完毕后，对应的k次读通道执行完毕

* 原子值

  * 包 sync/atomic 提供了原子操作功能，一个原子操作观察到另一协程原子操作写的值时：
  * 该写操作之前的语句以及当前写操作都执行完毕

* 互斥锁

  * 同步点：Mutex Lock 函数返回之前 => 另一协程的 Unlock 本身以及之前的语句执行完毕
  * 同步点：RWMutex Lock 返回之前还需要 => 其他所有协程的 RUnlock 执行完毕
  * 同步点：RWMutex RLock 返回之前 => 另一协程的 Unlock 执行完毕

* sync.Cond，其他的同步机制包括条件变量，以及以下一些机制：
* sync.Map
* sync.Once
* sync.Pool
* sync.WaitGroup
