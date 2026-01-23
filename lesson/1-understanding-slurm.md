## Slurm 이란 ##

Slurm은 Simple Linux Utility for Resource Management의 약자이다. 
이름은 'Simple'이라고 시작하지만, 실제로는 전 세계 슈퍼컴퓨터와 AI 클러스터에서 가장 많이 사용되는 오픈 소스 워크로드 관리자(Workload Manager)이자 작업 스케줄러이다.

### Slurm의 3가지 핵심 역할 ##
* 리소스 할당 (Allocate): 사용자가 요청한 시간 동안 특정 개수의 CPU, GPU, 메모리 등의 자원을 독점적으로 할당해 줍니다.
* 작업 실행 (Run): 할당된 노드에서 실제 계산 작업(예: AI 학습 모델)을 시작하고 실행 상태를 감시합니다.
* 대기열 관리 (Queue): 수많은 사용자가 동시에 작업을 던질 때, 우선순위(Priority)에 따라 줄을 세우고 자원이 나면 순서대로 투입합니다

### 아키텍처 ###


### 명령어 ###
* scontrol - 클러스터 상태 및 구성 정보를 모니터링하고 수정할 수 있습니다.
* squeue - 작업 상태에 대한 정보를 제공합니다.
* sacct - 실행 및 완료된 작업 및 작업 단계에 대한 정보를 제공합니다.
* srun - 작업을 시작할 수 있습니다.
* sview - 작업 상태, 시스템 상태 및 네트워크 토폴로지에 대한 그래픽 정보를 제공합니다.
* sacctmgr - 사용자 및 계정 확인, 클러스터 식별을 포함하여 데이터베이스를 관리할 수 있습니다.
* scancel - 실행 중이거나 대기 중인 작업을 중지할 수 있습니다.



## Accounting ##
Slurm에서 어카운팅(Accounting)은 시스템 자원을 "누가, 언제, 얼마나" 사용했는지 기록하고 관리하는 핵심 감사 시스템으로
단순한 기록을 넘어, 우선순위(Fairshare)를 결정하는 근거 데이터가 되기 때문에 매우 중요하다.

### 1. 어카운팅의 핵심 구조: SlurmDBD ###
* 어카운팅은 SlurmDBD(Slurm Database Daemon)라는 별도의 데몬과 MySQL/MariaDB를 통해 작동합니다.
* 사용자(User): 작업을 던지는 개별 계정.
* 어카운트(Account): 팀이나 프로젝트 단위 (예: ai_team, hpc_project). 사용자는 여러 어카운트에 속할 수 있습니다.
* 연합(Association): [사용자 + 어카운트 + 파티션 + 클러스터]를 하나로 묶는 논리적 단위입니다.

### 2. 어카운팅으로 할 수 있는 것 (Consultative Points) ###
* 자원 사용량 추적: 특정 프로젝트가 이번 달에 GPU를 몇 시간(GPU-hours) 사용했는지 정확히 계산하여 과금(Billing)하거나 보고서를 뽑을 수 있다.
* 우선순위(Fairshare) 계산: 어카운팅 데이터에 기록된 과거 사용량(Historical Usage)을 바탕으로, 최근에 자원을 많이 쓴 사용자의 우선순위는 낮추고, 덜 쓴 사용자는 높여주는 공평한 스케줄링을 구현한다.
* 할당량 제한(Resource Limits): 특정 어카운트가 동시에 사용할 수 있는 GPU 개수나 최대 CPU 시간을 설정하여 자원 독점을 방지한다.

### 3. 주요 명령어 ###
* sacct: 종료된 작업의 상세 기록(사용 시간, 메모리 피크 등)을 조회
* sacctmgr: 사용자, 어카운트, 쿼터(Quota) 등을 관리하는 관리자용 도구
* sreport: 주간/월간 자원 이용률 보고서를 생성.

### 4. 리소스 거버넌스 ###

