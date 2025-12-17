## QLoRA Fine-tuning 튜토리얼

## 개요

|항목|내용|
|---|---|
|목표|Llama-2-7B를 KorQuAD 데이터로 fine-tuning|
|방식|QLoRA (4bit 양자화 + LoRA)|
|환경|Ubuntu + RTX 5080 (16GB VRAM)|
|학습 시간|4분|
|결과|5/5 정답|

---

## QLoRA란?

```
QLoRA = Quantization + LoRA

┌─────────────────────────────────────┐
│ 베이스 모델 (Llama-2-7B)             │
│ - FP16 → 4bit 양자화 (동결)          │ ← Q
│                                     │
│ LoRA 어댑터 (16.7M params)           │
│ - FP16 유지 (학습)                   │ ← LoRA
└─────────────────────────────────────┘
```

|구분|일반 Fine-tuning|QLoRA|
|---|---|---|
|학습 파라미터|6.7B (100%)|16.7M (0.25%)|
|VRAM|~28GB|~7GB|
|저장 용량|~14GB|~67MB|

---
## QLoRA 과정 및 원리

```
1. Pre-trained 모델 (Llama-2-7B)
   │
   ▼
2. 4bit 양자화 (Q) - 동결
   │  - 14GB → 3.5GB로 압축
   │  - 이 부분은 학습 안 함
   │
   │  [원리]
   │  - FP16 → 4bit로 정밀도 낮춤
   │  - 0.123456789 → 0.12 (미세한 값 표현 불가)
   │  - gradient 업데이트 불가능 → 동결
   │
   ▼
3. LoRA 어댑터 추가 - 학습
   │  - 작은 가중치만 새로 학습
   │  - 0.25% 파라미터만 업데이트
   │
   │  [원리]
   │  - 어댑터는 FP16 유지 → 정밀한 gradient 계산 가능
   │  - 베이스 모델 출력 + LoRA 출력 = 최종 결과
   │  - 작은 어댑터만 학습해도 전체 모델 성능 변화
   │
   ▼
4. Fine-tuned 모델
   │
   │  [결과]
   │  - 베이스 모델: 기존 지식 보존 (동결)
   │  - LoRA 어댑터: 새로운 태스크 학습 (67MB)
   │  - 합쳐서 추론 시 사용
```

---


## 환경 설정
````bash
# 가상환경
python3 -m venv ~/ai-projects/venv
source ~/ai-projects/venv/bin/activate

# 패키지 설치
pip install torch transformers accelerate
pip install bitsandbytes peft trl
pip install datasets==3.6.0

````


---
## 파일 구조
```

~/ai-projects/llama2-korquad/
├── KorQuAD_v1.0_dev.json        # 원본 데이터
├── prepare_tutorial_data.py     # 1. 데이터 준비
├── korquad_tutorial.json        # 2. 학습 데이터
├── train_tutorial.py            # 3. 학습 (핵심)
├── test_tutorial.py             # 4. 테스트
└── llama2-korquad-tutorial/
    └── final/
        └── adapter_model.safetensors  # 67MB
```

---

## 1. 데이터 준비 (prepare_tutorial_data.py)

````python
import json

with open("KorQuAD_v1.0_dev.json", "r") as f:
    data = json.load(f)

refined_dict = {}
for topic in data['data']:
    for para in topic['paragraphs']:
        for qa in para['qas']:
            refined_dict[qa['question']] = qa['answers'][0]['text']

samples = []
for idx, (q, a) in enumerate(refined_dict.items()):
    if idx >= 20:
        break
    prompt = f"Below is an instruction that describes a task. Write a response that appropriately completes the request. ### Instruction: {q} ### Response: {a}"
    samples.append({"text": prompt})

with open("korquad_tutorial.json", "w", encoding="utf-8") as f:
    json.dump(samples, f, ensure_ascii=False, indent=2)

