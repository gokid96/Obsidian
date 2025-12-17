## QLoRA Fine-tuning 튜토리얼 (Unsloth 버전)

## 개요

|항목|내용|
|---|---|
|목표|Llama-3.1-8B를 KorQuAD 데이터로 fine-tuning|
|방식|QLoRA (Unsloth 사전 양자화)|
|특징|2배 빠름, 70% VRAM 절약, 코드 간결|
|환경|Ubuntu + RTX 5080 (16GB VRAM)|

---

## Self vs Unsloth 비교

|항목|Self 버전|Unsloth 버전|
|---|---|---|
|모델|Llama-2-7B|Llama-3.1-8B|
|양자화|직접 설정 (BitsAndBytesConfig)|사전 양자화 모델 사용|
|속도|1x|2x|
|VRAM|~7GB|~5GB|
|코드량|많음|간결|

---

## 파일 구조
```
~/ai-projects/llama2-korquad/
├── KorQuAD_v1.0_dev.json          # 원본 데이터
├── prepare_tutorial_data.py       # 데이터 준비 (공용)
├── korquad_tutorial.json          # 학습 데이터 (20개)
├── unsloth_train_qlora.py         # 학습 (Unsloth 버전)
├── unsloth_train_qlora_test.py    # 테스트
└── llama3-korquad-qlora/
    └── final/
        └── adapter_model.safetensors
```

---

## 환경 설정
```bash
source ~/ai-projects/venv/bin/activate

# Unsloth 설치
pip install unsloth
```

---

## 1. 학습 (unsloth_train_qlora.py)
```python
"""
Llama-3.1 QLoRA Fine-tuning with Unsloth
"""
from unsloth import FastLanguageModel
from datasets import load_dataset
from trl import SFTTrainer, SFTConfig

# ============================================
# 설정
# ============================================
MODEL_NAME = "unsloth/Meta-Llama-3.1-8B-bnb-4bit"
OUTPUT_DIR = "./llama3_8B_4bit_unsloth_train_qlora"

# ============================================
# 모델 + 토크나이저 로드 (Unsloth)
# ============================================
print("모델 로딩")
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name=MODEL_NAME,
    max_length=2048,
    load_in_4bit=True,
)

# ============================================
# LoRA 설정 (Unsloth)
# ============================================
print("LoRA 설정")
model = FastLanguageModel.get_peft_model(
    model,
    r=16,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj", 
                    "gate_proj", "up_proj", "down_proj"],
    lora_alpha=16,
    lora_dropout=0,
    bias="none",
    use_gradient_checkpointing="unsloth",
)

print("학습 파라미터:")
model.print_trainable_parameters()

# ============================================
# 데이터셋 로드
# ============================================
print("데이터셋 로딩")
dataset = load_dataset("json", data_files="korquad_train.json", split="train")
print(f"  데이터 수: {len(dataset)}")

# ============================================
# SFTConfig
# ============================================
sft_config = SFTConfig(
    output_dir=OUTPUT_DIR,
    num_train_epochs=100,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    learning_rate=2e-4,
    bf16=True,
    logging_steps=10,
    save_strategy="epoch",
    optim="adamw_8bit",
    warmup_ratio=0.03,
    lr_scheduler_type="cosine",
    report_to="none",
    max_length=2048,
    dataset_text_field="text",
)

# ============================================
# SFTTrainer
# ============================================
print("학습 시작")
trainer = SFTTrainer(
    model=model,
    args=sft_config,
    train_dataset=dataset,
    processing_class=tokenizer,
)

trainer.train()

# ============================================
# 저장
# ============================================
print("모델 저장")
trainer.save_model(f"{OUTPUT_DIR}/final")
tokenizer.save_pretrained(f"{OUTPUT_DIR}/final")
print(f"완료 저장 위치: {OUTPUT_DIR}/final")
```

---

## 2. 테스트 (unsloth_train_qlora_test.py)
```python
from unsloth import FastLanguageModel

print("모델 로딩")
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="./llama3_8B_4bit_unsloth_train_qlora/final",
    max_length=2048,
    load_in_4bit=True,
)
FastLanguageModel.for_inference(model)

prompt_template = "Below is an instruction that describes a task. Write a response that appropriately completes the request. ### Instruction: %s ### Response: "

def gen(question):
    prompt = prompt_template % question
    inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
    outputs = model.generate(**inputs, max_new_tokens=128, do_sample=False)
    return tokenizer.decode(outputs[0], skip_special_tokens=True).replace(prompt, "")

questions = [
    ("임종석이 여의도 농민 폭력 시위를 주도한 혐의로 지명수배 된 날은?", "1989년 2월 15일"),
    ("1989년 6월 30일 평양축전에 대표로 파견 된 인물은?", "임수경"),
    ("임종석이 여의도 농민 폭력 시위를 주도한 혐의로 지명수배된 연도는?", "1989년"),
    ("임종석을 검거한 장소는 경희대 내 어디인가?", "학생회관 건물 계단"),
    ("임종석이 조사를 받은 뒤 인계된 곳은 어딘가?", "서울지방경찰청 공안분실"),
]

print("\n" + "="*60)
print("테스트 (5개 샘플)")
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

python prepare_tutorial_data.py      # 데이터 준비 (이미 했으면 생략)
python unsloth_train_qlora.py        # 학습
python unsloth_train_qlora_test.py   # 테스트


watch -n 1 nvidia-smi # 
```

---

## Unsloth가 자동으로 해주는 것

|항목|Self 버전|Unsloth 버전|
|---|---|---|
|BitsAndBytesConfig|직접 설정|내부 처리|
|prepare_model_for_kbit_training()|직접 호출|내부 처리|
|gradient_checkpointing_enable()|직접 호출|`use_gradient_checkpointing="unsloth"`|
|LoraConfig|직접 설정|`FastLanguageModel.get_peft_model()`|

---

## 핵심 포인트

1. **사전 양자화 모델** — `unsloth/xxx-bnb-4bit` 다운받으면 바로 사용
2. **FastLanguageModel** — 모델 + 토크나이저 + LoRA 한 번에 처리
3. **for_inference()** — 추론 시 2배 속도 향상
4. **2025년 표준** — 실무에서 가장 많이 사용되는 방식