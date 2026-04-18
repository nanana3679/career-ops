# LLM + SRS 웹앱 설계

## Context

LLM(Claude 등)과 대화하면서 기억할 지식을 플래시카드로 저장하고, FSRS 기반 간격 반복으로 암기하는 시스템.

기존 obsicards 프로젝트는 Obsidian + SR 플러그인 + Git 동기화 구조였으나, 확장 플러그인 사용성이 떨어지고 설정이 복잡해서 **독립 웹앱**으로 전환.

hada 교훈: 오버엔지니어링 금지, 실제 배포할 것, 기존 오픈소스 활용.

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| 프레임워크 | Next.js (App Router) |
| 언어 | TypeScript |
| DB | Supabase (PostgreSQL) |
| 인증 | NextAuth.js (Google/GitHub OAuth) |
| SRS 알고리즘 | ts-fsrs |
| MCP 서버 | Next.js API route (Streamable HTTP) |
| 배포 | Vercel |
| 스타일링 | Tailwind CSS |

---

## 아키텍처

```
[claude.ai / Claude Mobile / Claude Desktop]
     |
     | MCP 커넥터 (Streamable HTTP)
     v
[Next.js API route: /api/mcp]  <- MCP 서버 (단일 배포)
     |
     v
[Next.js 웹앱]  <- SRS 학습 UI + 카드 관리 + 설정
     |
     +---> [Supabase PostgreSQL]  <- 카드 내용 + FSRS 상태
```

### 핵심 원칙
- MCP 서버와 웹앱이 **같은 코드베이스**, 같은 DB 접근
- MCP가 Vercel에서 안 되면 -> Cloudflare Workers로 분리 (obsicards 경험으로 fallback 확보)
- **MCP on Vercel 검증을 최우선으로** — 실패하면 "같은 코드베이스" 원칙이 무너지므로 Step 1에서 검증

---

## 데이터 모델

### users 테이블 (NextAuth.js 관리)
```sql
-- NextAuth.js adapter가 생성하는 테이블. 아래는 참조용 스키마.
-- Supabase adapter 사용: @auth/supabase-adapter (Auth.js v5)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT,
  email TEXT UNIQUE,
  email_verified TIMESTAMPTZ,
  image TEXT
);
-- NextAuth.js가 추가로 accounts, sessions, verification_tokens 테이블을 생성함.
```

### cards 테이블
```sql
CREATE TABLE cards (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),

  -- 카드 내용 (MVP: qa 타입만)
  type TEXT NOT NULL DEFAULT 'qa' CHECK (type IN ('qa')),
  content JSONB NOT NULL,
  -- qa: { question, answer }

  -- 카테고리 (평탄한 태그 배열)
  categories TEXT[] NOT NULL DEFAULT '{}',

  -- 저장소 출처 (MVP: local만)
  source TEXT NOT NULL DEFAULT 'local' CHECK (source IN ('local')),
  source_id TEXT,

  -- FSRS 상태 (ts-fsrs Card 필드)
  due TIMESTAMPTZ NOT NULL DEFAULT now(),
  stability REAL NOT NULL DEFAULT 0,
  difficulty REAL NOT NULL DEFAULT 0,
  elapsed_days INTEGER NOT NULL DEFAULT 0,
  scheduled_days INTEGER NOT NULL DEFAULT 0,
  reps INTEGER NOT NULL DEFAULT 0,
  lapses INTEGER NOT NULL DEFAULT 0,
  state INTEGER NOT NULL DEFAULT 0,  -- 0=New, 1=Learning, 2=Review, 3=Relearning
  last_review TIMESTAMPTZ,

  -- 전문 검색용
  search_text TSVECTOR GENERATED ALWAYS AS (
    to_tsvector('simple', coalesce(content->>'question', '') || ' ' || coalesce(content->>'answer', ''))
  ) STORED,

  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cards_due ON cards(user_id, due);
CREATE INDEX idx_cards_categories ON cards USING GIN(categories);
CREATE INDEX idx_cards_search ON cards USING GIN(search_text);
```

### 카테고리 설계: 덱 대신 태그

계층적 덱(Anki 방식) 대신 **평탄한 카테고리(태그)** 를 사용한다.

- 카드 하나에 여러 카테고리를 붙일 수 있음 (예: `['Node.js', 'Event Loop']`)
- 학습 시 원하는 카테고리만 필터링 가능
- 카테고리 없는 카드도 허용 (미분류)
- 별도 테이블 없이 `TEXT[]` 배열로 관리 — 카테고리 목록은 cards에서 DISTINCT 집계

