## 파티션 할당하기 ##

Slurm에서 파티션(Partition)은 여러 대의 컴퓨팅 노드를 논리적으로 묶어 놓은 '자원 그룹'이자, 사용자가 작업을 제출하는 '대기열(Queue)'이다. Slurm 공식 문서(SchedMD)에 따르면 파티션은 작업의 특성이나 하드웨어 사양에 따라 시스템을 효율적으로 나누어 관리하는 핵심 단위로, 다음과 같은 요소로 구성되어져 있다. 
* 노드 리스트 (Nodes): 해당 파티션에 포함된 실제 서버(컴퓨팅 노드)들의 목록으로 하나의 노드가 여러 파티션에 중복으로 속할 수도 있다.
* 시간 제한 (MaxTime): 한 작업이 해당 파티션에서 최대 몇 시간 동안 실행될 수 있는지 정의한다. (예: debug 파티션은 30분, batch 파티션은 2일 등)
* 사용자 권한 (AllowGroups): 특정 파티션을 사용할 수 있는 사용자 그룹을 제한하여 보안이나 우선순위를 관리할 수 있다.
* 우선순위 (Priority): 여러 파티션이 동일한 노드를 공유할 때, 어떤 파티션의 작업을 먼저 실행할지 결정한다. 

Slinky 에서는 파티션(Partition) 핸들링을 위해 신규 오브젝트인 Nodeset 오브젝트를 사용한다. 즉 Nodeset 오브젝트를 이용하여 slurm 의 파티션을 구현한다.  

### 1. AMEX CPU 파티션 생성 ###
eks 매니지드 노드 그룹 ng-amx 를 구성하는 4대의 m7i.8xlarge 인스턴스를 활용하여 AMEX CPU 파티션 생성할 예정이다. 라벨 정보가 필요하므로 라벨 부터 확인한다. 
```
aws eks describe-nodegroup --cluster-name ${CLUSTER_NAME} \
  --nodegroup-name ng-amx --query 'nodegroup.labels' --output text 
```
[결과]
```
{
    "alpha.eksctl.io/cluster-name": "slurm-on-eks",
    "alpha.eksctl.io/nodegroup-name": "ng-amx",
    "workload-type": "slurm-compute",
    "architecture": "amx-enabled"
}
```
ng-amx 노드 그룹의 taint 를 확인한다. slurm 전용으로 사용하고자 다음과 같은 taint 를 미리 생성하였다 (eksctl cluster.yaml 확인) 
```
aws eks describe-nodegroup --cluster-name ${CLUSTER_NAME} \
  --nodegroup-name ng-amx --query 'nodegroup.taints'
```
[결과]
```
[
    {
        "key": "workload",
        "value": "slurm",
        "effect": "NO_SCHEDULE"
    }
]
```

노드가 이미 생성되어 있으므로 Slinky에게 "동적으로 띄우지 말고, 이 라벨이 붙은 노드를 파티션으로 써라"고 알려준다. 이때 파드 스팩에 Toleration 도 함께 설정해야 slurmd 파드가 대상 노드에 스케줄링 될 수 있다. slinky 에서 slurmd 가 설치되는 노드를 식별하기 위해서 nodeset > podSpec > nodeSelector 의 라벨 설정을 이용한다.  
```
cat <<EOF > amx-nodeset.yaml
# nodesets 아래에 바로 이름을 키로 사용합니다 (리스트 '-' 제거)
nodesets:
  ns-amx:
    enabled: true
    replicas: 4                            # count 대신 replicas를 사용 (Slinky 1.0.1 규격)
    updateStrategy:
      type: RollingUpdate
    podSpec:                   
      nodeSelector:                        # node selector 를 이용하여 slurmd 가 설치될 노드를 식별한다.
        workload-type: "slurm-compute"     # workoad-type 라벨
        architecture: "amx-enabled"        # architecture 라벨 
      tolerations:                         # 노드그룹에 설정된 taint 를 무력화 시키기 위해서 설정
        - key: "workload"                  # slurmd 파드가 스케줄링 되면서 자동으로 이 toleration 이 slurmd 파드에 붙는다. 
          operator: "Equal"
          value: "slurm"
          effect: "NoSchedule"
    slurmd:
      image:
        repository: ghcr.io/slinkyproject/slurmd
        tag: 25.11-ubuntu24.04
      resources:
        limits:                            # slurmd 를 스케줄링 하면서 리소스를 선점해 놓는다. slurm 이외에 다른 워크로드의 접근을 차단.
          cpu: "30"                        # 32 vCPU (m7i.8xlarge) 
          memory: "120Gi"                  # 120 Gi 메모리 (m7i.8xlarge)
    # LogFile 사이드카 설정
    logfile:
      image:
        repository: docker.io/library/alpine
        tag: latest
    # slurm.conf의 NodeName 줄에 들어갈 내용
    extraConfMap:                          
      CPUs: "32"
      Features: "amx"
      RealMemory: "122880"

# partitions 하위는 리스트 형식을 유지하되, nodes 이름이 위와 정확히 일치해야 함
partitions:
  amx:
    enabled: true
    nodesets: 
      - "ns-amx"
    configMap:
      Default: "YES"
      MaxTime: "infinite"
      State: "UP"
EOF
```

