# mental_gdrl

정신건강 챗봇 가드레일: Jev 13축 신호 감지 → 프롬프트 뱅크 라우팅 (위험 · 우울 · 일반) → 유저 시뮬레이터로 평가.
2026-10-23 챗봇 대응 전략 검토 이후 작업을 [sdx_guardrail](https://github.com/sung00819/sdx_guardrail) 의 `chatbot/rlver` 에서 옮겨 왔다.

## 구성

| 경로 | 내용 |
|---|---|
| `chatbot/rlver/1025)jev.md` | Jev 13축 질문 정본 (위험 9 + 우울 + 추가 요인 3: 급성 맥락 · 정신과 병력 · 물질 사용) |
| `chatbot/rlver/1025)bank.json` | 라우팅용 프롬프트 뱅크 (meta · 정서 지원 전략 · general · crisis 12개) |
| `chatbot/rlver/bank.py` | `route` · `assemble_route` (턴마다 Jev 결과로 프롬프트 조합), 계획 칸 `plan_block` |
| `chatbot/rlver/episode.py` · `run.py` | 대화 실행 (`--template bank_jev` / `bank_jev_plan`) |
| `chatbot/rlver/simulator.py` · `prompts.py` · `compose.py` | 유저 시뮬레이터 (RLVER 기반, 배경 × 원하는 방식 3 × vanilla/challenging) |
| `chatbot/rlver/*_DRAFT.md` | 위기 항목 · 원하는 방식 · 사용자 말하기 방식 대응 초안 (전문가 검수 전) |
| `data/rlver/backgrounds_v2.csv` | 합성 사용자 배경 318개 (위험 198 · 우울 60 · 일반 60, 정답 위험 축 포함) |
| `results/1025)bank_jev/` | 대화 실험 결과 요약과 대화 전문 (원자료 jsonl 은 크기 때문에 뺐다) |
| `common/jev_client.py` | Jev · Kev 판정기 호출 |

자세한 실행 방법은 `chatbot/rlver/README.md`, 결과는 `results/1025)bank_jev/README.md`.

## 실행에 필요한 것

- 시뮬레이터 · 챗봇: vLLM 으로 띄운 Qwen3-30B-A3B-Instruct-2507-FP8 · Qwen3-8B (`chatbot/rlver/1025)config_84.json` 의 주소)
- 판정기: Jev (OpenRouter, `.env` 의 `OPENROUTER_API_KEY`). 키는 저장소에 넣지 않는다.

배경과 대화는 모두 LLM 으로 만든 합성 데이터다. 실제 사용자 정보는 없다.
