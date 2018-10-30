# GKEにBackend用のDeploymentを作ろう

## Backend用のGoのバイナリを内包したContainer Imageを作ろう

HTTP Requestを受け取り Hello Backend {Time}を返す、シンプルなアプリケーションです。

```
git clone git@github.com:sinmetal/backendhellotime.git
```

```
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"
)

func main() {
	http.HandleFunc("/", handler)

	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello Backend %s", time.Now())
}
```

### Cloud Build

#### Dockerfileの準備

Goのアプリケーションを実行するだけのシンプルなDockerfileを作成します。

```
FROM alpine:3.8
RUN apk add --no-cache ca-certificates
RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2
COPY ./backendhellotime /backendhellotime
ENTRYPOINT ["/backendhellotime"]
```

#### cloudbuild.yamlの準備

Go1.11でGoをビルドした後、Docker Buildを行うCloud Buildの設定ファイルを作成します。

```
steps:
  - name: 'golang:1.11.1-stretch'
    entrypoint: 'go'
    args: ['build', '.']
    env: ['GO111MODULE=on']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '--tag=gcr.io/$PROJECT_ID/sinmetal/backendhellotime/$BRANCH_NAME:$COMMIT_SHA', '.']
images: ['gcr.io/$PROJECT_ID/sinmetal/backendhellotime/$BRANCH_NAME:$COMMIT_SHA']
```

#### Cloud Buildの実行

`gcloud builds submit` を利用して、Google Cloud BuildにBuild Jobを投げます。
BRANCH_NAME,COMMIT_SHAは [Build Trigger](https://cloud.google.com/cloud-build/docs/running-builds/automate-builds) のための変数なので、今回は手動で設定します。

```
gcloud builds submit --config cloudbuild.yaml --substitutions BRANCH_NAME=manual,COMMIT_SHA=v1.0.0
```

#### Build 結果の確認

STATUSが `SUCCESS` になってれば、成功

```
gcloud builds list --limit 5
```

## Deploymentを作成

### yamlを作成

backendhellotime container imageをPodとして持つReplicaを2つ宣言するシンプルなDeploymentを作成します。
{your GCR image path} のところをPart2で作成したcontainer imageのpathに差し替えてください。
例えば `gcr.io/souzoh-demo-gcp-001/sinmetal/backendhellotime/manual:v1.0.0` のような値です。

``` backendhellotime-deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: backendhellotime-node
  name: backendhellotime-node
spec:
  replicas: 2
  template:
    metadata:
      labels:
        name: backendhellotime-node
    spec:
      containers:
      - image: {your GCR image path}
        name: backendhellotime-node
```

### yamlの適用

`kubectl apply` を利用して、作成したdeploymentを適用します。

```
kubectl apply -f backendhellotime-deployment.yaml
```

`kubectl get pods` でPod一覧を取得して、backendhellotimeが増えていることを確認します。

```
kubectl get pods

NAME                                     READY     STATUS    RESTARTS   AGE
backendhellotime-node-6d86cb698d-txwhg   1/1       Running   0          6m
hellotime-node-7dbc587dcd-4l7w2          1/1       Running   0          6m
```

## Serviceを作成

backendhellotimeはhellotimeから呼ばれるものなので、Serviceを作成します。
backendhellotimeのアプリケーションは8080をListenしているので、 `NodePort:8080` を指定します。

### yamlを作成

selectorとして `backendhellotime-node` を指定します。
portはNodeもContainer Imageも両方共8080をListenとします。
Container上のアプリケーションが8080以外をListenしている場合は、targetPortをそのportに変更します。

``` backendhellotime-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: backendhellotime-service
  name: backendhellotime-service
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    name: backendhellotime-node
```

### yamlを適用

`kubectl apply` を利用して、作成したserviceを適用します。

```
kubectl apply -f backendhellotime-service.yaml
```

`kubectl get services` でService一覧を取得して、backendhellotime-serviceが作成されていることを確認します。

```
kubectl get services

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
backendhellotime-service   NodePort    10.11.248.132   <none>        8080:31029/TCP   1d
kubernetes                 ClusterIP   10.11.240.1     <none>        443/TCP          1d
```

## hellotimeから、backendhellotimeを呼び出す

ServiceはGKE内部のDNSを利用して、アクセスすることができる。
以下は同じClusterにいるServiceを呼び出す時のDNS名。
`svc` はServiceであることを指している。

`http://{resource-name}.{namespace}.svc.cluster.local`

### hellotimeで、backendhellotimeを呼ぶようにする

以下のようにhellotimeの中身を変えてみよう。

```
func main() {
	for {
		res, err := http.Get("http://backendhellotime-service.default.svc.cluster.local:8080")
		if err != nil {
			fmt.Println(err)
		}
		defer res.Body.Close()
		b, err := ioutil.ReadAll(res.Body)
		if err != nil {
			fmt.Println(err)
		}
		fmt.Println(string(b))

		time.Sleep(5 * time.Second)
	}
}
```

### Cloud Buildの実行

`gcloud builds submit` を利用して、Google Cloud BuildにBuild Jobを投げます。
BRANCH_NAME,COMMIT_SHAは [Build Trigger](https://cloud.google.com/cloud-build/docs/running-builds/automate-builds) のための変数なので、今回は手動で設定します。
前回ビルドしたものと区別するために、 `v1.0.1` とします。

```
gcloud builds submit --config cloudbuild.yaml --substitutions BRANCH_NAME=manual,COMMIT_SHA=v1.0.1
```

#### Build 結果の確認

STATUSが `SUCCESS` になってれば、成功

```
gcloud builds list --limit 5
```

### Deployment yamlの編集

hellotimeのDeploymentのyamlのimageを新しく作成したhellotimeのv.1.0.1のものに差し替えます。

``` hellotime-deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: hellotime-node
  name: hellotime-node
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: hellotime-node
    spec:
      containers:
      - image: {your GCR image path} // v.1.0.1 のものに変更する
        name: hellotime-node
```

#### yamlの適用

`kubectl apply` を利用して、作成したdeploymentを適用します。

```
kubectl apply -f hellotime-deployment.yaml
```

`kubectl get pods` でPod一覧を取得します。
apply後、すぐに確認すれば、新しいVersionのPodが追加され、古いPodが削除されていくのを見ることができます。

```
kubectl get pods

NAME                                     READY     STATUS    RESTARTS   AGE
backendhellotime-node-6d86cb698d-txwhg   1/1       Running   0          1m
hellotime-node-7dbc587dcd-4l7w2          1/1       Running   0          6m
```

Deployできたら、hellotimeのログを確認して、backendhellotimeから返ってきたレスポンスが出力されているか確認してみよう。

```
Hello Backend 2018-10-27 03:48:03.755531262 +0000 UTC m=+2963.201806804
Hello Backend 2018-10-27 03:48:08.756320157 +0000 UTC m=+2968.202595636
Hello Backend 2018-10-27 03:48:13.757316924 +0000 UTC m=+2973.203592405
Hello Backend 2018-10-27 03:48:18.758167078 +0000 UTC m=+2978.204442558
```