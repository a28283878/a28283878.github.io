---
layout: post
title: "Golang --- net/websocket"
author: "Richard Lin"
categories: golang
tags: golang
image:
  feature: golang.png
---

## 簡單websocket server
* * *

```golang
package main

import (
	"fmt"

	"github.com/jasonsoft/napnap"
	"golang.org/x/net/websocket"
)
func main() {
	nap := napnap.New()
	router := napnap.NewRouter()
	go h.run()
	router.Get("/ws", napnap.WrapHandler(wsHandler()))
	nap.Use(router)
	httpEngine := napnap.NewHttpEngine(":8081")
	nap.Run(httpEngine)
}

```

這樣就會建立出listen 8081的handler


## 簡單client 連接
* * *

```golang
package main

import (
	"fmt"
	"os"
	"time"

	"golang.org/x/net/websocket"
)
const address string = "localhost:8081"

func main() {
	header := map[string][]string{}
	b := []string{"Penn"}
	name := []string{"vv"}
	header["topic"] = b
	header["name"] = name
	d, err := websocket.NewConfig(fmt.Sprintf("ws://%s/ws", address), fmt.Sprintf("http://%s/", address))
	d.Header = header
	fmt.Println("Starting Connect")
	ws, err := websocket.DialConfig(d)
	if err != nil {
		fmt.Printf("Dial failed: %s\n", err.Error())
		os.Exit(1)
	}
	readClientMessages(ws)
}

func readClientMessages(ws *websocket.Conn) {
	for {
		var message string
		// err := websocket.JSON.Receive(ws, &message)
		err := websocket.Message.Receive(ws, &message)
		if err != nil {
			fmt.Printf("Error::: %s\n", err.Error())
			ws = reconnect()
		}
		if len(message) > 0 {
			fmt.Println("message : " + message)
		}
	}
}

func reconnect() *websocket.Conn {
	ticker := time.NewTicker(time.Duration(5) * time.Second)
	var count = 1
	for _ = range ticker.C {
		header := map[string][]string{}
		b := []string{"Penn"}
		name := []string{"vv"}
		header["topic"] = b
		header["name"] = name
		d, err := websocket.NewConfig(fmt.Sprintf("ws://%s/ws", address), fmt.Sprintf("http://%s/", address))
		d.Header = header
		fmt.Printf("\nRetry Connect : %d times\n", count)
		ws, err := websocket.DialConfig(d)
		if err != nil {
			fmt.Printf("Dial failed: %s\n\n", err.Error())
		} else {
			return ws
		}
		count = count + 1
	}
	return nil
}

```

這樣會連去ws://localhost:8081/ws，並且會建立起websocket。<br>
若要使用Send或是Recieve，只要對`ws`操作就好。


## websocket轉接server
* * *

本次需要完成的是當有client使用websocket連接到server時，我們必須在建立出一個對應的websocket到更後方的server並保持連線。<br>

### 1.hub
需要一個管理的地方，集中管理這些websocket connection，目前為使用memory記住這些資訊，因此無法支援cluster。<br>
之後可能需要搭配Mysql或是Redis來完成支援cluster。<br>

```go
type hub struct {
    //所有的連線
	connections map[*connection]bool
    //推播所有的連線訊息
	broadcast chan []byte
    //註冊新的連線
	register chan *connection
    //取消註冊
	unregister chan *connection
}

//init hub
var h = hub{
	broadcast:   make(chan []byte),
	register:    make(chan *connection),
	unregister:  make(chan *connection),
	connections: make(map[*connection]bool),
}

//需要使用go h.run() 讓這個function持續運作
func (h *hub) run() {
	for {
		select {
		case c := <-h.register:
			h.connections[c] = true
			fmt.Printf("connection Name : %s  Topic : %s  From %s\n", c.Name, c.Topic, c.WsFront.Request().RemoteAddr)
		case c := <-h.unregister:
			if _, ok := h.connections[c]; ok {
				fmt.Printf("disconnection Name : %s  Topic : %s  From %s\n", c.Name, c.Topic, c.WsFront.Request().RemoteAddr)
				delete(h.connections, c)
				close(c.SendFront)
				close(c.SendBack)
			}
		case m := <-h.broadcast:
			for c := range h.connections {
				fmt.Printf("disconnection Name : %s  Topic : %s  From %s\n", c.Name, c.Topic, c.WsFront.Request().RemoteAddr)
				select {
				case c.SendFront <- m:
				default:
					delete(h.connections, c)
					close(c.SendFront)
					close(c.SendBack)
				}
			}
		}
	}
}
```

