# slurm-on-eks

본 워크숍은 Amazon EKS에서 Slurm과 Slinky를 이용해 Llama 3 등 대규모 모델을 최적으로 학습시키는 방법을 소개합니다. 전통적인 HPC 스케줄러인 Slurm은 모든 자원이 준비되었을 때 작업을 시작하는 '갱 스케줄링'을 통해 대규모 GPU 자원 관리의 안정성을 보장합니다. 오픈소스 프로젝트인 Slinky는 이러한 Slurm의 강점을 쿠버네티스 포드(Pod)에 투명하게 연결합니다. 이번 세션을 통해 참가자들은 별도의 복잡한 인프라 구성 없이도 쿠버네티스 환경 내에서 Slurm의 성능을 마스터하고, 최고 수준의 트레이닝 효율을 직접 경험하시게 될 것입니다.


* http://ec2-43-203-227-222.ap-northeast-2.compute.amazonaws.com:9090


### _Topics_ ###
  
* [C1. VPC 생성하기](https://github.com/gnosia93/slurm-on-k8s/tree/main/tf)
  
* [C2. EKS 클러스터 생성하기](https://github.com/gnosia93/slurm-on-k8s/blob/main/lession/2-building-eks-cluster.md)

* [C3. slinky 설치하기](https://github.com/gnosia93/slurm-on-k8s/blob/main/lession/3-install-slinky.md)

* [C4. 파티션 설정하기](https://github.com/gnosia93/slurm-on-eks/blob/main/lession/4-setup-partition.md)

* [C5. slurm 기본 명령어 익히기](https://github.com/gnosia93/slinky-on-k8s/blob/main/lession/5-get-started-slurm.md)

* [C6. 복원력 및 작업 재시작](https://github.com/gnosia93/slurm-on-eks/blob/main/lession/6-resilliency.md)

* [C7. 작업 모니터링 하기]

* [C8. 체크 포인트 저장하기]
    
* 노드 간 통신 병목을 해결하여 GPU 사용률(Utilization)을 90% 이상으로 끌어올리는 방법.
```
성능 확인 방법
학습 중 노드에 접속하여 아래 명령어로 확인해 보세요.
bash
# GPU 사용률 및 전력 소모 확인 (전력 소모가 높을수록 일을 많이 하는 중)
nvidia-smi dmon -s uc

# EFA 통신량 확인
watch -n 1 "fi_info -p efa"
```

* 내용: Slurm의 Job ID 기반으로 어떤 팀(또는 사용자)이 자원을 얼마나 썼는지 계산하고, 특정 예산을 초과하면 작업을 강제 종료하거나 알림을 보내는 정책 설정.
* 핵심: Karpenter의 Termination 정책과 연동하여 '좀비 노드' 방지하기.
* 내용: AWS FSx for Lustre를 Slurm 노드에 마운트하여 데이터 로딩 속도와 체크포인트 저장 속도를 극대화하는 법.
---

1. Pod 기반 로그인 노드 (가장 간편한 방식)
EKS 내부에 slurm-client용 Pod을 하나 띄워두고 kubectl exec로 접속해 사용하는 방식입니다. Slinky Helm 차트에 기본 포함되어 있는 경우가 많습니다.

```
loginNode:
  enabled: true
  image: "slinky-slurm-client:latest"
  # FSx 마운트 설정을 공유하여 로그인 노드에서도 데이터를 볼 수 있게 함
  extraVolumeMounts:
    - name: fsx-storage
      mountPath: /shared
```

```
kubectl exec -it deployment/slurm-login-node -n slurm -- /bin/bash

```
### __ref__ ##
* https://www.youtube.com/watch?v=Vi8chqAFuN0
