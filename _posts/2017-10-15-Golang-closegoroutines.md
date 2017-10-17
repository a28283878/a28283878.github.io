---
layout: post
title: "Golang-如何在多個goroutine中 關閉特定一個"
author: "Richard Lin"
categories: golang
tags: golang
image:
  feature: golang.jpg
---

最近在上班遇到一個問題，有多個goroutine會同時運行。<br>
每個goroutine的作用為定時更新某個物件。

可是有些物件會停止更新，要找個方法讓goroutine也停止運作，不然大量的goroutine會占用大量資源。

### 1. StackOverflow的方法
* * *
大部份的人都建議使用`channel`

```golang
package main

import "sync"
func main() {
    var wg sync.WaitGroup
    wg.Add(1)

    ch := make(chan int)
    go func() {
        for {
            foo, ok := <- ch
            if !ok {
                println("done")
                wg.Done()
                return
            }
            println(foo)
        }
    }()
    ch <- 1
    ch <- 2
    ch <- 3
    close(ch)

    wg.Wait()
}
```

### 問題

太多`channel`不好管理


### 2. 簡單Map法
* * *

使用map管理是否要繼續使用goroutine

`var m map[struct]bool`

使用struct當作key

```golang
t T{"something"}

go update(t)
```

```golang
func update(t T){
    for{

        if m[t]{//若沒有m[t] 預設會是false
            //刪除map中的t
            Delete(m,t)

            //關閉goroutine
            return
        } 

        dosomething()
    }
}
```
### 使用方法

當我們要關閉時，我們只要把map[t]設成true。<br>
等下次更新時goroutine就會自動關閉且把map[t]刪除。