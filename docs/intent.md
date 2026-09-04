# intent.md — 이 fork가 존재하는 이유와 현재 결정

> **성격**: 이 파일은 "무엇을 왜 결정했나"의 SSOT다. 코드가 답해주는 것(구조, 파일 위치)은
> `CLAUDE.md`, 여기엔 코드에서 읽을 수 없는 **의도와 제약**만 쓴다.
> 새 세션의 에이전트는 인프라·DB 관련 제안을 하기 전에 이 파일을 먼저 읽는다.

## 1. 목표

셀프호스팅 웹 애널리틱스를 **월 고정비를 최소화하면서** 계속 굴린다.

우선순위:

1. **월 비용** — 지금 Neon에 ~$20/월. 이게 이 리포의 1순위 제약이다.
2. 데이터 연속성 — 지금까지 쌓인 방문 기록을 잃지 않는다.
3. 유지보수 부담 최소화 — 1인 운영이다. 새벽에 깨는 인프라는 실패다.
4. 기능은 현상 유지로 충분하다. 새 기능이 필요해서 만든 fork가 아니다.

**명시적 비목표**: 확장성, 멀티테넌시, 고가용성. 트래픽 규모가 작고 커질 계획도 없다.

## 2. 제약

- 배포는 Vercel (`.vercel/project.json`, `prj_0qNE5O3NVCqzxMAZDaQzMlz3gRuT`)
- fork는 upstream보다 1커밋 앞선 상태를 유지 — 머지 비용을 0에 가깝게 둔다
- 운영자 1명, 전담 DBA 없음
- upstream(umami v3)은 **PostgreSQL 전용**이다. MySQL 지원은 v2에서 제거됐고
  MongoDB는 지원된 적이 없다

## 3. 결정 이력

### D-001 (2026-09-04) — MongoDB 포팅: **기각**

**맥락**: Neon $20/월이 아까워서 MongoDB Atlas Flex($8~30/월)로 옮길 수 있는지 검토.

**조사 결과**:

- umami의 분석 쿼리 `src/queries/sql/`는 **11,151줄의 손으로 쓴 raw SQL**이다.
  121개 JOIN, 26개 CTE, window function, `UNION ALL`, LATERAL 포함.
- `src/lib/prisma.ts`(616줄)는 필터를 SQL 문자열로 컴파일하는 방언 생성기다.
  MongoDB용 대응물을 통째로 새로 써야 한다.
- Prisma의 MongoDB 커넥터에는 raw 탈출구가 있지만 **SQL이 아니다** —
  `$runCommandRaw()` / `findRaw()` / `aggregateRaw()` 뿐이고 `$queryRaw`는 없다.
  즉 11k줄이 "고치면 되는" 게 아니라 **aggregation pipeline으로 전면 재작성** 대상이다.
  (탈출구의 존재가 이식을 쉽게 해주지 않는다. 입력 언어가 바뀐다.)
- upstream이 `src/queries/sql/`를 건드릴 때마다 그 재작성분이 전부 충돌한다.
  fork를 얇게 유지한다는 원칙과 정면 충돌.

**기각 사유 (결정적)**: **비용 문제가 DB 엔진 문제가 아니다.**
Neon Launch는 compute $0.106/CU-hour + storage $0.35/GB-month의 순수 사용량 과금이고
최소 요금이 없다. $20/월이 나온다는 건 **컴퓨트가 scale-to-zero 되지 않고 거의 상시
켜져 있다**는 뜻이다. Mongo Flex로 옮겨도 같은 상시 워크로드는 $8이 아니라 상한 $30에
붙는다. **수개월의 재작성을 하고 비용이 오를 수 있다.**

**대신 할 것**: 비용 구조를 먼저 측정한다. → D-002

### D-002 (2026-09-04) — 측정 결과: 서버리스 자체가 이 워크로드에 안 맞는다

프로덕션 DB를 읽기 전용으로 직접 조회해 **측정**했다 (추정 아님).

**스토리지 — 2,271 MB**

| 테이블 | 합계 | 데이터 | 인덱스 | 행 수 |
|---|---|---|---|---|
| `website_event` | 1,545 MB | 277 MB | **1,267 MB** | 1,169,529 |
| `event_data` | 647 MB | 274 MB | **372 MB** | 2,282,553 |
| `session` | 72 MB | 9 MB | 63 MB | 74,540 |

