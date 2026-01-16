Slinky(Slurm on EKS) 환경에서 로그를 그라파나(Grafana)로 보내는 가장 표준적인 방법은 Loki를 활용하는 것입니다.
분산 학습 로그는 워낙 양이 많기 때문에, 전체를 DB에 넣기보다는 [Promtail/Fluent Bit(수집) → Loki(저장) → Grafana(시각화)] 스택을 사용하는 것이 비용과 성능 면에서 유리합니다. Grafana Loki 공식 가이드를 참고하여 Slurm 맞춤형 구성을 정리해 드립니다.


### 1. 로그 수집기 설정 (Promtail 또는 Fluent Bit) ###
각 컴퓨팅 노드(p4dn)에서 발생하는 로그와 Slurm 컨트롤러 로그를 가로챕니다.

수집 대상:
* Slurm 시스템 로그: /var/log/slurmctld.log, /var/log/slurmd.log
* 작업 출력 로그: /shared/logs/*.out (FSx에 저장된 sbatch 결과물)
* 설정: Slinky 설치 시 values.yaml에 Loki 주소를 전달하여 로그를 쏘도록 설정합니다.

### 2. Grafana Loki 설치 (Helm) ###
EKS 클러스터에 로그 저장소인 Loki를 설치합니다.



* CloudWatch 활용
