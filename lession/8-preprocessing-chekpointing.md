대규모 모델 학습(LLM 등)에서 데이터 I/O는 전체 성능의 병목 지점이 될 수 있다. 특히 수천 개의 파드가 동시에 체크포인트를 저장하거나 데이터를 읽을 때 발생하는 I/O Storm을 방지하기 위한 아키텍처 설계가 핵심이다.

### 1. 고성능 병렬 파일 시스템 활용 (FSx for Lustre) ###
* Lustre의 강점: 수백 Gbps 수준의 대역폭을 제공하는 Amazon FSx for Lustre를 학습용 스토리지로 구성한다. 이는 Slurm의 다중 노드 학습 환경에서 각 워커 노드가 데이터에 지연 없이 접근할 수 있도록 돕는다.
* S3 연동: 학습 데이터 원본은 S3에 두고, FSx for Lustre를 통해 필요할 때만 데이터를 캐싱(Lazy Loading)하여 비용과 성능을 동시에 잡는 전략을 다룬다.

### 2. 체크포인팅 가속화 (Checkpointing Optimization) ###
* 분산 저장 전략: 모델 가중치를 저장할 때 모든 노드가 동시에 쓰기 작업을 수행하므로, EBS gp3의 IOPS/Throughput 설정을 최적화하거나 공유 파일 시스템의 스트라이핑(Striping) 설정을 조정하는 방법을 실습한다.
* 비동기 체크포인팅: 학습 프로세스가 멈추지 않고 배경에서 체크포인트를 저장할 수 있도록 하는 소프트웨어 기법과 Slurm 작업 스크립트 최적화를 제안한다.

### 3. 데이터 로딩 효율화 (Data Pre-fetching) ###
* 로컬 캐시 활용: 각 EKS 노드의 로컬 NVMe SSD를 임시 데이터 저장소(Local Path Provisioner 등)로 활용하여, 반복되는 에포크(Epoch) 동안 네트워크 트래픽을 최소화하는 하이브리드 스토리지 구성법을 배운다.
* 데이터 파이프라인 정체 해소: CPU 기반의 데이터 전처리(AMX 활용 등)와 GPU 기반의 학습 루프 사이의 속도 차이를 맞추기 위한 Prefetching 버퍼 설정 노하우를 공유한다.


## 데이터 전처리 ##

### 1. 데이터 수집 ###
```
import os
from datasets import load_dataset

# 1. 데이터셋 로드
print("Downloading dataset...")
dataset = load_dataset("wikitext", "wikitext-2-raw-v1", split="train")

# 2. 저장 경로 설정
raw_dir = "/data/raw"
os.makedirs(raw_dir, exist_ok=True)

# 3. 데이터를 10개의 파일로 쪼개서 저장
num_files = 10
lines_per_file = len(dataset["text"]) // num_files

for i in range(num_files):
    file_path = os.path.join(raw_dir, f"wiki_{i:02d}.txt")
    start_idx = i * lines_per_file
    end_idx = (i + 1) * lines_per_file if i != num_files - 1 else len(dataset["text"])
    
    with open(file_path, "w", encoding="utf-8") as f:
        # 지정된 범위의 데이터만 저장
        for text in dataset["text"][start_idx:end_idx]:
            text = text.strip()
            if text:
                f.write(text + "\n")
    
    print(f"Saved: {file_path}")

print("Done! Now you can run 'sbatch preprocess.sh'")
```
* S3 버킷으로 업로드 한다. 


### 2. 토크나이징(Tokenizing) ###

* lsutre 를 설치하여 S3 버킷으로 부터 데이터를 읽은 후 전처리
* 다시 S3 버킷에 데이터를 저장한다. 

[preprocess.py]
```
import os
import argparse
import webdataset as wds
from transformers import AutoTokenizer
from tqdm import tqdm

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--shard_id", type=int, required=True)
    parser.add_argument("--input_dir", type=str, default="/data/raw")
    parser.add_argument("--output_dir", type=str, default="/data/processed")
    parser.add_argument("--model_id", type=str, default="meta-llama/Meta-Llama-3-8B")
    args = parser.parse_args()

    # 1. 토크나이저 로드
    tokenizer = AutoTokenizer.from_pretrained(args.model_id)
    # Llama 3는 pad_token이 없으므로 필요시 설정
    if tokenizer.pad_token is None:
        tokenizer.pad_token = tokenizer.eos_token

    # 2. 입력 파일 선정 (Shard ID에 매칭)
    input_files = sorted([f for f in os.listdir(args.input_dir) if f.endswith(".txt")])
    target_file = os.path.join(args.input_dir, input_files[args.shard_id])
    
    output_path = os.path.join(args.output_dir, f"shard-{args.shard_id:05d}.tar")
    os.makedirs(args.output_dir, exist_ok=True)

    print(f"Processing {target_file} -> {output_path}")

    # 3. WebDataset 작성 (Lustre 위에서 병렬 쓰기)
    with wds.TarWriter(output_path) as sink:
        with open(target_file, 'r', encoding='utf-8') as f:
            for i, line in enumerate(tqdm(f)):
                line = line.strip()
                if not line: continue

                # 토크나이징 (정수 리스트로 변환)
                tokens = tokenizer.encode(line, add_special_tokens=True)
                
                # 데이터 저장 (.json 또는 .msgpack 형식)
                sink.write({
                    "__key__": f"sample_{args.shard_id}_{i:08d}",
                    "token_ids.json": tokens,
                    "text.txt": line
                })

if __name__ == "__main__":
    main()
```

