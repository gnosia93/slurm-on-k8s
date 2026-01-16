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
#### sbatch ####
sbatch 는 가장 일반적인 작업 제출 방식으로 쉘 스크립트(.sh)를 파라미터로 사용한다. sbatch -p 옵션으로 sinfo에서 확인한 가용 파티션을 지정할 수 있다.
```
sbatch -p [파티션명] job-script.sh
```
#### salloc / srun ####
salloc 또는 srun 대화형 작업 실행 명령어로 리소스를 즉시 할당받아 직접 터미널에서 작업하거나 실시간으로 실행 결과를 확인하고 싶을 때 사용한다.
```
srun -p [파티션명] --nodes=1 --pty bash
```
#### squeue ###
제출한 작업이 대기 중인지 혹은 실행 중인지 확인한다. 
```
squeue -u [사용자ID] 
```

#### 실제 훈련 샘플 ####
```
cat <<EOF > train_llama3.sh
#!/bin/bash
#SBATCH --job-name=llama3_training        # 작업 이름
#SBATCH --nodes=2                         # 사용할 노드 수 (p4dn 2대)
#SBATCH --ntasks-per-node=1               # 노드당 실행할 작업(프로세스) 수 (torchrun 의 개수)
#SBATCH --gres=gpu:8                      # 노드당 할당할 GPU 수 (A100 8개)
#SBATCH --cpus-per-task=96                # p4dn.24xlarge 전체 CPU 코어 활용
#SBATCH --partition=[당신의_GPU_파티션]       # sinfo에서 확인한 파티션 이름
#SBATCH --output=%x_%j.out                # 표준 출력 로그 파일

# 1. 분산 학습 환경 변수 설정
export MASTER_ADDR=$(scontrol show hostnames "$SLURM_JOB_NODELIST" | head -n 1)
export MASTER_PORT=12802
export WORLD_SIZE=$((SLURM_NNODES * 8))   # 총 GPU 수 (2노드 * 8개)

# 2. 모델 및 환경 준비 (예: Hugging Face Llama-3)
MODEL_PATH="meta-llama/Meta-Llama-3-8B"

# 3. 분산 학습 실행 (torchrun 활용)
srun torchrun \
    --nnodes=$SLURM_NNODES \
    --nproc_per_node=8 \
    --rdzv_id=$SLURM_JOB_ID \
    --rdzv_backend=c10d \
    --rdzv_endpoint=$MASTER_ADDR:$MASTER_PORT \
    training_script.py \
       --model_name_or_path $MODEL_PATH \
       --batch_size 4 \
       --use_fsdp True

# $? 는 쉘에서 앞선 명령어의 종료 상태를 의미 (0이면 성공, 그 외엔 실패)
if [ $? -ne 0 ]; then
    echo "⚠️ 작업 실패! 로그를 확인하세요: $SLURM_SUBMIT_DIR/%x_%j.out"
    # 여기에 Slack Webhook이나 다른 알림 명령어를 넣을 수 있다.
fi
EOF
```
* 최신 PyTorch 분산 학습(torchrun)은 Slurm이 각 노드에 딱 하나의 관리자 프로세스만 띄우기를 권장한다.
* Slurm은 노드 2개를 빌려오고, 각 노드에서 torchrun이라는 관리자 프로세스를 노드 마다 하나씩 (--ntasks-per-node=1) 실행한다.
* torchrun이 실행된 후, 해당 노드 안에 있는 8개의 GPU(--nproc_per_node=8)에 맞춰 8개의 실제 학습 프로세스를 알아서 쪼개서 실행 된다.

아래 명령어로 학습을 시작한다. 
```
sbatch train_llama3.sh
```
```
squeue -u $USER
tail -f p4dn_test_12345.out
```


