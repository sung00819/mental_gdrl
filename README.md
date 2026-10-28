# mental_gdrl

정신건강 챗봇 가드레일: Jev 13축 신호 감지 → 프롬프트 뱅크 라우팅 (위험 · 우울 · 일반) → 유저 시뮬레이터로 평가.
2026-10-23 챗봇 대응 전략 검토 이후 작업을 [sdx_guardrail](https://github.com/sung00819/sdx_guardrail) 의 `chatbot/rlver` 에서 옮겨 왔다.

## 구성

| 경로 | 내용 |
|---|---|
| `chatbot/rlver/guard.py` | **가드레일 단독 실행**: 대화 기록 → Jev 판정 → 프롬프트 조합 → 챗봇 답 (시뮬레이터 없이) |
| `chatbot/rlver/serve_vllm.sh` | 챗봇 · 시뮬레이터 vLLM 서버 띄우기 |
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

## 가드레일 실행

```bash
bash chatbot/rlver/serve_vllm.sh chatbot                 # Qwen3-8B 챗봇 서버 (포트 18602)
export OPENROUTER_API_KEY=...                            # Jev 판정기
python chatbot/rlver/guard.py --chatbot qwen3-8b --show  # 터미널에서 대화, 켜진 축 · 고른 모듈도 보인다
```

코드에서 쓸 때:

```python
from guard import Guardrail
g = Guardrail(config="config_local.json", chatbot="qwen3-8b")
state = g.new_state()                       # 대화마다 하나
out = g.respond(history, state)             # history = [{"role": "user"|"assistant", "content": ...}, ...]
out["reply"], out["jev"], out["route"]      # 보낼 답, 축별 P(예), 고른 모듈 (위험 · 우울 · 일반)
```

처리 순서는 발표 파이프라인과 같다: 최근 사용자 발화 3개로 Jev 13축 판정 (P ≥ 0.7) → 위험 축은 대화 끝까지 유지 →
meta + 정서 지원 / 추가 요인 / 일반 / 위기 지침 순으로 조합 → 위험 턴은 계획 칸을 채워 답 생성 → 앞 답과 거의 같으면 다시 쓰기.

## 실행에 필요한 것

- 챗봇 (· 시뮬레이터): vLLM 으로 띄운 Qwen3-8B · Qwen3-30B-A3B-Instruct-2507-FP8 (`serve_vllm.sh`, 주소는 `config_local.json`. 여러 서버로 돌린 1025 실험 설정은 `1025)config_84.json`)
- 판정기: Jev (OpenRouter `typesafe/jev-1.13`, `OPENROUTER_API_KEY`). 키는 저장소에 넣지 않는다.
- 시뮬레이터로 평가할 때: `python chatbot/rlver/run.py --config "chatbot/rlver/1025)config_84.json" --chatbot qwen3-8b --template bank_jev_plan --judge jev --backgrounds data/rlver/backgrounds_v2.csv --out out/rlver/<이름>`

배경과 대화는 모두 LLM 으로 만든 합성 데이터다. 실제 사용자 정보는 없다.
