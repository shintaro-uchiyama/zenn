---
title: "Go 1.18のgenericsとfuzzingに最低限触れといた"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["golang", "fuzzing", "generics"]
published: false
---

# 概要

Twitter眺めてたら  
https://go.dev/blog/tutorials-go1.18  
が流れてきたので来るリリースに向けて最低限触っといた

# 事前準備

まだGo 1.18はbeta版なので  
それ用のやつをインストールせねばならぬのじゃ

```zsh
$ go install golang.org/dl/go1.18beta1@latest
go: downloading golang.org/dl v0.0.0-20220106205509-1eec60721618
$
$ go1.18beta1 download
Downloaded   0.0% (    16384 / 143162528 bytes) ...
Downloaded 100.0% (143162528 / 143162528 bytes)
Unpacking /Users/su/sdk/go1.18beta1/go1.18beta1.darwin-amd64.tar.gz ...
Success. You may now run 'go1.18beta1'
$
$ go1.18beta1 version
go version go1.18beta1 darwin/amd64
$
$ alias go=go1.18beta1
$ go version
go version go1.18beta1 darwin/amd64
```

# generics

全部ここに書いてある  
https://go.dev/doc/tutorial/generics

## 実装

これまでなら`int64`と`float64`それぞれ関数定義しなきゃだったけど  
genericsでKやらVやら定義できまっせ。とそれだけ  

genericsガンガン使ってこう！！

```go: main.go
package main

import "fmt"

type Number interface {
	int64 | float64
}

func main() {
	// Initialize a map for the integer values
	ints := map[string]int64{
		"first": 34,
		"second": 12,
	}

	// Initialize a map for the float values
	floats := map[string]float64{
		"first": 35.98,
		"second": 26.99,
	}

	fmt.Printf("Generic Sums with Constraint: %v and %v\n",
		SumNumbers[string, int64](ints),
		SumNumbers[string, float64](floats))
}

// SumNumbers sums the values of map m. Its supports both integers
// and floats as map values.
func SumNumbers[K comparable, V Number](m map[K]V) V {
	var s V
	for _, v := range m {
		s += v
	}
	return s
}
```

実行結果

```zsh
$ go run main.go
Generic Sums with Constraint: 46 and 62.97
```

## 実装（省略系）

generics宣言された関数を呼び出す際に  
明示的に型を定義しなくても、理解してくれるみたい

```go: main.go
	fmt.Printf("Generic Sums with Constraint: %v and %v\n",
		SumNumbers(ints),
		SumNumbers(floats),
	)
```

# fuzzing

unitテストならぬfuzzテストができるみたい  
https://go.dev/doc/tutorial/fuzz

unitテストでは自分で用意したデータを通して  
実行した結果が準備した期待値とあってるか検証するけど  
fuzzテストでは事前にデータ準備しなくても  
いろんなデータを突っ込んで自身でケアしきれない範囲もテストしてくれる！  

## 準備

`go test`実行するのに`go mod init`しとかねば  
遊びで触るので適当なmodule名で

```zsh
$ go mod init example/fuzz
go: creating new go.mod: module example/fuzz
```

## 実装（不具合あり）

この文字列を逆順にする関数をテストする  
'abc' が入力なら 'cba' が出力で帰ってくる感じ

```go: main.go
package main

import (
	"fmt"
)

func main() {
    input := "The quick brown fox jumped over the lazy dog"
    rev := Reverse(input)
    doubleRev := Reverse(rev)
    fmt.Printf("original: %q\n", input)
    fmt.Printf("reversed: %q\n", rev)
    fmt.Printf("reversed again: %q\n", doubleRev)
}

func Reverse(s string) string {
    b := []byte(s)
    for i, j := 0, len(b)-1; i < len(b)/2; i, j = i+1, j-1 {
        b[i], b[j] = b[j], b[i]
    }
    return string(b)
}
```

## Fuzzテスト

単体テストと書きっぷりは似てる  
単体テストではTestXxxって書くとこをFuzzXxxって書いたり  
`*testing.T`じゃなくて`*testing.F`使ったり微妙に違うくらい

1. 2回reverseして元の文字列に戻るか
2. utf8として適切な文字列か検証

の2つを検証してる

```go: reverse_test.go
package main

import (
    "testing"
    "unicode/utf8"
)

func FuzzReverse(f *testing.F) {
    testcases := []string{"Hello, world", " ", "!12345"}
    for _, tc := range testcases {
        f.Add(tc)  // Use f.Add to provide a seed corpus
    }
    f.Fuzz(func(t *testing.T, orig string) {
        rev := Reverse(orig)
        doubleRev := Reverse(rev)
        if orig != doubleRev {
            t.Errorf("Before: %q, after: %q", orig, doubleRev)
        }
        if utf8.ValidString(orig) && !utf8.ValidString(rev) {
            t.Errorf("Reverse produced invalid UTF-8 string %q", rev)
        }
    })
}
```

