# 後片付け

## GKE Clusterの削除

```
gcloud container clusters delete "handson-cluster" --zone "us-central1-a"
```

Cluster Listを確認して、削除されていることを確認

```
gcloud container clusters list
```

## GCP ProjectのShut Down

GCP ProjectをShutdownして、完全に削除する。
Container RegistryにあるContainer Image含めてすべて削除する。
※ GCP Project IDは再利用できないので注意。

![Project Shutdown](https://github.com/sinmetal/gke_handson/blob/master/resources/project_shutdown.png)