---
created: 2026-01-12T10:44:06+01:00
modified: 2026-01-12T10:44:06+01:00
---

1. Install VM K8s Stack
```bash
kubectl create namespace vm
helm install vmks vm/victoria-metrics-k8s-stack -f 00-k8s-stack.yaml -n vm
```

2. Install new operator
```bash
kubectl apply -f ~/src/github.com/VictoriaMetrics/operator/config/rbac/role.yaml
kubectl create clusterrolebinding operator-binding --clusterrole=operator --serviceaccount=vm:vmks-victoria-metrics-operator
kubectl -n vm apply -f ~/src/github.com/VictoriaMetrics/operator/config/crd/overlay/crd.specless.yaml
kubectl apply -f 01-operator-image.yaml
```

3. Deploy VMDistributedCluster example
```bash
kubectl create namespace vmdistributed
kubectl apply -f 02-vmdistributedcluster.yaml
kubectl -n vmdistributed wait --for=jsonpath='{.status.updateStatus}'=operational vmdistributedcluster/vmd --timeout=30m
``` 

4. Install prometheus-benchmark
```bash
cd ~/src/github.com/VictoriaMetrics/prometheus-benchmark
git checkout vmdistributedcr
make install
```

5. Add Grafana datasources:

* Global 
  http://global-read-vmauth.vm.svc.cluster.local:8427/select/0/prometheus
* US East 1
  http://vmselect-vmcluster-us-east-1.vm.svc.cluster.local:8481/select/0/prometheus
* US West 1
  http://vmselect-vmcluster-us-west-2.vm.svc.cluster.local:8481/select/0/prometheus
* EU West 1
  http://vmselect-vmcluster-eu-west-1.vm.svc.cluster.local:8481/select/0/prometheus

7. Update clusters
```yaml
spec:
  zones:
    globalOverrideSpec:
      vminsert:
        replicaCount: 4
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1024Mi
      vmstorage:
        replicaCount: 4
        resources:
          requests:
            memory: 1Gi
          limits:
            memory: 3Gi
```

```bash
kubectl patch vmdistributedcluster vmd -n vmdistributed --type json --patch-file 07-vmd-extraargs-patch.json
```

8. After cluster update the following metrics show the process:

VMAgent dashboard:
![Pic1](pic1.png)

VMCluster Global dashboard - no disruption in write path:
![Pic2](pic2.png)

VMCluster Global dashboard - no disruption in read path:
![Pic3](pic3.png)

----
EU West cluster as a baseline:
![Pic4](pic4-eu-west-read.png)

----
US East read is droping right before update:
![Pic4](pic4-us-east-read.png)

----
US West read is droping right before update:
![Pic4](pic4-us-west-read.png)

9. Upgrade versions

```yaml
spec:
  zones:
    vmclusters:
    - name: vmcluster-us-east-1
      spec:
        clusterVersion: v1.131.0-cluster
    - name: vmcluster-us-west-2
      spec:
        clusterVersion: v1.131.0-cluster
```

```bash
kubectl patch vmdistributedcluster vmd -n vmdistributed --type merge --patch-file 09-vmd-version-patch.yaml
```

10. Distributed chart

Installation:
```bash
kubectl create namespace vm-distributed-chart
helm install vmd vm/victoria-metrics-distributed -f 06-distributed-chart-values.yaml -n vm-distributed-chart
```

Setup load test
```bash
cd ~/src/github.com/VictoriaMetrics/prometheus-benchmark
git checkout distributed-chart
make install
```