데이터는 560 MB인데 인덱스가 1.7 GB — **스토리지의 75%가 인덱스**다.
데이터 보존 기간: 2026-03-05 ~ 현재 (6개월).

**컴퓨트 — 유휴 구간이 사실상 없다** (최근 7일)

- 이벤트 81,653건, **이벤트 간 평균 간격 0.12분(약 7초)**
- 5분(Neon suspend 임계값) 초과 공백은 **79회뿐, 합계 9.7시간**
- 최대 공백 29.7분

**결론**: 168시간 중 회수 가능한 유휴는 **9.7시간(5.8%)**뿐이다. 트래커가 7초마다
컴퓨트를 깨우므로 scale-to-zero가 원리적으로 작동하지 않는다.
스토리지는 2.27 GB × $0.35 = **약 $0.79/월**. 즉 **$20 중 ~$19가 컴퓨트**이고,
이는 상시 가동 요금이다.

**따라서 D-001의 기각이 측정으로 확증됐다.** 비용의 원인은 DB 엔진이 아니라
"상시 워크로드를 사용량 과금 서버리스에 올려둔 것"이다. MongoDB Flex로 옮기면
같은 상시 트래픽이 상한 $30에 붙어 **비용이 오른다**.

**방향**: 서버리스를 떠나 **정액 요금제**로 간다. 부수적으로 인덱스/보존 정리로
스토리지를 줄인다(효과는 $1 미만이므로 우선순위 낮음 — 컴퓨트가 진짜 문제다).

## 4. 검토된 옵션 (비용순, 2026-09-04 기준)

각 항목의 `SQL?` 열이 핵심이다. **No면 11k줄 재작성**을 뜻한다.

| 옵션 | 월 비용 | SQL? | 코드 변경 | 판정 |
|---|---|---|---|---|
| 정액 VPS + Postgres (Hetzner 등) | ~$5 | 예 | 없음(연결 문자열) | **1순위** |
| 관리형 Postgres 정액제 (Supabase Pro 등) | $10~15 | 예 | 없음 | **2순위** (운영부담 회피) |
| Neon 튜닝만 | ~$19 | 예 | 없음 | **효과 없음** (유휴 5.8%) |
| Neon 유지 | ~$20 | 예 | 없음 | 기준선 |
| MongoDB Atlas Flex | $8~30 | **아니오** | ~11k줄 재작성 | **기각 (D-001)** |

D-002의 측정으로 "Neon 튜닝" 옵션은 사실상 탈락했다 — 회수 가능한 유휴가 5.8%뿐이라
suspend를 최적화해도 $1 남짓이다. 스토리지 정리도 $0.79 중 일부라 무의미하다.
**상시 워크로드에는 정액제가 구조적으로 맞다.**

**핵심 통찰**: Postgres를 유지하는 모든 옵션은 코드 변경이 **0**이다 — `DATABASE_URL`만
바꾸면 된다. MongoDB만 유일하게 리포를 다시 쓰게 만든다. 절감액이 같다면 코드를 안 건드리는
쪽이 언제나 낫다.

**VPS의 트레이드오프 (숨기지 말 것)**: 백업, 복구, 보안 패치, 디스크 모니터링이 전부
본인 책임이 된다. 목표 3(유지보수 부담 최소화)와 직접 충돌한다. $15 아끼려고 새벽 3시에
디스크 풀 대응을 하게 될 수 있다. 선택한다면 자동 백업을 첫날에 세팅할 것.

### D-003 (2026-09-04) — upstream v3.3.1 머지

`upstream/master` 239커밋 머지 (v3.2.0 → v3.3.1). **충돌 없음**, typecheck 0 에러,
테스트 739개 전부 통과 (95 파일).

fork 패치 2개는 그대로 살아남았다 (`src/lib/ip.ts`, `src/lib/detect.ts`).
upstream이 `ip.ts`를 `parseHeaderValue()`로 리팩터링했지만 우리 라인과 다른 영역이라
git이 자동 병합했고, `x-umami-client-ip`는 여전히 `CLOUD_MODE` 게이트 없이 무조건
읽힌다. **`FORK PATCH` 주석이 있어서 다음 머지에서도 판단이 가능하다 — 이 주석 관례를
유지할 것.**

**배포 전 반드시 할 것 — 신규 마이그레이션 4개**:

