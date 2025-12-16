## 정의
LoRA + 4bit 양자화

## 핵심 기술
1. **4-bit NormalFloat (NF4)**: 가중치를 4bit로 양자화
2. **Double Quantization**: 양자화 상수도 양자화
3. **Paged Optimizers**: GPU 메모리 스파이크 방지

## VRAM 비교

| 모델 | Full | LoRA | QLoRA |
|------|------|------|-------|
| 7B | 60GB | 16GB | 6GB |
| 13B | 120GB | 32GB | 10GB |
| 70B | 600GB | 160GB | 48GB |

## 성능
- Full Fine-tuning의 96~99% 성능
- 대부분의 작업에서 차이 느끼기 어려움

## 필요 라이브러리
```bash
pip install bitsandbytes peft transformers
```

## 코드 예시
```python
from transformers import AutoModelForCausalLM, BitsAndBytesConfig

# 4bit 양자화 설정
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_use_double_quant=True,
)

# 모델 로드
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=bnb_config,
    device_map="auto",
)

# LoRA 적용
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05,
)

model = get_peft_model(model, lora_config)
```

## 실무 팁
- Colab 무료 T4로 7B 모델 학습 가능
- RTX 3090/4090이면 13B도 가능
- 배치 사이즈 작게, gradient accumulation 활용