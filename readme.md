# KBO 야구규칙 및 리그 규정 안내 슬랙 봇 (KBO-RuleBot)

## 1. 목적
야구 경기 중 발생하는 판정 상황(인필드 플라이, 보크, 비디오 판독 대상, 연장전 및 우천·서스펜디드 규정 등)을 확인할 때, 수십 페이지에 달하는 KBO 공식 야구규칙과 리그 규정집에서 해당 조항을 직접 찾아야 하는 불편을 해결하기 위해 기획했습니다. 슬랙 채널이나 DM에서 자연어로 질문하면 관련 규정 청크를 검색하여 정확한 근거 조항 및 페이지 출처와 함께 같은 스레드에 답변하는 RAG 기반 질의응답 봇을 구축합니다.

## 2. 기능
- **슬랙 스레드 기반 자연어 질의응답**: 슬랙 채널 내 봇 멘션(`@야구봇`) 또는 DM으로 질문을 수신하고, 질문이 작성된 스레드에 답변을 반환합니다.
- **근거 중심 RAG 검색 및 출처 표기**: 질문과 의미적으로 가장 유사한 규정 청크를 검색하여 답변을 생성하며, 환각(Hallucination)을 방지하기 위해 `[문서명, 장/조, 페이지]` 형태의 출처를 함께 명시합니다.
- **메타데이터 필터링 검색**: 규정 기준 연도(`year`) 및 규정 카테고리(`category`: 경기규칙, 리그운영, 비디오판독 등) 메타데이터를 활용해 검색 범위를 좁혀 정확도를 높입니다.

## 3. 사용 모델과 선택 근거
- **생성 모델**: `gemma3:4b` (3.3GB, Ollama 로컬 구동)
- **임베딩 모델**: `qwen3-embedding:0.6b` (약 0.6GB, Ollama 로컬 구동) *(대안: `bge-m3`, 약 1.2GB)*
- **선택 근거**:
  - **한국어 지원 및 품질**: 한국어 문맥과 법률·규정 특유의 문어체 표현을 안정적으로 이해하고 임베딩할 수 있는 모델을 채택했습니다.
  - **비용 및 보안/오프라인 구동**: 외부 상용 API 상시 호출로 인한 토큰 비용 및 키 소진 문제를 방지하고, 로컬 환경에서 안정적으로 상시 응답할 수 있도록 구성했습니다.
  - **하드웨어(VRAM) 제약 충족**: 4주차 모델 비교 기준을 바탕으로 로컬 GPU VRAM 8GB 환경에서 생성 모델과 임베딩 모델이 동시에 적재되어도 메모리 초과 없이 동작하도록 선정했습니다.
- **VRAM 계산**:
  - `gemma3:4b (3.3GB)` + `KV 캐시·오버헤드 (1.5GB)` + `qwen3-embedding:0.6b (0.6GB)` = **약 5.4GB** (`bge-m3` 사용 시 **약 6.0GB**) → **8GB VRAM 한도 내 안정적 동작**

## 4. 사용 스택
- **Interface**: Slack (`slack_bolt` Socket Mode)
- **LLM & Embedding Runtime**: Ollama (`gemma3:4b`, `qwen3-embedding:0.6b`)
- **LangChain**:
  - Document Loaders (`PyPDFLoader`, `TextLoader`)
  - Text Splitters (`MarkdownHeaderTextSplitter`, `RecursiveCharacterTextSplitter`)
  - VectorStore (`Chroma`) 및 LCEL 기반 RAG 체인 (`RunnablePassthrough`, `StrOutputParser`)
- **LangSmith (필수)**: 슬랙 질문 입력 수신부터 VectorDB 문서 검색(Retriever), 프롬프트 조립, `gemma3:4b` 답변 생성까지의 전체 RAG 체인 실행 흐름과 지연 시간·검색 청크를 추적 및 디버깅하는 데 적용
- **LangGraph**: 미사용 (단일 선형 RAG 파이프라인으로 구성)

## 5. 실행 방법
1. **저장소 클론 및 Python 가상환경 활성화**
   ```bash
   git clone <repository-url>
   cd kbo-rule-bot
   python -m venv venv
   source venv/bin/activate        # macOS / Linux
   # venv\Scripts\activate         # Windows
