# C++ Concurrency In Action

## Why concurrency

- Separation of Concerns
- Performance
  - task parallelism
  - data parallelism

## Managing threads

```cpp
void update_data_for_widget(widget_id w,widget_data& data);
void oops_again(widget_id w) {
  widget_data data;
  # 启动线程 传递参数
  std::thread t(update_data_for_widget,w,data);
  display_status();
  # 等待线程完成
  t.join();
  process_widget_data(data);
}
```

> https://en.cppreference.com/w/cpp/thread/thread/thread

- 默认构造的 `std::thread` 啥也不干，本身的构造和析构也可能被优化掉
-  构造时传递了`Callable` 参数的 `std::thread` 对象会在另一个线程执行 `Callable` 参数
- 构造时传递的参数会拷贝到另一个线程中执行
  - 需要考虑性能和生命周期问题
- `std::thread` 是 move only 类型
- `std::thread` 对象析构时如果为 joinable 状态会调用 `std::terminal`()
  - 默认构造 `joinable() == false`
  - 带 callable 参数构造 `joinable() == true`
  - `join()` :  `joinable()` true -> false
  - `detach()`  :  `joinable()` true -> false
- 一般使用 `join()` 等待线程完成
- 静态成员函数 `hardware_concurrency()` 返回当前环境支持的并行线程个数，可以作为线程个数选择的参考
- `get_id()` 成员函数可以返回一个 `std::thread::id` 对象作为线程的标识
- c++ 20 提供了 `std::jthread` 作为自动 `join` 的线程类型，更推荐使用，在 c++20 之前可以用 `RAII` 包装类达到类似的效果

> https://en.cppreference.com/w/cpp/thread/jthread

## Sharing data between threads

> https://en.cppreference.com/w/cpp/thread

- 线程间数据共享要避免数据竞争(未定义行为)
- 可以使用互斥量和锁来保护数据
  - `std::mutex`
  - `std::shared_mutex` (c++17) 允许一写多读
  - `std::lock_gard` RAII 锁
  - `std::scope_lock` (c++17)锁多个互斥量可以避免死锁
  - `std::unique_lock` 可以灵活选择什么时候锁互斥量
- 可以使用 Double-Check 技巧保护初始化需要线程安全的数据

## Synchronizing concurrent operations

- 多次同步 `std::condition_variable`
  - `std::condition_variable` `wait()` 可能会被假唤醒
- 一次性同步 `std::future`
  - `std::promise`
  - `std:::packaged_task`
  - `std::async`
- `std::stop_token` 可用于取消线程 (c++20)
- `std::latch` 一次性屏障(c++20)
- `std::barrier` 多次使用屏障(c++20)

## The C++ memory model and operations on atomic types

> TBD

## Designing lock-based concurrent data structures

- 一个线程操作的造成的数据结构中间状态不应该被另一个线程看到
- 避免数据结构之间信息的隐含依赖造成的数据竞争，接口应该提供完整的操作(比如stack 的 empty 和 top 之间有隐含依赖)
- 注意数据结构的异常安全 (数据结构状态不变量不应该被异常破坏)
- 避免死锁 (比如在拿到锁的情况去执行外部提供的函数很危险，外部提供的函数可能会继续调用这个对象的接口造成死锁)
- 样例
  - stack
  - queue
  - lookup table
  - list

## Designing lock-free concurrent data structures

- 使用 `std::memory_order_seq_cst` 做原型设计
- 合理释放资源
  - 删除队列
  - hazard pointer
  - 引用计数
- 小心 ABA 问题
- 如果有忙等待可以考虑加 helper 代码
- 样例
  - stack
  - queue

  ## Designing concurrent code

- 划分线程工作的方法
  - 提前根据数据块划分 (accumulate)
  - 运行中递归划分(sort)
  - 根据任务类执行划分(pipeline)
- 影响并行代码性能的因素
  - 处理器数量
  - 数据竞争和 cache ping-pong
    - 避免多线程修改同一数据
- 伪共享
  - 避免多线程修改相邻数据, 可以使用 `std::hardware_destructive_interference_size` (c++17)
- 数据局部性 (缓存友好)
- Oversubscription 和频繁线程切换
- 考虑异常安全
  - 用好 `std::future`
- 拓展性和 Amdahl's law
- 避免线程等待
  - 合理划分任务让每个线程都有事干
- 提高响应速度 (ui)
- 样例
  - 矩阵乘法(数据划分)
  - std::for_each
  - std::find
  - std::partial_sum
- algorithm 库一些算法可以使用 execution policy (c++17)
> https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag

## Advanced thread management

- 线程池
  - 可使用 std::future 等待任务完成
  - 使用 thread local 变量避免数据争用
  - work stealing
  - 中断线程
    - std::stop_token (c++20)
    - 自己实现中断检查函数 (before c++20)

  ## Testing and debugging multithreaded applications

  - 并发 bug
    - 阻塞
      - 死锁
      - 活锁
      - IO 或线程间长时间等待
  - 竞态条件
    - 数据竞争
    - 破坏不变量
    - 生命周期问题
- 定位并发 bug
  - code review
  - 增加测试
  - 提高代码可测试性
    - 区分线程内工作和线程间工作
  - 多线程测试代码需要同步触发
    - `std::latch` (c++20)
    -  `std::futrue` 或者 your latch
  - benchmark
    - 如果你是为了性能选择了并发，证明你的选择


