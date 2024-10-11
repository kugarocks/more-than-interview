---
title: "乐观锁与悲观锁"
description: ""
summary: ""
date: 2024-10-11T15:00:00+08:00
lastmod: 2024-10-11T15:00:00+08:00
weight: 400
seo:
  title: "乐观锁与悲观锁"
  description: ""
  canonical: ""
  noindex: false
---

## 简介

在 Go 中，虽然没有直接的“乐观锁”和“悲观锁”概念，但可以通过相应的方式实现类似功能。
Go 的并发模型主要通过 `goroutines` 和通道（channels）来实现并发处理，
不过常见的锁机制，如 `sync.Mutex`（互斥锁）和 `CAS` 操作（Compare And Swap），
也可以用来模拟悲观锁和乐观锁的行为。

## 乐观锁

在 Go 中，乐观锁通常使用原子操作（atomic operations），
例如 `sync/atomic` 包中的 `CompareAndSwap` 函数。
通过版本号或计数器来检查数据是否被修改，类似于数据库中的乐观锁。

```go {frame="none"}
package main

import (
    "fmt"
    "sync/atomic"
)

type OptimisticCounter struct {
    value int64
}

func (o *OptimisticCounter) Increment() bool {
    for {
        // 获取当前值
        oldValue := atomic.LoadInt64(&o.value)
        // 新值
        newValue := oldValue + 1
        // 尝试原子更新值
        if atomic.CompareAndSwapInt64(&o.value, oldValue, newValue) {
            return true // 更新成功
        }
        // 更新失败，说明数据被其他操作修改，重试
    }
}

func (o *OptimisticCounter) GetValue() int64 {
    return atomic.LoadInt64(&o.value)
}

func main() {
    counter := &OptimisticCounter{value: 0}

    // 并发修改计数器
    for i := 0; i < 100; i++ {
        go counter.Increment()
    }

    // 模拟延迟，保证 goroutines 执行完成
    fmt.Println("Final Counter Value:", counter.GetValue())
}
```

`atomic.CompareAndSwapInt64` 是一个 CAS 操作，检查 `oldValue` 是否与当前值相同，如果相同则更新为 `newValue`。
这模拟了乐观锁的行为，即尝试更新，失败则重试。`Increment` 方法不断尝试增加计数器，直到成功，适合并发访问冲突较少的场景。

## 悲观锁

在 Go 中，悲观锁可以通过 `sync.Mutex` 实现，确保在资源访问时其他 `goroutine` 无法同时操作。

```go {frame="none"}
package main

import (
    "fmt"
    "sync"
)

type PessimisticCounter struct {
    mu    sync.Mutex
    value int
}

func (p *PessimisticCounter) Increment() {
    p.mu.Lock()   // 加锁，阻止其他 goroutine 访问
    defer p.mu.Unlock() // 在函数返回时解锁
    p.value++
}

func (p *PessimisticCounter) GetValue() int {
    p.mu.Lock()   // 加锁
    defer p.mu.Unlock() // 解锁
    return p.value
}

func main() {
    counter := &PessimisticCounter{value: 0}
    var wg sync.WaitGroup

    // 并发修改计数器
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment()
        }()
    }

    // 等待所有 goroutine 完成
    wg.Wait()

    fmt.Println("Final Counter Value:", counter.GetValue())
}
```

`sync.Mutex` 实现了典型的悲观锁，在 `Increment` 和 `GetValue` 操作中加锁，
保证同一时间只有一个 `goroutine` 能修改或读取计数器的值。
通过 `p.mu.Lock()` 和 `p.mu.Unlock()` 确保对计数器的操作是线程安全的，
适合高并发的写入场景。
