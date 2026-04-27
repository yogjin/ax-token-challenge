---
title: "병렬 에이전트로 캐싱 전략 분석 후 Opus로 팩트체크하기"
date: "2026-04-27"
tokens: "측정 중"
problems: 1
tools: ["Claude Code", "Parallel Agents (Haiku)", "Opus 4.6"]
---

## 뭘 했는가

Next.js + TanStack Query 기반 프론트엔드의 캐싱 전략을 병렬 AI 에이전트 2개(Explore, Architecture)로 분석했다. 결과가 나온 후 Opus로 모델을 전환해 직접 코드를 읽으며 재검증했더니, **방향은 맞았지만 수치와 메커니즘 설명이 부정확**한 부분들이 발견되었다.

### 해결한 문제

**"캐싱 전략이 없는 프로젝트"의 정확한 진단** — 4개 캐싱 계층(TanStack Query, HTTP 헤더, CDN, Next.js 서버)을 코드 기반으로 분석하고, AI 분석 결과의 과장/오류를 걸러내 실제 적용 가능한 가이드를 도출했다.

## 어떻게 했는가

### 1단계: 핵심 파일 파악

직접 `QueryProvider.tsx`, `next.config.js`, `clientAxios.ts`, `serverAxios.ts`를 읽어 전체 구조를 파악했다.

### 2단계: 병렬 에이전트 분석

Haiku 기반 에이전트 2개를 동시에 띄웠다:

```
에이전트 1 (Explore): React Query 설정, queryKey 패턴, 캐시 무효화 전략 탐색
에이전트 2 (Architecture): 캐싱 아키텍처 SWOT 분석, 개선안 제시
```

각 에이전트가 33~35개 도구를 호출하며 코드베이스를 독립적으로 분석. 약 2~3분 만에 상세한 보고서 두 개가 나왔다.

### 3단계: Opus로 재검증

사용자가 분석 결과에 질문을 던지면서 (`"서버 메모리는 문제없잖아"`, `"HTTP 캐싱이 Next.js 서버 캐싱이야?"`) Haiku의 설명이 부정확하다는 것이 드러났다. `/model`로 Opus로 전환 후 직접 검증한 결과:

| 항목 | Haiku 분석 | Opus 재검증 |
|------|----------|------------|
| staleTime 미설정 | 맞음 | 확인 |
| "서버 부하 1000배" | 과장 | 사용자당 추가 요청 수준 |
| "네트워크 60-70% 감소" | 근거 없음 | 패턴에 따라 다름 |
| HTTP 헤더 = 서버 캐싱 | 혼동 | 브라우저 지시사항일 뿐 |
| unstable_cache 권장 | 부분적 | Next.js 16은 `use cache` |

### 4단계: 최종 가이드 작성

Next.js 16 공식 문서를 조회해 최신 API(`use cache`, `cacheLife`, `cacheTag`)를 확인하고, 4계층(TanStack Query → HTTP 캐시 → CDN → Next.js 서버) 각각의 역할과 프로젝트에 맞는 적용 방안을 정리했다.

## 분석 결과: 현재 캐싱 현황

### 프로젝트 아키텍처

```
사용자 브라우저
  ↓ (clientAxios)
Next.js API Route (/api/[id]/route.ts)  ← 프록시 역할
  ↓ (serverAxios + Bearer 토큰 주입)
백엔드 API
  ↓
DB
```

모든 클라이언트 API 호출이 Next.js 서버를 경유하는 프록시 구조. 이 프록시의 목적은 **인증 토큰 은닉**(보안)이지 캐싱이 아니다.

### 4개 캐싱 계층의 현재 상태

```
┌───────────────────┬──────────────────────────────────┐
│ TanStack Query    │ gcTime: 5분, staleTime: 0 (기본값)│
│ (클라이언트 메모리) │ → 데이터 보관하지만 매번 refetch  │
├───────────────────┼──────────────────────────────────┤
│ HTTP 헤더         │ no-cache, must-revalidate        │
│ (브라우저 캐시)    │ → 브라우저가 응답 캐시하지 않음    │
├───────────────────┼──────────────────────────────────┤
│ CDN               │ 설정 없음                         │
│                   │ → CDN 캐시 불가                   │
├───────────────────┼──────────────────────────────────┤
│ Next.js 서버      │ 아무것도 없음                      │
│ (서버 캐시)        │ → force-dynamic, 캐시 API 미사용  │
└───────────────────┴──────────────────────────────────┘

결론: 사실상 캐싱 전략이 존재하지 않음.
```

### React Query 설정 실태

전역 설정 (`QueryProvider.tsx`):

```typescript
defaultOptions: {
  queries: {
    gcTime: FIVE_MINUTES,       // 5분
    retry: 1,
    refetchOnWindowFocus: false,
    // staleTime 없음 → 기본값 0
  },
}
```

65개 파일에서 `useQuery`를 사용하는데, `staleTime`을 명시한 곳은 **딱 1곳**. 나머지는 전부 기본값 0 → 매 마운트마다 refetch.

## 분석 결과: 캐싱 전략 가이드

### 4계층의 역할 분담

```
TanStack Query:  "같은 탭에서 같은 데이터를 반복 요청하지 않게"
                  → 페이지 이동 후 복귀 시 즉시 표시

HTTP 캐시:       "같은 브라우저에서 같은 URL을 반복 요청하지 않게"
                  → 새로고침 시에도 캐시 활용

CDN 캐시:        "같은 URL을 여러 사용자가 반복 요청하지 않게"
                  → 서버 부하 대폭 감소

Next.js 서버:    "백엔드 API를 반복 호출하지 않게"
                  → DB 부하 감소
```