**덱 대비 장점:**
- 한 카드가 여러 주제에 속할 수 있음 (교차 분류)
- 계층 관리 불필요 — 서브덱 문제 원천 차단
- MCP에서 카드 추가 시 카테고리 문자열만 넘기면 됨 (덱 ID 조회 불필요)

### review_logs 테이블
```sql
CREATE TABLE review_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  card_id UUID NOT NULL REFERENCES cards(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id),
  rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 4),  -- Again, Hard, Good, Easy
  state INTEGER NOT NULL,  -- 리뷰 시점의 카드 상태
  due TIMESTAMPTZ NOT NULL,
  stability REAL NOT NULL,
  difficulty REAL NOT NULL,
  elapsed_days INTEGER NOT NULL,
  scheduled_days INTEGER NOT NULL,
  review_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_logs_card ON review_logs(card_id);
```

### user_settings 테이블
```sql
CREATE TABLE user_settings (
  user_id UUID PRIMARY KEY REFERENCES users(id),
  daily_new_cards INTEGER NOT NULL DEFAULT 20,
  daily_review_limit INTEGER NOT NULL DEFAULT 200,
  -- FSRS 파라미터 (사용자별 커스텀 가능)
  fsrs_params JSONB,
  -- MCP 인증 (해시 저장, 평문 X)
  mcp_api_key_hash TEXT,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 카드 형식

### MVP: QA 타입만

```json
{ "question": "Node.js의 이벤트 루프 실행 순서는?", "answer": "콜 스택 -> nextTick -> 마이크로태스크 -> 매크로태스크 -> 반복" }
```

### Phase 2 확장 타입

**Definition (한 줄 정의)** — qa의 특수 케이스로, UI 표시만 다름
```json
{ "term": "libuv", "definition": "Node.js의 이벤트 루프와 비동기 I/O를 담당하는 C 라이브러리" }
```

**Cloze (빈칸 채우기)**
```json
{ "sentence": "ESM의 세 단계 동작 순서는 ==구문 분석==, ==인스턴스화==, ==평가==이다." }
```

---

## 리치 콘텐츠 렌더링

카드 content는 마크다운 문자열. 수식, 코드, 다이어그램, 이미지를 지원한다.

### 렌더링 파이프라인

```
마크다운 문자열
  │ remark-gfm        (표, 체크박스, 취소선)
  │ remark-math        ($...$ / $$...$$ 파싱)
  ▼
  │ rehype-katex       (수식 → KaTeX HTML)
  │ rehype-highlight   (코드 → 구문 강조)
  │ rehype-raw         (HTML 허용 — Mermaid CDN용)
  ▼
React 컴포넌트 트리 + Mermaid CDN 스크립트
```

### 수식 (LaTeX)

- 문법: `$인라인$`, `$$블록$$` (GitHub, Obsidian 공통)
- 라이브러리: `remark-math` + `rehype-katex` (KaTeX)
- KaTeX 선택 이유: 빌드타임 프리렌더, MathJax 대비 빠르고 가벼움 (~16kB + CSS)
- MathJax는 수식 커버리지/접근성이 더 좋지만, 플래시카드 수준에서는 KaTeX로 충분

### 코드 블록

- `rehype-highlight`로 구문 강조 (~5kB, 언어 선택 가능)

### 다이어그램 (Mermaid)

- Mermaid 번들 ~2.8MB로 매우 무거움 → **번들에 포함하지 않고 CDN으로 처리**
- `next/script` `afterInteractive` 전략으로 로드
- 마크다운에서 ` ```mermaid ` 코드블록 사용 → `react-markdown` components에서 `<pre class="mermaid">`로 변환

### 이미지

- Supabase Storage public bucket: `flashcard-images/{user_id}/{card_id}/`
- 마크다운 `![alt](url)` 문법 그대로 활용
- `react-markdown`의 `components.img`를 Next.js `<Image>`로 커스텀 (최적화)
- `next.config.js`에 Supabase 도메인 `remotePatterns` 등록 필요

### 오디오

- MVP: **Web Speech API** (`SpeechSynthesis`) — 무료, 패키지 불필요
- 제약: iOS Safari는 사용자 탭 없이 자동재생 불가, 브라우저별 음성 품질 차이
- Phase 2: 고품질 필요 시 ElevenLabs + Supabase Storage 캐싱

### MVP vs Phase 2

