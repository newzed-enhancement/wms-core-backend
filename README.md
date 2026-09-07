# B2B WMS AI Platform — Backend

중고·반품 도서가 물류센터에 입고될 때, 촬영된 사진으로 **외관 상태를 판정하고 품질 등급을
매기는** 시스템의 백엔드입니다. FastAPI API 서버와 Celery 기반 AI 검수 워커로 구성됩니다.

사람이 한 권씩 눈으로 보고 등급을 매기던 작업을, 사진 몇 장으로 대체하는 것이 목표입니다.
다만 AI가 애매하다고 판단한 건은 자동 확정하지 않고 관리자에게 넘깁니다(HITL).

---

## 검수가 흘러가는 방식

```
작업자 촬영 → S3 직접 업로드 → 검수 요청(202 Accepted)
                                      │
                                      ▼
                            Celery 워커 (inspection 큐)
                                      │
   Book Detector → Vision → Policy → Critic → Supervisor → Report
    (YOLO-World)  (GPT-4o) (UBCI규칙) (교차검증) (지휘·라우팅)  (보증서)
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
              자동 확정(APPROVED)                  HITL(관리자 판정)
                    │                                   │
                    └──────────► 재고 편입 ◄────────────┘
```

- **비동기 처리**: API는 접수만 하고 즉시 202로 응답합니다. 추론은 워커가 맡아
  트래픽이 몰려도 API가 막히지 않습니다.
- **이미지는 백엔드를 거치지 않습니다**: 클라이언트가 S3에 직접 올리고, 백엔드와 AI 모듈은
  CloudFront URL만 주고받습니다.
- **진행 상황은 SSE로 전달**합니다. 1회용 티켓으로 인증하며, 연결이 끊기면 지수 백오프로
  재연결하고 그래도 안 되면 폴링으로 넘어갑니다.

### 판정 파이프라인 (Multi-Agent)

LangGraph 위에 올린 Supervisor 구조입니다. Supervisor가 하위 에이전트의 보고를 종합해
다음 행동을 결정하고, **그 판단 근거를 state에 남깁니다**(관리자 화면에서 왜 이 경로로
갔는지 추적 가능).

| 단계 | 하는 일 | LLM |
|---|---|---|
| Book Detector | 사진에서 책 영역을 찾아 잘라냅니다 | 없음 (YOLO-World) |
| Vision | 결함 종류·위치·크기를 판독합니다 | GPT-4o + 검증용 GPT-4o-mini |
| Policy | UBCI 감점 매트릭스로 점수·등급을 확정합니다 | 없음 (규칙) + RAG 근거 검색 |
| Critic | Vision·Policy 결과가 서로 맞는지 교차 검증합니다 | GPT-4o-mini (판례 RAG) |
| Report | 고객이 읽는 품질보증서 문구를 만듭니다 | GPT-4o-mini (실패 시 규칙 폴백) |

**점수와 등급은 LLM이 정하지 않습니다.** 매입가에 직결되는 값이라 재현성과 감사 추적이
필요해서, Policy는 결정적 규칙으로만 계산합니다. LLM은 판독·검증·문장 생성에만 씁니다.

안전장치 몇 가지:
- 촬영한 컷 전부에서 책을 못 찾았는데 "결함 0건"이 나오면 자동 확정을 막고 관리자에게
  넘깁니다. **"검수하지 못했다"와 "검수했더니 흠이 없다"는 다릅니다.**
- 무결점(MINT) 건도 Policy·Critic 검증을 그대로 통과시킵니다. 우회 경로를 두면 판독이
  실패했을 때 그대로 자동 승인되는 구멍이 생깁니다.
- 노드마다 소요 시간과 LLM 토큰·비용을 수집해 `agent_logs`에 남깁니다.

---

## 구조

```
app/
├── main.py            # 라우터 등록 (20개)
├── core/              # 설정, DB, Celery, 보안, Redis Pub/Sub, WMS 클라이언트
├── domains/           # 도메인별로 라우터·서비스·스키마를 한 폴더에 모음
│   ├── auth/          admin/         books/       dev/
│   ├── inbound/       inspections/   inventory/   lpn/
│   ├── notifications/ orders/        pricing/     restock/
├── models/            # SQLModel 테이블 (도메인별 분할, wms.py가 집약 재수출)
├── ai/
│   ├── agents/        # detector · vision · policy · critic · report · human
│   ├── supervisor.py  # LangGraph 그래프 정의와 지휘 판단
│   ├── rag/           # UBCI 정책 검색, Critic 판례 저장·검색
│   └── instrumentation.py  # 노드별 지연·토큰·비용 계측
├── worker/            # Celery 태스크
└── batch/             # 주기 배치
```

