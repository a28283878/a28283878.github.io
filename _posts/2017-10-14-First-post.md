---
layout: post
title: "Jekyll 使用教學"
author: "Richard Lin"
categories: documents
image:
  feature: jekyll.png
---

## 1.安裝Jekyll
---------------------------------------

&nbsp;&nbsp;&nbsp;&nbsp;按照[Jekyll 安裝教學](http://jekyllcn.com/docs/installation/)中指示，將jekyll安裝完成。



## 2.選擇喜歡的Theme
---------------------------------------

&nbsp;&nbsp;&nbsp;&nbsp;可以到[Jekyll Theme](http://jekyllthemes.org/)中選擇自己喜歡的Theme,並去github `clone/fork`一份至local。



## 3.Build
---------------------------------------
1. 	`jekyll build` 來build部落格<br/>
	`jekyll serve` build部落格且開啟local server

2. 	通常port為4000，build完的網頁就會在`127.0.0.1:4000`

### 常見錯誤

build時出現 `you haven't included the 'jekyll-paginate' gem.`

``更新gem : gem install jekyll-paginate``<br>
``更新config.yml : 增加 plugins : [jekyll-paginate]``
