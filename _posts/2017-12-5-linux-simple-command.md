---
layout: post
title: "Linux --- linux指令手札"
author: "Richard Lin"
categories: documents
tags: documents
image:
  feature: linux.jpg
---

## tar
* * *
用於壓縮或是解壓縮。[超詳盡介紹](http://note.drx.tw/2008/04/command.html)

### --- option ---

-c  ：建立打包檔案，可搭配 -v 來察看過程中被打包的檔名(filename)<br>
-t  ：察看打包檔案的內容含有哪些檔名，重點在察看『檔名』就是了<br>
-x  ：解打包或解壓縮的功能，可以搭配 -C (大寫) 在特定目錄解開<br>
      **特別留意的是， -c, -t, -x 不可同時出現在一串指令列中。**<br><br>
-z  ：透過 gzip  的支援進行壓縮/解壓縮：此時檔名最好為 *.tar.gz<br>
-j  ：透過 bzip2 的支援進行壓縮/解壓縮：此時檔名最好為 *.tar.bz2<br>
-J  ：透過 xz    的支援進行壓縮/解壓縮：此時檔名最好為 *.tar.xz<br>
      **特別留意， -z, -j, -J 不可以同時出現在一串指令列中**<br><br>
-v  ：在壓縮/解壓縮的過程中，將正在處理的檔名顯示出來！<br>
-f filename：-f 後面要立刻接要被處理的檔名！建議 -f 單獨寫一個選項囉！(比較不會忘記)<br>
-C 目錄    ：這個選項用在解壓縮，若要在特定目錄解壓縮，可以使用這個選項。<br>

## touch
* * *
用於製造檔案，好用小東西。

```sh
touch temp.txt
```

如果想要創建有字在裡面的`.txt`可以這樣寫。

```sh
echo Text in file >word.txt
```

## chmod
* * *
更改權限。
r:4 可讀<br>
w:2 可寫<br>
x:1 可執行<br>
7 = rwx<br>
6 = rw，以此列推<br>

```sh
chmod 777 filename
```

chmod 777 filename 為root(rwx)、group(rwx)、other(rwx)