| 기능 | MVP | Phase 2 |
|------|-----|---------|
| 수식 (KaTeX) | O | |
| 코드 강조 | O | |
| 이미지 (Supabase) | O | |
| Mermaid (CDN) | | O |
| 오디오 (Web Speech) | | O |
| 오디오 (ElevenLabs) | | O |

MVP에서는 수식 + 코드 + 이미지만. Mermaid와 오디오는 Phase 2.

---

## 학습 세션 설계 (hada 경험 기반)

### 핵심 개념: 결정적 셔플

hada에서 가장 중요했던 설계 결정. **같은 날, 같은 학습 상태에서는 항상 같은 카드 순서가 나온다.**

```
seed = hash(user_id + date_string)
queue = deterministic_shuffle(collectable_cards, seed)
today_queue = queue.slice(0, daily_limit)
```

이 설계 덕분에:
- **학습량 조절이 자연스러움**: 위에서부터 잘라내므로 진행률 보존
- **새로고침해도 순서 복원**: 서버에서 순서를 다시 보내지 않아도 됨
- **계속 학습하기가 간단**: 슬라이스 범위만 확장
- **prefetch 대상이 명확**: 현재 위치 + N개를 미리 로드

### 유저 스토리 1: 일일 학습

```
1. 사용자가 /study 진입
2. 서버에서 오늘의 학습 대상 수집:
   a. 복습 카드: due <= now AND state != New (overdue)
   b. 새 카드: state == New (daily_new_cards 한도까지)
3. 수집된 카드를 결정적 셔플로 정렬
4. 셔플된 큐에서 daily_review_limit만큼 잘라서 오늘의 큐 확정
5. 카드 표시 (질문만 보임)
6. "정답 확인" 탭/클릭 -> 답변 표시
7. 4단계 평가: Again / Hard / Good / Easy
8. ts-fsrs로 다음 복습 일정 계산 -> DB 즉시 업데이트
9. 다음 카드로 이동
10. 모든 카드 완료 시 -> 완료 화면 + "계속 학습하기" 버튼
```

### 유저 스토리 2: 카테고리 필터 학습

```
1. /study 진입 시 카테고리 필터 UI 표시
   - 전체 학습 (기본)
   - 특정 카테고리만 선택 (복수 선택 가능)
2. 필터 적용 시 해당 카테고리의 카드만 수집
3. 이후 흐름은 유저 스토리 1과 동일
4. 카테고리 필터는 URL 파라미터로 관리 (/study?categories=Node.js,React)
   -> 공유/북마크 가능
```

### 유저 스토리 3: 계속 학습하기 (hada 방식)

```
상황: 오늘 할당된 카드를 전부 학습 완료함.

1. 완료 화면에서 "계속 학습하기" 버튼 표시
2. 버튼 클릭 시:
   - 결정적 셔플 큐에서 기존 daily_limit 이후의 카드를 추가로 가져옴
   - 예: 원래 한도 20장 -> "계속"하면 21~40번째 카드를 추가
   - 매번 +20장씩 확장 (설정의 daily_new_cards 단위)
3. 추가 학습분은 서버에 기록되지만, 다음 날 일일 한도에는 영향 없음
4. 결정적 셔플 덕분에 큐 확장이 자연스러움 — 이미 정해진 순서에서 더 가져올 뿐
```

**왜 이 설계가 좋은가 (hada 교훈):**
- 셔플 큐가 이미 전체 카드에 대해 결정되어 있으므로, "더 가져오기"가 단순한 슬라이스 확장
- 새로운 카드를 랜덤으로 뽑는 게 아니라 정해진 순서를 따르므로, 새로고침해도 동일한 추가 카드를 봄
- 서버 상태 변경 없이 클라이언트에서 처리 가능

### 유저 스토리 4: 학습량 조절

```
상황: 사용자가 일일 학습량을 변경하고 싶음.

A. 설정에서 변경 (영구적)
   1. /settings에서 daily_new_cards, daily_review_limit 수정
   2. 변경은 다음 날부터 적용 (hada 교훈: 즉시 적용하면 시간여행 문제 발생)
   3. 오늘 학습이 이미 진행 중이면 현재 큐는 유지

B. 학습 중 일시적 조절
   1. 학습 화면에서 "오늘만 줄이기 / 늘리기" 옵션
   2. 줄이기: 현재 큐에서 아직 안 한 카드 수를 줄임 (진행된 건 유지)
   3. 늘리기: 유저 스토리 3의 "계속 학습하기"와 동일한 메커니즘
   4. 일시적 변경이므로 다음 날에는 원래 설정으로 복귀
```

