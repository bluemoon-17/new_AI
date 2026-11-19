# 상세 구현 명세서: KBO 야구규칙 및 리그 규정 안내 슬랙 봇

## 1. 시스템 아키텍처 개요
본 시스템은 **오프라인 인덱싱 파이프라인(`ingest.py`)**과 **실시간 질의응답 봇 서버(`bot.py`)**의 2개 모듈 그룹으로 분리하여 구현합니다.

1. **인덱싱 파이프라인 (`ingest.py`)**: `data/` 디렉터리의 규정 문서를 로드한 뒤, 2단 분할(구조 기반 + 재귀적 분할)을 수행하고 필수 메타데이터를 주입하여 로컬 `Chroma` 벡터 데이터베이스(`chroma_db/`)에 영구 저장합니다.
2. **슬랙 질의응답 파이프라인 (`bot.py`, `rag_chain.py`)**: `slack_bolt`의 Socket Mode를 통해 슬랙 이벤트를 수신하고, LangChain LCEL 기반 RAG 체인을 호출하여 검색된 규정 근거로 답변을 생성한 뒤 해당 메시지의 `thread_ts`로 회신합니다.

---

## 2. 디렉터리 및 파일 구조
```text
kbo-rule-bot/
├── data/                         # 원본 KBO 규정 문서 (PDF, MD)
│   ├── kbo_official_rules_2026.pdf
│   └── kbo_league_regulations_2026.md
├── chroma_db/                    # 임베딩 벡터 및 메타데이터 영구 저장소 (Git 제외)
├── src/
│   ├── config.py                 # 모델명, 청크 파라미터, 경로 등 환경 설정 상수 관리
│   ├── ingest.py                 # 문서 로드 → 2단 분할 → 메타데이터 부착 → Chroma 적재
│   ├── rag_chain.py              # Retriever 구성, 프롬프트 템플릿, LCEL RAG 체인 정의
│   └── bot.py                    # Slack Bolt 이벤트 핸들러 및 스레드 응답 처리
├── .env                          # API 및 Slack 토큰 (Git 제외)
├── .env.example                  # 환경 변수 템플릿
├── requirements.txt              # 의존성 패키지 목록
├── README.md                     # 프로젝트 개요 및 실행 가이드
├── CONTEXT.md                    # 대상 문서, 분할·메타데이터 설계, 범위 명세
└── SPEC.md                       # 상세 구현 명세서
