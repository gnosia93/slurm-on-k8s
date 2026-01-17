## GPU 파티션 만들기 ##

Slurm에서 파티션(Partition)은 여러 대의 컴퓨팅 노드를 논리적으로 묶어 놓은 '자원 그룹'이자, 사용자가 작업을 제출하는 '대기열(Queue)'이다. Slurm 공식 문서(SchedMD)에 따르면 파티션은 작업의 특성이나 하드웨어 사양에 따라 시스템을 효율적으로 나누어 관리하는 핵심 단위로, 다음과 같은 요소로 구성되어져 있다. 
* 노드 리스트 (Nodes): 해당 파티션에 포함된 실제 서버(컴퓨팅 노드)들의 목록으로 하나의 노드가 여러 파티션에 중복으로 속할 수도 있다.
* 시간 제한 (MaxTime): 한 작업이 해당 파티션에서 최대 몇 시간 동안 실행될 수 있는지 정의한다. (예: debug 파티션은 30분, batch 파티션은 2일 등)
* 사용자 권한 (AllowGroups): 특정 파티션을 사용할 수 있는 사용자 그룹을 제한하여 보안이나 우선순위를 관리할 수 있다.
* 우선순위 (Priority): 여러 파티션이 동일한 노드를 공유할 때, 어떤 파티션의 작업을 먼저 실행할지 결정한다. 

### 1. 정적 노드 프로비저닝 ###
1. 노드그룹 생성
```
managedNodeGroups:
  - name: static-p4dn-group
    instanceType: p4dn.24xlarge
    minSize: 2
    maxSize: 2
    desiredCapacity: 2 # 2대 상시 유지
    volumeSize: 500
    efaEnabled: true   # p4dn의 핵심 기능
    labels:
      role: slurm-static-gpu # Slinky가 찾을 수 있게 라벨 부여
    taints:
      - key: "slinky.io/usage"
        value: "gpu-task"
        effect: "NoSchedule"

```

2. Slinky Helm values.yaml 연결
노드가 이미 떠 있으므로, Slinky에게 "동적으로 띄우지 말고, 이 라벨이 붙은 노드를 파티션으로 써라"고 알려줍니다.

```
yaml
clusters:
  - name: "slinky-cluster"
    partitions:
      - name: "static-gpu-partition"
        # 중요: Karpenter 설정 대신 고정된 노드 선택기 사용
        nodeSelector:
          role: slurm-static-gpu
        tolerations:
          - key: "slinky.io/usage"
            operator: "Equal"
            value: "gpu-task"
            effect: "NoSchedule"
        gres: "gpu:8"
```




### 2. 동적 프로비저닝 ###
Slinky는 Kubernetes 위에서 Slurm을 돌리는 구조이므로, 파티션 정의는 보통 values.yaml 파일의 clusters 섹션에서 이루어진다.

```
clusters:
  - name: "slinky-cluster"
    partitions:
      - name: "gpu-partition"
        instance_types: ["p4dn.24xlarge"] # AWS 인스턴스 타입 지정
        nodes: 2
        # EFA 활성화 핵심 설정
        efa_enabled: true  
        # 또는 annotation/label로 처리하는 경우
        labels:
          ://vpc.amazonaws.com: "true" 
        gres: "gpu:8"
      - name: "cpu-partition"
        instance_types: ["c5.24xlarge"]
        nodes: 10
```
helm 으로 설정을 업그레드 한다.
```
helm list -A
helm upgrade slinky slinky-chart/slinky -f values.yaml --namespace slurm
```

```
kubectl logs -l app.kubernetes.io/name=slinky-controller -n slurm
```
터미널에서 sinfo를 입력하여 gpu-partition과 cpu-partition이 리스트에 뜨는지 확인한다.
```
sinfo
```
사양이 GRES나 TRES에 제대로 반영되었는지 확인한다.
```
scontrol show partition gpu-partition
```

### 2. 동적 노드 프로비저닝 (Auto-scaling) ###
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

