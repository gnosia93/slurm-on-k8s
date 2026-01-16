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

### slurm 크럴스터 설치 ###
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

#### 3. IRSA 생성 ####
IAM 역할 생성 및 AWS 정책 연결 (EKS 전용 서비스 계정 생성) 한다.
```
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEKS_EBS_CSI_Driver_Policy \
  --approve \
  --role-only \
  --role-name EBS_CSI_DriverRole-${CLUSTER_NAME}
```

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
EOF
```


## 레퍼런스 ##
* https://github.com/SlinkyProject/slurm-operator
* https://aws.amazon.com/ko/blogs/containers/running-slurm-on-amazon-eks-with-slinky/
