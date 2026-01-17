## 파티션 할당하기 ##

Slurm에서 파티션(Partition)은 여러 대의 컴퓨팅 노드를 논리적으로 묶어 놓은 '자원 그룹'이자, 사용자가 작업을 제출하는 '대기열(Queue)'이다. Slurm 공식 문서(SchedMD)에 따르면 파티션은 작업의 특성이나 하드웨어 사양에 따라 시스템을 효율적으로 나누어 관리하는 핵심 단위로, 다음과 같은 요소로 구성되어져 있다. 
* 노드 리스트 (Nodes): 해당 파티션에 포함된 실제 서버(컴퓨팅 노드)들의 목록으로 하나의 노드가 여러 파티션에 중복으로 속할 수도 있다.
* 시간 제한 (MaxTime): 한 작업이 해당 파티션에서 최대 몇 시간 동안 실행될 수 있는지 정의한다. (예: debug 파티션은 30분, batch 파티션은 2일 등)
* 사용자 권한 (AllowGroups): 특정 파티션을 사용할 수 있는 사용자 그룹을 제한하여 보안이나 우선순위를 관리할 수 있다.
* 우선순위 (Priority): 여러 파티션이 동일한 노드를 공유할 때, 어떤 파티션의 작업을 먼저 실행할지 결정한다. 

### 1. AMEX CPU 파티션 생성 ###
ng-amx 매니지드 노드 그룹의 라벨을 확인한다.
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
ng-amx 노드 그룹의 taint 를 확인한다. 
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

노드가 이미 생성되어 있으므로 Slinky에게 "동적으로 띄우지 말고, 이 라벨이 붙은 노드를 파티션으로 써라"고 알려준다. 이때 Toleration 도 함께 설정한다. 
파티션 설정에 Toleration이 포함되는 이유는 "해당 파티션으로 제출된 모든 작업(Pod)에 이 출입증을 자동으로 달아주기 위함" 이다. 
```
cat <<EOF > amx-nodeset.yaml
# nodesets 아래에 바로 이름을 키로 사용합니다 (리스트 '-' 제거)
nodesets:
  ns-amx:
    enabled: true
    replicas: 4                # count 대신 replicas를 사용 (Slinky 1.0.1 규격)
    updateStrategy:
      type: RollingUpdate
    podSpec:
      nodeSelector:
        workload-type: "slurm-compute"
        architecture: "amx-enabled"
      tolerations:
        - key: "workload"
          operator: "Equal"
          value: "slurm"
          effect: "NoSchedule"
    slurmd:
      image:
        repository: ghcr.io/slinkyproject/slurmd
        tag: 25.11-ubuntu24.04
      resources:
        limits:
          cpu: "30"
          memory: "120Gi"
    # LogFile sidecar configurations.
    logfile:
      image:
        repository: docker.io/library/alpine
        tag: latest
    extraConfMap:
      CPUs: "32"
      Features: "amx"

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




Slinky는 Slurm의 작업 요청을 Kubernetes의 Pod 요청으로 변환하고, 이때 Karpenter(카펜터)가 이 Pod을 보고 "p4dn 2대가 필요하네?"라며 AWS EC2를 즉시 생성하여 클러스터에 붙인다.
sinfo에서 확인했을 때 파티션 상태가 idle 혹은 cloud로 보일 수 있는데, 이는 노드가 현재는 없지만, 작업 제출 시 자동으로 생성된다는 뜻이다.
GPU 파티션 설정 시 AWS EFA(Elastic Fabric Adapter) 활성화 옵션이 파티션 정의에 포함되어 있는지 꼭 확인해야 한다.


* 카펜터 설치
* 노드풀 설정

* 3. Slinky와의 연결 (Taint & Toleration)
이게 가장 중요합니다! Slurm 작업이 들어왔을 때 카펜터가 "아, 이건 Slurm용 노드구나"라고 알 수 있도록 Taint(용인) 설정을 맞춰야 합니다.
Slurm 파티션 설정: Helm values.yaml의 partitions 섹션에 해당 노드풀의 레이블이나 Taint를 기입합니다.
동작 원리: sbatch 제출 → Slinky가 Pod 생성 → Pod에 slurm-job 관련 Toleration 부여 → 카펜터가 이를 보고 일치하는 NodePool에서 p4dn 실행.

* 4. 주의사항 (Scale-down)
Time-to-Live (TTL): 작업이 끝나고 노드가 즉시 삭제되길 원한다면 카펜터 설정에서 disruption.consolidationPolicy: WhenEmpty를 설정하세요. Karpenter 정지 설정 가이드에서 상세 내용을 볼 수 있습니다.
결론적으로, 카펜터 설치 + 노드풀 설정 + Slinky 파티션 레이블 매칭 이 3박자가 맞으면 자동으로 p4dn이 생겼다 사라졌다 하는 동적 환경이 완성됩니다.
현재 노드풀 YAML을 직접 작성 중이신가요? 아니면 기존에 설치된 카펜터에 p4dn만 추가하려 하시나요? Spot 인스턴스 사용 여부를 알려주시면 비용 최적화 옵션도 덧붙여 드릴 수 있습니다.


* Slinky 환경에서 Slurm 파티션과 Karpenter 노드풀을 연결하는 핵심은 "이 파티션에 제출된 작업은 반드시 이 노드(Karpenter가 띄운 노드) 위에서만 실행되어야 한다"는 제약 조건을 거는 것입니다.
* Slinky Helm Chart 가이드와 일반적인 Slurm-on-K8s 구조에 따르면, values.yaml에 아래와 같이 nodeSelector와 tolerations를 명시해야 합니다.

[values.yaml]
```
clusters:
  - name: "slinky-cluster"
    partitions:
      - name: "gpu-partition"
        instance_types: ["p4dn.24xlarge"]
        # 1. 노드 선택 (NodePool의 labels와 일치해야 함)
        nodeSelector:
          karpenter.sh/nodepool: slurm-gpu-pool
        
        # 2. 테인트 허용 (NodePool에 설정된 taints가 있다면 필수)
        tolerations:
          - key: "slinky.io/usage"
            operator: "Equal"
            value: "gpu-task"
            effect: "NoSchedule"
        
        gres: "gpu:8"

