# CLAUDE.md — umami (fork)

## What this is

Personal fork of [umami-software/umami](https://github.com/umami-software/umami) v3.2.0,
self-hosted web analytics. Deployed on Vercel, backed by Neon Postgres.

- `origin` = `aesopfrom0/umami` (this fork), `upstream` = `umami-software/umami`
- The fork is intentionally **thin**: at time of writing, exactly one commit ahead of
  upstream (`0bf8e038d`, geo headers). Keep it that way — every local commit is a
  future merge conflict. Prefer configuration and env vars over patches.

## Target Users

- **Primary market**: 운영자 본인 (single-operator, self-hosted)
- **User-facing language**: English (upstream UI strings; do not localize)
- **Currency**: USD
- **Timezone context**: 대시보드는 브라우저 타임존, 저장은 UTC

이 프로젝트에 신규 user-facing 카피를 추가할 일은 거의 없다. 추가한다면 upstream i18n
구조(`src/i18n`)를 따르고 새 문자열 체계를 만들지 말 것.

## Primary constraint: cost

**이 리포의 1순위 제약은 기능이 아니라 월 비용이다.** Neon에 월 ~$20이 나가고 있고,
줄이는 게 목표다. 아키텍처 제안은 반드시 월 달러 영향과 함께 제시할 것.

비용 관련 조사·결정은 `docs/intent.md`가 SSOT. 새로 조사하기 전에 먼저 읽을 것 —
이미 기각된 옵션을 다시 제안하지 않기 위함이다.

## Architecture — the part that matters

데이터 접근은 **3층**이고, 층마다 이식성이 완전히 다르다. DB 교체를 논하는 사람은
이 구분을 먼저 이해해야 한다.

| 층 | 위치 | 규모 | 성격 |
|---|---|---|---|
| CRUD | `src/queries/prisma/` | ~1.4k lines | Prisma ORM. 이식 가능 |
| 분석 쿼리 | `src/queries/sql/` | ~11.2k lines | **손으로 쓴 raw SQL**. 이식 불가 |
| 디스패치 | `src/lib/db.ts` | ~40 lines | `runQuery({prisma, clickhouse})` 분기 |

`src/queries/sql/`의 각 파일은 같은 함수를 **두 번** 구현한다 — `relationalQuery()`
(Postgres) 와 `clickhouseQuery()`. 둘 다 문자열 SQL이다. 여기엔 121개 JOIN, 26개 CTE,
window function, `UNION ALL`, LATERAL이 들어 있다.

`src/lib/prisma.ts` (616줄) 와 `src/lib/clickhouse.ts` 는 단순 클라이언트 래퍼가 아니라
**SQL 방언 생성기**다 — `parseFilters()`, `getDateSQL()`, `getTimestampDiffSQL()` 등이
필터 객체를 SQL 조각으로 컴파일한다. 새 백엔드를 추가하려면 이 파일의 대응물을 통째로
써야 한다.

**따라서**: "DB를 X로 바꾸자"는 제안은 X가 SQL을 말하는지에 따라 난이도가 100배 차이난다.
SQL이면 방언 차이만 흡수하면 되고, 아니면 11k줄을 재작성해야 한다.

### 지금 켜져 있지 않은 것

`CLICKHOUSE_URL`, `REDIS_URL`, `KAFKA_URL`이 없으면 `runQuery`는 항상 Prisma/Postgres
경로로 간다. 즉 **현재 이 배포는 Postgres 단일 백엔드**다. ClickHouse 코드 경로는
존재하지만 죽어 있다.

## Working rules

- **패키지 매니저는 `pnpm`** (`pnpm-lock.yaml`, workspace). 글로벌 yarn 선호의 예외.
- 스키마를 만지면 `prisma/migrations/`에 마이그레이션을 추가한다. 기존 마이그레이션
  파일은 **수정하지 말 것** — 이미 적용된 것들이다.
- `src/generated/prisma/`는 빌드 산출물. 직접 편집 금지 (`pnpm build-db-client`).
- lint/format은 Biome (`pnpm lint`, `pnpm check`). ESLint 아님.
- 테스트는 vitest (`pnpm test`). e2e는 Playwright.

## Upstream 동기화

```
git fetch upstream && git log --oneline HEAD..upstream/master
```

업스트림이 `src/queries/sql/` 나 `src/lib/prisma.ts` 를 건드리면 로컬 DB 관련 변경과
충돌한다. 이것이 fork를 얇게 유지해야 하는 이유다.

## Anti-patterns

- ❌ 비용 논의 없이 인프라 제안 — 이 리포의 제약을 무시하는 것
- ❌ `src/queries/sql/` 에 세 번째 백엔드 분기 추가 — 11k줄 × N 유지보수
- ❌ upstream 파일을 스타일 이유로 리포맷 — 머지 충돌만 늘어난다
- ❌ `docs/` 에 날짜 없는 임시 조사 문서 — 컨벤션은 `YYMMDD-{설명}.md`
