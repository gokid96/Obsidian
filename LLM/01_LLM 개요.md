
# LLM (Large Language Model)

## LLM이란?
대규모 텍스트 데이터를 학습한 언어 모델
예: GPT-4, Claude, Llama, Gemini

## 학습의 큰 그림
```
┌──────────────────────────────────────┐
│           딥러닝 (Transformer)        │
│                                      │
│   1. Pre-training (사전학습)          │
│          ↓                           │
│   2. Fine-tuning (파인튜닝)           │
│       ├── SFT                        │
│       ├── RLHF                       │
│       └── DPO 등                     │
│                                      │
└──────────────────────────────────────┘
```

## 각 단계 요약

| 단계 | 하는 일 | 비유 |
|------|---------|------|
| Pre-training | 언어 패턴 학습 | 책 수만 권 읽기 |
| SFT | 대화 방법 학습 | 선생님이 모범답안 보여주기 |
| RLHF/DPO | 더 좋은 답변 학습 | "A가 B보다 낫다" 피드백 |

## 용어 정리

| 용어 | 설명 |
|------|------|
| 딥러닝 | LLM 전체를 관통하는 기반 기술 |
| Transformer | 현재 모든 LLM이 사용하는 아키텍처 (2017년 구글 발표) |
| Pre-training | 대규모 데이터로 언어 자체를 학습 |
| Fine-tuning | 목적에 맞게 조정 (= Post-training) |
| Alignment | 모델이 인간 의도대로 동작하게 만드는 것 |
| Base Model | Pre-training만 된 모델 (대화 못함) |

