## 핵심 아이디어
기존 가중치(W)는 freeze하고, 작은 행렬(A, B)만 추가로 학습

## 수식
```
W' = W + BA

W: 원본 가중치 (freeze)
B: d × r 행렬
A: r × k 행렬
r: rank (보통 8~64)
```

**원리 (수식 몰라도 됨):**
```
원래: 70억 개 파라미터 전부 학습 → GPU 터짐
LoRA: 작은 파라미터(0.1%)만 학습 → 개인 GPU로 가능
```

- 기존 가중치(W)는 건드리지 않고 freeze
- 작은 행렬 A, B 두 개만 추가해서 학습
- 나중에 W + (A×B)로 합침

**비유하자면:**
```
Full Fine-tuning = 집 전체 리모델링
LoRA = 벽지랑 조명만 바꾸기
```

→ 결과는 비슷한데 비용은 훨씬 적음

## 실무에서 알아야 할 것

```python
LoraConfig(
    r=16,           # 작을수록 가벼움, 클수록 성능↑
    lora_alpha=32,  # 보통 r의 2배
    target_modules=["q_proj", "v_proj"],  # 어디에 적용할지
)
```



## 왜 작동하는가
- 파인튜닝 시 가중치 변화량(ΔW)은 low-rank 특성을 가짐
- 즉, 적은 파라미터로도 충분히 표현 가능

## VRAM 비교 (7B 모델 기준)

| 방식 | VRAM |
|------|------|
| Full Fine-tuning | ~60GB |
| LoRA | ~16GB |
| QLoRA | ~6GB |

## 주요 하이퍼파라미터

| 파라미터 | 설명 | 추천값 |
|----------|------|--------|
| r (rank) | 행렬 크기 | 8~64 |
| lora_alpha | 스케일링 팩터 | r의 2배 |
| lora_dropout | 드롭아웃 | 0.05~0.1 |
| target_modules | 어떤 레이어에 적용할지 | q_proj, v_proj |

## 코드 예시
```python
from peft import LoraConfig, get_peft_model

config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)

model = get_peft_model(base_model, config)
```

## 장점
- 원본 모델 보존 (adapter만 저장하면 됨)
- 여러 adapter 쉽게 교체 가능
- 학습 빠름