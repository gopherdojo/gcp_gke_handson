# GKEにFrontend用のDeploymentを作成

## 操作するGKE Clusterを設定

`gcloud container clusters get-credentials` で操作するGKE Clusterを指定します。

```
gcloud container clusters get-credentials handson-cluster --zone us-central1-a --project {your project id}
```

## Deploymentを作成

### yamlを作成

hellotime container imageをPodとして持つReplicaを1つ宣言するシンプルなDeploymentを作成します。
{your GCR image path} のところをPart2で作成したcontainer imageのpathに差し替えてください。
例えば `gcr.io/souzoh-demo-gcp-001/sinmetal/hellotime/manual:v1.0.0` のような値です。

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
      - image: {your GCR image path}
        name: hellotime-node
```

### yamlの適用

`kubectl apply` を利用して、作成したdeploymentを適用します。

```
kubectl apply -f hellotime-deployment.yaml
```

`kubectl get pods` でPod一覧を取得できます。
STATUSがRunningになったら、動き始めています。

```
kubectl get pods

NAME                              READY     STATUS    RESTARTS   AGE
hellotime-node-5d56d45ddc-2lnf7   1/1       Running   0          3m
```

STATUSがRunningになったら、Podのログを確認して、Hello {Time} が出力されているかを確認しましょう。
`hellotime-node-5d56d45ddc-2lnf7` を自分のPodの名前に置き換えて実行してください。

```
kubectl logs -f hellotime-node-5d56d45ddc-2lnf7

Hello 2018-10-18 11:18:01.00845593 +0000 UTC m=+0.000262575
Hello 2018-10-18 11:18:06.009404119 +0000 UTC m=+5.001210752
Hello 2018-10-18 11:18:11.009586576 +0000 UTC m=+10.001393373
Hello 2018-10-18 11:18:16.009794563 +0000 UTC m=+15.001601221
Hello 2018-10-18 11:18:21.009952582 +0000 UTC m=+20.001759346
Hello 2018-10-18 11:18:26.010100972 +0000 UTC m=+25.001907619
Hello 2018-10-18 11:18:31.010266159 +0000 UTC m=+30.002072772
Hello 2018-10-18 11:18:36.010473825 +0000 UTC m=+35.002280460
Hello 2018-10-18 11:18:41.010641168 +0000 UTC m=+40.002447827
Hello 2018-10-18 11:18:46.010816821 +0000 UTC m=+45.002623461
Hello 2018-10-18 11:18:51.011011376 +0000 UTC m=+50.002818013
Hello 2018-10-18 11:18:56.011205904 +0000 UTC m=+55.003012553
```