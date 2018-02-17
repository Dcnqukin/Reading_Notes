# std::thread详解
---
* std::thread在头文件<thread>中声明，使用命令空间std。

- ** std::thread构造 **

|default		  |thread() noexcept;
|:----------------|:----------------
|initialization   |template <class Fn, class... Args> explicit thread(Fn&& fn, Args&&... args);
|copy[deleted]    |thread (const thread&) = delete;
|move			  |thread (thread&& x) noexcept;

---

> 默认构造函数，创建一个空的thread执行对象；
> 初始化构造函数，创建一个thread对象，该thread对象可以被joinable,新产生的线程会调用fn函数，该函数的参数由args给出。
> 拷贝构造函数（被禁用）
> move构造函数，调用成功后，x不代表任何thread执行对象。
> *PS：可被joinable的thread对象必须在他们销毁之前被主线程join()或者将其设置为detached（通过detach()）

----

* move赋值操作

move		  |thread& operator= (thread&& rhs) noexcept;
:-------------|:------------
copy[deleted] |thread& operator= (const thread&) = delete;

---

> move赋值操作，如果当前对象不可被joinable，则需要传递一个右值引用(rhs)给move赋值操作；如果当前对象可被joinable，则terminate()报错。
> 拷贝赋值操作被禁用，thread对象不可被拷贝。

---
* 其他成员函数

> get_id()，获取线程ID。
> joinable()，判断线程是否可被join/detach。
> join()，join线程，实际上是等待线程完成之后再执行销毁
> detach()， detach线程，不等待线程结束就执行其他操作（不一定是销毁）
> swap()；
> native_handle，返回native handle；
> hardware_concurrency[static]， 检测硬件并发特性。

---