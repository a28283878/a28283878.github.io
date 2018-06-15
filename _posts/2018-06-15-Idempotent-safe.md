---
layout: post
title: "Http Verbs --- Idempotent , Safe"
author: "Richard Lin"
categories: documents
tags: documents
image:
  feature: 
---

# Idempotent
* * *
一個http verb若是Idempotent代表說他送一次或是多次的request都會是一樣的影響並讓server維持在同一個狀態。<br>
所以像是`GET`,`HEAD`,`PUT`,`DELETE`這些動作送一次或是多次都不會讓server進入不同的狀態。

# Safe
* * *
一個http verb若是Safe的代表說這個操作不會改變server，是read-only的操作，所以像是`GET`,`HEAD`,`OPTIONS`都是safe method。