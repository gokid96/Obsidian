## 배경
- Full Fine-tuning: 모든 파라미터 학습 → 비용/시간 큼
- 7B 모델 전체 학습하려면 VRAM 60GB+ 필요
- 해결책: 일부 파라미터만 학습

## PEFT 기법들

| 기법 | 원리 | 특징 |
|------|------|------|
| LoRA | 작은 행렬 추가해서 학습 | 실무 표준 |
| Prefix-Tuning | 입력 앞에 학습 가능한 벡터 추가 | 생성 작업에 적합 |
| P-Tuning | 프롬프트를 학습 가능한 임베딩으로 | GPT 계열에 효과적 |
| Prompt Tuning | soft prompt만 학습 | 가장 파라미터 적음 |

## 왜 LoRA가 표준이 됐나
- 성능이 Full Fine-tuning에 가장 근접
- 구현이 간단
- 다양한 모델에 적용 가능
- Hugging Face에서 잘 지원 (peft 라이브러리)

## 핵심 라이브러리
```python
from peft import LoraConfig, get_peft_model
from transformers import AutoModelForCausalLM
```