---
layout: post
title: "Golang --- websocket proxy"
author: "Richard Lin"
categories: golang
tags: golang
image:
  feature: golang.jpg
---

## 需求
* * *

```
|Client|              |Proxy|              |Server|
        
        ----http--->
                             ----http--->
                                            upgrade
                             <-----ws----
                      upgrade
        <-----ws----
```

## Proxy
* * *

`Name`跟`Topic`用於傳給Server可以做訂閱等相關功能<br>
`WsFront` 為對client的websocket連線
`WsBack`  為對server的websocket連線
```golang
type connection struct {
	Name    string
	Topic   string
	Request *http.Request
	WsFront *websocket.Conn
	WsBack  *websocket.Conn
	API     *api
}
```

Read server傳來的message，若有錯誤則用writecontrol傳遞錯誤訊息及關閉socket。<br>
若有資料從server傳輸過來則傳遞到client
```golang
func (c *connection) backReader() {
	defer c.WsBack.Close()
	defer c.WsFront.Close()
	for {
		var message []byte
		_, message, err := c.WsBack.ReadMessage()
		if err != nil {
			c.WsFront.WriteControl(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseGoingAway, err.Error()), time.Time{})
			if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseNormalClosure, websocket.CloseServiceRestart) {
				log.Error(err)
				break
			}
			log.Debug(err)
			break
		}
		err = c.WsFront.WriteMessage(websocket.BinaryMessage, message)
		if err != nil {
			log.Error(err)
			break
		}
	}
	h.unregister <- c
}
```

道理同上，只是變成read client、write server
```golang
func (c *connection) frontReader() {
	defer c.WsBack.Close()
	defer c.WsFront.Close()
	for {
		var message []byte
		_, message, err := c.WsFront.ReadMessage()
		if err != nil {
			c.WsBack.WriteControl(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseGoingAway, err.Error()), time.Time{})
			if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseNormalClosure, websocket.CloseServiceRestart) {
				log.Error(err)
				break
			}
			log.Debug(err)
			break
		}
		err = c.WsBack.WriteMessage(websocket.BinaryMessage, message)
		if err != nil {
			log.Error(err)
			break
		}
	}
	h.unregister <- c
}
```

```golang
func wsHandler(c *napnap.Context, next napnap.HandlerFunc) {
	websocketURL := c.MustGet("url").(string)
	apiEntry := c.MustGet("api").(*api)
	outReq := c.MustGet("outReq").(*http.Request)
	conn := connection{
		Name:  c.Request.Header.Get("Name"),
		Topic: c.Request.Header.Get("Topic"),
	}

	outReq.Header["Name"] = []string{conn.Name}
	outReq.Header["Topic"] = []string{conn.Topic}
	wsTrans, response, err := websocket.DefaultDialer.Dial(websocketURL, outReq.Header)
	if err != nil {
		log.Error(err)
		return
	}
	p := newProxy()
	p.removeHeader(response.Header)
	ws, err := upgrader.Upgrade(c.Writer, c.Request, response.Header)
	if err != nil {
		log.Error(err)
		return
	}

	conn.WsFront = ws
	conn.WsBack = wsTrans
	conn.API = apiEntry
	conn.Request = c.Request
	defer ws.Close()
	defer wsTrans.Close()
	h.register <- &conn
	defer func() { h.unregister <- &conn }()
	go conn.backReader()
	conn.frontReader()
}
```