#### 1. 어카운트(그룹) 생성 ####
먼저 사용자가 소속될 프로젝트나 팀 단위의 어카운트를 만듭니다. 이때 어카운트 레벨에서 전체 쿼터를 미리 걸 수도 있습니다.
* 명령어: sacctmgr add account <어카운트명> Description="<설명>" Organization="<조직명>"
* 예시: sacctmgr add account ai_project Description="AI_Team_Project" Organization="Research_Dept"

#### 2. 사용자 생성 및 어카운트 연결 ####
사용자를 특정 어카운트에 귀속시키며 생성합니다.
* 명령어: sacctmgr add user <사용자명> Account=<어카운트명>
* 예시: sacctmgr add user gildong Account=ai_project

#### 3. 쿼터(Resource Limits) 설정 ####
이제 핵심인 쿼터를 부여합니다. 특정 사용자 혹은 어카운트 전체에 대해 CPU, GPU, 실행 가능 작업 수 등을 제한할 수 있습니다. 
* GPU 개수 제한 (사용자 단위):
```
sacctmgr modify user gildong set GrpTRES=gres/gpu=2
```
(설명: gildong 사용자가 동시에 사용할 수 있는 GPU 총합을 2개로 제한)

* 최대 실행 작업 수 제한 (어카운트 단위):
```
sacctmgr modify account ai_project set MaxJobs=10
```
(설명: ai_project 팀 전체가 동시에 돌릴 수 있는 작업 수를 10개로 제한)

* CPU 시간 제한 (어카운트 단위):
```
sacctmgr modify account ai_project set GrpCPURunMins=10000
```
(설명: 해당 팀이 사용 중인 총 CPU 시간 합계를 제한)

* 설정값 확인
```
sacctmgr show association user=<사용자명> format=Account,User,Partition,GrpTRES,MaxJobs
```

### 5. 사용자 관리 ###
Slurm에서 sacctmgr로 유저를 생성하는 것은 "자원 사용 권한(Accounting)"을 부여하는 과정이지, 실제 서버에 "로그인 계정"을 만드는 것은 아니다.
로그인을 위해서는 다음 두 가지가 모두 준비되어야 한다.

#### 1. OS 계정이 먼저 있어야 합니다 (LDAP/NIS/AD 등) ####
Slurm은 독자적인 인증 시스템이 아니라 리눅스 OS 계정 정보를 그대로 가져다 쓴다.
* 작동 원리: 사용자가 SSH로 마스터 노드(Login Node)에 접속할 때, Slurm은 사용자의 UID(User ID)를 보고 sacctmgr에 등록된 유저인지 확인.
* 권장 사항: 클러스터 환경(수십~수백 대의 서버)이므로, 각 서버마다 계정을 일일이 만들지 않고 OpenLDAP이나 Active Directory를 통해 계정 정보를 중앙 집중식으로 동기화.

#### 2. Slurmdbd와의 연결 ####
OS 계정이 있는 상태에서 sacctmgr add user <아이디>를 하면, 비로소 해당 사용자가 sbatch나 srun 명령어를 써서 GPU 자원을 요청할 수 있게 된다.
만약 OS 계정은 있는데 Slurm에 등록하지 않았다면? 로그인까지는 되지만, 작업을 던지려고 하면 Invalid account or account/partition combination specified라는 에러가 발생한다.

#### 3. 로그인 및 작업 제출 과정 ####
* SSH 접속: 사용자가 터미널에서 ssh user@login-node로 접속.
* 작업 제출: 접속 후 sbatch my_job.sh를 실행.
* 인증: Slurm이 "이 유저가 ai_project 어카운트에 등록되어 있고, 남은 GPU 쿼터가 있나?"를 확인.
* 실행: 승인되면 실제 계산 노드로 작업이 넘어간다.

