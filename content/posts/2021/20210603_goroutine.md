---
title: "goroutine"
date: 2021-06-03T11:25:47+09:00
draft: false
tags: 
  - golang
  - dev
---

# Goroutine

>A goroutine is a lightweight thread managed by the Go runtime.

- [A tour of go: goroutinne](https://tour.golang.org/concurrency/1)
- 引数の評価は既存のgoroutineで行い、実行だけ別のgoroutineを使用して行う。

```go
go f(x, y, z)
```

# Channel

>Channels are a typed conduit through which you can send and receive values with the channel operator, <-.

- [A tour of go: channel](https://tour.golang.org/concurrency/2)
- goの並列はMessage Passing方式。
    - ⇆ Shared Memory：複数プロセスがロックを取りながら共通メモリを利用する
- 並行実行されるgoroutine間を接続するパイプの役割で、データを共有する

```go
//　チャネル作成 make(chan [型])
ch := make(chan int)
```

```go
// Send v to channel ch.
ch <- v  
// Receive from ch, and assign value to v.
v := <-ch  
// 					
x, y := <-c, <-c
```


## 並列数を制限する

- 最大同時並列実行数をバッファサイズとしたチャネルを作成することで実現

```go
// 最大5並列の場合
limit := make(chan struct{}, 5)
result := make(chan downloadChan)
for _, w := range words {
	go func(w string) {
            // limitチャネルに空のstruct挿入
            limit <- struct{}{}
            // do something
            // チャネル解放
            <-limit
	}(w)
}
```
