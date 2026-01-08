VMDistributed demo
====

1. Delete default VMCluster
```
kubectl -n vm delete vmcluster vmks
```

2. Install new operator
```
kubectl apply -f ~/src/github.com/VictoriaMetrics/operator/config/rbac/role.yaml
kubectl create clusterrolebinding operator-binding --clusterrole=operator --serviceaccount=vm:vmks-victoria-metrics-operator
kubectl -n vm apply -f ~/src/github.com/VictoriaMetrics/operator/config/crd/overlay/crd.specless.yaml
kubectl apply -f 01-operator-image.yaml
```

3. Deploy VMDistributedCluster example
```
kubectl apply -f 02-vmdistributedcluster.yaml
kubectl -n vm wait --for=jsonpath='{.status.updateStatus}'=operational vmdistributedcluster/vmd --timeout=30m
``` 

4. Reconfigure VMAgent to send k8s and cluster metrics
```
kubectl apply -f 03-vmagent.yaml
```

4. Install prometheus-benchmark
```
cd https://github.com/VictoriaMetrics/prometheus-benchmark
make install
```
