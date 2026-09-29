# 📊 @upstash/agent-analytics 완전 분석 정리

> 이 문서는 `@upstash/agent-analytics` 저장소를 전수조사하고 분석한 내용을 정리한 것입니다.
> 코드 구조 분석부터 사용법, 수익화 전략까지 전부 담았습니다.

- **작성일:** 2026-09-29
- **분석 대상 버전:** `0.1.3`
- **분석 브랜치:** `claude/youthful-goldberg-rfu98e`

---

## 🔗 GitHub 저장소 주소

| 구분 | 주소 |
|---|---|
| 🍴 **이 저장소 (포크본)** | https://github.com/bmshin94/agent-analytics |
| ⭐ **원본 저장소 (Upstash 공식)** | https://github.com/upstash/agent-analytics |
| 🐛 이슈 트래커 | https://github.com/upstash/agent-analytics/issues |
| 📦 npm 패키지 | https://www.npmjs.com/package/@upstash/agent-analytics |
| 🖥️ Upstash 콘솔 | https://console.upstash.com |

### 관련 링크

- Upstash Redis 문서: https://upstash.com/docs/redis
- Upstash Redis SDK: https://github.com/upstash/redis-js
- Next.js Middleware 문서: https://nextjs.org/docs/app/building-your-application/routing/middleware
- Model Context Protocol (MCP): https://modelcontextprotocol.io

---

