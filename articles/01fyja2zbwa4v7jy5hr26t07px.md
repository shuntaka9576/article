---
title: "久しぶりにGoを書くときにやること(随時更新)"
type: "note"
description: ""
thumbnail: ""
publish: false
---

# はじめに
Goは2019年頃からCLIを作りたくなる度に環境構築して、作って、放置してを繰り返しているので、その度記憶を遡るのが億劫なので記事にすることにした。

# 環境
OSはインテル Mac OS Bug Surを想定

# 導入

go 1.18のインストール。brewだと妙に古いときがあるので、好きなバージョンを[Go公式](https://go.dev/dl/)からダウンロードして、ぽちぽちインストールが楽でミスがすくないのでは。


# プロジェクト作成

```bash
$ echo $GO111MODULE
on
```

```bash
$ export PROJECT_NAME=[プロジェクト名]
$ export REPO_OWNER=shuntaka9576

$ cd $PROJECT_NAME
$ mkdir $PROJECT_NAME
$ go mod init $REPO_OWNER/$PROJECT_NAME

# CLIを作る場合以下のディレクトリ構成が多いような(自由で構いません)
$ mkdir ./cmd/$PROJECT_NAME
$ touch ./cmd/$PROJECT_NAME/main.go
```

```go:cmd/$PROJECT_NAME/main.go
package main

import "fmt"

func main() {
	fmt.Println("Hello")
}
```

```bash: 動作することを確認
go run ./cmd/$PROJECT_NAME/main.go
```

# ライブラリの導入
`go.sum`が作成されて、`go.mod`が更新されていることを確認する

```bash
$ go get github.com/k0kubun/pp
go: added github.com/k0kubun/pp v3.0.1+incompatible
go: added github.com/mattn/go-colorable v0.1.12
go: added github.com/mattn/go-isatty v0.0.14
go: added golang.org/x/sys v0.0.0-20210927094055-39ccf1dd6fa6
```

ライブラリが動作することを確認する
```go:cmd/$PROJECT_NAME/main.go
package main

import (
	"github.com/k0kubun/pp"
)

func main() {
	pp.Println("Hello")
}
```

GO111MODULEが`on`なので、`~/go/pkg/mod/github.com/k0kubun/pp@v3.0.1+incompatible` にインストールされていることを確認


# CI

TODO


# ライブラリ

## Markdown関連
### yuin/goldmark-highlighting

* `github.com/alecthomas/chroma`と依存

## WebSocket

* [gorilla/websocket](https://github.com/gorilla/websocket/blob/master/examples/echo/client.go)
