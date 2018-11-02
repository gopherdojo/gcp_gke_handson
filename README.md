# gke-hands-on

[Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) のハンズオンです。

## Goal

GKE上にFrontendとBackendの2つのアプリケーションをDeployし、通信を行う。

![Goal](https://github.com/sinmetal/gke_handson/blob/master/resources/goal.png)

### GCP Resources

* [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/)
* [Google Cloud Container Registry](https://cloud.google.com/container-registry/)
* [Google Cloud Build](https://cloud.google.com/cloud-build/)

## Curriculum

### Part1 Clusterを作成

ハンズオンで利用するGKEのClusterを作成します。

### Part2 Goのバイナリを内包したContainer Imageの作成

GKE ClusterにDeployするためのContainer Imageを [Google Cloud Build](https://cloud.google.com/cloud-build/) を利用して作成します。

### Part3 Frontend用のDeploymentを作成

Frontend用のDeploymentを作成して、GKE Cluster上で動かします。

### Part4 Backend用のDeployment, Serviceを作成

Backend用のDeploymentとServiceを作成して、GKE Cluster上で動かします。

### part5 Stackdriver Traceの追加

Frontend, BackendにStackdriver Traceを追加します。

### Part99 後片付け

ハンズオンで作成したResourceを削除します。