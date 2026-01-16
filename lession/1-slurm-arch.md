### 1. slurmctld (중앙 관리자) ###
역할: 클러스터의 두뇌. 작업을 스케줄링하고 노드 상태를 감시합니다.
Slinky 특징: EKS의 컨트롤러 Pod에서 실행되며, 작업 요청이 들어오면 K8s API에 "노드가 필요해!"라고 신호를 보냅니다. slurmctld 가이드(SchedMD)

### 2. slurmdbd (데이터베이스 관리자) ###
역할: 기록관. 작업 이력, 팀별 사용량(Accounting), QOS 정책을 관리합니다.
Slinky 특징: 보통 Amazon RDS(MySQL)와 연결되어 있으며, 작업이 끝나면 누가 GPU를 얼마나 썼는지 정산 데이터를 저장합니다. slurmdbd 가이드(SchedMD) 

### 3. slurmd ###
역할: 노드 관리. 각 컴퓨팅 노드(p4dn 등)에서 실행되며, 컨트롤러의 명령을 받아 실제 작업을 띄웁니다.
Slinky 특징: Karpenter가 띄운 노드 내부의 워커 Pod에서 실행됩니다. 노드의 GPU/CPU 상태를 체크해 컨트롤러에 보고합니다.

### 4. slurmstepd ###
역할: 작업 단위 관리. slurmd가 작업을 실행할 때마다 생성되는 하위 프로세스입니다.
특징: 개별 작업(Job Step)이 할당된 자원을 넘지 않는지 감시하고, 작업이 끝나면 깔끔하게 정리하고 사라집니다.

### 5. munged (인증 파수꾼) ###
역할: 보안 통신. Slurm 데몬들 간의 통신이 안전한지 확인하는 인증 메커니즘입니다.
특징: 모든 노드가 동일한 munge.key를 공유해야 하며, 이게 다르면 노드가 클러스터에 합류하지 못합니다.

#### 데몬 간 관계도 ####
* 사용자 → sbatch 제출
* slurmctld (스케줄링 결정) ↔ slurmdbd (권한/기록 확인)
* slurmctld → slurmd (노드에게 실행 명령)
* slurmd → slurmstepd (실제 프로세스 관리)

----
### SlurmDBD(Slurm DataBase Daemon) ###
클러스터의 '기록보관소'이자 '관리자' 역할을 수행합니다. 

#### 1. 작업 회계 (Accounting) ####
가장 핵심적인 기능으로, 모든 작업의 시작부터 끝까지를 기록합니다. Slurm Accounting 가이드에 따르면 다음 데이터를 보관합니다.
* 리소스 사용량: 누가, 언제, 어떤 노드에서 GPU/CPU를 얼마나 썼는지 기록합니다.
* 결과 기록: 작업이 성공했는지, 에러(NODE_FAIL, TIMEOUT 등)로 끝났는지 상태 코드를 저장합니다.
* 조회: 사용자는 sacct 명령어로 며칠 전 끝난 작업의 로그와 성능을 다시 찾아볼 수 있습니다

#### 2. 자원 할당 및 정책 관리 (Resource Quotas) ####
단순히 기록만 하는 게 아니라, 누가 자원을 쓸 수 있는지 권한을 통제합니다.
* Account/User 관리: 팀(Account)별로 사용자를 묶고, 특정 팀에만 p4dn 파티션 사용 권한을 부여합니다.
* QOS(Quality of Service): "A팀은 GPU를 동시에 16개까지만 쓸 수 있다"거나 "한 작업당 최대 24시간만 점유 가능하다"는 식의 제한 정책을 데이터베이스에 저장하고 실시간으로 적용합니다.

#### 3. 우선순위 결정 (Fairshare) ####
여러 사용자가 동시에 작업을 던졌을 때, 누구의 것을 먼저 실행할지 계산합니다.
* Fairshare 알고리즘: 과거에 자원을 많이 쓴 사용자는 우선순위를 낮추고, 오랫동안 기다린 사용자는 높이는 식의 계산을 위해 과거 누적 사용량 데이터를 DB에서 가져와 사용합니다.