💡 컨설팅 포인트
고객사 인프라를 구축할 때, "유저 관리 자동화"가 핵심이다. Ansible 같은 자동화 도구나 스크립트를 짜서 [리눅스 계정 생성 + Slurm 유저 등록]을 한 번에 처리하도록 설계하는 것이 정석입니다.

#### 로그인 서버가 4대 인경우 ####
로그인 서버가 4대 정도라면 LDAP/AD 없이 수동 관리도 충분히 가능합니다. 다만, 4대의 서버가 동일한 사용자 ID(UID)와 그룹 ID(GID)를 갖도록 맞추는 것이 핵심입니다.
[host.ini]
```
[login_servers]
login1 ansible_host=10.0.0.1
login2 ansible_host=10.0.0.2
login3 ansible_host=10.0.0.3
login4 ansible_host=10.0.0.4

[slurm_master]
master_node ansible_host=10.0.0.10
```

[ansible.yaml]
```
---
- name: AI 인프라 유저 통합 생성 (OS 계정 + Slurm 어카운팅)
  hosts: all
  become: yes
  vars:
    # 생성할 유저 정보 정의
    user_id: "gildong"
    user_uid: 2001
    group_id: "ai_team"
    group_gid: 3001
    slurm_account: "ai_project"

  tasks:
    # 1. 모든 서버(로그인 4대 + 마스터)에 동일한 GID로 그룹 생성
    - name: "그룹 생성 (GID: {{ group_gid }})"
      group:
        name: "{{ group_id }}"
        gid: "{{ group_gid }}"
        state: present

    # 2. 모든 서버에 동일한 UID/GID로 유저 생성
    - name: "유저 생성 (UID: {{ user_uid }}, GID: {{ group_gid }})"
      user:
        name: "{{ user_id }}"
        uid: "{{ user_uid }}"
        group: "{{ group_id }}"
        shell: /bin/bash
        state: present

    # 3. Slurm 마스터 노드에서만 어카운팅 등록 실행
    - name: "Slurm 어카운팅 등록"
      command: "sacctmgr -i add user {{ user_id }} Account={{ slurm_account }}"
      delegate_to: "{{ groups['slurm_master'][0] }}"
      run_once: true
      register: slurm_result
      changed_when: "'Adding' in slurm_result.stdout"
      failed_when: 
        - slurm_result.rc != 0 
        - "'Already exists' not in slurm_result.stderr"
```
모듈 + 멱등성(Idempotency)
* 모듈(Module): user, group, command (실제 작업을 수행하는 부품)
* 태스크(Task): 모듈을 사용하여 정의한 하나의 작업 단위 (위 코드의 - name: 부분)
* 플레이북(Playbook): 여러 태스크를 모아놓은 전체 시나리오 파일 (.yml)

```
ansible-playbook -i hosts.ini create_user.yml
```

## Slurm을 이용한 GPU 전용 파티션 설정 ##
Slurm에서 GPU 전용 파티션(Partition)을 설정하는 것은 특정 노드들을 그룹화하고, 그 안에서 GPU 자원을 효율적으로 분배하는 과정이다.

#### 1. 노드 정의 (GRES 설정) ####
가장 먼저 각 노드가 어떤 GPU를 몇 개 가지고 있는지 선언해야 한다. Slurm GRES(Generic Resource) 가이드를 참고하면, Generic Resource 설정을 통해 GPU를 인식시킨다.
```
# slurm.conf 파일 하단에 노드 정의
NodeName=gpu-node[01-10] CPUs=64 RealMemory=256000 Sockets=2 CoresPerSocket=16 ThreadsPerCore=2 State=UNKNOWN GRES=gpu:h100:8
```
#### 2. 파티션(Partition) 정의 ####
이제 정의된 노드들을 묶어 GPU 전용 파티션을 만든다. Slurm Partition 설정을 통해 접근 권한과 우선순위를 제어한다.
```
# slurm.conf 파일에 파티션 정의
PartitionName=gpu_part Nodes=gpu-node[01-10] Default=NO MaxTime=INFINITE State=UP OverSubscribe=NO PriorityTier=100 TRESBillingWeights="CPU=1.0,Mem=0.25G,GRES/gpu=2.0"
```
* PriorityTier=100: 일반 파티션보다 우선순위를 높여 GPU 작업을 먼저 처리.
* OverSubscribe=NO: 하나의 GPU에 여러 작업이 겹치지 않도록 독점 사용을 강제.

