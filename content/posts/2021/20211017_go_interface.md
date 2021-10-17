---
title: "golang interfaceとは"
date: 2021-10-17T11:25:47+09:00
draft: false
toc: false
images:
tags:
  - golang
  - dev
---


# Interface

- 中身のない型


## 何でも入る型

```go
func main() {
	var i interface{}
	
	i = 4
	fmt.Println(i) //4
	
	i = 3.5
	fmt.Println(i) //3.5
	
	i = "文字列"
	fmt.Println(i) //文字列
}
```

- 型変換

```go
// (interfaceの変数).(型名)

str, _ := i.(string)
fmt.Printf("\n%s %T", str, str) // 文字列 string true
```

- switchで型判定できる

```go
func main() {
	var i interface{}
	
	i = 4
	printType(i)
	
	i = 3.5
	printType(i)
	
	i = "文字列"
	printType(i)
	
	i = false
	printType(i)
}


func printType(i interface{}) {
	switch i.(type) {
	case int:
		fmt.Println("int")
	case float64:
		fmt.Println("float64")
	case string:
		fmt.Println("string")
	default:
		fmt.Println("other")
	}
}
```

## 関数をまとめる

- 関数群をメソッドにもつ構造体を代入できる、クラスの役割

```go
type Interface interface {
	add(x int) int
}

func main() {
	var i Interface
	i = struct1{value: 1}
	i = struct2{value: 2}
}

type struct1 struct {
	value int
}

type struct2 struct {
	value int
}

func (s1 struct1) add(x int) int {
	return s1.value + x
}
```

- addメソッドはstruct1にしか実装していないため、Interface型のiにStruct2は代入できない

> cannot use struct2{...} (type struct2) as type Interface in assignment:
	struct2 does not implement Interface (missing add method)
