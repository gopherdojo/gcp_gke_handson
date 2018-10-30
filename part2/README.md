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

## Cloud Build

### Dockerfileの準備

Goのアプリケーションを実行するだけのシンプルなDockerfileを作成します。

```
FROM alpine:3.8
RUN apk add --no-cache ca-certificates
RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2
COPY ./hellotime /hellotime
ENTRYPOINT ["/hellotime"]
```

### cloudbuild.yamlの準備

Go1.11でGoをビルドした後、Docker Buildを行うCloud Buildの設定ファイルを作成します。

```
steps:
  - name: 'golang:1.11.1-stretch'
    entrypoint: 'go'
    args: ['build', '.']
    env: ['GO111MODULE=on']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '--tag=gcr.io/$PROJECT_ID/sinmetal/hellotime/$BRANCH_NAME:$COMMIT_SHA', '.']
images: ['gcr.io/$PROJECT_ID/sinmetal/hellotime/$BRANCH_NAME:$COMMIT_SHA']
```

### Cloud Buildの実行

`gcloud builds submit` を利用して、Google Cloud BuildにBuild Jobを投げます。
BRANCH_NAME,COMMIT_SHAは [Build Trigger](https://cloud.google.com/cloud-build/docs/running-builds/automate-builds) のための変数なので、今回は手動で設定します。

```
gcloud builds submit --config cloudbuild.yaml --substitutions BRANCH_NAME=manual,COMMIT_SHA=v1.0.0
```

### Build 結果の確認

STATUSが `SUCCESS` になってれば、成功

```
gcloud builds list --limit 5
```