#### 3. gres.conf 설정 (각 계산 노드) ####
Slurm이 물리적 GPU 장치 파일(/dev/nvidia0 등)과 통신할 수 있도록 각 계산 노드의 /etc/slurm/gres.conf를 작성한다.
```
# gres.conf (모든 GPU 노드 공통)
Name=gpu Type=h100 File=/dev/nvidia[0-7]
```
* TRES Billing 가중치: 위 설정의 TRESBillingWeights는 중요. 단순히 CPU 코어 수뿐만 아니라 GPU 사용량을 기준으로 과금(Accounting)이나 우선순위를 계산하도록 유도
* Affinity 연동: GPU 전용 파티션을 쓸 때는 반드시 TaskPlugin=task/affinity와 SelectTypeParameters=CR_Core_Memory,CR_CORE_BINDING 설정을 켜서, GPU와 가까운 NUMA 노드의 CPU가 할당되도록 해야 성능 병목을 막을 수 있다.

설정을 변경하신 후에는 반드시 서비스를 재시작해야 적용된다.
```
sudo scontrol reconfigure  # 마스터 노드에서 실행
```
#### 4. 작업 실행 ####
다음 명령어로 GPU 파티션에 작업을 던질 수 있다.
```
sbatch -p gpu_part --gres=gpu:1 my_job.sh 
```

#### 5. 리소스 사용 제한  ####
* 특정 그룹만 파티션 사용 허용 (AllowGroups)
slurm.conf의 파티션 설정 부분에 AllowGroups 옵션을 추가한다.
```
# slurm.conf 설정 예시
PartitionName=gpu_part Nodes=gpu-node[01-10] AllowGroups=ai_team,research_group Default=NO State=UP
```

* 작업당/사용자당 최대 GPU 사용 제한 (QoS 활용)
파티션 설정보다 더 강력하고 세밀한 제어는 QoS(Quality of Service)를 통해 이루어진다.
```
# 한 번에 최대 4개의 GPU만 사용 가능하고, 최대 2개 작업만 동시 실행 가능한 QoS 생성
sacctmgr add qos gpu_limit MaxTRESPerUser=gres/gpu=4 MaxJobsPerUser=2 Priority=100
# ai_project 어카운트에 이 제한을 적용
sacctmgr modify account ai_project set QoS=gpu_limit
```

* 작업당 최소/최대 GPU 할당량 강제 (Partition Limits)
유저가 실수로 노드 전체 GPU를 다 써버리는 것을 방지하기 위해 파티션 자체에 제약을 건다.
```
# slurm.conf 설정 추가
PartitionName=gpu_part ... MaxNodes=2 MaxCPUsPerNode=32 GrpTRES=gres/gpu=20
```
* MaxNodes=2: 한 작업이 노드 2개를 초과해서 잡을 수 없음.
* GrpTRES=gres/gpu=20: 이 파티션 전체에서 동시에 돌아가는 GPU 합계를 20개로 제한.

----
## 작업 간의 우선순위(Fairshare) 관리 전략 ###
Slurm의 Fairshare는 단순히 "누가 먼저 왔느냐"가 아니라, "과거에 자원을 얼마나 많이 썼느냐"를 기준으로 우선순위를 동적으로 조정하는 핵심 알고리즘입니다.

#### 1. 가중치 설정 (Multifactor Priority) ####
Slurm은 여러 요소를 합산해 최종 점수(Priority)를 냅니다. slurm.conf에서 Fairshare의 비중을 높여야 과거 사용량이 실제 우선순위에 반영됩니다