helm show values <chart-name> 를 사용하면 차트가 제공하는 values 상세 스펙을 확인할 수 있다. values.yaml 을 수정할때 참고해서 작성해야 한다. 
```
helm show values oci://ghcr.io/slinkyproject/charts/slurm
```

helm 차트를 업데이트 한다. 
```
helm upgrade --install slurm oci://ghcr.io/slinkyproject/charts/slurm \
  --reuse-values \
  --namespace=slurm -f amx-nodeset.yaml
```

slurm 오퍼레이터 로그에 오류가 없는지 확인한다.
```
kubectl logs -n slinky deployment/slurm-operator
```
파드가 대상 노드 그룹의 노드들에 제대로 스케줄링 되었는지 확인한다.
```
kubectl get pods -n slurm -l app.kubernetes.io/instance=slurm-worker-ns-amx 
```
[결과]
```
NAME                    READY   STATUS    RESTARTS   AGE
slurm-worker-ns-amx-0   2/2     Running   0          4m44s
slurm-worker-ns-amx-1   2/2     Running   0          4m44s
slurm-worker-ns-amx-2   2/2     Running   0          4m44s
slurm-worker-ns-amx-3   2/2     Running   0          4m44s
```

slurmctld 파드로 로그인하여 신규로 추가된 파티션을 확인하다.
```
kubectl exec -it slurm-controller-0 -n slurm -c slurmctld -- /bin/bash
slurm@slurm-controller-0:/tmp$ sinfo
```
[결과]
```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
slinky       up   infinite      1   idle slinky-0
all          up   infinite      5   idle ns-amx-[0-3],slinky-0
amx*         up   infinite      4   idle ns-amx-[0-3]
```
amx 파티션의 상세 정보를 확인한다. 
```
slurm@slurm-controller-0:/tmp$ scontrol show partition amx
```
[결과]
```
slurm@slurm-controller-0:/tmp$ scontrol show partition amx
PartitionName=amx
   AllowGroups=ALL AllowAccounts=ALL AllowQos=ALL
   AllocNodes=ALL Default=YES QoS=N/A
   DefaultTime=NONE DisableRootJobs=NO ExclusiveUser=NO ExclusiveTopo=NO GraceTime=0 Hidden=NO
   MaxNodes=UNLIMITED MaxTime=UNLIMITED MinNodes=0 LLN=NO MaxCPUsPerNode=UNLIMITED MaxCPUsPerSocket=UNLIMITED
   NodeSets=ns-amx
   Nodes=ns-amx-[0-3]
   PriorityJobFactor=1 PriorityTier=1 RootOnly=NO ReqResv=NO OverSubscribe=NO
   OverTimeLimit=NONE PreemptMode=OFF
   State=UP TotalCPUs=128 TotalNodes=4 SelectTypeParameters=NONE
   JobDefaults=(null)
   DefMemPerNode=UNLIMITED MaxMemPerNode=UNLIMITED
   TRES=cpu=128,mem=507024M,node=4,billing=128
```

### 2. NVIDIA GPU 파티션 생성 (Karpenter) ###

#### 사전준비 ####