### staleTime과 gcTime의 관계

```
staleTime = 0 (현재):
  마운트 → 캐시 반환 → 하지만 즉시 stale → 무조건 refetch
  
staleTime = 5분 (개선):
  마운트 → 캐시 반환 → 5분 내면 fresh → refetch 안 함
  5분 후 → 캐시 반환 → stale → 백그라운드 refetch → UI 자동 갱신
```

핵심: staleTime은 "이 데이터를 얼마나 믿을 것인가", gcTime은 "안 쓰는 캐시를 언제 버릴 것인가".

### HTTP Cache-Control 헤더의 실체

현재 `no-cache, must-revalidate`가 설정되어 있는데, 이것은 **Next.js가 브라우저에 보내는 응답 헤더**일 뿐이다. Next.js 서버 자체가 캐싱하는 것과는 무관하다.

```
Cache-Control 헤더 = "브라우저야, 이 응답을 캐시하지 마"
Next.js 서버 캐싱 = use cache, unstable_cache 같은 별도 API 필요
```

이 둘을 혼동하면 "HTTP 헤더를 바꾸면 서버가 캐싱한다"는 오해가 생긴다.

### Next.js 16의 새로운 캐싱 모델

```typescript
// use cache 디렉티브 (unstable_cache 대체)
export async function getArticles() {
  'use cache'
  cacheLife('hours')      // 1시간 캐시
  cacheTag('articles')    // 태그 기반 무효화

  return await fetch('/api/articles');
}

// 무효화 (Server Action에서)
async function createArticle() {
  'use server'
  await db.article.create({ ... });
  updateTag('articles');  // 태그로 캐시 무효화
}
```

### 데이터 유형별 권장 전략

| 데이터 유형 | TanStack staleTime | HTTP 캐시 | CDN | Next.js 서버 |
|-----------|-------------------|-----------|-----|-------------|
| 기사 목록 (공개) | 5분 | max-age 60초 | s-maxage 300초 | use cache |
| 기사 상세 (공개) | 10분 | max-age 60초 | s-maxage 600초 | use cache |
| 프로필 (사용자별) | 1분 | private, no-cache | 불가 | 불가 |
| 댓글 (실시간) | 0 | no-cache | 불가 | 불가 |
| 광고 (1회성) | Infinity | no-cache | 불가 | 불가 |
| 카테고리 (정적) | 30분 | max-age 300초 | s-maxage 1800초 | use cache |

### 단계별 적용 계획

**Phase 1 — 즉시 (1줄 수정)**:

```typescript
// QueryProvider.tsx에 staleTime 추가
defaultOptions: {
  queries: {
    gcTime: FIVE_MINUTES,
    staleTime: A_MINUTE,  // ← 이 한 줄
    retry: 1,
    refetchOnWindowFocus: false,
  },
}
```

**Phase 2 — 단기 (1주)**: HTTP Cache-Control을 데이터 유형별로 차등화

**Phase 3 — 중기 (2-4주)**: Next.js 서버 캐싱(`cacheComponents: true` + `use cache`) 도입

## 사용한 프롬프트

```
요즘IT의 캐싱 전략이 어떻게 되어있는지 코드 전체를 보고 분석해.
전문가 불러서 분석해. 병렬 에이전트 써도 돼.
```

이후 분석 결과에 대한 질문으로 검증을 유도:

```
그게 메모리 누수랑 관련이있어?
서버 메모리는 문제없는거잖아
http캐싱이라는게 nextjs 서버단의 캐싱이야?
니가 다시 판단해봐 opus야
```

마지막 프롬프트가 핵심이다. `/model`로 Opus로 전환한 뒤 **"니가 다시 판단해봐"** 한 마디로 이전 분석 전체를 재검증하게 만들었다.

## 깨달음

- **병렬 에이전트는 탐색에 강하고 판단에 약하다.** Haiku 에이전트가 65개 파일에서 캐싱 설정을 찾아내는 속도는 빨랐지만, "60-70% 감소" 같은 수치를 근거 없이 만들어냈다. 탐색은 Haiku에 맡기고, 판단은 Opus로 검증하는 2단계 구조가 효과적이다.
- **사용자의 질문이 최고의 팩트체크다.** "서버 메모리는 문제없잖아"라는 한 마디가 Haiku의 분석 오류를 정확히 짚어냈다. AI 결과를 그대로 신뢰하지 않고 질문을 던지는 것이 중요하다.
- **캐싱은 계층별로 하는 일이 다르다.** TanStack Query(탭 내 중복 방지), HTTP 캐시(브라우저 중복 방지), CDN(사용자 간 중복 방지), Next.js 서버(백엔드 보호). 이걸 혼동하면 "HTTP 헤더를 바꾸면 서버가 캐싱한다" 같은 오해가 생긴다.
- **가장 실용적인 첫 단계는 놀라울 정도로 간단하다.** `staleTime: A_MINUTE` 한 줄 추가만으로 "페이지 이동 후 1분 내 복귀 시 즉시 표시"가 된다. 캐싱 전략이라고 하면 거창하게 느껴지지만, 핵심은 "데이터를 얼마나 믿을 것인가"라는 하나의 숫자를 결정하는 것이다.