print(f"생성: {len(samples)}개")
```

**출력 형식:**
```
Below is an instruction that describes a task. Write a response that appropriately completes the request. ### Instruction: 임종석이 여의도 농민 폭력 시위를 주도한 혐의로 지명수배 된 날은? ### Response: 1989년 2월 15일
````

---

## 2. 학습 (train_tutorial.py) 핵심

```python
import torch
from datasets import load_dataset
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from trl import SFTTrainer, SFTConfig

MODEL_NAME = "TinyPixel/Llama-2-7B-bf16-sharded"
OUTPUT_DIR = "./llama2-korquad-tutorial"

# 1️⃣ 4bit 양자화 설정 (Q)
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_use_double_quant=True,
)

# 2️⃣ 모델 로드
model = AutoModelForCausalLM.from_pretrained(
    MODEL_NAME, quantization_config=bnb_config, device_map="auto"
)
model.gradient_checkpointing_enable()
model = prepare_model_for_kbit_training(model)

tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
tokenizer.pad_token = tokenizer.eos_token

# 3️⃣ LoRA 설정
lora_config = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    bias="none", task_type="CAUSAL_LM",
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
)
model = get_peft_model(model, lora_config)

# 4️⃣ 데이터 로드
dataset = load_dataset("json", data_files="korquad_tutorial.json", split="train")

# 5️⃣ 학습 설정
sft_config = SFTConfig(
    output_dir=OUTPUT_DIR,
    num_train_epochs=100,
    per_device_train_batch_size=4,
    learning_rate=2e-4,
    fp16=True,
    max_length=256,
    dataset_text_field="text",
    report_to="none",
)

# 6️⃣ 학습
trainer = SFTTrainer(
    model=model, args=sft_config,
    train_dataset=dataset, processing_class=tokenizer,
)
trainer.train()

# 7️⃣ 저장
trainer.save_model(f"{OUTPUT_DIR}/final")
tokenizer.save_pretrained(f"{OUTPUT_DIR}/final")
```

---

## 3. 테스트 (test_tutorial.py)
```python
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import PeftModel
import torch

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.float16,
)

# 베이스 모델 + LoRA 어댑터 로드
base_model = AutoModelForCausalLM.from_pretrained(
    "TinyPixel/Llama-2-7B-bf16-sharded",
    quantization_config=bnb_config,
    device_map="auto",
)
model = PeftModel.from_pretrained(base_model, "./llama2-korquad-tutorial/final")
tokenizer = AutoTokenizer.from_pretrained("TinyPixel/Llama-2-7B-bf16-sharded")

# 추론
prompt_template = "Below is an instruction that describes a task. Write a response that appropriately completes the request. ### Instruction: %s ### Response: "

def gen(question):
    prompt = prompt_template % question
    inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
    outputs = model.generate(**inputs, max_new_tokens=128, do_sample=False)
    return tokenizer.decode(outputs[0], skip_special_tokens=True).replace(prompt, "")

# 테스트
print(gen("임종석이 여의도 농민 폭력 시위를 주도한 혐의로 지명수배 된 날은?"))
# 출력: 1989년 2월 15일
```

---

## 실행

```bash
cd ~/ai-projects/llama2-korquad
source ~/ai-projects/venv/bin/activate

python prepare_tutorial_data.py  # 데이터 준비
python train_tutorial.py         # 학습 (4분)
python test_tutorial.py          # 테스트
```

---

## 학습 결과

|항목|값|
|---|---|
|Epochs|100|
|학습 시간|4분|
|최종 loss|0.086|
|정확도|5/5 (100%)|
|어댑터 크기|67MB|

---

## 핵심 포인트

1. **QLoRA = 4bit 양자화 + LoRA** → 7B 모델을 7GB VRAM으로 학습 가능
2. **프롬프트 일관성** → 학습과 추론 시 동일한 형식 사용 필수
3. **TRL 최신 버전** → `max_seq_length` → `max_length`, `tokenizer` → `processing_class`