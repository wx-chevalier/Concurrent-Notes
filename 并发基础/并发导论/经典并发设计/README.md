# 经典并发设计

|     |                                                                                            |                                                                                                                                                                                        |
| --- | ------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | How to implement a spinlock                                                                | [Github: link](https://github.com/dennyzhang/cheatsheet.dennyzhang.com/blob/master/cheatsheet-concurrency-A4/concurrency.org#how-to-implement-a-spinlock)                              |
| 2   | How to implement a mutex                                                                   | [Github: link](https://github.com/dennyzhang/cheatsheet.dennyzhang.com/blob/master/cheatsheet-concurrency-A4/concurrency.org#how-to-implement-a-mutex)                                 |
| 3   | How to implement a condition variable                                                      | [Github: link](https://github.com/dennyzhang/cheatsheet.dennyzhang.com/blob/master/cheatsheet-concurrency-A4/concurrency.org#how-to-implement-a-condition-variable)                    |
| 4   | How to implement a reader-writer locker                                                    | [Github: link](https://github.com/dennyzhang/cheatsheet.dennyzhang.com/blob/master/cheatsheet-concurrency-A4/concurrency.org#how-to-implement-a-reader-writer-locker)                  |
| 5   | How to implement a bounded blocking queue                                                  | [Github: link](https://github.com/dennyzhang/cheatsheet.dennyzhang.com/blob/master/cheatsheet-concurrency-A4/concurrency.org#how-to-implement-a-bounded-blocking-queue)                |
| 6   | Create two threads cooridnated by mutex in C                                               | [Github: code-example/threads/thread_mutex.c](https://github.com/dennyzhang/cheatsheet.dennyzhang.com/blob/master/cheatsheet-concurrency-A4/code-example/threads/thread_mutex.c)       |
| 7   | IPC: use shared memory without kernel copy                                                 | [Github: code-example/shared-memory](https://github.com/dennyzhang/cheatsheet.dennyzhang.com/blob/master/cheatsheet-concurrency-A4/code-example/shared-memory)                         |
| 8   | Support in-memory kv store transactions                                                    | [Github: link](https://github.com/dennyzhang/cheatsheet.dennyzhang.com/blob/master/cheatsheet-concurrency-A4/concurrency.org#support-in-memory-kv-store-transactions)                  |
| 9   | [Design a thread-safe Hashmap](https://architect.dennyzhang.com/design-concurrent-hashmap) |                                                                                                                                                                                        |
| 10  | [Delayed task scheduling](https://architect.dennyzhang.com/explain-delayedqueue)           |                                                                                                                                                                                        |
| 11  | Implement a lock-free queue with multiple readers/writers                                  | [Github: link](https://github.com/dennyzhang/cheatsheet.dennyzhang.com/blob/master/cheatsheet-concurrency-A4/concurrency.org#implement-a-lock-free-queue-with-multiple-readerswriters) |
| 12  | Implement a api rate limiter with token bucket algorithm                                   |                                                                                                                                                                                        |

## 编程挑战

| Num | Problem                             | Summary                                                                                                              |
| --- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| 1   | Semaphores to maintain the order    | [Leetcode: Building H2O](https://code.dennyzhang.com/building-h2o)                                                   |
| 2   | Web Crawler Multithreaded           | [LeetCode: Web Crawler Multithreaded](https://code.dennyzhang.com/web-crawler-multithreaded)                         |
| 3   | Print Zero Even Odd                 | [Leetcode: Print Zero Even Odd](https://code.dennyzhang.com/print-zero-even-odd)                                     |
| 4   | Map/Reduce: scheduler + workers     | [Leetcode: Fizz Buzz Multithreaded](https://code.dennyzhang.com/fizz-buzz-multithreaded)                             |
| 5   | Design Bounded Blocking Queue       | [Leetcode: Design Bounded Blocking Queue](https://code.dennyzhang.com/design-bounded-blocking-queue)                 |
| 6   | Avoid deadlock and starvation       | [Leetcode: The Dining Philosophers](https://code.dennyzhang.com/the-dining-philosophers)                             |
| 7   | Claim ownerhip of a single resource | [LeetCode: Traffic Light Controlled Intersection](https://code.dennyzhang.com/traffic-light-controlled-intersection) |
