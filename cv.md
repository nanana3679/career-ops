# 고명섭

- Next.js, TypeScript 기반의 프론트엔드 개발자
- 010-9274-4621
- nanana3679@gmail.com
- https://github.com/nanana3679/

---

## 학력

- 국민대학교 졸업 (2018.03 ~ 2024.08, 소프트웨어학부)

---

## 프로젝트

### HADA-reboot

기존 HADA 서비스를 서버리스 단일 스택으로 마이그레이션.

**기간**: 2026.03 (1개월)
**기술 스택**: Cloudflare Workers, Drizzle ORM, Cloudflare D1 (SQLite), KV, Auth.js
**참여 인원**: 1인(개인 프로젝트)

- 이중 서버 구조(Spring Boot + Next.js)에서 배포·모니터링 이중 관리 및 서버 간 네트워크 지연 발생 → Cloudflare Workers 단일 스택으로 마이그레이션 → 서버 운영 비용 0원 달성, 네트워크 지연 제거
- Server Action에서 자체 API Route를 fetch하는 구조로 같은 프로세스 내 불필요한 HTTP 왕복 발생 → Server Actions에서 D1 직접 접근으로 전환하여 직렬화 레이어 제거, 타입 안전성 확보
- 카테고리별 집계 쿼리에서 매 요청마다 58,000건 집계 → KV 캐싱 도입 — 글로벌/유저별 캐시 분리, 학습 시 선택 무효화하여 D1 읽기 부하 감소, 무료 티어 내 운영 유지
- 인증 시스템에서 JWT 직접 구현의 보안 취약점 위험 (XSS, CSRF) → Auth.js v5 + D1 Adapter, HttpOnly 쿠키 기반 세션으로 전환하여 인증 코드 대폭 감소, 보안 강화

### not4k

PC용 웹 리듬게임 플랫폼.

**기간**: 2026.01 ~ 2026.02 (2개월)
**기술 스택**: React, TypeScript, PixiJS, Web Audio API, Zustand, Supabase, Vite, Vitest
**참여 인원**: 1인(개인 프로젝트)

- 매 프레임 Sprite 생성/삭제로 인한 GC 압박과 프레임 드롭 발생 → PixiJS v8 기반 레이어드 컨테이너 + 상태별 오브젝트 풀링 적용 → 60~144fps 이상 안정 유지
- 스트리밍 방식의 오디오 재생이 네트워크 버퍼링으로 인한 불규칙 지연 유발 → Web Audio API의 AudioBuffer 전체 디코딩 방식으로 전환, 로드 시점에 전체 음원을 메모리에 적재 → 샘플 단위 정밀 타이밍 보장, 오프셋 캘리브레이션과 결합하여 입력-음원 간 동기화 달성
- 게임 루프가 requestAnimationFrame(16.67ms 간격) 기반이라 12ms 유예 시간이 0ms 또는 16.67ms로 이분화되는 문제 발생 → KeyboardEvent.timeStamp 기반 이벤트 큐 방식으로 전환, 프레임 주기 의존을 제거하여 서브밀리초 정밀도의 유예 시간 판정 구현

### wordleDecks

사용자가 직접 단어 목록을 만들어 Wordle 게임 덱을 생성·공유·플레이할 수 있는 웹 애플리케이션.

**기간**: 2025.10 (1개월)
**기술 스택**: Next.js, TypeScript, Tailwind CSS, Tanstack Query, shadcn/ui, Supabase, Vercel
**참여 인원**: 1인(개인 프로젝트)

- 좋아요 클릭 후 서버 응답까지 UI 반영이 지연되어 반응성 저하 → useOptimistic + startTransition으로 낙관적 업데이트 적용, 서버 실패 시 자동 롤백 처리 → 클릭 즉시 UI 반영으로 체감 응답 지연 제거
- 덱 공유 링크 전송 시 제목·설명·썸네일이 표시되지 않는 문제 → generateMetadata로 OG 태그 동적 생성 + Edge Runtime 기반 OG 이미지 API 구현, ISR로 정적 캐싱 적용 → 카카오톡·슬랙 등에서 덱 정보가 포함된 리치 미리보기 제공 및 메타데이터 응답 속도 개선
- 비공개 덱이 다른 사용자에게 노출될 수 있는 보안 문제 → Supabase RLS 정책으로 SELECT/UPDATE/DELETE에 소유권 기반 접근 제어 적용 → 서버 액션 레벨이 아닌 DB 레벨에서 권한 검증을 보장하여 보안 계층 강화

### HADA

58,000+ 한국어 단어를 FSRS 간격 반복 알고리즘으로 학습 및 최적 복습 시점 자동 스케줄링 웹 앱.

**기간**: 2025.01 ~ 2025.05 (5개월)
**기술 스택**: Next.js, TypeScript, Tanstack Query, JWT, Figma, Material Web Components (@lit/react), Motion, SCSS Modules
**참여 인원**: 4인 (프론트엔드 2명, 백엔드 2명)

- Material Web Components가 React 미지원 + 유지보수 모드 전환으로 필요한 컴포넌트 부재 → @lit/react로 기존 컴포넌트를 React로 래핑하고, 미제공 컴포넌트는 MD3 스펙에 맞춰 직접 구현. MD3 Window Size Class 기반 5단계 반응형 레이아웃 적용하여 MD3 디자인 시스템을 React 환경에서 일관되게 적용
- 학습 카드 전환 시 카드 데이터를 매번 요청하면 전환마다 로딩 발생 → React Query prefetch로 다음 5장을 미리 캐싱하여 카드 전환 시 로딩 없는 즉각적 UX 제공
- 학습 결과 저장 실패 시 낙관적 업데이트된 큐 상태와 서버 상태 불일치 발생 → useMutation onError에서 큐 revert 처리하여 네트워크 오류 시에도 학습 상태 정합성 유지
- 덱 내 단어 목록 조회 시 58,000건 중 카테고리별 수백~수천 건을 한 번에 로딩하면 초기 로딩 지연 → useInfiniteQuery 기반 무한스크롤 + 페이지네이션 적용, 초기 로딩 최소화 및 점진적 데이터 로딩

---

## 자격

- 정보처리기사
- 네트워크관리사 2급
