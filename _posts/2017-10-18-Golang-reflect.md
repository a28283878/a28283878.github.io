---
layout: post
title: "Golang --- reflect 帶我走的地獄路"
author: "Richard Lin"
categories: golang
tags: golang
image:
  feature: golang.png
---

# 需求
* * *
<div style="width:100%; height:2em;"></div>
原本公司load config的方法為使用yaml檔，配合官方套件輕鬆載入。<br>
然而新的需求是希望能夠直接從enviroment variable中直接拿到config的值<br>
於是一趟reflect的地獄巡禮即將展開。

# Reflect這東西
* * *
<div style="width:100%; height:2em;"></div>
由於需求所需要支援的型態各式各樣，除了基本`int` `float` `string` `bool`，甚至有`slice`跟`struct`，還有讓我在地獄走兩回的`slice struct`。<br>

因為傳進來的型態不固定，因此最外面的接口必須使用`interface{}`來當作parameter。<br>

再使用**reflect**取得值的型態跟賦予值。<br>

### reflect.Value
是變數的值，可以get或是set值

### reflect.Type
是變數的型態，可以get到型態的各種資訊

# 開工
* * *
<div style="width:100%; height:2em;"></div>

##  1.  首先我們需一個`func load(out reflect.Value)`來處理外部來的資訊。<br>

```go
func load(out reflect.Value){
    //check 有多少值在out
}
```

**Notice** : 
> `out reflect.Value` 該怎麼得到reflect.Value這型態<br>
> 假設我們從外部拿到`in interface{}`<br>
> 我們需要做的是`reflect.ValueOf(in)`<br>
> 但`in`因為要求關係會是pointer，後面的處理會無法處理，因此我們要再加上`Elem()`來取得pointer所指向的值<br>

> 結論：`reflect.ValueOf(in).Elem()`

##  2.  我們可以使用`out.NumField()`來知道有多少的值在out裡。<br>

**Example** :

```go
var number int

reflect.ValueOf(number).NumberField() = 1

type Mystruct struct{
    val1 int
    val2 string
}

reflect.ValueOf(Mystruct).NumberField() = 2
```

##  3.  把check完的值準備set值進去。<br>

這時候處理的型態有可能為`int` `float` `string` `bool` `slice` `struct` (`slice struct`為slice)<br>

所有的基本型態都有語法可以輸入，例如string有reflect.Value.SetString(string)<br>

麻煩的是struct與slice<br>

## struct slice
* * *
struct的處理方法，我是將他再塞進`load(out reflect.Value)`裡，讓他再解析一次<br><br>

slice的基本型態我是將值先使用`json.Unmarshal([]byte(value), &slice)`<br>
再使用`out.Set(reflect.ValueOf(slice))`將值塞入<br><br>

但如果是`slice struct`頭就大了<br>

##  4.  Slice Struct<br>
主要做法為使用`slice := reflect.MakeSlice(out.Type(), out.Len(), out.Cap())`來創出一個slice

再想辦法把值塞進slice裡，再講slice塞進out

```go
func readSlice(out reflect.Value, outField reflect.StructField) error {
	env := os.Getenv(outField.Tag.Get("env"))
	tokens := getTokens(env)
	elType := out.Type().Elem()
	slice := reflect.MakeSlice(out.Type(), out.Len(), out.Cap())

	if len(tokens) >= 1 {
		for _, ele := range tokens {
			el := reflect.New(elType).Elem()
			ele = strings.TrimLeft(ele, "{")
			ele = strings.TrimRight(ele, "}")
			vals := strings.Split(ele, ",")
			for index, val := range vals {
				if index >= el.NumField() {
					return fmt.Errorf("too many arguments. in %s", ele)
				}
				if err := setValue(el.Field(index), val); err != nil {
					return err
				}
			}
			slice = reflect.Append(slice, el)
		}
	}

	out.Set(slice)
	return nil
}
```

###  [github專案連結](https://github.com/a28283878/go_config_env_yaml)<br><br><br>