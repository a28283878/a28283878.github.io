---
layout: post
title: "Golang --- init 與 package 關係"
author: "Richard Lin"
categories: golang
tags: golang
image:
  feature: golang2.jpeg
---

本篇主要是紀錄一下package `init()` 這func，當package被引用時是否會觸發，如果會觸發則何時觸發。<br>

本篇使用兩個.go測試<br>

1.  main.go
    ```go
    package main

    import (
        "fmt"
        "mygo/test/other"
    )

    func main() {

        fmt.Println("HI i am main")
        other.NewOther()
    }
    ```

2.  init.go
    ```go
    package other

    import (
        "fmt"
    )

    func init() {
        fmt.Println("HI i am other")
    }

    func NewOther() {
        fmt.Println("Just new an other")
    }
    ```

執行後的結果為
```
HI i am other
HI i am main
Just new an other
```

代表說`init()`在package成功被引用後就會被執行