**hada에서 배운 날짜 경계 처리:**
- "하루의 기준 시각" 옵션을 넣지 않음 (hada에서 시간여행 문제 유발)
- 하루 경계 = UTC 자정 (또는 사용자 타임존 자정) 고정
- 설정 변경은 항상 다음 날부터 적용 -> 결정적 셔플의 전제(같은 날 = 같은 큐)가 유지됨

### 유저 스토리 5: 학습 중 카드 재출현 (Learning 상태)

```
상황: Again을 누른 카드가 같은 세션 내에서 다시 나와야 함.

1. ts-fsrs가 Again/Hard 평가 시 짧은 간격(1분, 10분 등)을 반환
2. 이 카드는 state=Learning 또는 state=Relearning이 됨
3. 학습 세션 내 처리:
   - due가 현재 세션 내 시간인 Learning 카드는 세션 큐 뒤쪽에 재삽입
   - 새 카드/복습 카드 사이사이에 Learning 카드가 끼어들어옴
   - 구체적으로: 현재 위치에서 N장(기본 3장) 뒤에 삽입
4. Learning 카드는 daily_review_limit에 카운트하지 않음 (Anki 방식)
```

### 유저 스토리 6: 새로고침 시 진도 보존

```
1. 카드 평가할 때마다 결과를 서버에 즉시 전송 (낙관적 업데이트)
2. 새로고침 시:
   a. 서버에서 카드 상태(FSRS 필드) 재조회
   b. 결정적 셔플로 큐 재구성 (같은 seed -> 같은 순서)
   c. 이미 학습한 카드(오늘 review_log가 있는)는 큐에서 제외
   d. -> 자연스럽게 이어서 학습
3. 멀티탭: 탭 간 통신 없이 수용 (hada 결정). 중복 학습 가능하지만, review_log에 기록됨.
```

### 유저 스토리 7: MCP로 카드 추가 후 학습

```
1. Claude와 대화 중 "이거 카드로 만들어줘"
2. MCP addCards 호출 -> DB에 카드 생성 (state=New, due=now)
3. 사용자가 웹앱으로 이동하여 /study 진입
4. 방금 추가된 카드가 새 카드 풀에 포함됨
5. 결정적 셔플 결과에 새 카드가 반영됨
   - 단, 이미 오늘의 큐가 생성된 후에 추가된 카드는 큐 재계산 필요
   - 처리: 학습 시작 시점에 큐를 계산하므로, /study 재진입 시 반영
   - 학습 중에 추가된 카드는 현재 세션에는 안 나옴 (다음 세션부터)
```

---

## MCP 서버 도구

Next.js API route `/api/mcp`에서 MCP 서버를 호스팅.

### 인증
- MCP API key를 `user_settings.mcp_api_key_hash`에 **해시로 저장** (SHA-256)
- 웹앱 설정 페이지에서 API key 생성 시 한 번만 평문 표시, 이후 해시만 보관
- claude.ai 커넥터: Authless + URL query parameter 방식 (`/api/mcp?key={API_KEY}`)
- 요청 시 key를 해시하여 DB의 해시와 비교 -> user_id 조회

### 도구 정의

**addCards** -- 카드 생성
```typescript
{
  categories: string[],  // 카테고리 (없으면 미분류)
  cards: Array<{
    question: string,
    answer: string,
  }>,  // 최대 50장
}
```

**searchCards** -- 카드 검색
```typescript
{
  query: string,         // 전문 검색 (tsvector 활용)
  category?: string,     // 카테고리 필터
  limit?: number,        // 기본 10
}
```

**getStudyStatus** -- 학습 현황
```typescript
// 입력 없음
// 반환: 오늘 새 카드/복습 카드 수, 카테고리별 현황
```

---

## 웹앱 페이지 구조

```
/                    -> 대시보드 (오늘의 학습 현황, 카테고리별 카드 수)
/study               -> SRS 학습 UI (카테고리 필터 + 학습)
/cards               -> 카드 브라우저 (검색, 카테고리 필터)
/cards/new           -> 카드 생성
/cards/[cardId]      -> 카드 상세/편집
/settings            -> 설정 (일일 학습량, FSRS 파라미터, MCP API key)
/auth/signin         -> 로그인
```

---

## 프로젝트 구조

