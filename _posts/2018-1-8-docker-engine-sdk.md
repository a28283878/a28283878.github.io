---
layout: post
title: "Docker --- docker engine sdk for golang"
author: "Richard Lin"
categories: documents
tags: documents
image:
  feature: docker.png
---

## Docker server & client
docker運作方法為，docker client會使用api去打docker server來取得資料，所有我們所使用的指令例如`docker ps`都是通過API來與server溝通的。因此為了方便開發，docker官方發布了golang與python的sdk給開發人員使用。今天要用的就是使用sdk來製作docker的為web terminal。

## 使用材料
[docker engine sdk for golang](https://godoc.org/github.com/moby/moby/client)<br>
[xterm.js](https://xtermjs.org/)

## 實作流程
1.  透過sdk與docker server建立出tty
2.  在建立與前端的連線(web socket)
3.  使用xterm.js顯示出terminal

## 透過sdk與docker server建立出tty
若我們要對docker daemon做任何操作，必須先建立出client物件。
```golang
    cli, err := client.NewClient("tcp://xxx.xxx.xx.x", "api的版本", nil, nil)
	if err != nil {
		log.Print(err)
		return
	}
```
之後建立出與container的連線。
```golang
    ctx := context.Background()
	execConfig := types.ExecConfig{
		AttachStderr: true,
		AttachStdin:  true,
		AttachStdout: true,
		Cmd:          []string("/bin/sh"),
		Tty:          true,
		Detach:       false,
	}

	//set target container
	exec, err := cli.ContainerExecCreate(ctx, "container id", execConfig)
	if err != nil {
		log.Print(err)
		return
	}
	execAttachConfig := types.ExecStartCheck{
		Detach: false,
		Tty:    true,
	}
	containerConn, err := cli.ContainerExecAttach(ctx, exec.ID, execAttachConfig)
	if err != nil {
		log.Print(err)
		return
	}
```
這樣就可以得到`containerConn`的Conn物件了，之後再與handler整合。

## 在建立與前端的連線(web socket)
我們可以把上方的東西整合成可以連線的web socket handler。
```golang
var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin: func(r *http.Request) bool {
		return true
	},
}

func handleWebsocket(w http.ResponseWriter, r *http.Request) {

	//upgrade http to websocket
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Print(err)
		return
	}

	cli, err := client.NewClient("tcp://xxx.xxx.xx.x", "api的版本", nil, nil)
	if err != nil {
		log.Print(err)
		return
	}

	ctx := context.Background()
	execConfig := types.ExecConfig{
		AttachStderr: true,
		AttachStdin:  true,
		AttachStdout: true,
		Cmd:          []string("/bin/sh"),
		Tty:          true,
		Detach:       false,
	}

	//set target container
	exec, err := cli.ContainerExecCreate(ctx, "container id", execConfig)
	if err != nil {
		log.Print(err)
		return
	}
	execAttachConfig := types.ExecStartCheck{
		Detach: false,
		Tty:    true,
	}
	containerConn, err := cli.ContainerExecAttach(ctx, exec.ID, execAttachConfig)
	if err != nil {
		log.Print(err)
		return
	}

	go func() {
		for {
			//docker reader and websocket writer
			buf := make([]byte, 4096)
			_, err = containerConn.Reader.Read(buf)
			if err != nil {
				log.Print(err)
				conn.Close()
				return
			}
			err = conn.WriteMessage(websocket.BinaryMessage, buf)
			if err != nil {
				log.Print(err)
				conn.Close()
				return
			}
		}
	}()

	for {
		//docker writer and websocket reader
		_, reader, err := conn.NextReader()
		if err != nil {
			log.Print(err)
			containerConn.Close()
			return
		}
		_, err = io.Copy(containerConn.Conn, reader)
		if err != nil {
			log.Print(err)
			containerConn.Close()
			return
		}
	}
}
```


## 使用xterm.js顯示出terminal
接下來使用專門處理terminal的前端js套件xterm.js，來呈現出terminal的樣子就可以了。
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <link rel="stylesheet" href="node_modules/xterm/dist/xterm.css">
    <script src="node_modules/xterm/dist/xterm.js"></script>
    <meta charset="UTF-8">
    <title>test</title>
</head>
<body>
    <script>
        var term
        var terminalContainer = document.getElementById('terminal')
        
        var websocket
        function ab2str(buf) {
            return new TextDecoder().decode(buf);
        }  
    </script>
    <form id="command-submit" onsubmit="event.preventDefault(); cmdFunction();">
        Command : <input type="text" id="cmd"><br>
        <input type="submit" value="Submit">
    </form>
    <div id="terminal">
        <script>
            term = new Terminal({
                cursorBlink: true, 
                cols: 120
            });
            term.open(terminalContainer, true);
        </script>
    </div>
    <script>
        function cmdFunction(){
            var cmd = document.getElementById('cmd').value
            websocket = new WebSocket('ws://127.0.0.1:8000/term?cmd=' + cmd);
            websocket.binaryType = "arraybuffer";
            websocket.onopen = function(evt){                                 
                term.off('data', sendWebsocketData);
                term.on('data', sendWebsocketData);

                websocket.onmessage = function(evt) {
                    if (evt.data instanceof ArrayBuffer) {
                            term.write(ab2str(evt.data));
                    } else {
                            alert(evt.data)
                    }
                }

                websocket.onclose = function(evt) {
                    term.off('data', sendWebsocketData);
                    term.write("Session terminated");
                    term.write("\r\n");
                }

                websocket.onerror = function(evt) {
                    if (typeof console.log == "function") {
                            console.log(evt)
                    }
                }
            }
        }   

        function sendWebsocketData(data){
            websocket.send(new TextEncoder().encode(data));
        }
    </script>
</body>
</html>
```