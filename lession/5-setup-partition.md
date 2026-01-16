## GPU 파티션 만들기 ##

### 1. Cloud-Native 설정 (Helm/K8s) ###
Slinky는 Kubernetes 위에서 Slurm을 돌리는 구조이므로, 파티션 정의는 보통 values.yaml 파일의 clusters 섹션에서 이루어진다.

```
clusters:
  - name: "slinky-cluster"
    partitions:
      - name: "gpu-partition"
        instance_types: ["p4dn.24xlarge"] # AWS 인스턴스 타입 지정
        nodes: 2
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
작업이 들어올 때만 p4dn 같은 고가 자원을 띄우게 할 수 있다.
sinfo에서 확인했을 때 파티션 상태가 idle 혹은 cloud로 보일 수 있는데, 이는 노드가 현재는 없지만, 작업 제출 시 자동으로 생성된다는 뜻이다.
GPU 파티션 설정 시 AWS EFA(Elastic Fabric Adapter) 활성화 옵션이 파티션 정의에 포함되어 있는지 꼭 확인해야 한다.


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

