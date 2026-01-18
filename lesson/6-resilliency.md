## 장애 대응 및 복구 ##

Slinky 환경에서 장애가 발생하면, Slurm의 작업 관리 메커니즘과 Kubernetes의 자가 치유(Self-healing) 기능이 동시에 작동합니다. Slurm fault tolerance(SchedMD) 원칙에 따른 시나리오별 동작은 다음과 같습니다.

### 1. 작업 중인 파드(Pod) 또는 프로세스 장애 ###
* Slurm의 감지: srun 프로세스가 비정상 종료되면 Slurm은 해당 작업을 FAILED 상태로 처리하고 로그(%j.out)에 에러를 기록합니다.
* 자동 재시도: 만약 sbatch 스크립트에 --requeue 옵션을 넣었다면, Slurm은 작업을 대기열(Queue)로 되돌려 다시 실행을 시도합니다.
* K8s의 역할: 개별 프로세스 장애 시 K8s는 개입하지 않고 Slurm의 판단에 맡깁니다.

### 2. Kubernetes 노드(EC2) 장애 (Node Panic/Termination) ###
* 노드 상태 변화: 노드가 응답하지 않으면 Slurm sinfo에서 해당 노드 상태가 down 또는 drain으로 변합니다.
* Karpenter/MNG의 대응:
  * Managed Node Group(정적): AWS가 노드 상태 불량을 감지하고 새 인스턴스를 띄워 Desired: 2 수치를 맞춥니다.
  * Karpenter(동적): 장애 노드를 삭제하고, 대기 중인 Slurm Pod을 위해 새로운 노드를 즉시 프로비저닝합니다. Karpenter Node Termination(Karpenter.sh) 문서에 자세한 과정이 나와 있습니다.

### 3. 장애 복구 후 재시작 (Checkpoint) ###
* 중요: 노드가 새로 떠도 학습은 처음부터 다시 시작됩니다.
* 대응책: 이를 방지하려면 Llama 3 모델 학습 시 Checkpoint 기능을 반드시 활성화해야 합니다. PyTorch Distributed Checkpoint를 쓰면 노드 장애 후 재투입 시 마지막 저장 지점부터 이어서 학습할 수 있습니다.

### 4. Slurm 컨트롤러(Slinky) 장애 ###
* 고가용성(HA): Slinky는 보통 K8s 상에서 Deployment로 돌아가므로, 컨트롤러 파드가 죽으면 K8s가 즉시 다른 노드에서 살려냅니다. 이때 작업 데이터는 DB(보통 외부 RDS)에 저장되어 있어 상태가 유지됩니다.
* slurmctld가 죽어도 작업이 유지되는 특성과 Slinky의 etcd 기반 자동 복구를 연결해서 설명

#### 💡 실무 팁: ####
sinfo에서 특정 노드가 drain 상태로 멈춰 있다면, 관리자가 scontrol update nodename=[노드명] state=resume 명령어로 수동 복구해줘야 할 수도 있습니다.
지금 발생한 특정 에러 메시지가 있나요? 아니면 장애 테스트(Chaos Testing)를 준비 중이신가요? 구체적인 상황을 알려주시면 대응 명령어를 정리해 드릴게요.


## 훈련 복구 ##
쿠버네티스(EKS) 환경이기 때문에 Slurm 설정이 없어도 파드(Pod) 수준의 재시작은 기본적으로 일어난다. 하지만 Slurm 관점에서 "단순히 파드가 다시 뜨는 것"과 "작업(Job)이 정상적으로 완수되는 것"의 차이를 구분해야 한다. 

#### 1. 쿠버네티스(K8s)가 해주는 재시작 (Restart) ####
* 작동: 파드 내부의 컨테이너가 죽으면 쿠버네티스의 디플로이먼트(Deployment)나 스테이트풀셋(StatefulSet)이 감지하고 컨테이너를 다시 실행한다.
* 한계: 쿠버네티스는 '프로세스가 살아있는지'는 보지만, Slurm의 '작업 큐(Queue)'와 '할당된 자원(Resource)' 상태까지는 알지 못한다. 즉, 파드만 덩그러니 다시 뜨고 Slurm 작업은 FAILED 상태로 멈춰버린다.

#### 2. Slurm(Slinky)이 해주는 재시작 (Requeue) ####
* 작동: --requeue 옵션을 쓰면, 노드 장애 시 Slurm 컨트롤러가 해당 작업을 '큐에 다시 대기' 시킨다. Slurm Job Requeue 기능은 자원이 확보되는 즉시 새로운 파드를 생성하여 작업을 처음부터(혹은 체크포인트부터) 다시 실행하도록 유도한다.
* 강점: 파드만 다시 뜨는 게 아니라, Slurm의 작업 이력과 의존성(Dependency) 체인이 깨지지 않고 유지된다.

