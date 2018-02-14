#深入应用C++11:代码优化与工程级应用
----
##篇一：C++11改进我们的程序
---
###章五：C++11多线程开发
---
+ 线程
	- 线程的创建
		> 用std::thread创建线程，只需要提供线程函数，或者函数对象即可，并且可以同时指定线程函数的参数。
		
	- 线程的基本用法
		1. 获取当前信息
		> 线程可以获取当前线程的ID，还可以获取CPU核心数量。( get_id() / std::thread::hardware_concurrency() )
		```
		#include <thread>
		#include <iostream>
		void func(){}
		int main()
		{
			std::thread t(func);
			std::cout<< t.get_id() << std::endl;
			std::cout<< std::thread::hardware_concurrency() << std::endl;
			return 0;
		}
		```
		2.线程休眠
		> 可以使线程休眠一定时间(std::this_thread::sleep_for() )
+ 互斥量
	- C++11提供了四种语义的互斥量(mutex);互斥量是一种同步原语，是一种线程同步的手段，用于保护多线程同时访问的共享数据
	> std::mutex: 独占的互斥量，不能递归使用；
	> std::timed_mutex: 带超时的独占互斥量，不能递归使用；
	> std::recursive_mutex: 递归互斥量
	> std::recursive_timed_mutex: 带超时的递归互斥量。
	> 互斥量的基本接口一般用法是，通过lock()方法来阻塞线程，直到获得互斥量的所有权为止。
	> 在线程获得互斥量并且完成任务后，必须使用unlock()来解除对互斥量的占用，lock()和unlock()必须成对出现
	> 即PV操作。通过try_lock()尝试锁定互斥量，如果成功则返回true，如果失败则返回false，它是非阻塞的。
		- 独占互斥量std::mutex
		- 递归互斥量std::recursive_mutex
		> 递归锁允许同一线程多次获得该互斥锁，可以用于解决同一线程需要多次获取互斥量时的死锁问题。
		- 带超时的互斥量std::timed_mutex和std::recursive_timed_mutex、
		> 超时锁，主要是在获取锁时增加超时等待功能。增加了两个接口：try_lock_for和try_lock_until。
		> 这两个接口是用于设置获取互斥量的超时时间，使用时可以用一个while循环去不断地获取互斥量。。
+ 条件变量
	- 条件变量是C++11提供的另外一种用于等待的同步机制，它能阻塞一个或多个线程，直到收到另外一个线程发出的通知或超时。
	> 才会唤醒当前阻塞的线程。条件便利需要和互斥量配合起来用。
	- C++11提供了两种条件变量：
	> condition_variable，配合std::unique_lock<std::mutex>进行wait操作；
	> condition_variable_any，和任意带有lock、unlock语义的mutex搭配使用，比较灵活，但效率较低。
	- 条件变量的使用过程：
	> 1）拥有条件变量的线程获取互斥量；
	> 2）循环检查某个条件，如果条件不满足，则阻塞直到条件满足；如果条件满足，则向下执行；
	> 3）某个线程满足条件执行完之后调用notify_one或notify_all唤醒一个或所有的等待线程
	- **同步队列的实现**
	```
#include <mutex>
#include <thread>
#include <list>
#include <condition_variable>

template<typename T>
class SyncQueue
{
    bool IsFull() const
    {
        return m_queue.size() == m_maxSize;
    }
    bool IsEmpty() const
    {
        return m_queue.empty();
    }
public:
    SyncQueue(int maxSize):m_maxSize(maxSize)
    {
    }
    void Put(const T& x)
    {
        std::lock_guard< std::mutex> locker(m_mutex);
        while(IsFull())
        {
            cout << "缓冲区满，需等待" << endl;
            m_notFull.wait(m_mutex);
        }
		//m_notFull.wait(locker, [this]{return !IsFull();}); // Lambda表达式。
        m_queue.push_back(x);
        m_notEmpty.notify_one();
    }
    void Take(T& x)
    {
        std::lock_guard<std::mutex> locker(m_mutex);
        while(IsEmpty())
        {
            cout << "缓冲区空，需等待" << endl;
            m_notEmpty.wait(m_mutex);
        }
        x = m_queue.front();
        m_queue.pop_front();
        m_notFull.notify_one();
    }
    bool Empty()
    {
        std::lock_guard<std::mutex> locker(m_mutex);
        return m_queue.empty();
    }
    bool Full()
    {
        std::lock_guard<std::mutex> locker(m_mutex);
        return m_queue.size() == m_maxSize;
    }
    size_t Size()
    {
        std::lock_guard<std::mutex> locker(m_mutex);
        return m_queue.size();
    }
    int Count()
    {
        return m_queue.size();
    }
private:
    std::list<T> m_queue;
    std::mutex m_mutex;
    std::condition_variable_any m_notEmpty;
    std::condition_variable_any m_notFull;
    int m_maxSize;
};
	```
	> 将上面的代码中std::lock_guard改写成std::unique_lock，把std::condition_variable_any改写为std::condition_variable,
	> 并且用等待一个判断式的方法来实现一个简单的线程池。
	```
#include <thread>
#include <condition_variable>
#include <mutex>
#include <list>
#include <iostream>
using namespace std;
template <typename T>
class SimpleSyncQueue
{
public:
    SimpleSyncQueue()
    {
    }
    void Put(const T& x)
    {
        std::lock_guard<std::mutex> locker(m_mutex);
        m_queue.push_back(x);
        m_notEmpty.notify_one();
    }
    void Take(T& x)
    {
        std::unique_lock<std::mutex> locker(m_mutex);
        m_notEmpty.wait(locker, [this]{return !m_queue.empty(); });
        x = m_queue.front();
        m_queue.pop_front();
    }
    bool Empty()
    {
        std::lock_guard<std::mutex> locker(m_mutex);
        return m_queue.empty();
    }
    size_t Size()
    {
        std::lock_guard<std::mutex> locker(m_mutex);
        return m_queue.size();
    }
private:
    std::list<T> m_queue;
    std::mutex m_mutex;
    std::condition_variable m_notEmpty;
};

	```