| 마이그레이션 | 성격 |
|---|---|
| `21_add_session_link` | 테이블 추가 (안전) |
| `22_add_2fa` | 테이블·컬럼 추가 (안전) |
| `23_update_session_data` | **파괴적** — 중복 `session_data` 행 `DELETE` 후 unique index |
| `24_lowercase_username` | **파괴적** — username 소문자화, 충돌 시 soft-deleted 행 rename |

23·24는 데이터를 되돌릴 수 없다. **적용 전 백업 필수.**

다만 운영 DB를 실제로 조회해 확인한 결과, **둘 다 이 DB에서는 no-op**이다:

- `session_data` 0행 → 23의 `DELETE`가 지울 게 없다
- 유저 1명, 대소문자 혼용 username 0건, soft-deleted 0건 → 24가 바꿀 행이 없다

즉 이번 머지의 마이그레이션 리스크는 실질적으로 없다. 그래도 백업은 하고 적용할 것
(위 수치는 2026-09-04 기준이고, 적용 시점에 달라질 수 있다).

### D-004 (2026-09-04) — v3.3.1 배포 런북 (마이그레이션 4개)

**핵심 사실 1 — `git push`가 곧 마이그레이션이다.**
`scripts/check-db.js`의 `applyMigration()`이 Vercel 빌드 중 `prisma migrate deploy`를
자동 실행한다. `SKIP_DB_MIGRATION`이 설정돼 있지 않으면 푸시하는 순간 21~24가 적용된다.
즉 "배포"와 "마이그레이션"을 분리하려면 그 환경변수를 써야 한다.

**핵심 사실 2 — `DATABASE_URL`이 pooled 엔드포인트다. 이게 최대 리스크.**

- 운영 DB 조회 결과 `application_name = pgbouncer`, 호스트가 `...-pooler.c-4.us-east-1.aws.neon.tech`
- Vercel 프로덕션 env에 **`DIRECT_DATABASE_URL`이 없다** (`CLIENT_IP_HEADER`, `DATABASE_TYPE`,
  `APP_SECRET`, `DATABASE_URL` 4개뿐)
- `applyMigration()`은 `DIRECT_DATABASE_URL || DATABASE_URL` 순으로 고르므로 지금은
  **pooler를 통해 마이그레이션이 돈다**
- Prisma 공식 문서: Schema Engine은 PgBouncer를 지원하지 않으며 마이그레이션은 direct
  connection을 써야 한다 ([docs](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/pgbouncer))

→ **배포 전에 `DIRECT_DATABASE_URL`을 추가한다.** Neon에서 호스트의 `-pooler`만 뺀 URL.

**리스크 평가 — 실제로는 낮다 (측정 기준 2026-09-04)**

| 마이그레이션 | 대상 | 영향 |
|---|---|---|
| 21 | `session_link` 신규 테이블 | 기존 데이터 무관 |
| 22 | 2FA 테이블 5개 + `user`/`team`에 `ADD COLUMN NOT NULL DEFAULT false` | PG 11+ 에서 메타데이터 전용, 테이블 rewrite 없음 |
| 23 | `session_data` 중복 DELETE + unique index | **0행 / 중복 그룹 0개 → no-op** |
| 24 | username 소문자화 | 유저 1명, 대소문자 혼용 0건 → **no-op** |

**대용량 테이블(`website_event` 1.17M행, `event_data` 2.28M행)에는 DDL이 전혀 없다.**
기존 테이블 중 인덱스가 추가되는 건 `session_replay`뿐인데 0행이다.
`_prisma_migrations`는 `20_add_heatmap`까지 깨끗하고 failed/pending 행이 없다.

**절차 (2026-09-04 실행)**

1. ✅ Vercel Production에 `DIRECT_DATABASE_URL` 추가 (`-pooler` 없는 호스트).
   `-pooler`를 뺀 URL로 `prisma migrate status`를 돌려 연결과 pending 4개를 사전 확인함
2. 롤백 수단 확인 — Neon 유료 플랜은 기본 **1일 PITR**(Instant restore, Settings →
   Instant restore). 별도 덤프 없이 이 창 안에서 되돌릴 수 있다
3. `git push origin master` → 빌드가 마이그레이션 자동 적용
4. 빌드 로그에서 `Database is up to date.` 확인
5. 사후 검증: `_prisma_migrations`에 21~24가 `finished_at NOT NULL`로 들어갔는지,
   로그인 되는지(24가 username을 건드리므로), 대시보드에 데이터가 보이는지

**실행 결과 (2026-09-04 07:18 UTC) — 성공**

