### [slurm 오토 스케일링 설정하기](https://slinky.schedmd.com/projects/slurm-operator/en/release-1.0/usage/autoscaling.html) ###
slrum 의 오토 스케일링, 정확하게는 slurm 파드의 오토 스케일링은 KEDA, Prometheus, Metric 서버 기반으로 동작을 한다. slurm job 이 생성되었을때 Karpenter 까지 연결되는 작동 메커니즘은 다음과 같다. 

#### 작동 메커니즘: Slurm → KEDA → Pod → Karpenter ####
* Slurm Job 발생: 사용자가 sbatch로 작업을 던집니다. 현재 컴퓨팅 노드가 0개이므로 작업은 PENDING 상태가 됩니다.
* KEDA의 감지: KEDA가 Slurm의 메트릭(예: squeue 대기 수)을 확인하고, NodeSet(또는 Deployment)의 Replica를 0에서 1 이상으로 올립니다.
* Pending Pod 생성: NodeSet이 파드를 생성하려고 하지만, 수용할 노드가 없으므로 파드는 Pending 상태로 쿠버네티스 스케줄러에 머뭅니다.
* Karpenter 출동: Karpenter가 이 Pending 파드를 발견하고, "아, 파드가 필요로 하는 리소스(CPU/GPU 등)에 맞는 실제 노드를 프로비저닝해야겠군!" 하며 인스턴스를 띄웁니다.
노드가 준비되면 파드가 배치되고, Slurm Worker가 활성화되어 작업을 수행합니다.

프로메테우스를 설치한다. 
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  --set 'installCRDs=true' \
  --namespace prometheus --create-namespace
```
KEDA 를 설치한다.
```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda \
  --namespace keda --create-namespace
```
slurm 을 업그레이드 한다. 
```
helm update --install slurm oci://ghcr.io/slinkyproject/charts/slurm \
  --set 'controller.metrics.enabled=true' \
  --set 'controller.metrics.serviceMonitor.enabled=true' \
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
