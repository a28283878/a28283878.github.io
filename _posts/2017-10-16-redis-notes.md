---
layout: post
title: "Redis-基本使用方法"
author: "Richard Lin"
categories: redis
image:
  feature: redis.jpg
---

# 1. 設定單一Key對上單一值
* * *
<div style="width:100%; height:2em;"></div>
### SET
* * *

`SET key apple`

### GET
* * *

`GET key`<br>
return `apple`

# 2. 設定單一Key對上多個值
* * *
<div style="width:100%; height:2em;"></div>
### LPUSH
* * *

`LPUSH key apple`<br>
`LPUSH key lemon`<br>

### LRANGE
* * *
#### LRANGE key start stop

`LRANGE key 0 -1`//允許輸出重複元素<br>

#### start
-1 為最後一個 element<br>
-2 為倒數第二 and so on