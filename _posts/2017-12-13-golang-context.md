---
layout: post
title: "Golang --- context package"
author: "Richard Lin"
categories: golang
tags: golang
image:
  feature: golang.jpg
---

## Context 套件包
* * *
於GO 1.7版本，從/net/context升級為標準套件。用於處理多個goroutine中資料的同步與當呼叫goroutine的主程序結束時關閉開啟的goroutine。<br>
是個相當實用但卻需要花點心思了解的套件包。<br>
目前主要用到的部分是當主程序關閉時如何關掉執行中的goroutine與如何透過context時做timeout的機制。

## Context 新手上路
* * *
在官方的文件中這麼說到。

1.  不要傳遞為nil的Context，官方有時做出兩種初始的Context(`Context.Background()`、`Context.TODO()`)，請使用這兩種作為最一開始的context。
2.  Context.Value所傳遞的數據不要含有Optional Value。
3.  不要將Context放入struct中，應該要傳入需要的func裡，且放在第一個argument中並叫做ctx，如func DoSomething（ctx context.Context,arg Arg）

## Context 運用方式
* * *
1. Timeout 機制
```golang
ctx := context.Background()
ctxWithTimeout, cancelWithTimeout := context.WithTimeout(ctx, 5*time.Second)
defer cancancelWithTimeout()

select {
	case <-ctxWithTimeout.Done():
		fmt.Println(ctx.Err()) // prints "context deadline exceeded"
}
```

2. 停止 goroutines
```golang
func work(ctx context.Context) error {
    for i := 0; i < 1000; i++ {
        select {
        case <-time.After(2 * time.Second):
            fmt.Println("Doing some work ", i)

        // we received the signal of cancelation in this channel    
        case <-ctx.Done():
            fmt.Println("Cancel the context ", i)
            return ctx.Err()
        }
    }
    return nil
}

func main() {   
    ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
    defer cancel()
    fmt.Println("Hey, I'm going to do some work")

    go work(ctx)
    fmt.Println("Finished. I'm going home")
}
```