한 도메인을 고칠 때 여러 폴더를 오가지 않도록, 라우터와 서비스와 스키마를 도메인 폴더
안에 함께 둡니다.

---

## 시작하기

### 1. 사전 준비

- Docker Desktop, [uv](https://docs.astral.sh/uv/)
- `.env` 파일 (필수 항목과 값은 [로컬 개발 가이드](docs/Team_Local_Dev_Guide.md) 참고)
- **YOLO 가중치 4종**을 `models/` 폴더에 배치 — Git에 올라가지 않으므로 별도로 받아야
  합니다. 없으면 검수 요청이 실패합니다. 목록은 [models/README.md](models/README.md) 참고.

### 2. 실행

```bash
docker compose up -d --build       # DB·Redis·Chroma·API·워커 전체 기동
docker compose run --rm rag-seed   # UBCI 정책을 벡터 DB에 적재 (최초 1회)
```

- API: http://localhost:8080 (Swagger: `/docs`)
- Celery 모니터링(Flower): http://localhost:5556

포트는 개인 트랙과 겹치지 않도록 분리되어 있습니다 (DB 5433 / Redis 6380 / Chroma 8002).

### 3. 테스트

```bash
uv sync --dev
uv run pytest -q
```

---

## 품질 게이트

PR을 올리면 아래 네 가지가 **모두 통과해야** 머지할 수 있습니다. 로컬에서 먼저 돌려 보세요.

```bash
uv run ruff check app/          # 린트
uv run ruff format --check app/ # 포맷
uv run mypy app/                # 타입
uv run pytest -q                # 테스트 (64개 파일)
```

규칙을 무시한 항목은 전부 `pyproject.toml`에 **사유와 함께** 적혀 있습니다. 새로 무시할
때도 이유를 남겨 주세요. 자세한 내용은 [로컬 개발 가이드](docs/Team_Local_Dev_Guide.md)에
있습니다.

---

## 문서

| 문서 | 내용 |
|---|---|
| [고도화 참여자 온보딩](docs/ONBOARDING.md) | **새로 합류했다면 먼저 읽으세요.** 바뀐 구조, 지켜야 할 규칙, 알려진 함정 |
| [API 명세서](docs/API_Specification.md) | 엔드포인트 70개. **OpenAPI에서 자동 생성**하므로 손으로 고치지 않습니다 |
| [로컬 개발 가이드](docs/Team_Local_Dev_Guide.md) | 환경변수, 실행, RAG 시딩, 품질 게이트 |
| [기획서](docs/B2B_WMS_AI_Platform_기획서_ver1.6.0.0.md) | 전체 시스템 구조와 요구사항 |
| [워크플로우](docs/B2B_WMS_AI_Platform_워크플로우_ver1.6.0.0.md) | 시퀀스 다이어그램, 데이터 흐름 |
| [용어집](docs/B2B_WMS_AI_Platform_용어집_ver1.6.0.0.md) | UBCI, LPN 등 도메인 용어 |
| [DB 마이그레이션 가이드](docs/Database_Migration_Guide.md) | Alembic 사용법 |
| [Git 협업 규칙](docs/Git_Collaboration_Guide.md) | 브랜치 규칙, PR 절차 |

API 명세서를 다시 뽑으려면:

```bash
uv run python scripts/generate_api_spec.py
```

---

## 관련 저장소

- 프론트엔드: [wms-core-frontend](https://github.com/newzed-enhancement/wms-core-frontend)

## 만든 사람들

KT AIVLE School 9기 AI 2반 5조 빅프로젝트.

- **PM · 아키텍처**: 장문경
- **백엔드**: 박민우, 서다은
- **프론트엔드**: 박준희, 고영빈, 소한민
- **AI**: 홍경표

아키텍처 설계와 기획의 저작권은 장문경에게 있으며, 각 팀원의 구현 기여는 커밋 이력에
남아 있습니다.