実際に実行してみると

```zsh
$ go test -v
=== RUN   FuzzReverse
=== RUN   FuzzReverse/seed#0
=== RUN   FuzzReverse/seed#1
=== RUN   FuzzReverse/seed#2
--- PASS: FuzzReverse (0.00s)
    --- PASS: FuzzReverse/seed#0 (0.00s)
    --- PASS: FuzzReverse/seed#1 (0.00s)
    --- PASS: FuzzReverse/seed#2 (0.00s)
PASS
ok  	example/fuzz	0.178s
```

4つのデータがPASSしてる！  
...これunit testと何が違うの？って思うけど

こんな感じで`-fuzz` flagをつけて実行すると

```zsh
$ go test -fuzz=Fuzz -v
=== FUZZ  FuzzReverse
fuzz: elapsed: 0s, gathering baseline coverage: 0/3 completed
fuzz: elapsed: 0s, gathering baseline coverage: 3/3 completed, now fuzzing with 8 workers
fuzz: minimizing 32-byte failing input file
fuzz: elapsed: 0s, minimizing
--- FAIL: FuzzReverse (0.22s)
    --- FAIL: FuzzReverse (0.00s)
        reverse_test.go:20: Reverse produced invalid UTF-8 string "\xae\xcd"

    Failing input written to testdata/fuzz/FuzzReverse/9b504024244a9afd5840f8f96d7a0cfd880663007b6495535a0e3c28bdea6241
    To re-run:
    go test -run=FuzzReverse/9b504024244a9afd5840f8f96d7a0cfd880663007b6495535a0e3c28bdea6241
FAIL
exit status 1
FAIL	example/fuzz	0.561s
```

UTF-8として適切な文字列じゃないって怒られた！！  

`-fuzz` flagつけないと`f.Add(tc)`で突っ込んだ自分のテストデータだけテストして  
flagをつけると勝手にいろんなデータでテストが走り出す  

そんでこのFuzzテストはこけるまで永遠に続く  
ので、`-fuzztime 5s` 引数で何秒実行するか指定が必要みたい

## 実装を直す

何回か走らせて色々怒られたのを直すとこんな感じ

```go: main.go
package main

import (
	"errors"
	"fmt"
	"log"
	"unicode/utf8"
)

func main() {
	input := "The quick brown fox jumped over the lazy dog"
	rev, err := Reverse(input)
	if err != nil {
		log.Fatal(err)
	}
	doubleRev, err := Reverse(rev)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("original: %q\n", input)
	fmt.Printf("reversed: %q\n", rev)
	fmt.Printf("reversed again: %q\n", doubleRev)
}

func Reverse(s string) (string, error) {
	if !utf8.ValidString(s) {
		return s, errors.New("input is not valid UTF-8")
	}
	r := []rune(s)
	for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
		r[i], r[j] = r[j], r[i]
	}
	return string(r), nil
}
```

## Fuzzテスト再実行

`Reverse`がエラーならnil返すか`t.Skip()`する

```go: reverse_test.go
package main

import (
	"testing"
	"unicode/utf8"
)

func FuzzReverse(f *testing.F) {
	testcases := []string{"Hello, world", " ", "!12345"}
	for _, tc := range testcases {
		f.Add(tc) // Use f.Add to provide a seed corpus
	}
	f.Fuzz(func(t *testing.T, orig string) {
		rev, err1 := Reverse(orig)
		if err1 != nil {
			return
		}
		doubleRev, err2 := Reverse(rev)
		if err2 != nil {
			return
		}
		if orig != doubleRev {
			t.Errorf("Before: %q, after: %q", orig, doubleRev)
		}
		if utf8.ValidString(orig) && !utf8.ValidString(rev) {
			t.Errorf("Reverse produced invalid UTF-8 string %q", rev)
		}
	})
}
```

これで再度実行すると

```zsh
$ go test -fuzz=Fuzz -fuzztime 5s -v
=== FUZZ  FuzzReverse
fuzz: elapsed: 0s, gathering baseline coverage: 0/37 completed
fuzz: elapsed: 0s, gathering baseline coverage: 37/37 completed, now fuzzing with 8 workers
fuzz: elapsed: 3s, execs: 576016 (191999/sec), new interesting: 0 (total: 35)
fuzz: elapsed: 5s, execs: 975339 (190528/sec), new interesting: 0 (total: 35)
--- PASS: FuzzReverse (5.10s)
PASS
ok  	example/fuzz	5.204s
```

okで成功してる！！

# まとめ

どっちも速攻で使えそうなやつ  
正式リリースされたら早速使っていこう！