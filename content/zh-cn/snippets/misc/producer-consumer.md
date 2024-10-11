---
title: "生产者与消费者"
description: ""
summary: ""
date: 2024-10-10T14:00:00+08:00
lastmod: 2024-10-10T14:00:00+08:00
weight: 200
seo:
  title: "生产者与消费者"
  description: ""
  canonical: ""
  noindex: false
---

## 简介

实现一个简单的生产者与消费者模型。

## Go

```go {frame="none"}
package main

import (
  "fmt"
  "math/rand"
  "sync"
  "time"
)

const (
    numItems = 10
)

type Item struct {
    Action string
    Number int
}

func producer(ch chan<- *Item, wg *sync.WaitGroup, r *rand.Rand) {
    defer wg.Done()
    for i := 0; i < numItems; i++ {
        num := r.Intn(100)
        fmt.Printf("produce item: %d\n", num)
        ch <- &Item{
            Action: "print",
            Number: num,
        }
        time.Sleep(time.Millisecond * 500)
    }
    close(ch)
}

func consumer(ch <-chan *Item, wg *sync.WaitGroup) {
    defer wg.Done()
    for item := range ch {
        switch item.Action {
        case "print":
            fmt.Printf("consume item: %d\n", item.Number)
        default:
            fmt.Printf("unsupported action: %s", item.Action)
        }
        time.Sleep(time.Millisecond * 200)
    }
}

func main() {
    r := rand.New(rand.NewSource(time.Now().UnixNano()))
    ch := make(chan *Item, 5)
    wg := &sync.WaitGroup{}

    wg.Add(1)
    go producer(ch, wg, r)

    wg.Add(1)
    go consumer(ch, wg)

    wg.Wait()
    fmt.Println("Done")
}
```