### 2.connection
這邊寫有server還有`connection struct`以及未實作的根據連線topic連接至不同的後台socket。

```golang
type connection struct {
	Name    string
    Topic   string
    //client 與 此server的ws
    WsFront *websocket.Conn
    //後台server 與 此server的ws
	WsBack  *websocket.Conn
    //準備傳送到client的訊息buff
    SendFront chan []byte
    //準備傳送到後台server的訊息buff
	SendBack  chan []byte
}

//用來對後台server 傳送訊息
func (c *connection) backWriter() {
	for message := range c.SendBack {
		err := websocket.Message.Send(c.WsBack, message)
		if err != nil {
			break
		}
	}
	h.unregister <- c
	c.WsFront.Close()
	c.WsBack.Close()
}
//用來監聽後台server所傳送的訊息
func (c *connection) backReader() {
	for {
		var message []byte
		err := websocket.Message.Receive(c.WsBack, &message)
		if err != nil {
			fmt.Println(err.Error())
			break
		}
		c.SendFront <- message
	}
	h.unregister <- c
	c.WsFront.Close()
	c.WsBack.Close()
}
//用來對前台client 傳送訊息
func (c *connection) frontWriter() {
	for message := range c.SendFront {
		err := websocket.Message.Send(c.WsFront, message)
		if err != nil {
			break
		}
	}
	h.unregister <- c
	c.WsFront.Close()
	c.WsBack.Close()
}
//用來監聽前台client所傳送的訊息
func (c *connection) frontReader() {
	for {
		var message []byte
		err := websocket.Message.Receive(c.WsFront, &message)
		if err != nil {
			fmt.Println(err.Error())
			break
		}
	}
	h.unregister <- c
	c.WsFront.Close()
	c.WsBack.Close()
}

//handler
func wsHandler() websocket.Handler {
	return websocket.Handler(func(ws *websocket.Conn) {
        req := ws.Request()
        //取得header的一些值
		conn := connection{
			Name:  req.Header.Get("name"),
			Topic: req.Header.Get("topic"),
		}
		// Send Success Connect
		err := websocket.Message.Send(ws, "connect middle")
		if err != nil {
			log.Println(err)
		}

        //todo : 根據conn.Topic來取得不同的backAddr
        //       應該會與api一起搭配使用
		//backAddr := getBackAddress(conn.Topic)
        d, err := websocket.NewConfig(fmt.Sprintf("ws://%s/ws", address), ws.Config().Origin.String())
        //將收到的header也複製到要傳送到後台的header裡面
		d.Header = req.Header
		wsTrans, err := websocket.DialConfig(d)
		if err != nil {
			websocket.Message.Send(ws, fmt.Sprintf("Dial failed: %s\n", err.Error()))
			return
		}
		conn.WsFront = ws
        conn.WsBack = wsTrans
        //buff 預設為1024，大量資料傳輸必須更寬
		conn.SendBack = make(chan []byte, 1024)
        conn.SendFront = make(chan []byte, 1024)
        //註冊連線
		h.register <- &conn
		defer func() { h.unregister <- &conn }()
		go conn.frontWriter()
		go conn.backWriter()
        go conn.frontWriter()
        //一個不使用go 不然handler會關閉 ws也會中斷連線
		conn.backReader()
	})

}

func getBackAddress(topic string) string {
	return fmt.Sprintf("ws://%s/ws", address)
}
```