```
src/
  app/
    page.tsx                  # 대시보드
    study/page.tsx            # 학습 UI (카테고리 필터 포함)
    cards/
      page.tsx                # 카드 브라우저
      new/page.tsx            # 카드 생성
      [cardId]/page.tsx       # 카드 상세/편집
    settings/page.tsx         # 설정
    api/
      mcp/route.ts            # MCP 서버 엔드포인트
      cards/route.ts          # 카드 CRUD API
      study/route.ts          # 학습 세션 API (큐 조회, 리뷰 제출)
  lib/
    db.ts                     # Supabase 클라이언트
    fsrs.ts                   # ts-fsrs 래퍼 (스케줄링 로직)
    shuffle.ts                # 결정적 셔플 (seed 기반)
    mcp/
      server.ts               # MCP 서버 설정 + 도구 등록
      tools/
        addCards.ts
        searchCards.ts
        getStudyStatus.ts
    types.ts                  # 공유 타입 정의
    validators.ts             # 입력 검증 (Zod 스키마)
```

---

## 개발 계획

### Phase 1 -- 핵심 기능 (MVP)

**Step 1: MCP 검증 + 프로젝트 셋업**
- [ ] Next.js API route에서 최소 MCP 서버 배포 (echo 도구 하나만)
- [ ] Vercel에 배포 후 claude.ai 커넥터 연결 테스트
- [ ] **실패 시 -> Cloudflare Workers로 분리** (이 시점에 결정해야 이후 구조가 달라짐)
- [ ] 성공 시 -> Next.js + TypeScript + Tailwind 프로젝트 본격 셋업
- [ ] Supabase 프로젝트 생성 + 스키마 마이그레이션
- [ ] Auth.js v5 설정 (Google OAuth + Supabase adapter)
- **완료 기준**: claude.ai에서 커넥터 추가 후 echo 도구 호출 성공

**Step 2: 카드 CRUD + 웹앱**
- [ ] 카드 CRUD API (생성, 조회, 수정, 삭제)
- [ ] 카드 생성 UI (qa 타입, 카테고리 태그 입력)
- [ ] 카드 브라우저 (카테고리 필터, tsvector 전문 검색)
- [ ] 카드 상세/편집 UI
- **완료 기준**: 웹앱에서 카드 생성/검색/편집/삭제가 동작

**Step 3: SRS 학습**
- [ ] ts-fsrs 통합 -- 스케줄링 로직 (fsrs.ts 래퍼)
- [ ] 결정적 셔플 구현 (shuffle.ts)
- [ ] 학습 UI -- 카드 큐, 정답 확인, 4단계 평가
- [ ] Learning 카드 세션 내 재삽입 로직
- [ ] 계속 학습하기 (큐 확장)
- [ ] 일일 학습량 설정 (변경은 다음 날 적용)
- [ ] 대시보드 -- 오늘의 학습 현황 (카테고리별)
- **완료 기준**: 카드 10장을 만들고 학습 세션을 완료할 수 있음. 새로고침해도 진도 유지.

**Step 4: MCP 도구 구현**
- [ ] addCards 도구 (카테고리 + qa 카드 배열)
- [ ] searchCards 도구 (전문 검색 + 카테고리 필터)
- [ ] getStudyStatus 도구
- [ ] MCP API key 생성/인증 (해시 저장)
- **완료 기준**: claude.ai에서 addCards로 카드 3장 생성 -> 웹앱 대시보드에서 확인 가능

### Phase 2 -- 실사용 개선
- [ ] Notion 연동 (외부 저장소)
- [ ] Definition, Cloze 카드 타입 추가
- [ ] 모바일 반응형 최적화 (PWA)
- [ ] 학습 통계 (일별/주별 리뷰 수, 정답률)
- [ ] 카드 마크다운 렌더링 (수식, 코드 블록)

### Phase 3 -- 확장
- [ ] FSRS 파라미터 최적화 (리뷰 로그 기반)
- [ ] 카드 가져오기/내보내기 (Anki 호환)
- [ ] 추가 외부 저장소 어댑터

---

## 검증 포인트

1. **MCP Streamable HTTP on Vercel**: **Step 1에서 최우선 검증**. 실패 시 Workers fallback이 아키텍처 전체에 영향.
2. **ts-fsrs 통합**: 카드 상태 업데이트가 DB 스키마와 매끄럽게 매핑되는지.
3. **결정적 셔플**: 같은 입력에 대해 항상 같은 순서가 나오는지. 큐 확장 시 기존 순서가 유지되는지.
4. **claude.ai 커넥터 인증**: Authless + query parameter API key 방식이 정상 동작하는지.

