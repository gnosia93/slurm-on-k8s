## 사전준비 ##
slurm 클러스터에서 GPU 및 efa 디바이스를 사용하기 위해 NVIDIA 및 efa 디바이스 플러그인을 설치하고, 카펜터를 활용한 동적 노드 프로비저닝을 위해 EC2Nodeclass 및 노드풀을 생성한다.   

### [1. NVIDIA 디바이스 플러그인 설치](https://docs.aws.amazon.com/eks/latest/userguide/ml-eks-k8s-device-plugin.html) ###
```
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
helm search repo nvdp --devel

helm install nvdp nvdp/nvidia-device-plugin \
  --namespace nvidia \
  --create-namespace \
  --version 0.18.0 \
  --set gfd.enabled=true
```
```
kubectl get daemonset -n nvidia
```
[결과]
```
NAME                                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
nvdp-node-feature-discovery-worker                2         2         0       2            0           <none>                        5s
nvdp-nvidia-device-plugin                         0         0         0       0            0           <none>                        5s
nvdp-nvidia-device-plugin-gpu-feature-discovery   0         0         0       0            0           <none>                        5s
nvdp-nvidia-device-plugin-mps-control-daemon      0         0         0       0            0           nvidia.com/mps.capable=true   5s
```
실제 노드의 갯수는 6대 이지만, ng-amx 에 속한 4대의 경우 taint 가 설정되어 있어 nvidia-device-plug-in 데몬셋이 랜딩하지 못한다.

### 2. efa 디바이스 플러그인 설치 ###
```
helm repo add eks https://aws.github.io/eks-charts
helm install aws-efa-k8s-device-plugin eks/aws-efa-k8s-device-plugin --namespace kube-system

kubectl patch ds aws-efa-k8s-device-plugin -n kube-system --type='json' -p='[
  {"op": "add", "path": "/spec/template/spec/tolerations/-", "value": {"operator": "Exists"}}
]'

kubectl get ds aws-efa-k8s-device-plugin -n kube-system
```
[결과]
```
NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
aws-efa-k8s-device-plugin   0         0         0       0            0           <none>          13s
```

### 3. GPU 노드풀 생성 ###
```
cat <<EOF > nodepool-gpu.yaml 
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu
spec:
  template:
    metadata:
      labels:
        nodeType: "nvidia" 
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["g", "p"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: gpu
      expireAfter: 720h # 30 * 24h = 720h
      taints:
      - key: "nvidia.com/gpu"            # nvidia-device-plugin 데몬은 nvidia.com/gpu=present:NoSchedule 테인트를 Tolerate 한다. 
        value: "present"                 # value 값으로 present 와 다른값을 설정하면 nvidia-device-plugin 이 동작하지 않는다 (GPU를 찾을 수 없다)   
        effect: NoSchedule               # nvidia-device-plugin 이 GPU 를 찾으면 Nvidia GPU 관련 각종 테인트와 레이블 등을 노드에 할당한다.  
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmpty       # 이전 설정값은 WhenEmptyOrUnderutilized / 노드의 잦은 Not Ready 상태로의 변경으로 인해 수정  
    consolidateAfter: 20m
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: gpu
spec:
  role: "eksctl-KarpenterNodeRole-training-on-eks"
  amiSelectorTerms:
    # Required; when coupled with a pod that requests NVIDIA GPUs or AWS Neuron
    # devices, Karpenter will select the correct AL2023 accelerated AMI variant
    # see https://aws.amazon.com/ko/blogs/containers/amazon-eks-optimized-amazon-linux-2023-accelerated-amis-now-available/
    # EKS GPU Optimized AMI: NVIDIA 드라이버와 CUDA 런타임만 포함된 가벼운 이미지 (Karpenter가 자동으로 선택 가능) 가 설치됨.
    # 특정 DLAMI 가 필요한 경우 - name : 필드에 정의해야 함. 
    - alias: al2023@latest
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "training-on-eks" 
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "training-on-eks" 
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 300Gi
        volumeType: gp3
EOF
```

```
kubectl apply -f nodepool-gpu.yaml
```

## 테스트 하기 ##

