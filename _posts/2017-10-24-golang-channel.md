---
layout: post
title: "Golang --- channel 注意事項"
author: "Richard Lin"
categories: golang
tags: golang
image:
  feature: golang.jpg
---

### Channel
* * *
使用於goroutine中與外界傳遞訊息的方式。

### 常見錯誤
* * *
```golang
import "fmt"

func main(){
     s := make(chan string) //宣告一個 channel 變數
     s <- "hello"           //寫入 channel (sender)
     val := <- s               //讀取 channel (receiver)
     fmt.Println(val)
}
```
此時會出現deadlock，因為執行到`s <- "hello"`時執行緒會停在這邊等待reciver接收，因此整個程式會卡在那邊。<br>

### 正確方式
```golang
import "fmt"

func main(){
     s := make(chan string) //宣告一個 channel 變數
     go func(){
         s <- "hello"
     }()                    //寫入 channel (sender)
     val := <- s               //讀取 channel (receiver)
     fmt.Println(val)
}
```

### 遞迴channel
* * *
``` golang
func main(){
    ch := make(chan int)
    go recursiveChannel(ch ,4)
    for resp := range ch{
        fmt.Println("to" ,resp)
    }
}

func recursiveChannel(ch chan int, num int){
    defer close(ch)

    if num >0{
        ch <- num
        chh := make(chan int)
        go recursiveChannel(chh ,num - 1)
        for resp := range chh{
            ch <- resp 
        }
    }

    return 
}
```

### Result
```golang
to  4
to  3
to  2
to  1
```