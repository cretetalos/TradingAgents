# TradingAgents — 프로그램 설명 문서

> 멀티 에이전트 LLM 기반 금융 트레이딩 리서치 프레임워크
> (버전 0.3.1 기준 · 문서 작성일 2026-07-08)

---

## 1. 이 프로그램은 무엇인가

**TradingAgents**는 실제 트레이딩 회사의 조직 구조와 의사결정 흐름을 모방한 **멀티 에이전트 트레이딩 리서치 프레임워크**입니다. 여러 개의 LLM 기반 전문 에이전트(펀더멘털·감성·뉴스·기술 분석가, 강세/약세 리서처, 트레이더, 리스크 관리팀, 포트폴리오 매니저)가 시장 데이터를 수집·분석하고, **구조화된 토론**을 거쳐 최종 매매 신호(BUY / SELL / HOLD 등)를 도출합니다.

- **성격**: 연구(research) 목적 프레임워크. 실거래 자동매매 봇이 아니며 투자 자문도 아닙니다.
- **인터페이스**: 대화형 CLI(`tradingagents`) + 파이썬 패키지(`tradingagents`) 두 가지.
- **핵심 엔진**: [LangGraph](https://github.com/langchain-ai/langgraph) 기반 상태 그래프(state graph)로 에이전트 실행 순서·토론·라우팅을 오케스트레이션.

---

## 2. 아키텍처 — 에이전트 파이프라인

의사결정은 복잡한 트레이딩 과제를 **역할별 전문 에이전트**로 분해하여 단계적으로 진행됩니다.

```
                          ┌─────────────────────────────────────┐
   시장/뉴스/감성 데이터  │            분석가 팀 (Analysts)         │
   ───────────────────▶  │  펀더멘털 · 감성 · 뉴스 · 기술 분석가   │
                          └───────────────────┬─────────────────┘
                                              │ 분석 리포트
                                              ▼
                          ┌─────────────────────────────────────┐
                          │        리서처 팀 (Researchers)        │
                          │   강세(Bull) ↔ 약세(Bear) 구조적 토론  │
                          │        → 리서치 매니저가 종합           │
                          └───────────────────┬─────────────────┘
                                              ▼
                          ┌─────────────────────────────────────┐
                          │           트레이더 (Trader)           │
                          │   리포트를 종합해 매매 제안 작성        │
                          └───────────────────┬─────────────────┘
                                              ▼
                          ┌─────────────────────────────────────┐
                          │       리스크 관리팀 (Risk Mgmt)        │
                          │  공격적 · 중립 · 보수적 관점의 토론      │
                          └───────────────────┬─────────────────┘
                                              ▼
                          ┌─────────────────────────────────────┐
                          │     포트폴리오 매니저 (Portfolio Mgr)  │
                          │       최종 승인/거부 → 매매 신호        │
                          └─────────────────────────────────────┘
```

### 2.1 분석가 팀 (`tradingagents/agents/analysts/`)
| 에이전트 | 역할 |
|----------|------|
| 펀더멘털 분석가 (`fundamentals_analyst`) | 재무제표·실적 지표를 평가해 내재가치와 위험 신호 식별 |
| 감성 분석가 (`sentiment_analyst`, `social_media_analyst`) | 뉴스 헤드라인·StockTwits·Reddit 여론을 단일 감성 지표로 집계 |
| 뉴스 분석가 (`news_analyst`) | 글로벌 뉴스·거시 지표를 모니터링해 이벤트의 시장 영향 해석 |
| 기술 분석가 (`market_analyst`) | MACD·RSI 등 기술 지표로 패턴·추세 분석 |

### 2.2 리서처 팀 (`tradingagents/agents/researchers/`)
- **강세(Bull) 리서처 ↔ 약세(Bear) 리서처**가 분석 결과를 놓고 구조적 토론을 벌여 잠재 수익과 리스크를 저울질.
- **리서치 매니저(`managers/research_manager`)**가 토론을 종합.

### 2.3 트레이더 (`tradingagents/agents/trader/`)
- 분석가·리서처 리포트를 종합해 매매 타이밍과 규모가 담긴 제안 작성.

### 2.4 리스크 관리팀 (`tradingagents/agents/risk_mgmt/`)
- **공격적·중립·보수적** 세 관점의 토론자가 변동성·유동성 등 리스크를 다각도로 평가.

### 2.5 포트폴리오 매니저 (`tradingagents/agents/managers/portfolio_manager`)
- 매매 제안을 최종 승인/거부하고, 승인 시 (시뮬레이션) 실행 신호 생성.

### 2.6 오케스트레이션 (`tradingagents/graph/`)
| 파일 | 역할 |
|------|------|
| `trading_graph.py` | 전체 그래프 구성의 진입점 |
| `setup.py` / `conditional_logic.py` | 노드/엣지 구성 및 토론 라운드 라우팅 |
| `analyst_execution.py` | 분석가 도구 실행(ToolNode) 처리 |
| `signal_processing.py` | 최종 신호 파싱·정규화 |
| `reflection.py` | 결정에 대한 사후 회고 |
| `checkpointer.py` | LangGraph 체크포인트 저장/재개 |

---

## 3. 데이터 소스 & LLM 프로바이더

### 3.1 시장/데이터 벤더 (`tradingagents/dataflows/`)
| 카테고리 | 기본 벤더 | 비고 |
|----------|-----------|------|
| 주가·기술지표·펀더멘털·뉴스 | **yfinance** | 무료, 키 불필요 (또는 Alpha Vantage 선택) |
| 거시 지표 | **FRED** | `FRED_API_KEY` 필요(무료) |
| 예측 시장 | **Polymarket** | 공개 Gamma API, 키 불필요 |
| 감성/여론 | **Reddit, StockTwits** | 공개 엔드포인트, 키 불필요 |

벤더는 `default_config.py`의 `data_vendors` / `tool_vendors`에서 카테고리별·도구별로 교체 가능.

### 3.2 지원 LLM 프로바이더 (`tradingagents/llm_clients/`)
OpenAI(GPT), Anthropic(Claude), Google(Gemini), xAI(Grok), DeepSeek, Qwen, GLM, MiniMax, Kimi, Mistral, Groq, NVIDIA NIM, OpenRouter, **AWS Bedrock**, **Azure OpenAI**, 그리고 **모든 OpenAI 호환 서버**(vLLM·LM Studio·llama.cpp·Ollama 등 로컬 포함).

- 프로바이더는 `llm_provider` 설정으로 선택하고, API 키는 해당 프로바이더 전용 환경변수(`OPENAI_API_KEY`, `ANTHROPIC_API_KEY` 등)로 주입.
- "빠른 사고(quick_think)"와 "깊은 사고(deep_think)" 모델을 분리 지정 가능(`quick_think_llm`, `deep_think_llm`).

---

## 4. 설정 (`tradingagents/default_config.py`)

주요 설정 키와 대응 환경변수(`TRADINGAGENTS_*`):

| 항목 | 키 | 기본값 |
|------|-----|--------|
| LLM 프로바이더 | `llm_provider` | `openai` |
| 깊은 사고 모델 | `deep_think_llm` | `gpt-5.5` |
| 빠른 사고 모델 | `quick_think_llm` | `gpt-5.4-mini` |
| 토론 라운드 수 | `max_debate_rounds` | `1` |
| 리스크 토론 라운드 | `max_risk_discuss_rounds` | `1` |
| 출력 언어 | `output_language` | `English` |
| 체크포인트 재개 | `checkpoint_enabled` | `False` |
| 결과 저장 경로 | `results_dir` | `~/.tradingagents/logs` |
| 캐시 경로 | `data_cache_dir` | `~/.tradingagents/cache` |

거의 모든 설정은 CLI 인자, `TRADINGAGENTS_*` 환경변수, `.env` 파일로 오버라이드할 수 있습니다.

---

## 5. 설치 및 실행

```bash
# 설치
git clone https://github.com/TauricResearch/TradingAgents.git
cd TradingAgents
pip install .

# API 키 설정 (택1: 환경변수 또는 .env)
cp .env.example .env      # 편집해서 키 입력
export OPENAI_API_KEY=...  # 사용할 프로바이더 키

# 대화형 CLI 실행
tradingagents
```

Docker로도 실행 가능:
```bash
cp .env.example .env
docker compose run --rm tradingagents
# 로컬 모델(Ollama) 프로파일
docker compose --profile ollama run --rm tradingagents-ollama
```

---

## 6. 데이터 취급 및 보안 특성

> 이 절은 코드 정적 분석 + mitmproxy 런타임 트래픽 캡처 검증(2026-07-08) 결과를 반영합니다.

- **아웃바운드 전송 경로**는 두 종류뿐입니다:
  1. **사용자가 명시적으로 선택한 LLM 프로바이더**로 분석 프롬프트 송신 (정상 동작).
  2. **금융/여론 데이터 API**로부터의 조회(GET) — 티커 등 조회 대상만 URL에 포함.
- **API 키**는 로컬 `.env` 파일과 프로세스 환경변수에만 저장되며, 해당 프로바이더 인증에만 사용됩니다. 제3자로 전송되지 않습니다.
- `api.tauric.ai/v1/announcements`로의 호출은 **페이로드 없는 순수 GET(공지 수신 전용)**이며 사용자 데이터·키·헤더를 싣지 않습니다. (런타임 캡처로 요청 본문 0바이트, 인증 헤더 없음 확인)
- 텔레메트리·백도어·비밀정보 유출 코드는 발견되지 않았습니다. `subprocess`/`eval`/`exec`/`pickle.load` 등 코드 실행 위험도 없습니다.
- `.gitignore`가 `.env`·`.envrc`·`.env.enterprise`를 제외하여 비밀정보가 저장소에 커밋되지 않습니다.

> ⚠️ 주의: 분석 프롬프트에는 사용자가 입력한 종목·전략 정보가 포함되어 **선택한 LLM 프로바이더로 전송**됩니다. 이는 도구의 정상 동작이므로, 민감 정보를 프롬프트에 넣지 않도록 유의하세요.

---

## 7. 면책 (Disclaimer)

TradingAgents는 **연구 목적**으로 설계되었습니다. 트레이딩 성능은 백본 LLM, 모델 temperature, 거래 기간, 데이터 품질 등 여러 비결정적 요인에 따라 달라집니다. **금융·투자·트레이딩 자문이 아닙니다.** ([공식 면책 고지](https://tauric.ai/disclaimer/))

---

## 8. 참고 자료
- 프로젝트: <https://github.com/TauricResearch/TradingAgents>
- 논문: TradingAgents (arXiv:2412.20138), Trading-R1 Technical Report (arXiv:2509.11420)
- 변경 이력: [`CHANGELOG.md`](../CHANGELOG.md)
