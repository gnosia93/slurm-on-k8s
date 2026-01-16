### 1. slurm Pod 확인 ###
```
kubectl get pods -n slurm
```
[결과]
```
NAME                             READY   STATUS             RESTARTS         AGE
slurm-controller-0               3/3     Running            0                45m
slurm-restapi-5468d6d478-xmxdk   1/1     Running            0                45m
slurm-worker-slinky-0            1/2     CrashLoopBackOff   15 (4m35s ago)   45m
```
pvc 를 확인한다.
```
kubectl get pvc -n slurm
```
[결과]
```
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
statesave-slurm-controller-0   Bound    pvc-aa96c336-889e-4186-985f-3d99857a3631   4Gi        RWO            gp3            <unset>                 73m
```

### 2. slurm 로그인 ###
이 워크샵에서는 별도의 로그인 파드를 두지 않고 controller 를 사용한다.
```
kubectl exec -it slurm-controller-0 -n slurm -c slurmctld -- /bin/bash

slurm@slurm-controller-0:/tmp$ sinfo
```
[결과]
```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
slinky       up   infinite      1   idle slinky-0
all*         up   infinite      1   idle slinky-0
```