* 설정 예시 (slurm.conf):
```
PriorityType=priority/multifactor
PriorityWeightFairshare=10000  # Fairshare 비중을 크게 설정
PriorityWeightAge=1000        # 대기 시간 비중은 보조적으로 설정
PriorityDecayHalfLife=7-0     # 과거 사용량의 영향력을 7일마다 절반으로 감소
```
* 전략: DecayHalfLife를 통해 최근 사용량에 더 높은 벌점을 줍니다. 1~2주 전 기록은 잊어주고, 어제 많이 쓴 사람의 우선순위를 오늘 낮추는 방식입니다.

#### 2. 어카운트별 지분(Share) 할당 ####
모든 팀에 동일한 권한을 주기보다, 투자 비용이나 중요도에 따라 Raw Shares를 다르게 배정합니다
```
# A팀(메인 프로젝트)은 지분 100 부여
sacctmgr modify account team_a set share=100
# B팀(테스트/지원 팀)은 지분 50 부여
sacctmgr modify account team_b set share=50
```
결과: 자원이 꽉 찼을 때, A팀 유저는 B팀 유저보다 2배 더 높은 확률로 먼저 자원을 할당받습니다.

#### 3. '자원 독식' 방지를 위한 계층 구조 (Parent-Child) ####
어카운트를 Tree 구조로 설계하여 상위 조직의 쿼터 내에서 하위 팀들이 경쟁하게 만듭니다.
```
구조: Research_Dept (Share: 100) -> AI_Lab (Share: 50), Bio_Lab (Share: 50)
전략: AI_Lab의 한 명의 헤비 유저가 자원을 다 써버려도, 그 피해는 같은 AI_Lab 팀원들에게만 가고 Bio_Lab의 우선순위에는 영향을 주지 않도록 격리하는 효과가 있습니다
```

## 팀별 GPU 사용량을 시각화 보고서로 만드는 방법 ##
#### 1. 데이터 추출: sreport 활용 ####
Slurm에 내장된 sreport 도구를 사용하면 특정 기간 동안의 GPU 사용량(TRES)을 팀(Account)별로 집계할 수 있습니다
```
sreport cluster AccountUtilizationByUser cluster=<클러스터명> start=now-30days format=Account,Login,Used -t Percent
```
#### 2. 시각화 방법  ####
Grafana + Prometheus (가장 추천)
* 실시간 대시보드를 구축하여 웹에서 상시 확인할 수 있는 방법입니다. Slurm-exporter를 활용합니다.
* 방법: Slurm Exporter가 수집한 데이터를 Prometheus 저장소에 쌓고, Grafana에서 "GPU Usage by Account" 파이 차트나 시계열 그래프를 그립니다.

## Slurm과 Docker/Singularity 컨테이너 연동 ##

#### 1. Singularity (HPC 환경의 표준) ####
HPC 클러스터에서 가장 권장되는 방식입니다. 루트 권한이 필요 없고, Slurm의 자원 제한(cgroups)과 완벽하게 호환됩니다. SingularityCE 공식 문서
* 작동 방식: 컨테이너가 일반 프로세스처럼 실행되므로, Slurm이 GPU와 CPU 할당량을 그대로 제어할 수 있습니다.
* 연동 방법: sbatch 스크립트 내에서 직접 실행합니다.
```
# sbatch 스크립트 예시
#SBATCH --gres=gpu:1

# Docker 이미지를 자동으로 가져와 실행 (Singularity가 변환)
singularity exec --nv docker://pytorch/pytorch:latest python train.py
```
MPI와 GPU(NV) 지원이 강력하며, 사용자가 루트 권한을 가질 수 없어 보안상 매우 안전

