# Goのバイナリを内包したContainer Imageを作ろう

## sample codeの取得

```
git clone git@github.com:sinmetal/hellotime.git
```

Hello {Time}を標準出力に5sec毎に出力するシンプルなアプリケーションです。

```
package main

import (
	"fmt"
	"time"
)

func main() {
	for {
		fmt.Printf("Hello %s\n", time.Now())
		time.Sleep(5 * time.Second)
	}
}
```

## Dockerfileの準備

Goのアプリケーションを実行するだけのシンプルなDockerfileを作成します。

```
FROM alpine:3.7
RUN apk add --no-cache ca-certificates
RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2
COPY gopath/bin/hellotime /hellotime
CMD ["/hellotime"]
```

## cloudbuild.yamlの準備

```
steps:
  - name: 'gcr.io/cloud-builders/go'
    args: ['test', './...']
    env: ['PROJECT_ROOT=github.com/sinmetal/hellotime']
  - name: 'gcr.io/cloud-builders/go'
    args: ['install', '-a', '-ldflags', '-s', '-installsuffix', 'cgo', 'github.com/sinmetal/hellotime']
    env: [
      'PROJECT_ROOT=github.com/sinmetal/hellotime',
      'CGO_ENABLED=0',
      'GOOS=linux'
    ]
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '--tag=gcr.io/$PROJECT_ID/sinmetal/hellotime:v1.0.0', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ["push", "gcr.io/$PROJECT_ID/sinmetal/hellotime:v1.0.0"]

```

## Cloud Buildの実行

```
gcloud builds submit --config cloudbuild.yaml .
```