- 빌드 2분, 상태 Ready. 로그에 `✓ Database is up to date.` (07:18:51)
- `_prisma_migrations`에 21~24 전부 `finished_at NOT NULL`, 실패/롤백 0건
- 데이터 무손실: 이벤트 1,169,557 → 1,169,701 (배포 중에도 수집 계속됨),
  세션 74,554 / 유저 1 / 웹사이트 8 그대로. DB 크기 2,272 MB 변동 없음
- 신규 테이블 `session_link`, `two_factor_auth` 생성됨 (0행, 정상)
- 프로덕션 응답 확인: `/`, `/login`, `/script.js` 전부 200
- **수집 경로 E2E 검증**: 실제 website_id로 `/api/send` POST → HTTP 200 →
  DB에 행 적재 확인

  이때 남긴 테스트 행(이벤트 1 + 세션 1)은 같은 날 삭제 완료

**되돌리기**: 코드는 `git revert`로 되지만 **마이그레이션은 자동 롤백이 없다.**
21·22는 추가만 하므로 사실상 무해하고, 23·24는 이 DB에서 no-op이라 실제 롤백 대상이
없다. 그래도 1번(백업)을 건너뛰지 말 것 — 위 수치는 오늘 기준이다.

## 5. 열린 항목

- [x] ~~배포 검증용 테스트 행 삭제~~ — 2026-09-04 완료. `event_id`/`session_id` 지정으로
      이벤트 1행 + 세션 1행만 삭제 (1,169,705 → 1,169,704). 잔여 0건 확인.

- [ ] Neon 청구서 실물로 compute/storage 분해 교차검증 (D-002는 DB 측 측정 기반 추정)
- [ ] 정액 이전 대상 확정: VPS 직접 운영(~$5) vs 관리형 정액(~$10~15)
      — 판단 기준은 "백업/패치를 내가 지겠는가"
- [ ] 이전 시 데이터 이관 리허설 (`pg_dump`/`pg_restore`, 2.27 GB)
- [ ] 미사용 인덱스 정리 (측정 완료, 아래 참조)

### 부록 — 미사용 인덱스 측정 (2026-09-04)

`pg_stat_user_indexes` 기준 **61개 인덱스가 `idx_scan = 0`**, 사용 중은 40개.
`pg_stat_database.stats_reset`이 `null`이고 40개는 스캔이 잡히므로 카운터는 살아 있다
— 즉 이 0들은 통계 초기화 아티팩트가 아니라 **실제 미사용**이다.

상위 후보:

| 인덱스 | 크기 | 스캔 |
|---|---|---|
| `website_event_website_id_created_at_page_title_idx` | 226 MB | 0 |
| `website_event_website_id_created_at_url_query_idx` | 102 MB | 0 |
| `website_event_visit_id_idx` | 13 MB | 0 |
| `session_website_id_created_at_{os,browser,screen,city,device,language}_idx` | 각 ~7 MB | 0 |

**주의 (지우기 전에 읽을 것)**:

- 이건 umami **upstream 기본 스키마**다. 지우면 `prisma/schema.prisma`와 실제 DB가
  어긋나고, 다음 `prisma migrate`에서 되살아나거나 충돌한다. fork를 얇게 유지한다는
  원칙과 충돌하므로 스키마 수정이 아니라 **운영 DB에서만** 적용할지 결정할 것.
- 스캔 0은 "지금까지의 대시보드 사용 패턴에서 안 쓰였다"는 뜻이다. 특정 필터
  (page title, url query별 분석)를 쓰기 시작하면 필요해진다.
- **절감액은 최대 ~$0.13/월** (380 MB × $0.35). 비용 관점에선 사실상 무의미하다.
  쓰기 성능(인덱스 유지 비용) 개선이 실제 이득이고, 그건 컴퓨트에 간접적으로 도움된다.

## 6. 이 파일 관리 규칙

- 결정은 **append-only**. D-00N 번호로 추가하고, 뒤집을 땐 지우지 말고 새 항목에서
  "D-00X를 뒤집는다"고 명시한다. 기각 사유가 남아야 같은 논의를 반복하지 않는다.
- 사고 흔적(왜 그렇게 생각했나, 시뮬레이션)은 `~/notes/`로, 결정 결과만 여기로.
- 이 파일은 자체로 완결이어야 한다 — 다른 머신엔 `~/notes/`가 없다.
