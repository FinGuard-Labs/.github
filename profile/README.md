<p align="center">
  <img src="https://raw.githubusercontent.com/FinGuard-Labs/.github/main/profile/logo.png" width="140" alt="FinGuard AI 로고" />
</p>

<h1 align="center">FinGuard AI</h1>

<p align="center">
  망분리 환경을 가정한 로컬 AI 기반 주식 포트폴리오 분석 및 리밸런싱 플랫폼
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Java_17-007396?style=flat-square&logo=openjdk&logoColor=white" alt="Java 17" />
  <img src="https://img.shields.io/badge/Spring_Boot_3.5-6DB33F?style=flat-square&logo=springboot&logoColor=white" alt="Spring Boot 3.5" />
  <img src="https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white" alt="FastAPI" />
  <img src="https://img.shields.io/badge/Vue_3-4FC08D?style=flat-square&logo=vuedotjs&logoColor=white" alt="Vue 3" />
  <img src="https://img.shields.io/badge/PostgreSQL_16_+_pgvector-4169E1?style=flat-square&logo=postgresql&logoColor=white" alt="PostgreSQL 16 + pgvector" />
  <img src="https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white" alt="Redis" />
  <img src="https://img.shields.io/badge/Ollama-000000?style=flat-square&logo=ollama&logoColor=white" alt="Ollama" />
</p>

## 무엇을 만드는가

보유 종목과 자산 데이터를 외부로 내보내지 않고, 로컬 환경 안에서 시세 수집부터 AI 리포트 생성까지 처리하는 백엔드 파이프라인입니다.

- **데이터는 밖으로 나가지 않습니다.** 상용 LLM API 대신 로컬 Ollama 모델을 사용합니다.
- **계산은 코드가, 설명은 LLM이 합니다.** 수익률, MDD, 리밸런싱 수량은 Python으로 계산하고, LLM은 계산된 수치와 검색된 공시 원문만 받아 리포트를 작성합니다.
- **비용 0원으로 동작합니다.** 모든 구성 요소가 로컬과 Docker에서 실행됩니다.

## 아키텍처

```mermaid
flowchart TB
    FE["frontend<br/>Vue 3 · Vite · Pinia<br/>:5173"]

    subgraph BE["backend · Spring Boot 3.5 · :8080"]
        direction LR
        B1["포트폴리오·자산 관리<br/>JPA + QueryDSL"]
        B2["일별 시세·공시 수집<br/>Spring Batch"]
        B3["외부 API 장애 격리<br/>Resilience4j"]
    end

    subgraph AI["ai-engine · FastAPI · :8000"]
        direction LR
        A1["1. 정량 분석<br/>Pandas / NumPy"] --> A2["2. RAG 검색<br/>pgvector"] --> A3["3. 리포트 생성"]
    end

    EXT["외부 시세 API"]
    REDIS[("Redis :6379<br/>시세 캐싱 · 분산 락")]
    PG[("PostgreSQL 16 + pgvector<br/>:5432")]
    OLLAMA["Ollama :11434<br/>로컬 LLM"]

    FE -- "REST API / SSE" --> BE
    BE -- "내부 REST" --> AI
    B3 -. "호출" .-> EXT
    BE <--> REDIS
    BE --> PG
    AI --> PG
    A3 --> OLLAMA
```

## ERD

```mermaid
erDiagram
    users ||--o{ portfolios : "보유"
    users ||--o{ ai_analysis_reports : "분석 이력"

    users {
        bigint user_id PK
        varchar email UK
        varchar risk_profile "STABLE, NEUTRAL, AGGRESSIVE"
        numeric target_cash_ratio "목표 현금 비중"
        timestamptz created_at
    }
    portfolios {
        bigint portfolio_id PK
        bigint user_id FK
        varchar ticker "종목코드"
        numeric quantity "보유수량"
        numeric avg_buy_price "평균매수가"
        numeric target_weight "목표비중"
        bigint version "낙관적 락"
        timestamptz updated_at
    }
    ai_analysis_reports {
        bigint report_id PK
        bigint user_id FK
        numeric total_return_rate "수익률"
        numeric mdd_rate "최대낙폭"
        jsonb rebalancing_guide "종목별 매수·매도 수량"
        text ai_summary_text "LLM 생성 리포트"
        timestamptz created_at
    }
    daily_stock_prices {
        bigint price_id PK
        varchar ticker "ticker + trade_date 유니크"
        date trade_date
        numeric open_price
        numeric high_price
        numeric low_price
        numeric close_price
        bigint volume
    }
    disclosure_vectors {
        bigint doc_id PK
        varchar ticker
        varchar title
        text content_chunk "원문 텍스트"
        vector embedding "768차원"
        timestamptz published_at
    }
```

`daily_stock_prices`와 `disclosure_vectors`는 외래 키 없이 `ticker` 값으로 연결된다.

## 저장소

| 저장소 | 역할 | 스택 |
| :--- | :--- | :--- |
| backend | 포트폴리오 관리, 시세 수집 배치, 외부 API 장애 격리 | Java 17, Spring Boot, JPA, QueryDSL, Spring Batch, Resilience4j |
| ai-engine | 정량 지표 계산, 공시 RAG 검색, 리포트 생성 | Python 3.11, FastAPI, Pandas, NumPy, pgvector |
| frontend | 대시보드, 분석 진행 상태 스트리밍 화면 | Vue 3, Vite, Pinia |
| infra | 로컬 실행 환경과 DB 스키마 | Docker Compose, PostgreSQL, Redis |
| docs | 아키텍처 설계서, 트러블슈팅과 성능 측정 기록 | Markdown |

## 진행 상황

- [x] 인프라와 서비스별 초기 프로젝트 세팅
- [ ] 포트폴리오 관리와 일별 시세 수집 배치
- [ ] 정량 분석과 RAG 기반 리포트 생성
- [ ] 장애 격리, 캐싱, 부하 테스트