#### Gang Scheduling의 연속성 #### 
* 대규모 GPU 학습은 여러 노드가 동시에 떠야 하는 갱 스케줄링(Gang Scheduling)이 핵심이다. 노드 하나가 죽었을 때 쿠버네티스가 그 노드만 살리면 나머지 노드들과의 싱크가 깨질 수 있다.
* 전체 재시작: Slurm의 --requeue는 전체 워커 노드 세트를 다시 조율하여 학습이 일관성 있게 시작되도록 보장한다.



## 장애 테스트(Chaos Testing) ##
Slurm on EKS(Slinky) 환경에서의 카오스 테스트(Chaos Testing)는 "컴퓨팅 자원의 유동성"과 "대규모 모델 학습의 연속성"을 검증하는 데 초점을 맞춰야 합니다. 특히 p4dn 같은 고가 가용 자원을 쓰시므로, 장애 시 자원이 방치되지 않고 잘 회수되는지도 중요합니다.

### 1. 테스트 도구 추천 ###
직접 노드를 끄는 방식도 좋지만, 자동화된 도구를 쓰면 재현성이 높아집니다.
* AWS Fault Injection Service (FIS): AWS 공식 도구로, 특정 EC2 인스턴스를 중단시키거나 네트워크 레이턴시를 강제로 발생시킬 수 있습니다.
* Chaos Mesh: 쿠버네티스 네이티브 도구입니다. Slinky 컨트롤러 파드만 골라 죽이거나(Pod Kill), GPU 노드 간 통신을 끊는(Network Chaos) 테스트에 최적입니다.

### 2. 카오스 테스트 시나리오 디자인 (4단계) ###

#### 컴퓨팅 노드 급사 (Node Termination) ####
* 가설: 학습 중 p4dn 노드 하나가 점검 등으로 삭제되어도, Slurm은 작업을 재제출(Requeue)하고 Karpenter는 새 노드를 즉시 띄울 것이다.
* 방법: 학습이 한창일 때 kubectl delete node [노드명] 수행.
* 관찰 포인트:
  * Slurm 작업이 FAILED로 끝나는지, 아니면 대기열로 다시 들어가는지.
  * Karpenter가 5분 이내에 새 p4dn을 바인딩하는지.

#### EFA 네트워크 지연 (Network Latency) ####
* 가설: 노드 간 통신이 느려지면 NCCL 타임아웃이 발생하며, 시스템은 안전하게 작업을 중단하고 에러 로그를 남길 것이다.
* 방법: Chaos Mesh의 NetworkChaos를 사용해 100ms 이상의 지연 발생.
* 관찰 포인트: 분산 학습 프레임워크(PyTorch/DeepSpeed)가 좀비 프로세스가 되지 않고 깔끔하게 종료되는가?

#### Slurm 컨트롤러(Slinky) 장애 ####
* 가설: Slurm 컨트롤러 파드가 재시작되어도 실행 중인 작업(Job)은 영향 없이 계속 돌아갈 것이다.
* 방법: kubectl delete pod -l app.kubernetes.io/name=slinky-controller -n slurm
* 관찰 포인트: 컨트롤러 복구 후 squeue를 쳤을 때 작업 상태가 유실 없이 복구되는가?

#### 리소스 부족 (Insufficient Capacity) ####
* 가설: AWS 가용 영역(AZ)에 p4dn 재고가 없을 때, Slurm 작업은 PENDING 상태로 대기하며 시스템이 뻗지 않을 것이다.
* 방법: 카펜터 NodePool에서 사용 불가능한 인스턴스 타입을 강제로 지정하거나 대수를 초과해서 요청.


## sbatch 설정 ##
```
#SBATCH --requeue                  # <--- 핵심: 자동 재제출 활성화
```
* 장애 발생: 학습 중 p4dn 노드 하나를 kubectl delete node로 강제 삭제합니다.
* Slurm 감지: Slurm 컨트롤러가 노드 통신 두절을 감지하고 작업을 NODE_FAIL 상태로 전환합니다.
* 재제출: --requeue가 설정되어 있으므로, 작업은 즉시 PENDING 상태로 대기열로 돌아갑니다.
* 복구: Karpenter가 새로운 노드를 프로비저닝하면, Slurm은 동일한 작업 ID(또는 새 ID)로 작업을 다시 시작합니다.

## 주의 사항 및 팁 ##
* 체크포인트(Checkpoint): Slurm은 작업을 다시 던져줄 뿐, 학습 데이터를 복구해주지는 않습니다. 반드시 PyTorch Checkpoint 기능을 사용하여 /shared 폴더 같은 공용 스토리지에 진행 상황을 저장해야 합니다.
* 무한 루프 방지: 스크립트 자체의 결함으로 종료될 때도 재제출되면 무한 루프에 빠질 수 있습니다. Slurm Job Exit Codes 문서에 따르면, 특정 종료 코드(예: 1)일 때는 재제출하지 않도록 관리자 설정(RequeueExit)이 가능합니다.
* 상태 확인: 재제출된 작업은 squeue에서 R 상태 옆에 (REQUEUED) 표시가 붙거나 노출됩니다.


