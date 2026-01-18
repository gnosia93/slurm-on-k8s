## 파이프라인 구성 전략: Slurm Job Chaining ##
대규모 실험에서는 이전 단계가 성공적으로 끝나야 다음 단계가 시작되어야 한다. Slurm Slinky를 통해 쿠버네티스 자원을 점유하면서도 고전적인 HPC의 정교한 파이프라인 제어 방식을 적용한다.

#### 1. 작업 의존성(Dependency) 관리 ####
* Sequential Workflow: 데이터 전처리(CPU/AMX) → 모델 학습(GPU) → 체크포인트 검증 및 평가 순으로 이어지는 단계를 구성
* 의존성 옵션 활용: Slurm의 --dependency 옵션을 사용하여 앞선 작업이 정상 종료(afterok)되었을 때만 다음 작업이 큐에서 활성화되도록 설정.
```
sbatch --dependency=afterok:<JobID> train.sh
```

작업 제출 후 squeue 명령어를 입력하면, 의존성이 걸린 뒤쪽 작업들은 (Dependency)라는 상태 메시지와 함께 대기(PD) 상태로 표시되는 것을 확인할 수 있다.
```
#!/bin/bash

# 1. 데이터 전처리 단계 (AMX 파티션 활용)
# --parsable 옵션은 Job ID만 깔끔하게 반환하게 한다.
PRE_JOB_ID=$(sbatch --parsable --partition=amx-partition preprocess.sh)
echo "Step 1: Preprocessing submitted (Job ID: ${PRE_JOB_ID})"

# 2. 모델 학습 단계 (GPU 파티션 활용)
# --dependency=afterok:<JobID> 는 앞선 작업이 에러 없이 끝나야 실행됨을 의미한다.
TRAIN_JOB_ID=$(sbatch --parsable --partition=gpu-partition --dependency=afterok:${PRE_JOB_ID} train.sh)
echo "Step 2: Training submitted (Job ID: ${TRAIN_JOB_ID})"

# 3. 결과 정리 및 평가 단계
FINAL_JOB_ID=$(sbatch --parsable --partition=amx-partition --dependency=afterok:${TRAIN_JOB_ID} postprocess.sh)
echo "Step 3: Post-processing submitted (Job ID: ${FINAL_JOB_ID})"

echo "------------------------------------------------"
echo "Full pipeline has been queued successfully."
echo "Use 'squeue' to monitor the dependency chain."
```
* preprocess.sh: AMX 파티션에서 데이터 전처리 수행
* train.sh: GPU 파티션에서 모델 학습 수행
* postprocess.sh: 결과 데이터 정리 및 리포트 생성

파이프라인의 마지막 단계인 결과 정리(Post-processing)는 학습이 끝난 후 생성된 방대한 데이터를 실제 서비스에 사용 가능한 형태로 가공하거나, 실험의 성패를 기록하는 중요한 과정이다.
* 모델 체크포인트 변환 및 저장: 훈련 도중 생성된 수많은 임시 체크포인트 중 가장 성능이 좋은 것을 골라 Hugging Face 형식이나 배포용 형식(ONNX, TensorRT 등)으로 직렬화(Serialization)하여 최종 모델 저장소(S3 등)에 업로드.
* 평가 메트릭 산출: 테스트 데이터셋을 사용하여 모델의 정확도, 손실값, 추론 속도 등을 계산하고 이를 실험 관리 도구(예: Weights & Biases, MLflow)에 기록하여 시각화.
* 데이터 클렌징: 훈련 과정에서 일시적으로 사용했던 Amazon FSx for Lustre 내의 대용량 중간 데이터나 임시 캐시 파일을 삭제하여 클라우드 비용을 절감.
* 리포트 생성 및 알림: 실험의 주요 결과와 지표를 요약한 리포트를 생성하고, 팀원들에게 Slack이나 이메일로 성공/실패 알림.


#### 2. 이기종 자원 파이프라인 최적화 ####
* 파티션 스위칭: 전처리 단계는 AMX 전용 파티션(amx-partition)에서 수행하고, 학습 단계는 GPU 전용 파티션(gpu-partition)으로 자동 전달되는 파이프라인을 구축하여 고가의 GPU 자원 대기 시간을 최소화.
* 자원 선점 및 해제: 작업이 종료되는 즉시 쿠버네티스 파드가 반납되도록 구성하여 클라우드 비용을 최적화

#### 3. 자동화된 재시작 및 에러 핸들링 ####
* 에러 트래핑: 파이프라인 중간에 작업이 실패할 경우, 관리자에게 알림을 보내거나 특정 체크포인트 지점부터 자동으로 작업을 재시작하는 Self-healing 파이프라인 구성.
* 로그 통합 관리: 각 단계에서 생성되는 로그를 Amazon CloudWatch 또는 중앙 집중형 로그 시스템(Lokii) 으로 수집하여 전체 파이프라인의 상태를 한눈에 모니터링.

#### 4. CI/CD 및 Argo Workflows 연동 ####
코드 변경 시 Argo Workflows가 트리거되어 Slurm에 작업을 제출하고, 완료 후 결과를 대시보드에 반영하는 최신 MLOps 아키텍처 구현.
