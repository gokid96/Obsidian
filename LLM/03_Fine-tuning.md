
### QLoRA 방식이 사실상 표준 
## 왜 QLoRA가 대세냐면

- RTX 3090/4090 (24GB)으로 7B~13B 모델 파인튜닝 가능
- Colab 무료 티어로도 돌아감
- LoRA 대비 성능 차이 거의 없음 (논문상 96~99% 수준)
## 실무에서는

```
QLoRA + SFT → 90%의 유즈케이스 커버
```

RLHF/DPO는 OpenAI, Anthropic 같은 데서 베이스 모델 만들 때나 쓰고, 일반 개발자가 "우리 회사 데이터로 튜닝하자"할 때는 거의 QLoRA야.



# Fine-tuning (파인튜닝)

Pre-training된 모델을 목적에 맞게 조정하는 단계
= Post-training 이라고도 부름

---

## 1. SFT (Supervised Fine-Tuning)

### 개념
사람이 만든 (질문, 답변) 쌍으로 대화 방법을 가르침

### 예시
- Input: "서울의 인구는?"
- Output: "서울의 인구는 약 950만 명입니다."
- 이런 쌍을 수만~수십만 개 학습

### 장점
- 구현이 단순함
- 결과가 예측 가능함

### 한계
- 사람이 작성한 "정답"만 학습 → 창의성 부족
- "뭐가 더 좋은 답인지"는 모름

---

## 2. RLHF (Reinforcement Learning from Human Feedback)

### 개념
사람의 선호도 피드백을 기반으로 강화학습

### 학습 과정

Step 1: 질문 하나에 답변 A, B 생성
Step 2: 사람이 "A가 B보다 낫다" 판단
Step 3: 이 피드백으로 Reward Model 학습
Step 4: Reward Model 점수가 높아지도록 LLM 업데이트

### 특징

| 항목 | 내용 |
|------|------|
| 핵심 알고리즘 | PPO (Proximal Policy Optimization) |
| 사용처 | GPT-4, Claude, Gemini 등 대부분의 상용 모델 |

### 장점
- 주관적 품질도 학습 가능
- 안전성, 유용성 등 복합적 기준 반영
- 가장 검증된 alignment 방법

### 단점
- 구현 복잡
- Reward Model 별도 학습 필요
- 비용 높음

---

## 3. DPO (Direct Preference Optimization)

### 개념
RLHF의 간소화 버전. Reward Model 없이 직접 학습

### 비교

RLHF: 선호도 데이터 → Reward Model → LLM 학습
DPO:  선호도 데이터 → 바로 LLM 학습

### 특징

| 항목 | 내용 |
|------|------|
| 발표 | 2023년 스탠포드 |
| 사용처 | Llama 3, Zephyr 등 오픈소스 모델 |

### 장점
- 구현이 SFT만큼 단순
- 학습 안정적
- 비용 효율적

### 단점
- RLHF만큼 성능 나오는지 논쟁 중

---

## 4. LoRA / QLoRA

### 개념
전체 파라미터가 아닌 일부만 학습하는 효율화 기법

### 비교

| 기법 | VRAM (7B 기준) | 특징 |
|------|----------------|------|
| Full Fine-tuning | ~60GB | 최고 성능, 비용 높음 |
| LoRA | ~16GB | 효율적 |
| QLoRA | ~6GB | 개인 GPU로 가능 |

### 사용 시점
- 개인/소규모 팀이 커스텀 모델 만들 때
- 특정 도메인 적응 (의료, 법률 등)

---

## 5. 선택 기준

| 상황 | 추천 |
|------|------|
| 대화 형식만 가르치고 싶다 | SFT |
| 답변 품질 높이고 싶다 | RLHF 또는 DPO |
| 리소스 충분, 최고 품질 | RLHF |
| 리소스 제한, 간단하게 | DPO |
| 개인 GPU로 파인튜닝 | QLoRA + SFT |

### RLHF vs DPO

| 기준 | RLHF | DPO |
|------|------|-----|
| 구현 난이도 | 높음 | 낮음 |
| 학습 안정성 | 불안정 가능 | 안정적 |
| 성능 | 검증됨 | 대부분 비슷 |
| 비용 | 높음 | 낮음 |

