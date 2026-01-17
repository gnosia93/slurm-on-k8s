Slurm Slinky는 초거대 규모 HPC 클러스터의 병목 현상을 해결하기 위해 SchedMD가 설계한 쿠버네티스 기반의 차세대 제어 계층 아키텍처이다. 기존의 단일 slurmctld 구조를 마이크로서비스로 재설계하고 고성능 저장소인 etcd 및 메시지 버스를 도입하여, 수만 개의 노드 환경에서도 시스템 응답성과 자원 오케스트레이션의 확장성을 극대화한 것이 핵심이다. 특히 기존 Slurm 명령어 및 설정과의 완벽한 하위 호환성을 유지함으로써 운영의 연속성을 보장하며, 온프레미스와 클라우드 자원을 단일 파티션 내에서 통합 관리할 수 있는 하이브리드 클라우드 최적화 환경을 제공한다.

Slurm Slinky 아키텍처를 구성하는 핵심 기술 요소는 etcd, 쿠버네티스 CRD, 그리고 메시지 버스 이다.
* etcd 기반 상태 동기화: 기존 Slurm이 로컬 메모리와 파일 시스템에 의존하여 상태를 저장하던 방식과 달리, 분산 키-값 저장소인 etcd를 활용한다. 이를 통해 수만 개의 노드와 작업 상태 데이터를 여러 컨트롤러가 실시간으로 공유하며, 특정 노드나 프로세스에 장애가 발생해도 데이터 유실 없이 즉각적인 고가용성(High Availability)을 보장한다.
* 쿠버네티스 커스텀 리소스(CRD) 활용: Slurm의 노드, 작업(Job), 파티션 정보를 쿠버네티스의 표준 객체처럼 다룰 수 있도록 CRD(Custom Resource Definition)로 추상화한다. 이를 통해 쿠버네티스의 기본 기능인 자동 복구(Self-healing)와 오토스케일링 메커니즘을 Slurm 자원 관리에 그대로 적용하여 운영 자동화 수준을 높인다.
* 고성능 메시지 버스 아키텍처: 컨트롤러와 수많은 에이전트(slurmd) 간의 통신을 직접 연결 방식이 아닌 분산 메시지 큐 방식으로 전환한다. 이 구조는 대규모 작업이 한꺼번에 제출될 때 발생하는 네트워크 트래픽 급증을 완충하는 버퍼 역할을 수행하며, 전체 시스템의 응답 속도를 일정하게 유지하는 데 기여한다.

또한 하이브리드 환경에서 서로 다른 인프라(온프레미스 베어메탈, 클라우드 가상 머신 등)를 동일한 방식으로 제어할 수 있도록 통신 프로토콜을 표준화하여 관리자는 하드웨어의 물리적 위치에 상관없이 단일 Slurm 컨트롤러 하에서 통합된 자원 풀을 운영할 수 있다.

### cert-manager 설치 ###
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --set 'crds.enabled=true' \
  --namespace cert-manager --create-namespace
```

### slurm CRD/오퍼레이터 설치 ###
```
helm install slurm-operator-crds oci://ghcr.io/slinkyproject/charts/slurm-operator-crds
helm install slurm-operator oci://ghcr.io/slinkyproject/charts/slurm-operator \
  --namespace=slinky --create-namespace
```

### slurm 클러스터 설치 ###
```
helm install slurm oci://ghcr.io/slinkyproject/charts/slurm \
  --namespace=slurm --create-namespace
```
[결과]
```
Pulled: ghcr.io/slinkyproject/charts/slurm:1.0.1
Digest: sha256:a11e2e84e528299884b72ce15d9bf548c3b1d7d391ef834a25f0a38244b5f4f9
NAME: slurm
LAST DEPLOYED: Fri Jan 16 12:15:56 2026
NAMESPACE: slurm
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
********************************************************************************

                                 SSSSSSS
                                SSSSSSSSS
                                SSSSSSSSS
                                SSSSSSSSS
                        SSSS     SSSSSSS     SSSS
                       SSSSSS               SSSSSS
                       SSSSSS    SSSSSSS    SSSSSS
                        SSSS    SSSSSSSSS    SSSS
                SSS             SSSSSSSSS             SSS
               SSSSS    SSSS    SSSSSSSSS    SSSS    SSSSS
                SSS    SSSSSS   SSSSSSSSS   SSSSSS    SSS
                       SSSSSS    SSSSSSS    SSSSSS
                SSS    SSSSSS               SSSSSS    SSS
               SSSSS    SSSS     SSSSSSS     SSSS    SSSSS
          S     SSS             SSSSSSSSS             SSS     S
         SSS            SSSS    SSSSSSSSS    SSSS            SSS
          S     SSS    SSSSSS   SSSSSSSSS   SSSSSS    SSS     S
               SSSSS   SSSSSS   SSSSSSSSS   SSSSSS   SSSSS
          S    SSSSS    SSSS     SSSSSSS     SSSS    SSSSS    S
    S    SSS    SSS                                   SSS    SSS    S
    S     S                                                   S     S
                SSS
                SSS
                SSS
                SSS
 SSSSSSSSSSSS   SSS   SSSS       SSSS    SSSSSSSSS   SSSSSSSSSSSSSSSSSSSS
