---
title: "golang エスケープ解析を調べてみる"
date: 2021-06-02T11:25:47+09:00
draft: false
toc: false
images:
tags:
  - golang
  - dev
---

## エスケープ解析　Escape Analysis

- Goコンパイラはエスケープ解析の際に基本的にスタック領域を割り当てし、 必要な時のみヒープ割り当てを行う。
- 関数内でのみ使うならスタック、他の関数で使う可能性あればヒープ。
- スタックはヒープと比較し、軽量かつ高速処理。


ビルド時にオプションつけることで、エスケープ解析の際にどこでヒープ領域が使われたか確認できる


```go
go build -gcflags '-m'

// 詳細
go build -gcflags '-m -m'
```


can inline = インライン展開の条件を満たしている
inlining call to = インライン展開
does not escape = スタック領域
does escape = ヒープ領域


## 例

- スタックには上限があるため、メモリを多く必要とする場合はヒープ領域にエスケープされる

```go
func child(data []int) {
   if len(data) == 0 {
     fmt.Fprintln(os.Stderr, len(data))
   }
}

func BenchmarkTooLarge(b *testing.B) {
    for i := 0; i < b.N; i++ {
        data := make([]int, 10000)
        child(data)
    }
}
```

```
$ go test escape_analysis_test.go -bench . -gcflags=-m -benchmem
# command-line-arguments [command-line-arguments.test]
./escape_analysis_test.go:10:6: can inline child
./escape_analysis_test.go:19:17: inlining call to child
./escape_analysis_test.go:10:15: data does not escape
./escape_analysis_test.go:12:18: ... argument does not escape
./escape_analysis_test.go:12:33: len(data) escapes to heap　// fmt外部関数への引数はヒープへ
./escape_analysis_test.go:16:24: b does not escape
./escape_analysis_test.go:18:21: make([]int, 10000) escapes to heap　// 長さ10000のスライス割り当てはヒープへ
./escape_analysis_test.go:19:17: ... argument does not escape
./escape_analysis_test.go:19:17: len(data) escapes to heap

BenchmarkTooLarge-8   	  150694	      7932 ns/op	   81920 B/op	       1 allocs/op
```

- インライン展開された関数内で確保したメモリはエスケープしない
    - allocate()でmake()でメモリ確保してreturnしているが、can inline allocate インライン展開されているためスタック領域が割り当てられる

```go
func child(data []int) {
   if len(data) == 0 {
     fmt.Fprintln(os.Stderr, len(data))
   }
}

func allocate() []int { 
    return make([]int, 1) 
}

func BenchmarkAllocateInFunction(b *testing.B) {
    for i := 0; i < b.N; i++ {
        data := allocate()
        child(data)
    }
}
```

```
$ go test escape_analysis_test.go -bench . -gcflags=-m -benchmem
# command-line-arguments [command-line-arguments.test]
./escape_analysis_test.go:10:6: can inline child
./escape_analysis_test.go:16:6: can inline allocate // BenchmarkAllocateInFunction内でインライン展開される
./escape_analysis_test.go:39:25: inlining call to allocate
./escape_analysis_test.go:40:17: inlining call to child
./escape_analysis_test.go:10:15: data does not escape
./escape_analysis_test.go:12:18: ... argument does not escape
./escape_analysis_test.go:12:33: len(data) escapes to heap
./escape_analysis_test.go:16:36: make([]int, 1) escapes to heap 
./escape_analysis_test.go:37:34: b does not escape
./escape_analysis_test.go:39:25: make([]int, 1) does not escape // スタックへ
./escape_analysis_test.go:40:17: ... argument does not escape
./escape_analysis_test.go:40:17: len(data) escapes to heap

BenchmarkAllocateInFunction-8   	1000000000	         0.502 ns/op	       0 B/op	       0 allocs/op
```

- new()でもスタック使われることはある

```
type candy struct {
	color string
	count int
}

func main() {
	_ = child()
}

func child() *candy {
	a := new(candy)
	a.color = "red"
	a.count = 3
	return &candy{}
}
```

```
$ go build -gcflags -m test.go
# command-line-arguments
./test.go:12:6: can inline child
./test.go:8:6: can inline main
./test.go:9:14: inlining call to child
./test.go:9:14: new(candy) does not escape 
./test.go:9:14: &candy literal does not escape
./test.go:13:11: new(candy) does not escape　// new()でもスコープが関数内のみの場合はスタック
./test.go:16:10: &candy literal escapes to heap // 関数の外で扱われる（と判断された）ためエスケープ
```
 