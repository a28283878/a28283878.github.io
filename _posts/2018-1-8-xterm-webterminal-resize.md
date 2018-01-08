---
layout: post
title: "Golang --- xterm.js resize terminal"
author: "Richard Lin"
categories: golang
tags: golang
image:
  feature: golang.png
---

在使用webterminal時如果沒有處理連線過去的terminal的長與寬時，可能就會出現一些奇怪的現象，例如顯示出來還有位子可是卻自動換行等等。<br>
所以我們要使用前端xterm.js去監聽是否有`resize`事件發生，並在後端時另外做處理。

## 使用材料
[kr/pty](https://godoc.org/github.com/kr/pty)<br>
[xterm.js](https://xtermjs.org/)

## 實作流程
1. 使用xterm.js監聽，並區別出與一般傳輸data的差別
2. 後端處理

## 使用xterm.js監聽，並區別出與一般傳輸data的差別
xterm.js已經有提供一個叫做resize的事件可以讓我們去監聽。
```js
term.on('data', function(data) {
    console.log(data)
    websocket.send(new TextEncoder().encode("\x00" + data));
});

term.on('resize', function(evt) {
    websocket.send(new TextEncoder().encode("\x01" + JSON.stringify({cols: evt.cols, rows: evt.rows})))
});
```
並且為了區分`data`我們在一般傳輸資料的資料最前面加一個0的byte，在`resize`的資料傳輸我們加一個1的byte。這樣在後端會好處理很多。

## 後端處理
在後端處理時我們必須要先讀取第一個byte來確認他是一般data還是resize。<br>
```golang
dataTypeBuf := make([]byte, 1)
    _, err := conn.Read(dataTypeBuf)
    if err != nil {
        log.Print(err.Error())
        conn.Write([]byte(err.Error()))
        return
}
```
之後我們可以拿`dataTypeBuf`去做判斷。<br>
當為0時，我們就當作一般資料傳輸。<br>
當為1時，代表說要resize tty。我們就將resize資訊decode後放進function裡即可。
```golang
type windowSize struct {
	Rows uint16 `json:"rows"`
	Cols uint16 `json:"cols"`
	X    uint16
	Y    uint16
}

switch dataTypeBuf[0] {
case 0:
    _, err := io.Copy(tty, conn)
    if err != nil {
        log.Print(err.Error())
        conn.Write([]byte(err.Error()))
        return
    }
case 1:
    decoder := json.NewDecoder(conn)
    resizeMessage := windowSize{}
    err := decoder.Decode(&resizeMessage)
    if err != nil {
        log.Print(err.Error())
        conn.Write([]byte(err.Error()))
        continue
    }
    _, _, errno := syscall.Syscall(
        syscall.SYS_IOCTL,
        tty.Fd(),
        syscall.TIOCSWINSZ,
        uintptr(unsafe.Pointer(&resizeMessage)),
    )
    if errno != 0 {
        log.Print(errno.Error())
        conn.Write([]byte(errno.Error()))
        return
    }
default:

}
```