SSSSSSSSSSSSS   SSS   SSSS       SSSS   SSSSSSSSSS  SSSSSSSSSSSSSSSSSSSSSS
SSSS            SSS   SSSS       SSSS   SSSS        SSSS     SSSS     SSSS
SSSS            SSS   SSSS       SSSS   SSSS        SSSS     SSSS     SSSS
SSSSSSSSSSSS    SSS   SSSS       SSSS   SSSS        SSSS     SSSS     SSSS
 SSSSSSSSSSSS   SSS   SSSS       SSSS   SSSS        SSSS     SSSS     SSSS
         SSSS   SSS   SSSS       SSSS   SSSS        SSSS     SSSS     SSSS
         SSSS   SSS   SSSS       SSSS   SSSS        SSSS     SSSS     SSSS
SSSSSSSSSSSSS   SSS   SSSSSSSSSSSSSSS   SSSS        SSSS     SSSS     SSSS
SSSSSSSSSSSS    SSS    SSSSSSSSSSSSS    SSSS        SSSS     SSSS     SSSS

********************************************************************************

CHART NAME: slurm
CHART VERSION: 1.0.1
APP VERSION: 25.11

slurm has been installed. Check its status by running:
  $ kubectl --namespace=slurm get pods -l helm.sh/chart=slurm-1.0.1 --watch

Learn more about Slurm:
  - Overview: https://slurm.schedmd.com/overview.html
  - Quickstart: https://slurm.schedmd.com/quickstart.html
  - Documentation: https://slurm.schedmd.com/documentation.html
  - Support: https://www.schedmd.com/slurm-support/our-services/
  - File Tickets: https://support.schedmd.com/

Learn more about Slinky:
  - Overview: https://www.schedmd.com/slinky/why-slinky/
  - Documentation: https://slinky.schedmd.com
```

## Storage Class 설정(gp3) ##
```
export AWS_REGION=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].RegionName' --output text)
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME="slinky-on-k8s"
```
#### 1. OIDC 확인 ####
OIDC 가 설치되어 있는지 확인한다. 
```
aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text
```
[결과]
```
https://oidc.eks.ap-northeast-2.amazonaws.com/id/FD17E419F758EAAE2455EEEF9F2D40B5
```
#### 2. 애드온 확인 ####
aws-ebs-csi-driver 애드온이 설치되어 있는지 확인한다. 
```
aws eks list-addons --cluster-name ${CLUSTER_NAME} --output=text
```
[결과]
```
ADDONS  aws-ebs-csi-driver
ADDONS  coredns
ADDONS  eks-pod-identity-agent
ADDONS  kube-proxy
ADDONS  metrics-server
ADDONS  vpc-cni
```

#### 3. Role 및 IRSA 생성 ####
IAM 역할 생성 및 AWS 정책 연결 (EKS 전용 서비스 계정 생성) 한다.
```
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEKS_EBS_CSI_Driver_Policy \
  --approve \
  --role-name EBS_CSI_DriverRole-${CLUSTER_NAME}
```
* 클라우드 포메이션 에러가 발생하였다. 콘솔에서 확인하고 재설치 해야 한다..


#### 4. 스토리지 클래스 생성 ####
default 스토리지 클래스를 gp3 타입으로 생성한다.
```
cat <<EOF | kubectl apply -f - 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  #gp3의 기본 IOPS와 Throughput을 지정할 수 있습니다. (선택 사항)
  #iopsPerGB: "3000"
  #throughput: "125"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```
생성된 스토리지 클래스를 조회한다. 
```
kubectl get sc
```
[결과]
```
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2             kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  20h
gp3 (default)   ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   82s
```

## 레퍼런스 ##
* https://github.com/SlinkyProject/slurm-operator
* https://aws.amazon.com/ko/blogs/containers/running-slurm-on-amazon-eks-with-slinky/
