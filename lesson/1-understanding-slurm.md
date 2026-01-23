## Slurm 이란 ##

Slurm은 Simple Linux Utility for Resource Management의 약자이다. 
이름은 'Simple'이라고 시작하지만, 실제로는 전 세계 슈퍼컴퓨터와 AI 클러스터에서 가장 많이 사용되는 오픈 소스 워크로드 관리자(Workload Manager)이자 작업 스케줄러이다.

### Slurm의 3가지 핵심 역할 ##
* 리소스 할당 (Allocate): 사용자가 요청한 시간 동안 특정 개수의 CPU, GPU, 메모리 등의 자원을 독점적으로 할당해 줍니다.
* 작업 실행 (Run): 할당된 노드에서 실제 계산 작업(예: AI 학습 모델)을 시작하고 실행 상태를 감시합니다.
* 대기열 관리 (Queue): 수많은 사용자가 동시에 작업을 던질 때, 우선순위(Priority)에 따라 줄을 세우고 자원이 나면 순서대로 투입합니다