### 1. GPU 테스트 ###
```
cat <<EOF | kubectl apply -f - 
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  restartPolicy: Never                                # 재시작 정책을 Never로 설정 (실행 완료 후 다시 시작하지 않음)
  containers:                                         # 기본값은 Always - 컨테이너가 성공적으로 종료(exit 0)되든, 에러로 종료(exit nonzero)되든 상관없이 항상 재시작
    - name: cuda-container                            # nvidia-smi만 실행하고 끝나는 파드에 이 정책이 적용되면, 종료 후 다시 실행을 반복하다가 결국 CrashLoopBackOff 상태가 됨.
      image: nvidia/cuda:13.0.2-runtime-ubuntu22.04    
      command: ["/bin/sh", "-c"]
      args: ["nvidia-smi && sleep 300"]                # nvidia-smi 실행 후 300초(5분) 동안 대기
      resources:
        limits:
          nvidia.com/gpu: 1
  tolerations:                                             
    - key: "nvidia.com/gpu"
      operator: "Exists"                      # 노드의 테인트는 nvidia.com/gpu=present:NoSchedule 이나,   
      effect: "NoSchedule"                    # Exists 연산자로 nvidia.com/gpu 키만 체크         
EOF
```

파드를 생성하고 nvidia-smi 가 동작하는 확인한다.  
```
kubectl get pods
kubectl logs gpu-pod
```
[출력]
```
Wed Dec 10 06:44:46 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.195.03             Driver Version: 570.195.03     CUDA Version: 13.0     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA T4G                     On  |   00000000:00:1F.0 Off |                    0 |
| N/A   49C    P8              9W /   70W |       0MiB /  15360MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

### 2. EFA 테스트 ### 
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: efa-test-pod
  labels:
    app: efa-test
spec:
  nodeSelector:
    karpenter.sh/nodepool: gpu                
  tolerations:                                             
    - key: "nvidia.com/gpu"
      operator: "Exists"                      # 노드의 테인트는 nvidia.com/gpu=present:NoSchedule 이나, Exists 연산자로 nvidia.com/gpu 키만 체크  
      effect: "NoSchedule"
  containers:
    - name: efa-container                               # public.ecr.aws/deep-learning-containers/pytorch-training:2.8.0-gpu-py312-cu129-ubuntu22.04-ec2-v1.0 
      image: public.ecr.aws/hpc-cloud/nccl-tests:latest           # EFA 드라이버와 NCCL 테스트 도구가 포함된 이미지 사용 (NVIDIA 공식 이미지 권장)
      command: ["/bin/bash", "-c", "sleep infinity"]
      resources:
        limits:
          vpc.amazonaws.com/efa: 1                      # EFA 장치를 파드에 직접 할당 (VPC CNI가 이 장치를 인식함)
          nvidia.com/gpu: 1                             # GPU 인스턴스인 경우
      securityContext:
        capabilities:                                   # EFA 통신을 위해 메모리 잠금 권한 필요
          add: ["IPC_LOCK"]
EOF
```
EFA는 하드웨어가 시스템 메모리에 직접 접근하여 데이터를 읽고 쓰는 RDMA(Remote Direct Memory Access) 기술을 사용한다. 통신에 사용되는 메모리 주소가 스왑 처리되어 디스크로 이동해버리면 하드웨어가 메모리를 찾지 못해 시스템 장애나 통신 에러가 발생한다. IPC_LOCK은 해당 메모리를 RAM에 "고정"시켜 이 문제를 방지한다. 
```
kubectl exec -it efa-test-pod -- /bin/bash
fi_info -p efa
```
[결과]
```
provider: efa
    fabric: efa-direct
    domain: rdmap47s0-rdm
    version: 201.0
    type: FI_EP_RDM
    protocol: FI_PROTO_EFA
provider: efa
    fabric: efa
    domain: rdmap47s0-rdm
    version: 201.0
    type: FI_EP_RDM
    protocol: FI_PROTO_EFA
provider: efa
    fabric: efa
    domain: rdmap47s0-dgrm
    version: 201.0
    type: FI_EP_DGRAM
    protocol: FI_PROTO_EFA
```
* 성공 시: provider: efa, fabric: efa와 같은 정보가 상세하게 출력된다.
* 실패 시: fi_info 결과에 아무것도 나오지 않거나 에러가 발생한다. 이 경우 보안 그룹의 인/아웃 바운드 셀프 참조 존재여부를 확인한다. 
