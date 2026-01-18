# slurm-on-eks

본 워크숍은 Amazon EKS에서 Slurm과 Slinky를 이용해 Llama 3 등 대규모 모델을 최적으로 학습시키는 방법을 소개한다.
전통적인 HPC 스케줄러인 Slurm은 모든 자원이 준비되었을 때 작업을 시작하는 '갱 스케줄링(Gang Scheduling)'을 통해 대규모 GPU 자원 관리의 안정성을 보장한다. 오픈소스 프로젝트인 Slinky는 이러한 Slurm의 강점을 쿠버네티스 파드(Pod)에 투명하게 연결한다.
참가자들은 이 워크숍을 통해 복잡한 인프라 구성 없이도 쿠버네티스 환경 내에서 Slurm을 활용하는 핵심 아키텍처와 실전 운용 노하우를 배우게 된다.


### _Topics_ ###
  
* [C1. VPC 생성하기](https://github.com/gnosia93/slurm-on-k8s/tree/main/tf)
  
* [C2. EKS 클러스터 생성하기](https://github.com/gnosia93/slurm-on-k8s/blob/main/lession/2-building-eks-cluster.md)

* [C3. slinky 설치하기](https://github.com/gnosia93/slurm-on-k8s/blob/main/lession/3-install-slinky.md)

* [C4. 파티션(큐) 할당하기](https://github.com/gnosia93/slurm-on-eks/blob/main/lession/4-creating-slurm-partition.md)

* [C5. slurm 명령어 살펴보기](https://github.com/gnosia93/slurm-on-eks/blob/main/lession/5-slurm-command.md)

* C6. 작업 제출하기
  * AMX Lllama
  * GPU Lllama

* [C6. 복원력 및 작업 재시작](https://github.com/gnosia93/slurm-on-eks/blob/main/lession/6-resilliency.md)

* [C7. 작업 모니터링 하기](https://github.com/gnosia93/slurm-on-eks/blob/main/lession/7-job-monitoring.md)

* [C8. 데이터 전처리와 체크포인팅](https://github.com/gnosia93/slurm-on-eks/blob/main/lession/8-preprocessing-chekpointing.md)

---
* 스토리지 ??? : Llama 3 같은 대용량 모델은 데이터 로딩 속도가 중요.
* 3. 고속 전처리 전략 (Slurm Job 활용)
FSx for Lustre는 수백 Gbps의 대역폭을 제공하므로, 전처리 작업 자체를 Slurm의 CPU 전용 파티션에서 병렬로 돌리는 것이 좋습니다.
방법: Raw 데이터를 FSx 위에서 병렬로 읽어 토크나이징(Tokenizing)하거나 TFRecord/WebDataset 형태로 변환한 뒤, 동일한 FSx 경로에 저장합니다.
성능 팁: Lustre 스트라이핑(Striping) 설정을 통해 대용량 파일을 여러 객체 저장소에 분산 저장하면 대규모 GPU 학습 시 읽기 성능이 극대화됩니다.

