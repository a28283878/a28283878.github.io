---
layout: post
title: "Golang --- golang web terminal"
author: "Richard Lin"
categories: golang
tags: golang
image:
  feature: golang2.jpeg
---

### 本篇[程式碼](https://github.com/a28283878/go_terminal)

除了docker的web terminal之外也要做一個主機的web terminal，但是網路上的範本都是在主機架起server後直接連接前端。而這次的需求為，主機跟前端中間還會有一個中繼的server，所以只好自幹出一個簡單的server了。

## 使用材料
[kr/pty](https://godoc.org/github.com/kr/pty)<br>
[xterm.js](https://xtermjs.org/)

## 實作流程
裝載主機的server需要做的事情不多。
1.  與terminal建立連線
2.  open tcp port讓中繼主機可以連線

## 與terminal建立連線
我們需要使用kr/pty來建立出與terminal的io連線。<br>
注意kr/pty只支援unix的terminal

```golang
//set and start command
	cmd := exec.Command("/bin/bash", "-l")
	cmd.Env = append(os.Environ(), "TERM=xterm")
	tty, err := pty.Start(cmd)
	if err != nil {
		log.Print(err.Error())
		return
	}
```

別忘了建立連線後要記得用`defer`來讓執行結束時自動關閉。
```golang
defer func() {
  cmd.Process.Kill()
  cmd.Process.Wait()
  tty.Close()
}()
```

## open tcp port讓中繼主機可以連線
再來我們需要開啟一個port讓中繼server可以連線的到。<br>
這次因為是主機對主機所以就直接用tcp不用websocket了。
```golang
func main() {
	l, err := net.Listen("tcp", ":8081")
	if err != nil {
		log.Fatal(err.Error())
	}
	defer l.Close()
	for {
		// Wait for a connection.
		conn, err := l.Accept()
		if err != nil {
			log.Fatal(err.Error())
		}
		// Handle the connection in a new goroutine.
		// The loop then returns to accepting, so that
		// multiple connections may be served concurrently.
		go serverConn(conn)
	}
}
```

## 中繼server
中繼server就跟docker web terminal很像，只要串接前端與後端就能成功讓資料傳遞。