[preprocess.sh]
```
#!/bin/bash
#SBATCH --job-name=llama3-prep
#SBATCH --partition=amx-part      # CPU 전용 파티션 사용
#SBATCH --array=0-9                
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8         # 8코어씩 할당
#SBATCH --output=logs/prep_%a.out

# FSx for Lustre 스트라이핑 설정 (최초 1회만 실행하면 됨)
# lfs setstripe -c 8 /data/processed

# 작업 수행
# SLURM_ARRAY_TASK_ID가 자동으로 파이썬의 shard_id로 들어감
srun python preprocess.py \
    --shard_id $SLURM_ARRAY_TASK_ID \
    --input_dir /data/raw \
    --output_dir /data/processed \
    --model_id "meta-llama/Meta-Llama-3-8B"
```
* Job Array(--array)
  독립적인 작업의 갯수를 설정하는 것으로 slurm 스케줄링의 대상이다. Array ID($SLURM_ARRAY_TASK_ID)는 서로 완전히 독립적인 프로세스로 하나가 죽어도 다른 번호 작업에 영향을 주지 않습니다. 
* nTasks (--ntasks) 
  각 작업당 사용할 프로세스의 갯수를 설정한다. 보통 전처리의 경우 1 이다.
* CPUs-per-task (--cpus-per-task)   
  Task(프로세스)가 사용할 CPU 코어의 개수를 설정한다. 


```
mkdir -p logs               # logs 디렉토리가 없으면 생성 (스크립트에서 지정한 출력 경로)
sbatch preprocess.sh        # Slurm에 작업 제출
squeue -u $USER             # 현재 내 작업 목록 확인
tail -f logs/prep_0.out     # 실시간 로그 확인 (예: 0번 작업 로그)

scancel <JOB_ID>            # 특정 작업 ID 취소
scancel -u $USER            # 내 모든 작업 취소
```

## 3. 훈련 하기 ##
* lustre 에 저장된 파일을 읽어온다. 

```
import torch
import webdataset as wds
from torch.utils.data import DataLoader
from transformers import DataCollatorForLanguageModeling, AutoTokenizer

def get_llama3_dataloader(input_shards, batch_size, model_id="meta-llama/Meta-Llama-3-8B"):
    tokenizer = AutoTokenizer.from_pretrained(model_id)
    if tokenizer.pad_token is None:
        tokenizer.pad_token = tokenizer.eos_token

    # 1. WebDataset 파이프라인 구성
    dataset = (
        wds.WebDataset(input_shards, shardselection=wds.shardlists.split_by_node) # 노드별 샤드 분산
        .shuffle(1000)                      # 버퍼 셔플
        .decode("json")                     # preprocess.py에서 저장한 token_ids.json 디코딩
        .rename(input_ids="token_ids.json") # 키 이름 변경
        .map(lambda x: {"input_ids": x["input_ids"]}) # 학습에 필요한 필드만 추출
    )

    # 2. Data Collator 설정 (Padding 및 Masking 처리)
    # Llama 3는 Causal LM이므로 mlm=False 설정
    collator = DataCollatorForLanguageModeling(tokenizer=tokenizer, mlm=False)

    # 3. PyTorch DataLoader로 변환
    loader = DataLoader(
        dataset,
        batch_size=batch_size,
        num_workers=4,        # FSx I/O를 병렬로 처리할 워커 수
        collate_fn=collator,
        pin_memory=True
    )
    
    return loader

# 사용 예시 (FSx 경로의 모든 tar 파일 지정)
shard_path = "/data/processed/shard-{00000..00009}.tar"
train_loader = get_llama3_dataloader(shard_path, batch_size=4)

for batch in train_loader:
    # batch['input_ids']: [batch_size, sequence_length]
    # batch['labels']: [batch_size, sequence_length] (Causal LM용 자동 생성)
    print(f"Batch shape: {batch['input_ids'].shape}")
    break
...
```
FSx for Lustre 활용 시 핵심 성능 포인트
* split_by_node & split_by_worker: 멀티 GPU 학습(DDP/DeepSpeed) 시, WebDataset의 분산 처리 기능을 사용해야 각 GPU가 서로 다른 데이터를 읽어 중복 학습을 방지합니다.
* num_workers 최적화: FSx for Lustre는 병렬 읽기에 강하므로 num_workers를 4~8 정도로 설정하여 I/O 대역폭을 최대한 활용하세요. AWS FSx Lustre 성능 가이드에 따르면 병렬 요청이 많을수록 처리량이 증가합니다.
* Local Caching 방지: WebDataset은 메모리에 데이터를 쌓지 않고 바로 GPU로 넘기기 때문에, 수백 GB의 데이터셋도 FSx의 고속 네트워크망(최대 수백 Gbps)을 통해 병목 없이 학습할 수 있습니다.


## 체크 포인팅 ##