+ 原子变量
	- C++11提供了一个原子类型std::atomic<T>，可以使用任意类型作为模板参数，C++11内置了整型的原子变量。
	> 有操作系统基础就能够理解原子量是什么了。
+ call_once/once_flag的使用
	> call_once用于保证函数在多线程环境中只被调用一次。
	> 在使用std::call_once时，需要一个once_flag作为call_once的入参。
	```
#include <iostream>
#include <thread>
#include <mutex>
std::once_flag flag;
void do_once()
{
    std::call_once(flag, [](){ std::cout << "Called once" << std::endl; });
}
int main()
{
    std::thread t1(do_once);
    std::thread t2(do_once);
    std::thread t3(do_once);

    t1.join();
    t2.join();
    t3.join();
}

	```
+ 异步操作
	- C++11提供了异步操作相关的类，有std::future、std::promise和std::package_task。
	> std::future作为异步结果的传输通道，可以获取线程函数的返回值；
	> std::promise用于包装一个值，将数据和future绑定起来，方便线程赋值；
	> std::package_task用于包装一个可调用对象，将函数和future绑定起来，以便异步调用。
	
	- std::future包含可以查询future的状态（future_status）：
	> Deferred，异步操作尚未开始；<std::future::deferred>
	> Ready，异步操作已经完成；<std::future::ready>
	> Timeout，异步操作超时。<std::future::timeout>
	> 我们可以查询future的状态，通过它的内部状态可以知道异步任务的执行情况。
	```
	std::future_status status;
		do{
			status = future.wait_for(std::chrono::seconds(1));
			if (status == std::future_status::deferred){
				std::cout << "deferred\n";
			}else if(status == std::future_status::timeout){
				std::cout << "timeout\n";
			}else if(status == std::future_status::ready){
				std::cout << "ready!\n";
			}
		} while(status != std::future_status::ready);
	```
	> 获取future的结果的三种方式：get、wait、wait_for，其中get等待异步操作结束并返回结果，wait只是等待异步操作完成，没有返回值，wait_for是超时等待返回结果。
	
	- std::promise、std::packaged_task和std::future三者之间的关系
	> std::future提供了一个访问异步操作结果的机制，它和线程是一个级别的，属于低层次的对象。std::packaged_task和std::promise在std::future之上。
	> std::future是不可拷贝的，std::shared_future是可拷贝的，当需要将future放到容器之中则需要用shared_future。

+ 线程异步操作函数async
	> std::async比std::promise、std::packaged_task和std::thread更高一层，它可以用来直接创建异步的task，异步任务返回的结果也保存在future中。
	> 当需要获取异步任务的结果时，只需要调用future.get()方法即可，否则调用future.wait()方法。
	- std::async的原型为async(std::launch::async | std::launch::deferred, f, args...)，第一个参数是线程的创建策略，默认的策略是立即创建线程。
	> std::launch::async： 在调用async时就开始创建线程；
	> std::launch::deferred： 延迟加载方式创建线程。调用async时不创建线程，直到调用了future的get或者wait才创建线程。
	> 第二个参数是线程函数，第三个参数是线程函数的参数。
	- std::async是更高层次的异步操作，使得我们不用关注线程创建内部细节，就能方便地获取异步执行状态和结果，还可以指定线程创建策略：应该用std::async替代线程的创建，让它成为我们做异步操作的首选。
	