#### 2. Docker (NVIDIA Pyxis/Enroot) ####
NVIDIA Pyxis와 Enroot는 Slurm 클러스터에서 Docker 컨테이너를 마치 일반 프로세스처럼 가볍고 빠르게 실행하기 위해 NVIDIA가 개발한 기술 세트입니다. 
쉽게 비유하자면, Enroot는 컨테이너를 실행하는 '엔진'이고, Pyxis는 Slurm에서 그 엔진을 쉽게 쓸 수 있게 해주는 '리모컨(플러그인)'입니다
Docker를 직접 실행하는 대신, NVIDIA가 개발한 Pyxis 플러그인과 Enroot 런타임을 사용하여 Slurm에서 Docker 컨테이너를 네이티브하게 지원할 수 있다.
```
# 별도 변환 없이 Docker 이미지 바로 실행
srun --container-image=nvcr.io#nvidia/tensorflow:23.02-tf1-py3 python -c "import tensorflow"
```
사용자 권한(unprivileged)으로 컨테이너를 실행하며, MPI 및 GPU 가속 기능을 완벽히 지원합니다

#### 3. Docker 데몬 직접 연동 (비권장) ####
모든 컴퓨팅 노드에 Docker를 설치하고 사용자를 docker 그룹에 추가하여 실행하는 방식입니다.
* 구현: sbatch 스크립트 안에서 docker run 명령어를 직접 호출합니다.
```
# sbatch 스크립트 예시
srun docker run --rm --gpus all --net=host -v /shared_data:/data my_image:latest python train.py
```
* 문제점:
  * 보안: 사용자가 호스트의 Root 권한을 탈취할 위험이 커 관리자가 기피합니다.
  * EFA 연동: --net=host 및 장치 마운트(--device) 설정을 수동으로 정교하게 세팅해야 성능이 나옵니다.
  * 멀티 노드: 여러 대의 서버를 묶어 학습할 때(MPI 등) 컨테이너 간 통신 설정이 매우 복잡합니다.

* 직접적인 명령어(sbatch, squeue 등) 활용법


## EFA 사용을 위해 memlock 설정 ##
```
#!/bin/bash
#SBATCH --job-name=efa-container-train
#SBATCH --nodes=2              # 2대 이상의 노드에서 분산 학습
#SBATCH --ntasks-per-node=8    # 노드당 GPU 개수에 맞춤
#SBATCH --gres=gpu:8           # GPU 할당
#SBATCH --partition=p4d        # EFA 지원 인스턴스 파티션

# [필수] 메모리 잠금 제한 해제 (OS 레벨의 허가증)
# ulimit -l (Max Locked Memory)
# ulimit -s (Max Stack Size)
ulimit -l unlimited
ulimit -s unlimited

# [선택] EFA 관련 환경 변수 설정 (성능 최적화)
export FI_EFA_USE_DEVICE_RDMA=1 # GPU Direct RDMA 활성화
export FI_PROVIDER=efa          # EFA 프로바이더 강제 지정
export NCCL_DEBUG=INFO          # EFA 작동 여부 확인용 로그

# Pyxis/Enroot를 사용하는 경우 --container-image 옵션 활용
srun --container-image="nvcr.io#nvidia/pytorch:23.10-py3" \
     --container-mounts=/shared:/shared \
     python -u /shared/train.py
```
* ulimit -l unlimited를 실행 노드에서 먼저 선언하면, 그 뒤에 오는 srun 프로세스들이 이 '무제한 권한'을 상속받습니다.
* Pyxis를 사용 중이라면, srun이 컨테이너를 띄울 때 호스트의 이 설정을 컨테이너 내부 환경까지 그대로 복사해 넣습니다.

아래 코드로 설정이 제대로 되었는지 확인한다. 
```
import os
# 시스템 ulimit 확인 (결과가 18446744073709551615 또는 매우 큰 수면 성공)
print(f"Memlock limit: {os.resource.getrlimit(os.resource.RLIMIT_MEMLOCK)}")
```

