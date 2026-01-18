## 데이터 전처리 ##

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

* Job Array(--array)
  독립적인 작업의 갯수를 설정하는 것으로 slurm 스케줄링의 대상이다. Array ID($SLURM_ARRAY_TASK_ID)는 서로 완전히 독립적인 프로세스로 하나가 죽어도 다른 번호 작업에 영향을 주지 않습니다. 
* nTasks (--ntasks) 
  각 작업당 사용할 프로세스의 갯수를 설정한다. 보통 전처리의 경우 1 이다.
* CPUs-per-task (--cpus-per-task)   
  Task(프로세스)가 사용할 CPU 코어의 개수를 설정한다. 
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
* I/O 효율성: 수백만 개의 텍스트 파일을 읽는 대신, 큰 .tar 파일(WebDataset)로 묶어 저장하므로 나중에 PyTorch WebDataset DataLoader를 쓸 때 GPU 데이터 로딩 병목이 사라집니다.
* FSx 최적화: lfs setstripe 명령을 통해 결과물을 분산 저장하면, 수백 개의 GPU가 동시에 이 데이터를 읽을 때 Amazon FSx for Lustre 성능을 100% 활용합니다.
* Llama 3 전용: Meta에서 제공하는 공식 토크나이저를 사용하므로 학습 시 즉시 사용 가능한 데이터가 생성됩니다.

---
Llama 3와 같은 대규모 모델 학습 시, 전처리된 WebDataset(.tar) 파일을 가장 효율적으로 읽어오는 방법은 webdataset 라이브러리를 사용하는 것입니다. 이 방식은 데이터를 로컬에 다운로드하지 않고 FSx for Lustre에서 직접 스트리밍하므로 메모리 점유율이 매우 낮습니다.
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
```
FSx for Lustre 활용 시 핵심 성능 포인트
* split_by_node & split_by_worker: 멀티 GPU 학습(DDP/DeepSpeed) 시, WebDataset의 분산 처리 기능을 사용해야 각 GPU가 서로 다른 데이터를 읽어 중복 학습을 방지합니다.
* num_workers 최적화: FSx for Lustre는 병렬 읽기에 강하므로 num_workers를 4~8 정도로 설정하여 I/O 대역폭을 최대한 활용하세요. AWS FSx Lustre 성능 가이드에 따르면 병렬 요청이 많을수록 처리량이 증가합니다.
* Local Caching 방지: WebDataset은 메모리에 데이터를 쌓지 않고 바로 GPU로 넘기기 때문에, 수백 GB의 데이터셋도 FSx의 고속 네트워크망(최대 수백 Gbps)을 통해 병목 없이 학습할 수 있습니다.


## 체크 포인팅 ##