```

```
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: slurm-gpu-pool
spec:
  template:
    spec:
      requirements:
        - key: "node.kubernetes.io/instance-type"
          operator: In
          values: ["p4dn.24xlarge"]
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["on-demand"] # 또는 spot
      nodeClassRef:
        name: slurm-gpu-nodeclass
---
apiVersion: karpenter.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: slurm-gpu-nodeclass
spec:
  amiFamily: AL2 # 또는 Bottlerocket
  subnetSelectorTerms:
    - tags: { "karpenter.sh/discovery": "my-cluster" }
  securityGroupSelectorTerms:
    - tags: { "karpenter.sh/discovery": "my-cluster" }
  # p4dn을 위한 EFA 설정은 AMI 내부에 구성되거나 UserData로 처리
```


### 3. 파티션 확인하기 ###
```
scontrol show config | grep ClusterName
scontrol show partition gpu-partition
```

🚀 다음 액션 제안 :

* Slinky 환경은 Node Selector나 Toleration 같은 쿠버네티스 개념이 Slurm 파티션과 연결되어 작동한다. 
* 혹시 현재 새로운 인스턴스 타입을 추가하려 하시나요, 아니면 기존 파티션의 타임아웃(Timeout) 설정을 변경하려 하시나요? 


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
* slurm 차트 value 확인
```
helm show values oci://ghcr.io/slinkyproject/charts/slurm | grep -A 50 "partitions"
```

* 차트 내려 받기
```
# 1. 차트 파일을 현재 디렉토리에 내려받기
helm pull oci://ghcr.io/slinkyproject/charts/slurm --version 1.0.1
# 2. 압축 풀기
tar -zxvf slurm-1.0.1.tgz
# 3. 파일 위치로 이동하여 내용 보기
cat slurm/templates/nodeset/nodeset-cr.yaml
```
