## [Nodeset 오토스케일링을 위한 KEDA 설치](https://slinky.schedmd.com/projects/slurm-operator/en/release-1.0/usage/autoscaling.html) ##
slrum 의 오토 스케일링, 정확하게는 slurm 파드의 오토 스케일링은 KEDA, Prometheus, Metric 서버 기반으로 동작을 한다. slurm job 이 생성되었을때 Karpenter 까지 연결되는 작동 메커니즘은 다음과 같다. 

#### 작동 메커니즘: Slurm → KEDA → Pod → Karpenter ####
* Slurm Job 발생: 사용자가 sbatch로 작업을 던집니다. 현재 컴퓨팅 노드가 0개이므로 작업은 PENDING 상태가 됩니다.
* KEDA의 감지: KEDA가 Slurm의 메트릭(예: squeue 대기 수)을 확인하고, NodeSet(또는 Deployment)의 Replica를 0에서 1 이상으로 올립니다.
* Pending Pod 생성: NodeSet이 파드를 생성하려고 하지만, 수용할 노드가 없으므로 파드는 Pending 상태로 쿠버네티스 스케줄러에 머뭅니다.
* Karpenter 출동: Karpenter가 이 Pending 파드를 발견하고, "아, 파드가 필요로 하는 리소스(CPU/GPU 등)에 맞는 실제 노드를 프로비저닝해야겠군!" 하며 인스턴스를 띄웁니다.
노드가 준비되면 파드가 배치되고, Slurm Worker가 활성화되어 작업을 수행합니다.

### KEDA 설치 ###

프로메테우스를 설치한다. 
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  --set 'installCRDs=true' \
  --namespace prometheus --create-namespace

kubectl get all -n monitoring
```
[결과]
```
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          15h
pod/prometheus-grafana-5d49b89749-zhs5k                      3/3     Running   0          15h
pod/prometheus-kube-prometheus-operator-55dfb9bbbc-vdrqr     1/1     Running   0          15h
pod/prometheus-kube-state-metrics-857895cb8d-wmscq           1/1     Running   0          15h
pod/prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          15h
pod/prometheus-prometheus-node-exporter-4kpps                1/1     Running   0          15h
pod/prometheus-prometheus-node-exporter-crv7w                1/1     Running   0          15h
pod/prometheus-prometheus-node-exporter-fth5d                1/1     Running   0          15h
pod/prometheus-prometheus-node-exporter-kzxrw                1/1     Running   0          15h
pod/prometheus-prometheus-node-exporter-wjx4k                1/1     Running   0          15h
pod/prometheus-prometheus-node-exporter-xsmz2                1/1     Running   0          15h

NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   15h
service/prometheus-grafana                        ClusterIP   172.20.251.150   <none>        80/TCP                       15h
service/prometheus-kube-prometheus-alertmanager   ClusterIP   172.20.234.31    <none>        9093/TCP,8080/TCP            15h
service/prometheus-kube-prometheus-operator       ClusterIP   172.20.59.144    <none>        443/TCP                      15h
service/prometheus-kube-prometheus-prometheus     ClusterIP   172.20.36.175    <none>        9090/TCP,8080/TCP            15h
service/prometheus-kube-state-metrics             ClusterIP   172.20.219.176   <none>        8080/TCP                     15h
service/prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     15h
service/prometheus-prometheus-node-exporter       ClusterIP   172.20.169.242   <none>        9100/TCP                     15h

NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/prometheus-prometheus-node-exporter   6         6         6       6            6           kubernetes.io/os=linux   15h

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-grafana                    1/1     1            1           15h
deployment.apps/prometheus-kube-prometheus-operator   1/1     1            1           15h
deployment.apps/prometheus-kube-state-metrics         1/1     1            1           15h

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-grafana-5d49b89749                    1         1         1       15h
replicaset.apps/prometheus-kube-prometheus-operator-55dfb9bbbc   1         1         1       15h
replicaset.apps/prometheus-kube-state-metrics-857895cb8d         1         1         1       15h

NAME                                                                    READY   AGE
statefulset.apps/alertmanager-prometheus-kube-prometheus-alertmanager   1/1     15h
statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus       1/1     15h
```


KEDA 를 설치한다.
```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda \
  --namespace keda --create-namespace
```
기존 설정을 유지하면서 slurm 을 업그레이드 하기 위해 --reuse-values 옵션을 사용한다. 
```
helm update --install slurm oci://ghcr.io/slinkyproject/charts/slurm \
  --set 'controller.metrics.enabled=true' \
  --set 'controller.metrics.serviceMonitor.enabled=true' \
  --reuse-values \
  --namespace slurm --create-namespace
```

KEDA api 서비스를 확인한다. 
```
kubectl get apiservice -l app.kubernetes.io/instance=keda
```
[결과]
```
...
```

### KEDA ScaleObject 생성 ###
```
export PROMETHEUS_URL=$(..)
```

gpu 노드셋 정보를 조회한다.
```
slurm-worker-ns-gpu
...
```

```
cat <<EOF > keda-scaleobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: scale-gpu
spec:
  scaleTargetRef:                                
    apiVersion: slinky.slurm.net/v1beta1        # KEDA 스케일링의 대상을 NodeSet 으로 설정
    kind: NodeSet
    name: slurm-worker-ns-gpu
  idleReplicaCount: 0
  minReplicaCount: 0                            # 최소 0 
  maxReplicaCount: 1000                         # 최대 1000 - 인스턴스 1000대 까지 오토 스케일링.  
  triggers:
    - type: prometheus
      metricType: Value
      metadata:
        serverAddress: ${PROMETHEUS_URL}                # http://prometheus-kube-prometheus-prometheus.prometheus:9090
        query: slurm_partition_jobs_pending{partition="gpu"}
        threshold: '1'                          # 이 값을 0 으로 잡아야 할지? 1로 잡아야 할지? 
EOF
```
```
kubectl apply -f keda-scaleobject.yaml
```


