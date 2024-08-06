## C++

### 并发编程

#### 异步

##### std::future, std::promise, std::packaged_task

- `std::future`：是最基本的类型，它是在不同个线程中操作的关键，是对异步操作的封装。

  - `std::future::get()`：阻塞获取结果
    - 这个结果是怎么来的，它可以是`std::promise:set(value)`来的，也可以是`std::packaged_task`中的func返回结果
  - `std::future::wait(std::chrono::duration)`：等待结果，限时阻塞
  - `std::shared_future`：`std::future`只能调用一次，而`std::shared_future`可以调用多次，它存储了结果

- `std::promise`：设置结果，不关心结果怎么得来的

  - `std::promise::get_future()`：返回`std::future`对象，可以给其它线程去用

- `std::packaged_task`：构造时传递回调函数

  - 也有`get_future()`函数

  - 实例

    ```cpp
    std::packaged_task<int(int, int)> tsk(countdown); // set up packaged_task
    std::future<int> ret = tsk.get_future(); // get future
    // 仿函数对象，operator()可以像函数一样调用
    std::thread th(std::move(tsk), 5, 0); // spawn thread to count down from 5 to 0
    int value = ret.get(); // wait for the task to finish and get result
    ```

##### std::aysnc

高级封装，封装了`std::packaged_task`和`std::thread`，其返回`std::future`

```cpp
std::future<T> std::async(std::launch policy, F&& f, Args&&... args)
```

- `std::launch`
  - `std::launch::async`：会创建一个线程，执行
  - `std::launch::deferred`：表示调用线程入口函数将会被延迟到 std::future 的 wait() 或 get() 调用，当 wait() 或者 get() 没有被调用时，线程入口函数不会被调用(线程不会被创建)；

#### 条件变量

> [转 C++11 并发指南std::condition_variable详解 - 大老虎打老虎 - 博客园 (cnblogs.com)](https://www.cnblogs.com/wangshaowei/p/9593201.html)

作用：多个线程对一个竞争资源的按条件同步访问。

其需要一个`std::unique_lock`，对竞争资源保护。条件是：要等待A线程完成工作，之后该线程再`notify_one()`或`notify_all()`唤醒其它线程，实现同步。

```cpp
// 判断条件
bool predicate() {
    // ......
}

// A线程调用
void producer(int id) {
    std::unique_lock<std::mutex> lck(mtx);
    /* 对竞争资源进行操作 */
    cv.notify_one();
}

// B线程
void consumer() {
    std::unique_lock<std::mutex> lck(mtx);
    cv.wait(lck, predicate);
    /* 对竞争资源进行操作 */
}
```

- `cv.wait( lck, predicate )`
  *带唤醒后判断的*

  - lck：是锁，在阻塞前，会先`lck.unlock()`

  - predicate：解锁后，会根据predicate返回值判断，若为true则退出方法，false则继续阻塞。

    ```cpp
    // 等价于：
    while (!predicate())
        cv.wait(lck);
    ```

  - 当被唤醒，并且`predicate()`返回true，则：
    - 先给lck上锁
    - 再退出wait