## 📑 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [폴더 및 파일 구조](#2-폴더-및-파일-구조)
3. [핵심 동작 원리](#3-핵심-동작-원리)
4. [쉬운 비유로 이해하기](#4-쉬운-비유로-이해하기)
5. [설치 및 사용법](#5-설치-및-사용법)
6. [플러그인 / 스킬 / MCP 구분](#6-플러그인--스킬--mcp-구분)
7. [API 토큰 관련](#7-api-토큰-관련)
8. [왜 주목받는가](#8-왜-주목받는가)
9. [로컬 에이전트 구축에 주는 도움](#9-로컬-에이전트-구축에-주는-도움)
10. [React / PHP 구현 가능성](#10-react--php-구현-가능성)
11. [수익화 아이디어 10선](#11-수익화-아이디어-10선)
12. [한계점 및 개선 여지](#12-한계점-및-개선-여지)
13. [추천 로드맵](#13-추천-로드맵)

---

## 1. 프로젝트 개요

### 한 줄 정의

> **ChatGPT, Claude, Perplexity 같은 AI 에이전트/크롤러가 내 웹사이트를 방문(인용)한 횟수를 Upstash Redis에 기록하고 집계하는 초경량 TypeScript SDK**

### 기본 정보

| 항목 | 내용 |
|---|---|
| 패키지명 | `@upstash/agent-analytics` |
| 버전 | `0.1.3` (초기 단계, breaking change 이력 있음) |
| 라이선스 | MIT (상업적 이용 / 수정 / 재배포 가능) |
| 언어 | TypeScript 100% |
| 런타임 | Bun (개발/테스트), 배포는 ESM + CJS 듀얼 |
| 빌드 도구 | tsup |
| 코드 규모 | 소스 7개 파일 약 500줄 + 테스트 437줄 |
| 필수 의존성 | `@upstash/redis ^1.38.0` (peerDependency) |

### 해결하는 문제 — GEO 시대의 측정 공백

```
2010년대  : SEO 시대   → "구글 검색 1페이지에 올라가자"
2023년~   : AI 등장    → 사람들이 검색 대신 ChatGPT에게 질문
2025년~   : GEO 시대   → "AI가 우리를 인용하게 만들자"
                         ↑ 그런데 측정할 도구가 없다!
```

**핵심 문제:** AI 크롤러는 **JavaScript를 실행하지 않는다.**
→ Google Analytics 같은 클라이언트 스크립트 기반 도구로는 **AI 봇 방문이 전혀 잡히지 않는다.**

**해결:** 서버 미들웨어에서 HTTP 헤더를 직접 검사해 AI 봇을 식별하고 카운트.

관련 용어:
- **GEO** (Generative Engine Optimization) — 생성형 엔진 최적화
- **AEO** (Answer Engine Optimization) — 답변 엔진 최적화

---

## 2. 폴더 및 파일 구조

```
agent-analytics/
├── CLAUDE.md                 # 프로젝트 커스텀 지침 (페르소나 설정)
├── README.md                 # 설치 안내 (25줄, 쿼리 API 설명 없음)
├── LICENSE                   # MIT
├── package.json              # 패키지 메타 + 빌드 스크립트
├── tsconfig.json             # strict 모드 TS 설정
├── bun.lock                  # 의존성 잠금 파일
├── .gitignore                # .env*, dist, node_modules 등 제외
├── docs/
│   └── AGENT-ANALYTICS-분석-정리.md   # 👈 이 문서
├── public/
│   └── agentanalytics.png    # Upstash 콘솔 "AI Tracking" 대시보드 스크린샷
└── src/
    ├── index.ts              # (14줄)  공개 API 진입점
    ├── types.ts              # (90줄)  타입 정의
    ├── analytics.ts          # (322줄) 핵심 엔진 (읽기/쓰기)
    ├── request.ts            # (54줄)  AI 봇 탐지
    ├── script.ts             # (29줄)  Redis Lua 스크립트
    ├── hash.ts               # (63줄)  결정론적 해시 키 생성
    ├── time.ts               # (28줄)  시간 버킷 계산
    ├── duration.ts           # (30줄)  "28d" 같은 기간 문자열 파싱
    ├── analytics.test.ts     # (332줄) 통합 테스트
    ├── hash.test.ts          # (48줄)
    ├── time.test.ts          # (36줄)
    └── duration.test.ts      # (21줄)
```

### 파일별 역할

| 파일 | 역할 |
|---|---|
| `index.ts` | `AgentAnalytics`, `IndexNotFoundError` 클래스 + 전체 타입 export |
| `types.ts` | `Provider`, `TrackedEvent`, `AgentAnalyticsConfig`, `TimeRange` 등 정의 |
| `request.ts` | `detectProvider()` — User-Agent / Referer 헤더로 AI 봇 판별 |
| `hash.ts` | `dimensionPairs()`, `canonicalize()`, `dataHash()` — cyrb53 기반 키 생성 |
| `time.ts` | `dateToHourInt()`, `hourIntToDate()` — 1시간 단위 버킷 변환 |
| `duration.ts` | `parseDuration()` — `"28d"`, `"24h"`, `"90m"`, `"3600s"` → 초 |
| `script.ts` | `INGEST_SCRIPT` — 원자적 카운터 증가 Lua 스크립트 |
| `analytics.ts` | `AgentAnalytics` (쓰기) + `AnalyticsQuery` (읽기) |

### 대시보드 스크린샷 (`public/agentanalytics.png`)

Upstash 콘솔의 **AI Tracking** 화면:

- 상단: Database 선택 + Prefix 입력 (`@upstash/agent-analytics`)
- 필터: `Last 7 days`
- 통계 카드: ChatGPT `36,601` (+29.6%) / Perplexity `15,551` (+28.2%) / Claude `10,576` (+17.7%) / Other `8,915`
- 하단: **Requests Over Time** 프로바이더별 누적 막대 차트

> ⚠️ **중요:** 이 대시보드 UI는 **저장소에 포함되어 있지 않다.** Upstash 콘솔에 내장된 기능이며,
> 이 SDK는 **데이터 수집만** 담당한다. → 이 공백이 수익화의 핵심 기회다.

---

## 3. 핵심 동작 원리

### 3.1 전체 데이터 흐름

```
  AI 봇이 사이트 방문
         │
         ▼
 ┌────────────────────┐
 │ proxy.ts (미들웨어)  │
 │ analytics.track()   │
 └─────────┬──────────┘
           ▼
   detectProvider()  ─── 미확인 봇 ──→ null 반환 (기록 안 함)
           │ 알려진 AI
           ▼
   { provider: "claude", path: "/blog" }
           │
           ▼
   키 이름순 정렬 → URL 인코딩 → cyrb53 해시
           │
           ▼
   키: <prefix>:event:<해시>:<hourInt>
           │
           ▼
   ┌──────────────────────┐
   │ Lua EVAL (왕복 1회)   │
   │ HINCRBY count +1     │
   │ 최초 1회만 HSET+EXPIRE│
   └──────────┬───────────┘
              ▼
        Upstash Redis
              │
       (Redis Search 비동기 인덱싱)
              │
              ▼
   query.aggregateBy() / query.timeseries()
              │
              ▼
        대시보드 / 차트
```

### 3.2 AI 봇 탐지 (`request.ts`)

User-Agent와 Referer를 소문자로 합쳐 키워드 매칭한다.

```typescript
const source = `${userAgent} ${referer}`;

if (source.includes("chatgpt")   || source.includes("openai"))          return "chatgpt";
if (source.includes("claude")    || source.includes("anthropic"))       return "claude";
if (source.includes("perplexity"))                                      return "perplexity";
if (source.includes("gemini")    || source.includes("google-extended")) return "gemini";
if (source.includes("copilot")   || source.includes("bing"))            return "copilot";

return undefined;   // 미확인 → 아예 기록하지 않음
```

**지원 프로바이더 (5종):** `chatgpt` | `claude` | `perplexity` | `gemini` | `copilot`

**설계 결정:**
- 커밋 `d3b413d`에서 `"other"` 통합 카테고리를 **의도적으로 제거** (breaking change)
- URL 프래그먼트(`#섹션`)를 제거해 `/a#x`와 `/a#y`가 따로 집계되지 않도록 정규화
- **IP, 원본 User-Agent 문자열은 저장하지 않음** → PII / 고카디널리티 회피 (GDPR 친화적)

### 3.3 키 설계 (`hash.ts` + `time.ts`)

```
<prefix>:event:<data-hash>:<hourInt>
   │        │        │          │
   │        │        │          └─ 에포크 이후 흐른 시간 수 (Math.floor(ms / 3600000))
   │        │        └─ cyrb53(정렬된 차원 문자열).toString(36)
   │        └─ 고정 문자열
   └─ 기본값 "@upstash/agent-analytics"
```

**핵심 아이디어 1 — 로그가 아닌 카운터**

```
❌ 로그 방식:  방문 1건 = 레코드 1줄  → 100만 방문 = 100만 줄 (용량 폭발)
✅ 카운터 방식: 방문 100만건 = 해시 1개, count = 1000000 (용량 극소)
```

**핵심 아이디어 2 — 정렬을 통한 순서 독립성**

```
{ provider: "claude", path: "/blog" }  →  정렬  →  "path=%2Fblog&provider=claude"
{ path: "/blog", provider: "claude" }  →  정렬  →  "path=%2Fblog&provider=claude"
                                                    ↑ 동일한 해시
```

정렬하지 않으면 같은 논리적 이벤트가 두 개의 서로 다른 키로 갈라져 통계가 손상된다.

**핵심 아이디어 3 — 카디널리티 제어**

```typescript
// types.ts 주석 원문
/**
 * This is intentionally a small, pruned set: every distinct combination
 * of these values becomes its own counter (one hash key per hour), so we
 * only keep fields that are useful to group and filter analytics by.
 * Time is **not** a dimension — it lives in the hour bucket of the key.
 */
```

차원이 늘어날수록 키 개수가 곱셈으로 증가한다.
→ **"무엇을 기록하지 않을지 정하는 것"이 시계열 설계의 핵심.**

### 3.4 Lua 스크립트 (`script.ts`) — 가장 잘 설계된 부분

```lua
local count = redis.call('HINCRBY', KEYS[1], 'count', 1)
if count == 1 then
  local fields = {'hour', ARGV[2]}
  for i = 3, #ARGV do
    fields[#fields + 1] = ARGV[i]
  end
  redis.call('HSET', KEYS[1], unpack(fields))
  redis.call('EXPIRE', KEYS[1], tonumber(ARGV[1]))
end
return count
```

**ARGV 레이아웃:**
```
ARGV[1]  = TTL (초)
ARGV[2]  = hourInt
ARGV[3~] = key1, value1, key2, value2, ...
```

**네 가지 이점:**

| 이점 | 설명 |
|---|---|
| ⚛️ **원자성** | Redis가 스크립트를 분할 불가능하게 실행. 서버 N대 동시 요청에도 카운트 정확 |
| 🚀 **왕복 1회** | 읽기+쓰기+TTL을 1 round-trip으로. 엣지 미들웨어 응답 속도에 직결 |
| 🧠 **조건부 메타데이터 쓰기** | `HINCRBY`가 1을 반환 = 최초 생성. 이때만 불변 메타데이터를 기록 |
| 💰 **비용 절감** | `track()` 1회 = Redis 커맨드 **1개**. (분리했다면 3개 → 3배 비용) |

### 3.5 읽기 API (`analytics.ts` → `AnalyticsQuery`)

Redis Search 인덱스 기반.

```typescript
const EVENT_SCHEMA = {
  count:    { type: "U64", fast: true },   // 범위 필터 + 합산
  hour:     { type: "U64", fast: true },   // 시간 범위 필터 + 히스토그램
  provider: { type: "KEYWORD" },           // 정확값 그룹핑
  path:     { type: "KEYWORD" },
};
```

| 메서드 | 설명 | 호출 시점 |
|---|---|---|
| `getIndex()` | 검색 인덱스 생성 (idempotent) | **setup 시 1회만** |
| `aggregateBy({ field, since, until })` | 차원별 합계 | 읽기 경로 |
| `timeseries({ since, until, groupBy })` | 시간별 시계열 (갭필링 포함) | 읽기 경로 |
| `waitIndexing()` | 인덱싱 완료 대기 | 테스트 / read-after-write |
| `dropIndex()` | 인덱스 삭제 | 정리 |

**고급 설계 3가지:**

1. **스키마 자동 조정 (reconciliation)**
   `existsOk: true`는 인덱스가 이미 있으면 no-op이지만 **스키마 차이를 반영하지 않는다.**
   그래서 `describe()`로 기존 스키마를 확인하고, 필드 집합/타입이 다르면 **drop 후 재생성**한다.

2. **에러 경로 최적화**
   ```typescript
   private async run<T>(query: () => Promise<T>): Promise<T> {
     try {
       return await query();
     } catch (error) {
       if ((await this.index().describe()) === null) {
         throw new IndexNotFoundError(this.indexName);
       }
       throw error;
     }
   }
   ```
   인덱스 존재 확인을 **에러가 났을 때만** 수행 → 정상 경로는 요청 1회 유지.

3. **갭필링 + 균일 키셋**
   `timeseries()`는 데이터가 없는 시간대도 `0`으로 채우고, 모든 버킷이 **동일한 키 집합**을 갖도록 맞춘다.
   → Recharts / Chart.js에 변환 없이 바로 투입 가능.

### 3.6 공개 API 전체

```typescript
// 생성
new AgentAnalytics({ redis, prefix?, retention?, indexName? })
AgentAnalytics.fromEnv({ prefix?, retention?, indexName? })

// 쓰기 (오버로드 2개)
analytics.track(request: Request): Promise<number | null>
analytics.track(event: TrackedEvent, time?: Date): Promise<number>

// 읽기
analytics.query.getIndex(): Promise<Index>
analytics.query.aggregateBy({ field, since, until? }): Promise<Record<string, number>>
analytics.query.timeseries({ since, until?, groupBy? }): Promise<TimeseriesBucket[]>
analytics.query.waitIndexing(): Promise<void>
analytics.query.dropIndex(): Promise<void>
```

**기본 설정값:**

| 옵션 | 기본값 |
|---|---|
| `prefix` | `"@upstash/agent-analytics"` |
| `retention` | `"28d"` (28일) |
| `indexName` | `prefix`에서 특수문자를 `_`로 치환 후 `-events` 접미사 |

---

## 4. 쉬운 비유로 이해하기

### 편의점 로봇 출입 기록부

오빠가 편의점 사장님이고, 요즘 **사람 손님뿐 아니라 배달 플랫폼 로봇**도 상품을 스캔하러 온다고 상상하자.

- 로봇이 상품을 스캔해 가면, 나중에 앱 사용자가 "라면 어디가 싸?"라고 물었을 때 **내 가게가 추천된다.**
- 그래서 **어떤 로봇이 몇 번 왔는지** 알아야 한다.
- 그런데 기존 CCTV(= Google Analytics)는 **사람 얼굴만 인식**한다. 로봇은 안 잡힌다.

**이 SDK = 로봇 전용 출입 기록부**

| 단계 | 비유 | 실제 코드 |
|---|---|---|
| 1 | 문 앞에 경비원 배치 | `analytics.track(request)` |
| 2 | 명찰 확인 (모르면 통과만 시킴) | `detectProvider()` |
| 3 | [누가+어디를+몇시에] 서랍에 숫자만 +1 | Lua `HINCRBY` |
| 4 | 28일 지나면 서랍 자동 폐기 | `EXPIRE` (TTL) |

### 서랍 구조 예시

```
📦 "Claude  + /블로그 + 14시"  → 5
📦 "Claude  + /블로그 + 15시"  → 2
📦 "ChatGPT + /블로그 + 14시"  → 12
📦 "ChatGPT + /가격   + 14시"  → 3
```

방문이 100만 번이어도 **서랍 하나의 숫자만 커진다.** 이것이 비용 절약의 핵심이다.

### Lua = "한 호흡 마법 주문"

마법 없이 하면:

```
서버A: 서랍 읽음 → "5개"
서버B: 서랍 읽음 → "5개"     ← 동시에 읽음
서버A: 6개로 저장
서버B: 6개로 저장            ← 7이어야 하는데 6. 데이터 손실!
```

Lua 스크립트는 Redis 서버 내부에서 **쪼갤 수 없게** 실행되므로 이 문제가 원천 차단된다.

### 주의사항 3가지

| 주의 | 이유 |
|---|---|
| `getIndex()`는 **setup에서 1회만** | 매 요청마다 부르면 불필요한 네트워크 왕복 발생 |
| 인덱싱은 **비동기** | 방금 `track()`한 게 즉시 쿼리에 안 나올 수 있음 → `waitIndexing()` |
| 응답을 막지 말 것 | `waitUntil(analytics.track(request))` 로 백그라운드 처리 |

---

## 5. 설치 및 사용법

### STEP 0 — Upstash Redis 준비 (필수)

1. https://console.upstash.com 가입
2. **Create Database**
3. Region 선택 (한국이면 `ap-northeast-1` 도쿄 권장)
4. Type: Regional (저비용) 또는 Global (전역 저지연)
5. **REST API** 탭에서 URL / TOKEN 복사

> 무료 티어: 일 10,000 커맨드 / 256MB
> ⚠️ Redis Search 기능이 지원되는 리전/플랜인지 확인 필요

### STEP 1 — 설치

```bash
npm install @upstash/agent-analytics @upstash/redis
# 또는
pnpm add @upstash/agent-analytics @upstash/redis
bun add  @upstash/agent-analytics @upstash/redis
```

> `@upstash/redis`는 **peerDependency**이므로 반드시 별도 설치해야 한다.
> (커밋 `3cb5f8d`에서 버전 충돌 방지를 위해 의도적으로 분리)

### STEP 2 — 환경변수

```bash
# .env.local
UPSTASH_REDIS_REST_URL=https://xxxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=AbCdEf...
```

### STEP 3 — Redis 클라이언트

```typescript
// redis.ts
import { Redis } from "@upstash/redis";
export const redis = Redis.fromEnv();
```

### STEP 4 — 미들웨어에 추적 삽입

```typescript
// proxy.ts  (Next.js 최신) / middleware.ts (구버전)
import { NextResponse, type NextRequest } from "next/server";
import { AgentAnalytics } from "@upstash/agent-analytics";
import { redis } from "./redis";

const analytics = new AgentAnalytics({ redis });

export const proxy = async (request: NextRequest) => {
  await analytics.track(request);
  return NextResponse.next();
};

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

**권장: 응답을 막지 않는 버전**

```typescript
import { waitUntil } from "@vercel/functions";

export const proxy = (request: NextRequest) => {
  waitUntil(analytics.track(request));   // 백그라운드 처리
  return NextResponse.next();
};
```

**옵션 커스터마이징**

```typescript
const analytics = new AgentAnalytics({
  redis,
  prefix:    "myapp:ai",   // 기본 "@upstash/agent-analytics"
  retention: "90d",        // 기본 "28d"
  indexName: "my-index",   // 기본 prefix 기반 자동 생성
});

// 환경변수 자동 로드 버전
const analytics = AgentAnalytics.fromEnv({ retention: "90d" });
```

### STEP 5 — 인덱스 생성 (1회)

```typescript
// scripts/setup.ts
import { AgentAnalytics } from "@upstash/agent-analytics";
import { redis } from "../redis";

await new AgentAnalytics({ redis }).query.getIndex();
console.log("인덱스 준비 완료");
```

```bash
npx tsx scripts/setup.ts
```

### STEP 6 — 데이터 조회

**방법 A — Upstash 콘솔**
콘솔 → DB 선택 → 우상단 `⋯` → **AI Tracking** → Prefix 입력

**방법 B — 자체 API (커스텀 대시보드용)**

```typescript
// app/api/ai-stats/route.ts
import { AgentAnalytics } from "@upstash/agent-analytics";
import { redis } from "@/redis";

const analytics = new AgentAnalytics({ redis });

export async function GET() {
  const since = new Date(Date.now() - 7 * 24 * 3600_000);

  const [byProvider, byPath, series] = await Promise.all([
    analytics.query.aggregateBy({ field: "provider", since }),
    analytics.query.aggregateBy({ field: "path", since }),
    analytics.query.timeseries({ since, groupBy: "provider" }),
  ]);

  return Response.json({ byProvider, byPath, series });
}
```

### 개발용 테스트

```bash
curl -H "User-Agent: PerplexityBot/1.0" http://localhost:3000/blog
curl -H "User-Agent: ChatGPT-User/1.0"  http://localhost:3000/pricing
curl -H "User-Agent: Claude-Web/1.0"    http://localhost:3000/docs
```

### 소스 직접 빌드

```bash
git clone https://github.com/bmshin94/agent-analytics
cd agent-analytics
bun install
bun run typecheck   # 타입 체크
bun run build       # dist/ 생성 (ESM + CJS + .d.ts)
bun test            # ⚠️ 실제 Upstash 인스턴스 필요
```

---

## 6. 플러그인 / 스킬 / MCP 구분

### 결론: 셋 다 아니다. **평범한 npm 라이브러리(SDK)** 다.

| 구분 | 정체 | 설치 위치 | 소비 주체 | 해당 여부 |
|---|---|---|---|---|
| **npm SDK** | 코드에 import하는 라이브러리 | `package.json` | 내 애플리케이션 코드 | ✅ **이것** |
| **MCP 서버** | AI가 도구로 호출하는 서버 | `.mcp.json` 등 | Claude / Cursor 등 | ❌ |
| **Claude Skill** | `SKILL.md` + 스크립트 묶음 | `.claude/skills/` | Claude Code | ❌ |
| **플러그인** | 특정 앱의 확장 | 해당 앱 설정 | 그 앱 | ❌ |

### 근거

```json
// package.json — 표준 라이브러리 구조
{
  "main":    "./dist/index.cjs",
  "module":  "./dist/index.js",
  "types":   "./dist/index.d.ts",
  "exports": { ... }
}
```

- `mcpServers` 설정 없음
- `SKILL.md` 파일 없음
- `bin` 필드 없음 (CLI 아님)
- `@modelcontextprotocol/sdk` 의존성 없음

### 이름 때문에 생기는 혼동

```
"agent-analytics"
     │          │
  AI 에이전트를   분석하는 라이브러리
  (추적 대상)

❌ 에이전트가 사용하는 분석 도구
✅ 에이전트를 분석하는 도구
```

### 다만, MCP 서버로 감쌀 수는 있다

```typescript
server.tool("get_ai_citations", { days: z.number() }, async ({ days }) => {
  const data = await analytics.query.aggregateBy({
    field: "provider",
    since: new Date(Date.now() - days * 86400_000),
  });
  return { content: [{ type: "text", text: JSON.stringify(data) }] };
});
```

→ Claude에게 "이번 주 우리 사이트 AI 인용 어때?"라고 물어볼 수 있게 된다. (수익화 아이디어 #3)

---

## 7. API 토큰 관련

### 결론: **Upstash Redis 토큰만 필요.** AI 회사 토큰은 전혀 필요 없다.

| 토큰 | 필요 여부 | 이유 |
|---|---|---|
| **Upstash Redis URL / TOKEN** | ✅ 필수 | 데이터 저장 및 조회 |
| OpenAI API 키 | ❌ 불필요 | OpenAI 서버에 요청하지 않음 |
| Anthropic API 키 | ❌ 불필요 | Claude API 호출 없음 |
| Perplexity / Google 키 | ❌ 불필요 | 동일 |

### 왜 AI 회사 토큰이 필요 없는가

```
❌ 이 SDK가 하는 일이 아님:
   내 서버 ──API 호출──> OpenAI "우리 사이트 인용했어?"

✅ 실제 동작:
   AI 봇 ──방문──> 내 서버 ──"ChatGPT군요"──> Redis에 +1
```

AI 봇이 **제 발로 내 서버에 오는 것**을 받아 적는 구조라서 외부 API 호출이 전혀 없다.

### 보안 체크리스트

- 🚫 `NEXT_PUBLIC_` 접두사 **절대 금지** (클라이언트 노출 시 DB 탈취 위험)
- ✅ `.gitignore`에 `.env*` 포함 확인 (이 저장소는 이미 설정됨)
- ✅ Vercel / Netlify는 대시보드의 Environment Variables에 등록
- 💡 Upstash는 **읽기 전용 토큰** 발급 가능 → 대시보드 전용 서버에 사용
- ⚠️ `track()`은 쓰기가 필요하므로 미들웨어 측은 쓰기 토큰 필수

### 비용

| 티어 | 가격 | 한도 |
|---|---|---|
| Free | $0 | 일 10,000 커맨드 / 256MB |
| Pay-as-you-go | 약 $0.2 / 10만 커맨드 | 무제한 |

- `track()` 1회 = Redis 커맨드 **1개** (Lua로 묶은 덕분)
- 하루 AI 봇 방문 1,000회 = 무료 티어의 10%만 사용
- Lua 없이 구현했다면 커맨드 3배 → **최적화가 곧 비용 절감**

---

## 8. 왜 주목받는가

> ⚠️ 이 분석은 코드와 시장 맥락에 근거한 **구조적 이유**다.
> 원본 저장소의 실제 스타 수치는 이 세션의 접근 권한 밖이라 검증하지 않았다.

### ① 타이밍 — 측정 공백의 시대

```
Gartner 예측: 2026년까지 검색 트래픽 25% 감소
사람들이 구글 대신 ChatGPT에게 질문
→ 그런데 AI 봇은 JS를 실행하지 않아 GA로는 안 잡힘
→ 시장에 뚫린 구멍 + 딱 맞는 도구 = 폭발적 관심
```

### ② Upstash 브랜드 신뢰

- `@upstash/redis` — 서버리스 Redis 사실상 표준
- `@upstash/ratelimit` — 레이트리밋 표준
- `@upstash/qstash` — 메시지 큐
- Vercel 공식 파트너

→ "Upstash가 만들었으면 일단 믿고 본다"는 축적된 신뢰.

### ③ 코드가 교과서급

500줄 남짓에 담긴 프로덕션 패턴:

- Lua 기반 원자적 연산 (Redis 고급 패턴)
- TTL 기반 자동 데이터 정리 (비용 설계)
- 해시 기반 카디널리티 제어 (시계열 DB 설계 원리)
- Read / Write 책임 분리 (CQRS 축소판)
- 네트워크 round-trip 최소화 (엣지 최적화)
- 소스보다 많은 테스트 (437줄)

### ④ 진입장벽이 거의 없음

```typescript
await analytics.track(request);   // 이 한 줄이 전부
```

설치 5분, 코드 1줄, 무료 시작, 프론트엔드 수정 0.

### ⑤ 엣지 런타임 네이티브

Next.js 미들웨어는 Edge Runtime이라 제약이 많지만, 이 SDK는 HTTP REST 기반 Redis를 쓰므로 완벽히 동작한다.

### ⑥ 스크린샷 한 장의 설득력

README 최상단의 대시보드 이미지(`ChatGPT 36,601` 등)가
**"나도 이 숫자를 보고 싶다"**는 욕구를 즉시 유발한다.

---

## 9. 로컬 에이전트 구축에 주는 도움

### 결론: **직접적 도움 ✗ / 간접적 도움 ★★★★★**

### 직접 도움이 안 되는 부분

이것은 **AI를 만드는 도구가 아니라 AI를 측정하는 도구**다.

```
❌ LLM 호출        ❌ 프롬프트 관리     ❌ 벡터 DB / RAG
❌ Tool calling    ❌ 에이전트 루프     ❌ MCP 서버
```

### 간접 도움 5가지

#### ① 에이전트 텔레메트리 청사진 (가장 큼)

로컬 에이전트를 만들면 반드시 이런 질문이 생긴다:
"어떤 도구를 제일 많이 호출했나? 토큰을 얼마나 썼나? 어떤 작업이 자주 실패하나?"

**이 SDK의 구조를 거의 그대로 전용할 수 있다.**

```typescript
// 원본
type TrackedEvent = { provider: Provider; path: string };

// 에이전트용으로 변형
type AgentEvent = {
  agent:  string;   // "researcher" | "coder" | "reviewer"
  tool:   string;   // "read_file" | "web_search" | "bash"
  status: string;   // "success" | "error" | "timeout"
  model:  string;   // "opus" | "sonnet" | "haiku"
};

// 키 전략 그대로: agent-metrics:event:<해시>:<hourInt>
```

`analytics.ts` / `hash.ts` / `script.ts` / `time.ts` / `duration.ts` 는 거의 수정 없이 재사용 가능하다.

#### ② 카디널리티 제어 = 에이전트 설계의 핵심 교훈

에이전트를 만들다 보면 모든 것을 기록하고 싶어지지만, 그러면 스토리지 폭발 / 쿼리 지연 / 신호 실종이 온다.
**"무엇을 기록하지 않을지 정하는 게 더 중요하다"** — 이 코드의 가장 큰 교훈.

#### ③ 레이트리밋 / 쿼터 시스템 원형

```lua
local used = redis.call('HINCRBY', KEYS[1], 'tokens', ARGV[1])
if used == tonumber(ARGV[1]) then
  redis.call('EXPIRE', KEYS[1], 86400)   -- 일일 리셋
end
if used > tonumber(ARGV[2]) then return -1 end   -- 쿼터 초과
return used
```

#### ④ MCP 서버로 감싸면 "자기 관측 에이전트"

```typescript
server.tool("agent_stats", async ({ days }) =>
  analytics.query.aggregateBy({ field: "tool", since: ... })
);
```

→ Claude에게 "내가 만든 에이전트 이번 주 어때?"라고 물으면 스스로 분석한다.

#### ⑤ 엣지 친화 I/O 설계 학습

"왕복 3번 → 1번"이라는 사고방식은 로컬 에이전트의 I/O 설계에도 그대로 적용된다.

### 추천 스택 조합

```
🧠 에이전트 로직    → Claude Agent SDK / LangGraph
🔌 도구 연결        → MCP 서버
📊 관측/텔레메트리  → 이 SDK 구조 응용  ⭐
💾 상태 저장        → Redis (Upstash 또는 로컬)
🎨 대시보드         → Next.js + Recharts
```

> 이 SDK는 "에이전트의 두뇌"가 아니라 **"에이전트의 계기판"** 설계도다.

---

## 10. React / PHP 구현 가능성

### 10.1 React — 대시보드는 최적, 추적은 불가

#### ❌ React로 추적은 불가능

```jsx
// 작동하지 않음
useEffect(() => {
  analytics.track();   // AI 봇은 JavaScript를 실행하지 않는다
}, []);
```

AI 크롤러는 HTML만 받아가고 JS를 실행하지 않는다.
→ 추적은 **반드시 서버사이드(미들웨어)** 에서 해야 한다.
(이것이 GA가 AI 봇을 못 잡는 이유와 동일하다.)

#### ✅ React로 대시보드는 완벽하게 가능

```tsx
// app/api/ai-stats/route.ts (서버)
export async function GET() {
  const since = new Date(Date.now() - 7 * 864e5);
  return Response.json({
    byProvider: await analytics.query.aggregateBy({ field: "provider", since }),
    series:     await analytics.query.timeseries({ since, groupBy: "provider" }),
  });
}
```

```tsx
// components/AIDashboard.tsx (클라이언트)
"use client";
import useSWR from "swr";
import { BarChart, Bar, XAxis, YAxis, Tooltip, Legend } from "recharts";

export function AIDashboard() {
  const { data } = useSWR("/api/ai-stats", (u) => fetch(u).then((r) => r.json()));
  if (!data) return <div>로딩중...</div>;

  // timeseries가 갭필링 + 균일 키셋이라 변환이 단순하다
  const chartData = data.series.map((b) => ({
    time: new Date(b.time).toLocaleTimeString("ko-KR", { hour: "2-digit" }),
    ...b.values,
  }));

  return (
    <>
      <div className="grid grid-cols-4 gap-4">
        {Object.entries(data.byProvider).map(([name, count]) => (
          <div key={name} className="rounded-xl border p-4">
            <p className="text-sm text-gray-500">{name}</p>
            <p className="text-3xl font-bold">{count.toLocaleString()}</p>
          </div>
        ))}
      </div>

      <BarChart width={800} height={300} data={chartData}>
        <XAxis dataKey="time" /><YAxis /><Tooltip /><Legend />
        <Bar dataKey="chatgpt"    stackId="a" fill="#111111" />
        <Bar dataKey="claude"     stackId="a" fill="#E8734A" />
        <Bar dataKey="perplexity" stackId="a" fill="#20808D" />
      </BarChart>
    </>
  );
}
```

#### 다른 프레임워크

```typescript
// Remix - entry.server.tsx
export default function handleRequest(request: Request, ...) {
  analytics.track(request);
  // ...
}

// Astro - middleware.ts
export const onRequest = async (context, next) => {
  await analytics.track(context.request);
  return next();
};
```

### 10.2 PHP — 가능하지만 직접 포팅 필요

> npm 패키지 자체는 PHP에서 사용 불가. **로직 포팅은 100% 가능** (MIT 라이선스).

포팅 대상:
1. User-Agent 탐지 로직
2. cyrb53 해시 (⚠️ PHP는 64bit 정수라 JS의 32bit 연산 재현 필요)
3. Lua 스크립트 (그대로 복사 가능)
4. Upstash REST API 호출

```php
<?php
// AgentAnalytics.php

class AgentAnalytics {
    private string $url;
    private string $token;
    private string $prefix;
    private int $ttl;

    const INGEST_SCRIPT = <<<'LUA'
local count = redis.call('HINCRBY', KEYS[1], 'count', 1)
if count == 1 then
  local fields = {'hour', ARGV[2]}
  for i = 3, #ARGV do fields[#fields + 1] = ARGV[i] end
  redis.call('HSET', KEYS[1], unpack(fields))
  redis.call('EXPIRE', KEYS[1], tonumber(ARGV[1]))
end
return count
LUA;

    public function __construct(string $prefix = 'php-agent-analytics', int $ttl = 2419200) {
        $this->url    = getenv('UPSTASH_REDIS_REST_URL');
        $this->token  = getenv('UPSTASH_REDIS_REST_TOKEN');
        $this->prefix = $prefix;
        $this->ttl    = $ttl;
    }

    public function detectProvider(): ?string {
        $ua  = strtolower($_SERVER['HTTP_USER_AGENT'] ?? '');
        $ref = strtolower($_SERVER['HTTP_REFERER'] ?? '');
        $s   = $ua . ' ' . $ref;

        if (str_contains($s,'chatgpt')    || str_contains($s,'openai'))          return 'chatgpt';
        if (str_contains($s,'claude')     || str_contains($s,'anthropic'))       return 'claude';
        if (str_contains($s,'perplexity'))                                        return 'perplexity';
        if (str_contains($s,'gemini')     || str_contains($s,'google-extended')) return 'gemini';
        if (str_contains($s,'copilot')    || str_contains($s,'bing'))            return 'copilot';
        return null;
    }

    /** JS Math.imul 재현 (32bit 곱셈) */
    private function imul(int $a, int $b): int {
        $a &= 0xFFFFFFFF; $b &= 0xFFFFFFFF;
        $ah = ($a >> 16) & 0xffff; $al = $a & 0xffff;
        $bh = ($b >> 16) & 0xffff; $bl = $b & 0xffff;
        return (($al * $bl) + ((($ah * $bl + $al * $bh) << 16) & 0xFFFFFFFF)) & 0xFFFFFFFF;
    }

    private function cyrb53(string $str): string {
        $h1 = 0xdeadbeef; $h2 = 0x41c6ce57; $M = 0xFFFFFFFF;
        for ($i = 0; $i < strlen($str); $i++) {
            $ch = ord($str[$i]);
            $h1 = $this->imul($h1 ^ $ch, 2654435761);
            $h2 = $this->imul($h2 ^ $ch, 1597334677);
        }
        $h1 = $this->imul($h1 ^ (($h1 >> 16) & $M), 2246822507);
        $h1 ^= $this->imul($h2 ^ (($h2 >> 13) & $M), 3266489909);
        $h2 = $this->imul($h2 ^ (($h2 >> 16) & $M), 2246822507);
        $h2 ^= $this->imul($h1 ^ (($h1 >> 13) & $M), 3266489909);
        return base_convert((string)(4294967296 * (2097151 & $h2) + ($h1 & $M)), 10, 36);
    }

    public function track(): ?int {
        $provider = $this->detectProvider();
        if ($provider === null) return null;

        $scheme = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ? 'https' : 'http';
        $path   = $scheme . '://' . $_SERVER['HTTP_HOST'] . strtok($_SERVER['REQUEST_URI'], '#');

        $dims = ['path' => $path, 'provider' => $provider];
        ksort($dims);   // 반드시 키 이름순 정렬 (TS 버전과 해시 호환)

        $parts = [];
        foreach ($dims as $k => $v) $parts[] = $k . '=' . rawurlencode($v);
        $hash = $this->cyrb53(implode('&', $parts));

        $hour = intdiv(time(), 3600);
        $key  = "{$this->prefix}:event:{$hash}:{$hour}";

        $argv = [(string)$this->ttl, (string)$hour];
        foreach ($dims as $k => $v) { $argv[] = $k; $argv[] = $v; }

        return $this->evalScript($key, $argv);
    }

    private function evalScript(string $key, array $argv): ?int {
        $body = json_encode([self::INGEST_SCRIPT, [$key], $argv]);
        $ch = curl_init($this->url . '/eval');
        curl_setopt_array($ch, [
            CURLOPT_POST           => true,
            CURLOPT_POSTFIELDS     => $body,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => 2,   // 페이지 지연 방지
            CURLOPT_HTTPHEADER     => [
                'Authorization: Bearer ' . $this->token,
                'Content-Type: application/json',
            ],
        ]);
        $res = curl_exec($ch);
        curl_close($ch);
        return $res ? (json_decode($res, true)['result'] ?? null) : null;
    }
}
```

**WordPress 플러그인으로:**

```php
<?php
/*
Plugin Name: AI Citation Tracker
Description: ChatGPT, Claude, Perplexity 봇 방문을 추적합니다
Version: 1.0
*/
add_action('init', function () {
    if (is_admin()) return;
    require_once __DIR__ . '/AgentAnalytics.php';
    (new AgentAnalytics())->track();
});
```

**Laravel 미들웨어로:**

```php
class TrackAIAgents {
    public function handle($request, Closure $next) {
        dispatch(fn() => (new AgentAnalytics())->track())->afterResponse();
        return $next($request);
    }
}
```

#### PHP 포팅 시 함정

| 함정 | 해결 |
|---|---|
| 해시 불일치 | PHP는 64bit, JS는 32bit 연산 → `imul()` 재현 필수, 테스트로 검증 |
| 동기 blocking | `fastcgi_finish_request()` 또는 큐 사용 |
| 키 정렬 누락 | `ksort()` 빼먹으면 TS 버전과 다른 키 생성 |

### 10.3 언어/프레임워크 정리표

| 환경 | 추적 | 대시보드 | 방법 |
|---|---|---|---|
| Next.js | ✅ | ✅ | 패키지 그대로 |
| Remix / Astro / SvelteKit | ✅ | ✅ | 서버 훅에서 `track()` |
| Express / Hono / Fastify | ✅ | ✅ | 미들웨어 |
| React (CSR 전용) | ❌ | ✅ | 추적엔 서버 필요 |
| PHP / Laravel / WordPress | ✅ | ✅ | 직접 포팅 |
| Python / Django | ✅ | ✅ | 직접 포팅 |
| Go / Rust | ✅ | ✅ | 직접 포팅 |

---

## 11. 수익화 아이디어 10선

### 시장 구조 — "빈칸이 곧 기회"

```
Upstash가 제공: 데이터 수집 SDK + 자사 콘솔 대시보드 (무료, MIT)
                          ↓
아무도 안 채운 영역:
  ❌ 임베드 가능한 대시보드      ❌ 슬랙/이메일 알림
  ❌ 다중 사이트 통합 관리        ❌ 경쟁사 벤치마킹
  ❌ AI 인사이트 & 추천          ❌ WordPress/PHP 지원 (시장 40%)
  ❌ 봇 진위 검증 (IP 역방향)    ❌ 팀 협업 / 클라이언트 리포트
  ❌ 컨설팅 / 교육
```

**왜 Upstash가 직접 안 하는가:**
Upstash의 본업은 Redis 인프라 판매이고, 이 SDK는 리드 제너레이션 도구다.
GEO 분석 SaaS로 확장할 동기가 없다. 그리고 **MIT 라이선스**라 상업적 활용이 전부 합법이다.

---

### 티어 1 — 빠른 시작 (1~4주)

#### 💡 #1 임베드형 대시보드 컴포넌트 ★★★★★

가장 명백한 공백. 데이터는 쌓이는데 볼 곳이 Upstash 콘솔뿐이다.

```tsx
import { AIDashboard } from "@yourname/agent-analytics-ui";
<AIDashboard endpoint="/api/ai-stats" theme="dark" />
```

**구성:** `<ProviderCards />`, `<RequestsOverTime />`, `<TopPagesTable />`,
`<ProviderPieChart />`, `<DateRangePicker />`, `<ExportButton />`

**수익 모델**

| 방식 | 가격 |
|---|---|
| 오픈소스 + 유료 Pro | 무료 / $49 일회성 |
| 템플릿 판매 (Gumroad / LemonSqueezy) | $29~99 |
| shadcn 스타일 배포 | 무료 → 인지도 → 컨설팅 유입 (권장) |

**기술:** Next.js + Recharts(또는 Tremor) + shadcn/ui + Tailwind + SWR

**장점:** 1~2주 MVP, 리스크 제로, 포트폴리오 활용, #4 SaaS로 확장 가능

---

#### 💡 #2 WordPress 플러그인 ★★★★★ (최우선 추천)

```
전 세계 웹사이트의 약 40% = WordPress
→ 이 SDK는 PHP 지원이 전무하다
```

**무료 버전 (WordPress.org 등록)**
- 자동 AI 봇 추적 (설정 0)
- 관리자 대시보드 위젯 / 최근 7일 통계 / 5개 프로바이더

**Pro 버전 ($49/년)**
- 무제한 히스토리 / 주간 이메일 리포트 / Slack·Discord 알림
- 페이지별 상세 분석 / WooCommerce 연동 / Yoast·RankMath 연동
- 멀티사이트 / CSV·PDF 내보내기

**수익 시뮬레이션 (전환율 2% 가정)**
```
10,000 설치 × 2% = 200명  × $49 = 연 $9,800
50,000 설치 × 2% = 1,000명 × $49 = 연 $49,000
```

> 💡 Redis 없이 **WordPress DB 저장 버전**도 제공하면 진입장벽이 0이 된다.

**장점:** 경쟁자 사실상 부재, WordPress.org = 무료 마케팅 채널, 유료 전환 문화 존재, 수동적 수입

---

#### 💡 #3 MCP 서버 + Claude 연동 ★★★★

```json
{
  "mcpServers": {
    "ai-citations": {
      "command": "npx",
      "args": ["-y", "@yourname/ai-citations-mcp"],
      "env": { "UPSTASH_REDIS_REST_URL": "...", "UPSTASH_REDIS_REST_TOKEN": "..." }
    }
  }
}
```

**제공 도구:** `get_citations`, `top_cited_pages`, `citation_trend`,
`compare_providers`, `geo_recommendations`

**사용 경험**
```
질문: "이번 주 우리 블로그 AI 인용 어때?"
답변: "지난 7일 총 62,728건입니다.
      ChatGPT 36,601 (58%, +29.6%) / Perplexity 15,551 (25%, +28.2%) / Claude 10,576 (17%, +17.7%)
      최다 인용 페이지는 /blog/redis-tips (1,204건)인데,
      FAQ 형식 + 구조화 데이터가 잘 돼 있습니다.
      같은 패턴을 /blog/nextjs-guide에도 적용하길 권합니다."
```

**장점:** MCP 생태계 초기 = 선점 효과, 얇은 래퍼라 개발 빠름, 화제성 높음

---

### 티어 2 — 본격 사업 (1~6개월)

#### 💡 #4 GEO 분석 SaaS ★★★★★ (최대 잠재력)

**Phase 1 (1~2개월):** 원클릭 설치, 실시간 대시보드, 프로바이더/페이지별 통계, 주간 이메일
**Phase 2 (3~4개월):** 봇 진위 검증, 프로바이더 확장(Grok/DeepSeek/Meta AI), 경쟁사 벤치마킹, GEO 최적화 제안, 알림
**Phase 3 (5~6개월):** 팀/에이전시, 화이트라벨, API/Webhook, GA4·Search Console 연동, A/B 테스트

**가격 설계**
```
Free       $0/월    1 사이트, 7일, 월 1만 이벤트
Starter    $19/월   3 사이트, 90일, 월 10만, 이메일 리포트
Pro        $49/월   10 사이트, 1년, 월 100만, 알림 + 인사이트
Agency     $199/월  무제한, 화이트라벨, API, 우선 지원
Enterprise 협의     온프레미스, SSO, SLA
```

**수익 시뮬레이션**
```
6개월차:  50명  × $30 = 월  $1,500
1년차:   200명  × $35 = 월  $7,000
2년차:   800명  × $40 = 월 $32,000 (연 $384K)
```

**기술 스택:** Next.js 15 + Tailwind + shadcn/ui + Recharts / Upstash Redis / Clerk·Auth.js / Stripe·LemonSqueezy / Resend / Vercel

**리스크 대응**

| 리스크 | 대응 |
|---|---|
| 대기업 진입 (Ahrefs, Semrush) | 개발자·기술블로그 틈새 특화 |
| AI 봇 헤더 변경 | 탐지 규칙을 원격 설정으로 분리 |
| 스토리지 비용 | TTL 계층화 (원본 28일 → 일별 요약 1년) |
| 초기 고객 확보 | 공격적 무료 티어 + 오픈소스 SDK 유입 |

**핵심 차별화**
```
❌ "AI 봇이 몇 번 왔습니다"                          ← 누구나 함
✅ "이 글이 많이 인용되는 이유는 FAQ 구조 +          ← 진짜 가치
    Schema.org 마크업 때문입니다. 다른 글 5개에
    같은 패턴을 적용하면 인용이 약 40% 늘 것으로 예상됩니다."
```

→ **데이터 제공자가 아니라 인사이트 제공자가 되어야 수익이 난다.**

---

#### 💡 #5 크로스 플랫폼 SDK 패밀리 ★★★★

```
agent-analytics-py / -php / -go / -rb / -rs / -java / -dotnet
```

모두 **동일한 Redis 스키마**를 사용 → 하나의 대시보드로 통합 조회.

**수익원:** SDK는 전부 무료 오픈소스(생태계 장악) → SaaS(#4) 유입 / 기업 지원 계약($500~2,000/월) / 커스텀 통합 컨설팅 / GitHub Sponsors

**우선순위:** ① PHP (WordPress 40%) ② Python (AI·데이터 커뮤니티) ③ Go (인프라)

---

#### 💡 #6 GEO 컨설팅 + 리테이너 ★★★★ (자본 0, 최단 수익화)

**GEO 감사 (1회성, ₩150만~500만)**
현황 진단 / 경쟁사 대비 포지션 / 콘텐츠 구조 개선안 / robots.txt·llms.txt 최적화 / Schema.org 설계 / 실행 로드맵

**월간 리테이너 (₩100만~300만/월)**
추적 시스템 운영 / 월간 리포트 + 전략 미팅 / 콘텐츠 최적화 실행 / A/B 테스트 / 신규 플랫폼 대응

**워크숍 (₩50만~200만/회)**

**타겟:** 기술 블로그 운영 SaaS, B2B 마케팅팀, 미디어·퍼블리셔, 이커머스, 전문 서비스(법률·의료·금융)

**수익 시뮬레이션**
```
리테이너 3곳 × ₩150만 = 월 450만원
감사 월 1건  × ₩300만 = 월 300만원
──────────────────────────────
                        월 750만원
```

**영업 전략**
1. 이 SDK로 타겟 회사 사이트를 2주간 추적
2. "귀사는 월 3,200회 인용되는데 경쟁사 X는 8,900회입니다. 왜일까요?"
3. 무료 진단 미팅 → 유료 전환

---

### 티어 3 — 장기 플레이 (6개월+)

#### 💡 #7 AI 봇 진위 검증 서비스 ★★★★

**해결하는 문제**
```bash
curl -H "User-Agent: ChatGPT-User/1.0" https://내사이트.com
# 가짜인데 ChatGPT로 기록된다
```

**해결책**
```typescript
async function verifyBot(ip: string, claimed: Provider): Promise<boolean> {
  const hostname = await reverseDns(ip);
  const VERIFIED = {
    chatgpt:    [/\.openai\.com$/, /chatgpt-user/],
    claude:     [/\.anthropic\.com$/],
    perplexity: [/\.perplexity\.ai$/],
    gemini:     [/\.googlebot\.com$/, /\.google\.com$/],
  };
  return VERIFIED[claimed]?.some((re) => re.test(hostname)) ?? false;
}
```
\+ 각 회사가 공개하는 공식 IP 대역 리스트 유지 관리

**수익 모델:** API 종량제 $0.0001/검증 / 구독형 $29월(월 100만 검증) / 검증된 봇 IP DB 구독 $99월

**장점:** 데이터 신뢰도는 분석 도구의 생명, 다른 모든 아이디어의 부가가치 레이어

---

#### 💡 #8 AI 트래픽 벤치마크 데이터 ★★★★

```
"2026 AI Citation Benchmark Report"
— 1,000개 사이트, 5억 건 인용 데이터 분석

SaaS 업계 평균: 월 8,400건 / 상위 10%: 월 42,000건
프로바이더 점유율: ChatGPT 52% / Perplexity 24% / Claude 18%
```

| 상품 | 가격 |
|---|---|
| 무료 요약 리포트 | $0 (리드 수집) |
| 상세 리포트 PDF | $299 |
| 데이터 API 구독 | $499/월 |
| 커스텀 업종 분석 | $2,000+ |

**장점:** 데이터 네트워크 효과(고객↑ = 데이터 가치↑), 언론 인용 = 무료 마케팅, 경쟁자가 못 따라오는 해자
**⚠️ 주의:** 익명화 + 집계 필수, 고객 동의 명시적 확보 (개인정보보호법)

---

#### 💡 #9 콘텐츠 최적화 자동화 에이전트 ★★★★★

**작동 루프**
```
① 인용 데이터 수집 (이 SDK)
      ↓
② 패턴 분석 — 어떤 글이 왜 인용되나
      ↓
③ 콘텐츠 개선안 생성 (LLM)
      ↓
④ GitHub PR 자동 생성 / CMS 초안 작성
      ↓
⑤ 변경 후 인용 변화 측정 (A/B)
      ↓
⑥ 학습 → ①로 (자기 개선 루프)
```

**기능 예시**
- "이 글은 결론이 맨 아래라 AI가 인용하기 어렵습니다 → 상단 요약 추가"
- "FAQ 섹션을 추가하면 인용률이 평균 35% 상승합니다"
- "Schema.org Article 마크업이 없습니다 → 자동 생성"
- "llms.txt 자동 생성/업데이트"
- "경쟁사 X가 인용되는 질문 10개 중 우리가 커버 못하는 것 3개"

**가격:** $99/월 (개인) / $299/월 (팀, PR 자동화) / $999/월 (엔터프라이즈)

**스택:** Claude Agent SDK·LangGraph / 이 SDK 구조 응용 / MCP(GitHub·CMS·Search Console) / Upstash Redis / Next.js

**장점:** 측정을 넘어 **문제 해결**까지, 로컬 에이전트 관심사와 완전 일치, 높은 진입장벽 = 강한 방어력

---

#### 💡 #10 교육 콘텐츠 & 커뮤니티 ★★★

유튜브 "GEO 완전정복" / 뉴스레터 "AI Citation Weekly" / 온라인 강의 / 유료 커뮤니티 / 전자책

```
강의 판매:      ₩99,000 × 200명  = ₩1,980만
뉴스레터 스폰서: $500 × 4회       = 월 $2,000
커뮤니티:       ₩29,000 × 100명  = 월 ₩290만
```

**장점:** 다른 모든 제품의 마케팅 채널, 개인 브랜드 구축 → 컨설팅 단가 상승, 낮은 초기 투자

---

### 법적 / 윤리적 체크리스트

| 항목 | 상태 | 비고 |
|---|---|---|
| MIT 라이선스 | ✅ | 상업 이용 / 수정 / 재배포 가능 |
| 저작권 고지 | ⚠️ | 코드 사용 시 LICENSE 포함 필수 |
| 상표 | ⚠️ | "Upstash" 이름을 제품명에 사용하지 말 것 |
| 개인정보 | ✅ | 원본이 PII를 저장하지 않음 |
| GDPR | ⚠️ | 봇 추적이라 대부분 비해당, 확장 시 검토 |
| 벤치마크 데이터 | ⚠️ | 고객 동의 + 익명화 필수 |
| 기여 예의 | 💡 | 원본에 개선사항 PR 제출 시 신뢰도 상승 |

---

## 12. 한계점 및 개선 여지

| 한계 | 설명 | 개선 아이디어 |
|---|---|---|
| 🔒 **Upstash 종속** | `redis.search`는 Upstash 전용 기능 | 어댑터 레이어 추상화 |
| 🎭 **User-Agent 위조 가능** | `curl -H "User-Agent: ChatGPT"` 로 조작 가능 | IP 역방향 DNS 검증 (#7) |
| 📉 **5개 프로바이더만** | Grok, DeepSeek, Meta AI 등 미지원 | 탐지 규칙을 설정 가능하게 |
| 🔀 **크롤링 vs 인용 미구분** | 학습용 크롤링인지 실제 인용인지 모름 | Referer 패턴 세분화 |
| 🖥️ **UI 없음** | 대시보드는 Upstash 콘솔에만 존재 | #1 임베드 대시보드 |
| 📄 **README 부실** | 25줄, 쿼리 API 설명 전무 | 문서 보강 (PR 기회) |
| 🧪 **테스트에 실 DB 필요** | `Redis.fromEnv()` — 모킹 없음 | 인메모리 Redis 모의 객체 |
| 🍼 **0.1.3 버전** | breaking change 이력 2회 (`!` 커밋) | 버전 고정 사용 권장 |
| 🌏 **지역/기기 차원 없음** | 국가별 분석 불가 | 차원 추가 (카디널리티 주의) |

### 원본에 기여할 만한 PR 아이디어

1. README에 쿼리 API 사용 예제 추가
2. `waitUntil` 패턴 문서화
3. 프로바이더 탐지 규칙을 설정 가능하게 (`customProviders` 옵션)
4. 봇 검증 훅 인터페이스 추가
5. 테스트용 인메모리 Redis 모의 객체

---

## 13. 추천 로드맵

```
┌──────────────────────────────────────────────────┐
│ STEP 1 (1~2개월) — 신뢰 쌓기 & 학습               │
├──────────────────────────────────────────────────┤
│ ① 임베드 대시보드 오픈소스 공개 (#1)               │
│    → GitHub 스타, 포트폴리오, 기술 검증            │
│ ② MCP 서버 배포 (#3)                              │
│    → 얇은 래퍼, 화제성 확보                        │
│ 수익: 거의 없음 (투자 기간)                        │
│ 획득: 실력 + 인지도 + 피드백                       │
└──────────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────┐
│ STEP 2 (2~6개월) — 현금 흐름 만들기                │
├──────────────────────────────────────────────────┤
│ ③ WordPress 플러그인 출시 (#2)                    │
│    → 시장 40%, 경쟁자 없음, 수동 수입              │
│ ④ 컨설팅 병행 (#6)                                │
│    → 즉시 현금 + 고객 문제 학습                    │
│ 목표 수익: 월 ₩300만~800만                         │
└──────────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────┐
│ STEP 3 (6개월~) — 스케일업                        │
├──────────────────────────────────────────────────┤
│ ⑤ SaaS 런칭 (#4)                                  │
│    → STEP 1,2에서 모은 유저를 전환                 │
│ ⑥ AI 최적화 에이전트 (#9)                         │
│    → 최고 가치 + 로컬 에이전트 관심사와 일치       │
│ 목표 수익: 월 $5,000~30,000                        │
└──────────────────────────────────────────────────┘
```

### 하나만 고른다면 → **#2 WordPress 플러그인**

1. 경쟁자가 없다 — 지금이 선점 타이밍
2. 시장이 거대하다 — WordPress 40%
3. 수동적 수입 — 한 번 만들면 계속 수익
4. 무료 마케팅 채널 — WordPress.org 검색
5. 기술 학습 효과 — PHP 포팅 과정에서 코드를 완전히 이해하게 됨

### 그다음 → **#9 AI 최적화 에이전트**

로컬 에이전트 구축 관심사 + 이 SDK + 시장 기회가 모두 만나는 지점.

---

## 📌 최종 요약 (3줄)

1. **정체:** ChatGPT·Claude·Perplexity 등 AI 봇의 사이트 방문을 서버 미들웨어에서 감지해 Upstash Redis에 카운트하는 초경량 TypeScript SDK. 플러그인·스킬·MCP가 아닌 **평범한 npm 라이브러리**다.
2. **가치:** AI 봇은 JS를 실행하지 않아 Google Analytics로 안 잡히는 **GEO 시대의 측정 공백**을 채운다. 코드 자체가 Redis Lua·TTL·카디널리티 제어·CQRS를 담은 **교과서급 학습 자료**다.
3. **기회:** 대시보드 UI·PHP 지원·봇 검증·인사이트가 전부 비어 있고 **MIT 라이선스**라 상업적 활용이 합법이다. WordPress 플러그인(#2)이 가장 빠른 수익화 경로, AI 최적화 에이전트(#9)가 가장 높은 가치다.

---

*본 문서는 `@upstash/agent-analytics` 저장소 전수조사 결과를 정리한 것입니다.*
*저장소: https://github.com/bmshin94/agent-analytics*
*원본: https://github.com/upstash/agent-analytics*
