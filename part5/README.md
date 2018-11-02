# Stackdriver Traceの追加

frontend pod, backend pod それぞれで処理時間をトレースできるようにしてみます。

## frontendにTraceを追加

main関数にtraceのexporterを登録する処理を追加します。
ProjectIDはひとまず自分のGCP ProjectIDをハードコードしてもよいですし、 https://github.com/gopherdojo/gcp_hellotime/blob/add-stackdriver-trace/metadata.go のように実行環境のProjectIDを取得するようにしてもかまいません。

``` main()
exporter, err := stackdriver.NewExporter(stackdriver.Options{
	ProjectID: projectID,
})
if err != nil {
	panic(err)
}
trace.RegisterExporter(exporter)
```

process関数を編集して、Spanを追加するのと、backendへのリクエストにSpanが伝播するようにします。

```
func process(ctx context.Context) (string, error) {
	ctx, span := trace.StartSpan(ctx, "/process", trace.WithSampler(trace.AlwaysSample())) // 通常traceはサンプリングされますが、5sec毎に定期実行されている場合、ほとんど取得されないので、すべて取得するようにオプションを指定します。
	defer span.End()

	client := &http.Client{
		Transport: &ochttp.Transport{
			// Use Google Cloud propagation format.
			Propagation:    &propagation.HTTPFormat{},
			FormatSpanName: formatSpanName,
		},
	}

	req, err := http.NewRequest("GET", "http://backendhellotime-service.default.svc.cluster.local:8080", nil)
	if err != nil {
		return "", err
	}

	// The trace ID from the incoming request will be
	// propagated to the outgoing request.
	req = req.WithContext(ctx)

	// The outgoing request will be traced with r's trace ID.
	res, err := client.Do(req)
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	b, err := ioutil.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(b), nil
}

func formatSpanName(r *http.Request) string {
	return r.URL.Host
}
```

できあがったサンプルコードは https://github.com/gopherdojo/gcp_hellotime/tree/add-stackdriver-trace にあります。

### Cloud Buildの実行

`gcloud builds submit` を利用して、Google Cloud BuildにBuild Jobを投げます。
BRANCH_NAME,COMMIT_SHAは [Build Trigger](https://cloud.google.com/cloud-build/docs/running-builds/automate-builds) のための変数なので、今回は手動で設定します。
前回ビルドしたものと区別するために、 `v1.0.2` とします。

```
gcloud builds submit --config cloudbuild.yaml --substitutions BRANCH_NAME=manual,COMMIT_SHA=v1.0.2
```

#### Build 結果の確認

STATUSが `SUCCESS` になってれば、成功

```
gcloud builds list --limit 5
```

### Deployment yamlの編集

hellotimeのDeploymentのyamlのimageを新しく作成したhellotimeのv.1.0.2のものに差し替えます。

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
      - image: {your GCR image path} // v.1.0.2 のものに変更する
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

## backendにTraceを追加

backendにもtraceを追加します。
frontendから渡ってきているTraceIDに乗せるために "go.opencensus.io/plugin/ochttp" を利用して、http handlerをラップしています。

```
import (
	"fmt"
	"log"
	"net/http"
	"time"

	"contrib.go.opencensus.io/exporter/stackdriver"
	"contrib.go.opencensus.io/exporter/stackdriver/propagation"
	"go.opencensus.io/plugin/ochttp"
	"go.opencensus.io/trace"
)

func main() {
	projectID, err := GetProjectID()
	if err != nil {
		panic(err)
	}
	// Create and register a OpenCensus Stackdriver Trace exporter.
	exporter, err := stackdriver.NewExporter(stackdriver.Options{
		ProjectID: projectID,
	})
	if err != nil {
		log.Fatal(err)
	}
	trace.RegisterExporter(exporter)

	trace.ApplyConfig(trace.Config{DefaultSampler: trace.AlwaysSample()}) // defaultでは10,000回に1回のサンプリングになっているが、リクエストが少ないと出てこないので、とりあえず全部出す

	server := &http.Server{
		Addr: ":8080",
		Handler: &ochttp.Handler{
			Handler:        http.DefaultServeMux,
			Propagation:    &propagation.HTTPFormat{},
			FormatSpanName: formatSpanName,
		},
	}

	http.Handle("/", ochttp.WithRouteTag(func() http.Handler { return http.HandlerFunc(handler) }(), "/backendhellotime/"))
	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello Backend %s", time.Now())
}

func formatSpanName(r *http.Request) string {
	return fmt.Sprintf("/backendhellotime%s", r.URL.Path)
}
```

できあがったサンプルコードは https://github.com/gopherdojo/gcp_backendhellotime/tree/add-stackdriver-trace にあります。

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

backendhellotimeのDeploymentのyamlのimageを新しく作成したhellotimeのv.1.0.1のものに差し替えます。

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
      - image: {your GCR image path}  // v.1.0.1 のものに変更する
        name: backendhellotime-node
```

#### yamlの適用

`kubectl apply` を利用して、作成したdeploymentを適用します。

```
kubectl apply -f backendhellotime-deployment.yaml
```

`kubectl get pods` でPod一覧を取得します。
apply後、すぐに確認すれば、新しいVersionのPodが追加され、古いPodが削除されていくのを見ることができます。

```
kubectl get pods

NAME                                     READY     STATUS    RESTARTS   AGE
backendhellotime-node-6d86cb698d-txwhg   1/1       Running   0          1m
hellotime-node-7dbc587dcd-4l7w2          1/1       Running   0          6m
```

## Stackdriver Traceの確認

https://console.cloud.google.com/traces のページでトレースが出力されているかを確認します。

![Stackdriver Trace](https://github.com/sinmetal/gke_handson/blob/master/part5/stackdriver-trace.png)