* [디바이스 플러그인을 설치 및 카펜터 GPU 노드풀 생성](https://github.com/gnosia93/slurm-on-eks/blob/main/lession/4-prerequisite.md)

* [Nodeset 오토스케일링을 위한 KEDA 설치](https://github.com/gnosia93/slurm-on-eks/blob/main/lession/4-keda-based-autoscaling.md)

#### GPU 파티션 생성하기 ####
```
cat <<EOF > gpu-nodeset.yaml
nodesets:
  ns-gpu:
    enabled: true
    replicas: 1                            # KEDA/카펜터에 의한 동적 프로비저닝 사용 - 초기값은 0 으로 설정
    updateStrategy:
      type: RollingUpdate
    podSpec:                   
      nodeSelector:                        # node selector 를 이용하여 slurmd 가 설치될 노드를 식별.
        karpenter.sh/nodepool: gpu         # 카펜터 gpu 노드풀로 설정
        node.kubernetes.io/instance-type: p4d.24xlarge        # 특정 인스턴스로 강제
      tolerations:                                             
        - key: "nvidia.com/gpu"            # 노드그룹에 설정된 taint 를 무력화 시키기 위해서 설정
          operator: "Exists"               # 노드의 테인트는 nvidia.com/gpu=present:NoSchedule 이나,   
          effect: "NoSchedule"             # Exists 연산자로 nvidia.com/gpu 키만 체크     
    slurmd:
      image:
        repository: ghcr.io/slinkyproject/slurmd
        tag: 25.11-ubuntu24.04
      resources:
        requests:                          # slurmd 를 스케줄링 하면서 리소스를 선점해 놓는다. slurm 이외에 다른 워크로드의 접근을 차단.
          cpu: "96"                        # 96 vCPU (p4d.24xlarge) 
          memory: "1152Gi"                 # 1152 Gi 메모리 (p4d.24xlarge)
    # LogFile 사이드카 설정
    logfile:
      image:
        repository: docker.io/library/alpine
        tag: latest
    # slurm.conf의 NodeName 줄에 들어갈 내용
    extraConfMap:                          
      CPUs: "96"
      RealMemory: "1152Gi"
      GPUs: "8"

# partitions 하위는 리스트 형식을 유지하되, nodes 이름이 위와 정확히 일치해야 함
partitions:
  gpu:
    enabled: true
    nodesets: 
      - "ns-gpu"
    configMap:
      Default: "NO"
      MaxTime: "infinite"
      State: "UP"
EOF
```
slurm 클러스터에 gpu 노드셋을 추가한다. 
```
helm upgrade --install slurm oci://ghcr.io/slinkyproject/charts/slurm \
  --reuse-values \
  --namespace=slurm -f gpu-nodeset.yaml
```

```
kubectl exec -it slurm-controller-0 -n slurm -c slurmctld -- /bin/bash
slurm@slurm-controller-0:/tmp$ sinfo
```


```
scontrol show node ns-gpu-0
# GPU 1개를 요청하는 인터랙티브 세션 실행
srun --gres=gpu:1 --partition=gpu hostname
Slinky는 GRES 자동 구성을 지원하지만, gres.conf 파일이 컨테이너 내부에 적절히 생성되었는지 kubectl exec으로 들어가 /etc/slurm/gres.conf를 확인
```

* Slinky Autoscaler 활성화: 워크숍 커리큘럼 중 "C3. slinky 설치하기" 단계에서 Autoscaler 옵션이 켜져 있어야 합니다. 이 옵션이 꺼져 있으면 srun을 해도 replicas가 변하지 않아 Karpenter가 반응하지 않습니다.
* Resume/Suspend Program: Slurm 설정(slurm.conf)에 Slinky가 제공하는 스크립트가 ResumeProgram으로 등록되어 있는지 확인이 필요합니다. AWS HPC 가이드에서는 이 스크립트가 Kubernetes API를 호출해 파드 개수를 조절하도록 안내합니다.




## GRES / TRES ##
Slurm 리소스 관리의 핵심인 두 용어는 "무엇을 관리하느냐"와 "어떻게 카운팅하느냐"의 차이입니다. Slinky(AWS) 환경에서는 특히 GPU와 네트워크 대역폭 할당을 위해 이 개념을 정확히 쓰는 것이 중요합니다. SchedMD GRES 문서를 참고하여 정리해 드립니다.

### 1. GRES (Generic Resources) ###
CPU/메모리 외에 사용자가 요청하는 특수 하드웨어로 gpu, mps, fpga 등을 의미한다.
사용자가 sbatch --gres=gpu:8과 같이 요청(Request)할 때 사용된다. 
p4dn.24xlarge 인스턴스가 생성될 때 "이 노드엔 GPU 8개가 있다"고 Slurm에 알려주는 꼬리표 역할을 한다.

### 2. TRES (Trackable Resources) ###
Slurm이 추적하고 기록(Accounting)할 수 있는 모든 자원을 통칭하는 것으로 GRES보다 더 넓은 개념이다.
cpu, mem, node, energy + 모든 GRES 가 포함되는데 주로 관리자가 사용량 제한(Quota)을 걸거나, 나중에 사용자가 자원을 얼마나 썼는지 통계를 낼 때 사용된다.
AWS 비용 최적화를 위해 "특정 사용자가 GPU(TRES)를 100시간 이상 쓰지 못하게 제한"하는 등의 과금 및 관리 정책에 쓰일 수 있다.


## 참고 ##

#### 차트 내려 받기 ####
```
# 1. 차트 파일을 현재 디렉토리에 내려받기
helm pull oci://ghcr.io/slinkyproject/charts/slurm --version 1.0.1
# 2. 압축 풀기
tar -zxvf slurm-1.0.1.tgz
# 3. 파일 위치로 이동하여 내용 보기
cat slurm/templates/nodeset/nodeset-cr.yaml
```