#### 4. 다중 클러스터 연동 (Multi-Cluster Management) ####
여러 개의 Slurm 클러스터가 있을 때, 하나의 중앙 DB(slurmdbd)에 연결하여 모든 클러스터의 자원을 통합 관리하고 작업 현황을 한곳에서 볼 수 있게 합니다. 


작업 실행 흐름 (Architecture Workflow)
* 제출: 사용자가 로그인 노드에서 sbatch 실행 → slurmctld로 전달.
* 기록: slurmctld가 slurmdbd에 "누가 작업을 시작함"이라고 기록 요청.
* 할당: slurmctld가 가용 리소스를 확인(Slinky라면 Karpenter를 통해 노드 생성 트리거).
* 실행: slurmctld가 해당 노드의 slurmd에게 명령 전달 → slurmd가 slurmstepd를 띄워 학습 시작.
* 완료: 작업 종료 후 slurmdbd에 사용량 데이터 최종 저장 및 노드 반납

#### Slinky(EKS)만의 아키텍처 특징 ####
* 고가용성(HA): 컨트롤러가 K8s Pod으로 실행되므로, 장애 시 Amazon EKS가 즉시 새로운 컨트롤러를 살려냅니다.
* 유연한 확장: slurmd가 떠 있는 컴퓨팅 노드 자체가 K8s Pod이며, 이는 필요할 때만 생성되는 Karpenter 노드 위에 올라갑니다.

#### 작업 스케줄링 ####
Slurm의 스케줄러가 작업을 처리하는 과정은 단순히 빈자리를 찾는 것을 넘어, 우선순위(Priority)와 백필(Backfill)이라는 두 가지 핵심 메커니즘을 통해 클러스터 효율을 극대화합니다

#### 1. 작업 제출 및 검증 (Submission) ####
사용자가 sbatch를 실행하면 slurmctld가 이를 수신합니다.
* 검증: 사용자가 해당 파티션을 쓸 권한이 있는지, 요청한 GPU 개수가 파티션 제한 내에 있는지 확인합니다.
* 대기: 검증을 통과하면 작업은 PENDING 상태로 대기열(Queue)에 들어갑니다.

#### 2. 우선순위 계산 (Priority Tiering) ####
대기열에 작업이 쌓이면, 스케줄러는 누구를 먼저 실행할지 점수를 매깁니다. Slurm Multifactor Priority 가이드에 따르면 다음 요소들이 합산됩니다.
* Fairshare: 최근에 자원을 적게 쓴 팀의 작업에 가산점을 줍니다. (DB 데이터 활용)
* Job Age: 대기열에서 오래 기다린 작업일수록 점수가 높아집니다.
* Partition: 특정 파티션(예: 긴급 전용)에 더 높은 가중치를 줄 수 있습니다. 

#### 3. 스케줄링 알고리즘 (Scheduling Logic) ####
가장 높은 점수의 작업을 실행하기 위해 리소스를 매칭합니다.
* Main Scheduler: 가장 높은 우선순위의 작업을 위해 리소스를 예약합니다.
* Backfill Scheduling (핵심): 만약 1순위 작업이 큰 자원(p4dn 10대 등)을 기다리느라 리소스가 비어 있다면, 그 틈새 시간에 끝날 수 있는 작은 작업들을 먼저 끼워 넣어 GPU 유휴 시간을 없앱니다. 

#### 4. Slinky의 동적 할당 (Cloud-Bursting) ####
Slinky 아키텍처만의 독특한 단계입니다.
* Trigger: 스케줄러가 실행할 작업을 확정했지만 노드가 없다면, Slinky 컨트롤러가 Kubernetes에 작업용 Pod 생성을 요청합니다.
* Karpenter 개입: K8s API가 이 요청을 받으면 Karpenter가 즉시 AWS에서 p4dn.24xlarge 인스턴스를 프로비저닝합니다.
* Launch: 노드 준비 완료 → slurmd 구동 → 작업 시작.
