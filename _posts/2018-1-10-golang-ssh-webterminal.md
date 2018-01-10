---
layout: post
title: "Golang --- golang web terminal via ssh"
author: "Richard Lin"
categories: golang
tags: golang
image:
  feature: golang.png
---

### 本篇[程式碼](https://github.com/a28283878/go_ssh_terminal)
本次使用golang ssh套件包來完成使用金鑰連線的web terminal。

## 使用材料
[golang/crypto/ssh](https://godoc.org/golang.org/x/crypto/ssh)<br>
[xterm.js](https://xtermjs.org/)

## 實作流程

首先我們必須建好金鑰並把公鑰先丟給遠端的主機，教學可以參照這篇[文章](https://blog.gtwang.org/linux/linux-ssh-public-key-authentication/)。<br>

再來我們就可以在本地拿對應的私鑰來跟遠端主機連線了。<br>
1. 先對私鑰做處理，得到能夠與公鑰相符的signer
    <hr>

    ```golang
        //get private key of user
        key, err := ioutil.ReadFile("私鑰位置 /xxx/xxx/.ssh/id_rsa")
        if err != nil {
            log.Fatalf("unable to read private key: %v", err)
        }

        // Create the Signer for this private key.
        signer, err := ssh.ParsePrivateKey(key)
        if err != nil {
            log.Fatalf("unable to parse private key: %v", err)
        }
    ```

2. 設定出登入的資訊，例如使用者名稱或是驗證hostkey的方法。
    <hr>
    ```golang
        config := &ssh.ClientConfig{
            //登入的使用者名稱
            User: "richard_lin",
            
            //設定驗證的方法，這次是用金鑰驗證也能使用帳號密碼。
            Auth: []ssh.AuthMethod{
                // Use the PublicKeys method for remote authentication.
                ssh.PublicKeys(signer),
            },
            
            //驗證hostkey的方法，這次簡單示範就不去驗證遠端主機的正當性
            HostKeyCallback: ssh.InsecureIgnoreHostKey(),
        }
    ```
3. 與遠端主機建立起連線。
    <hr>
    ```golang
        // Connect to the remote server and perform the SSH handshake.
        sshConn, err := ssh.Dial("tcp", "10.200.252.123:22", config)
        if err != nil {
            log.Fatalf("unable to connect: %v", err)
        }
        //若有建立連線，使用defer讓他關閉是個好習慣。
        defer sshConn.Close()

        // Set up new Session between server and host terminal via ssh
        session, err := sshConn.NewSession()
        if err != nil {
            log.Fatal("unable to create session: ", err)
        }
        defer session.Close()
    ```
4. 使用建立好的連線來產生出tty。
    <hr>
    ```golang
        // Set up terminal modes
        modes := ssh.TerminalModes{
            ssh.ECHO:          1,     // enable echoing
            ssh.TTY_OP_ISPEED: 14400, // input speed = 14.4kbaud
            ssh.TTY_OP_OSPEED: 14400, // output speed = 14.4kbaud
        }
        // Request pseudo terminal
        if err := session.RequestPty("xterm", 80, 30, modes); err != nil {
            log.Fatal("request for pseudo terminal failed: ", err)
        }
    ```
5. 建立與前端的websocket跟read以及write的動作。
    <hr>
    ```golang
        //set io.Reader and io.Writer from terminal session
        sshReader, err := session.StdoutPipe()
        if err != nil {
            log.Fatal(err)
        }
        sshWriter, err := session.StdinPipe()
        if err != nil {
            log.Fatal(err)
        }

        //read from terminal and write to frontend
        go func() {
            defer func() {
                conn.Close()
                sshConn.Close()
                session.Close()
            }()

            for {
                buf := make([]byte, 4096)
                n, err := sshReader.Read(buf)
                if err != nil {
                    log.Print(err)
                    return
                }
                err = conn.WriteMessage(websocket.BinaryMessage, buf[:n])
                if err != nil {
                    log.Print(err)
                    return
                }
            }
        }()

        //read from frontend and write to terminal
        go func() {
            defer func() {
                conn.Close()
                sshConn.Close()
                session.Close()
            }()

            for {
                // set up io.Reader of websocket
                _, reader, err := conn.NextReader()
                if err != nil {
                    log.Print(err)
                    return
                }
                // read first byte to determine whether to pass data or resize terminal
                dataTypeBuf := make([]byte, 1)
                _, err = reader.Read(dataTypeBuf)
                if err != nil {
                    log.Print(err)
                    return
                }

                switch dataTypeBuf[0] {
                // when pass data
                case 0:
                    buf := make([]byte, 1024)
                    n, err := reader.Read(buf)
                    if err != nil {
                        log.Print(err)
                        return
                    }
                    _, err = sshWriter.Write(buf[:n])
                    if err != nil {
                        log.Print(err)
                        conn.WriteMessage(websocket.BinaryMessage, []byte(err.Error()))
                        return
                    }
                // when resize terminal
                case 1:
                    decoder := json.NewDecoder(reader)
                    resizeMessage := windowSize{}
                    err := decoder.Decode(&resizeMessage)
                    if err != nil {
                        log.Print(err.Error())
                        continue
                    }
                    err = session.WindowChange(resizeMessage.Rows, resizeMessage.Cols)
                    if err != nil {
                        log.Print(err.Error())
                        conn.WriteMessage(websocket.BinaryMessage, []byte(err.Error()))
                        return
                    }
                // unexpected data
                default:
                    log.Print("Unexpected data type")
                }
            }
        }()
    ```
6. 建立出最後的shell，讓資料流可以正常傳遞。
    <hr>
    ```golang
        // Start remote shell
        if err := session.Shell(); err != nil {
            log.Println("failed to start shell: ", err)
        }

        if err := session.Wait(); err != nil {
            log.Println("failed to wait shell: ", err)
        }
    ```