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
이 워크샵에서는 별도의 로그인 파드를 두지 않고 controller 를 사용한다. sinfo 로 파티션 정보를 확인한다. STATE 는 idle 이어야 한다.
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
sinfo는 클러스터 내의 노드(Node)와 파티션(Partition) 상태를 확인할 때 사용한다. STATE 값으로는 idle: 대기 중, alloc: 작업 중, down: 장애 발생 등이 있다.

### 3. 작업 제출 ###
#### 1. sbatch ####
sbatch 는 가장 일반적인 작업 제출 방식으로 쉘 스크립트(.sh)를 파라미터로 사용한다. sbatch -p 옵션으로 sinfo에서 확인한 가용 파티션을 지정할 수 있다.
```
sbatch -p [파티션명] job-script.sh
```
#### 2. salloc / srun ####
salloc 또는 srun 대화형 작업 실행 명령어로 리소스를 즉시 할당받아 직접 터미널에서 작업하거나 실시간으로 실행 결과를 확인하고 싶을 때 사용한다.
```
srun -p [파티션명] --nodes=1 --pty bash
```
#### 3. squeue ###
제출한 작업이 대기 중인지 혹은 실행 중인지 확인한다. 
```
squeue -u [사용자ID] 
```



