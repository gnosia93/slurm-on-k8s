# slurm-on-eks

본 워크숍은 Amazon EKS에서 Slurm Slinky를 이용하여 대규모 모델을 최적으로 학습시키는 방법에 대해서 다룬다. Slurm은 오픈 소스 작업 스케줄러이자 리소스 관리자로 대규모 클러스터 환경에서 한정된 하드웨어 자원을 효율적으로 분배하기 위해 만들어졌다. 전통적인 HPC 스케줄러인 Slurm은 모든 자원이 준비되었을 때 작업을 시작하는 '갱 스케줄링(Gang Scheduling)'을 통해 대규모 GPU 자원 관리의 안정성을 보장하는데, Slinky는 이러한 Slurm의 강점을 쿠버네티스 파드(Pod)에 투명하게 연결한다. 이 워크숍을 통해 쿠버네티스 환경에서 Slurm을 활용하는 핵심 아키텍처와 관련 노하우를 배우게 된다.

![](https://github.com/gnosia93/slurm-on-eks/blob/main/image/slurm-on-eks-archi.png)

### _Topics_ ###
  
* [C1. VPC 생성하기](https://github.com/gnosia93/slurm-on-k8s/tree/main/tf)
  
* [C2. EKS 클러스터 생성하기](https://github.com/gnosia93/slurm-on-k8s/blob/main/lesson/2-building-eks-cluster.md)

* [C3. slinky 설치하기](https://github.com/gnosia93/slurm-on-k8s/blob/main/lesson/3-install-slinky.md)

* [C4. 파티션(큐) 할당하기 - 작성중](https://github.com/gnosia93/slurm-on-eks/blob/main/lesson/4-creating-slurm-partition.md)

* C5. 작업 제출하기
  * AMX Lllama 전처리
  * GPU Lllama 훈련

* [C6. 복원력 및 작업 재시작 - 작성중](https://github.com/gnosia93/slurm-on-eks/blob/main/lesson/6-resilliency.md)

* [C7. 작업 모니터링 하기 - 작성중](https://github.com/gnosia93/slurm-on-eks/blob/main/lesson/7-job-monitoring.md)

* [C8. 데이터 전처리와 체크포인팅 - 작성중](https://github.com/gnosia93/slurm-on-eks/blob/main/lesson/8-preprocessing-chekpointing.md)

* [C9. 파이프라인 구성하기 - 작성중](https://github.com/gnosia93/slurm-on-eks/blob/main/lesson/9-making-pipeline.md)

### _Revison History_ ###

* _2026-01-18 Draft Version Uploaded_

### _refs_ ###

* https://slurm.schedmd.com/slinky.html
