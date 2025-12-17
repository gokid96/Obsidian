## QLoRA Fine-tuning 튜토리얼 (Self 버전)

## 개요

|항목|내용|
|---|---|
|목표|Llama-2-7B를 KorQuAD 데이터로 fine-tuning|
|방식|QLoRA (4bit 양자화 + LoRA)|
|특징|직접 양자화 설정 (BitsAndBytesConfig)|
|환경|Ubuntu + RTX 5080 (16GB VRAM)|

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

## 파일 구조
```
~/ai-projects/llama2-korquad/
├── KorQuAD_v1.0_dev.json          # 원본 데이터
├── prepare_tutorial_data.py       # 데이터 준비
├── korquad_tutorial.json          # 학습 데이터 (20개)
├── self_train_qlora.py            # 학습 (Self 버전)
├── self_train_qlora_test.py       # 테스트
└── llama2-korquad-qlora/
    └── final/
        └── adapter_model.safetensors
```

---

## 환경 설정
```bash
python3 -m venv ~/ai-projects/venv
source ~/ai-projects/venv/bin/activate

pip install torch transformers accelerate
pip install bitsandbytes peft trl
pip install datasets
```

---

## 1. 데이터 준비 (prepare_tutorial_data.py)
```python
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

---

## 2. 학습 (self_train_qlora.py)
```python
"""
Llama-2 QLoRA Fine-tuning - Self 버전 (직접 양자화)
"""

import torch
from datasets import load_dataset
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from trl import SFTTrainer, SFTConfig

MODEL_NAME = "TinyPixel/Llama-2-7B-bf16-sharded"
OUTPUT_DIR = "./llama2_7B_slef_tain_qlora"

print("설정")
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_use_double_quant=True,
)

print("모델 로딩")
model = AutoModelForCausalLM.from_pretrained(
    MODEL_NAME, quantization_config=bnb_config, device_map="auto"
)
model.gradient_checkpointing_enable()
model = prepare_model_for_kbit_training(model)

tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
tokenizer.pad_token = tokenizer.eos_token

print("LoRA 설정")
lora_config = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    bias="none", task_type="CAUSAL_LM",
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
)
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()

print("데이터 로딩")
dataset = load_dataset("json", data_files="korquad_train.json", split="train")

sft_config = SFTConfig(
    output_dir=OUTPUT_DIR,
    num_train_epochs=100,  # 튜토리얼과 동일
    per_device_train_batch_size=4,
    gradient_accumulation_steps=1,
    learning_rate=2e-4,
    fp16=True,
    logging_steps=50,
    save_strategy="epoch",
    save_total_limit=2,
    optim="paged_adamw_8bit",
    max_length=256,
    dataset_text_field="text",
    report_to="none",
)

print("학습 시작")
trainer = SFTTrainer(
    model=model, args=sft_config,
    train_dataset=dataset, processing_class=tokenizer,
)
trainer.train()

print("저장")
trainer.save_model(f"{OUTPUT_DIR}/final")
tokenizer.save_pretrained(f"{OUTPUT_DIR}/final")
print("완료")
```

---

## 3. 테스트 (self_train_qlora_test.py)
```python
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import PeftModel
import torch

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.float16,
)

print("모델 로딩")
base_model = AutoModelForCausalLM.from_pretrained(
    "TinyPixel/Llama-2-7B-bf16-sharded",
    quantization_config=bnb_config,
    device_map="auto",
)
model = PeftModel.from_pretrained(base_model, "./llama2_7B_slef_tain_qlora/final")
tokenizer = AutoTokenizer.from_pretrained("TinyPixel/Llama-2-7B-bf16-sharded")

prompt_template = "Below is an instruction that describes a task. Write a response that appropriately completes the request. ### Instruction: %s ### Response: "

def gen(question):
    prompt = prompt_template % question
    inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
    outputs = model.generate(**inputs, max_new_tokens=128, do_sample=False)
    return tokenizer.decode(outputs[0], skip_special_tokens=True).replace(prompt, "")

# 튜토리얼과 동일한 20개 테스트
questions = [
    ("임종석이 여의도 농민 폭력 시위를 주도한 혐의로 지명수배 된 날은?", "1989년 2월 15일"),
    ("1989년 6월 30일 평양축전에 대표로 파견 된 인물은?", "임수경"),
    ("임종석이 여의도 농민 폭력 시위를 주도한 혐의로 지명수배된 연도는?", "1989년"),
    ("임종석을 검거한 장소는 경희대 내 어디인가?", "학생회관 건물 계단"),
    ("임종석이 조사를 받은 뒤 인계된 곳은 어딘가?", "서울지방경찰청 공안분실"),
]

print("\n" + "="*60)
print("튜토리얼 방식 테스트 (5개 샘플)")
print("="*60)

correct = 0
for q, answer in questions:
    result = gen(q)
    match = "✅" if answer in result else "❌"
    if answer in result:
        correct += 1
    print(f"\n질문: {q[:50]}...")
    print(f"정답: {answer}")
    print(f"생성: {result[:50]}")
    print(f"일치: {match}")

print(f"\n정확도: {correct}/{len(questions)}")
```

---

## 실행
```bash
cd ~/ai-projects/llama2-korquad
source ~/ai-projects/venv/bin/activate

python prepare_tutorial_data.py   # 데이터 준비
python self_train_qlora.py        # 학습
python self_train_qlora_test.py   # 테스트
```

---

## 핵심 포인트

1. **직접 양자화** — BitsAndBytesConfig로 4bit 양자화 설정
2. **prepare_model_for_kbit_training()** — 양자화 모델 학습 준비
3. **LoraConfig** — 수동으로 LoRA 어댑터 설정
4. **원리 이해에 적합** — 각 단계가 명시적으로 분리됨