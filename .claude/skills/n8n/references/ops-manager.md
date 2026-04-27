# Ops Manager 모듈

워크플로우 **실행 확인 / 실패 원인 분석 / 상태 점검 / 운영 리포트** 요청을 처리합니다.

## 운영 절차

### 1. 상태 점검
- 항상 `n8n_health_check`로 서버 상태를 먼저 확인합니다.
- `n8n_list_workflows`로 활성/비활성 워크플로우 현황을 파악합니다.

### 2. 실행 이력 조회
- `n8n_executions`로 최근 실행 이력을 확인합니다.
- 필요하면 워크플로우 ID, 상태(success/error/waiting), 기간 필터를 사용합니다.
- 실행 단위로 시작 시각, 종료 시각, 소요 시간, 상태를 정리합니다.

### 3. 에러 로그 조회
- 실패한 실행 ID를 식별한 뒤, 상세 로그를 조회하여 어떤 노드에서 실패했는지 확인합니다.
- 에러 메시지, HTTP 상태 코드, 입력 데이터를 함께 검토합니다.

## 실패 원인 분석 원칙

실패가 발견되면 다음 순서로 분석합니다.

1. **에러 메시지 우선 확인** — 가장 직접적인 원인 단서입니다.
2. **노드 종류별 흔한 원인 점검**
   - HTTP Request: SSL 오류, 인증 실패, 타임아웃, 잘못된 URL/메서드
   - 자격 증명 노드: 만료된 토큰, 권한 부족
   - Set/Function: 잘못된 표현식, undefined 참조
   - Trigger: 활성화 여부, 웹훅 URL 만료
3. **재현 가능성 확인** — 일시적 오류인지 구조적 결함인지 판단합니다.
4. **해결 방안 제시** — 단순 재실행으로 해결되는지, 워크플로우 수정이 필요한지 명확히 안내합니다.
5. **재발 방지 권고** — 에러 핸들링 노드(Error Trigger), 재시도 옵션, 알림 추가 등을 제안합니다.

## 운영 리포트 형식

운영 리포트를 요청받으면 다음 형식으로 정리합니다.

- **점검 시각**
- **서버 상태**: health check 결과
- **워크플로우 요약**: 전체 / 활성 / 비활성 개수
- **최근 실행 요약**: 성공 / 실패 / 대기 건수
- **주목할 실패 사례**: 워크플로우 이름, 실패 노드, 원인 요약
- **권장 조치**: 우선순위 순으로 1~3개

## 사용자 계정 정리

n8n MCP 도구에는 사용자(계정) 관리 기능이 없으므로 **REST API 직접 호출**로 처리합니다. n8n 은 두 가지 REST 가 있고 노출 정보가 다릅니다.

### 두 종류의 REST API

| 엔드포인트 | 인증 | 노출 필드 | 페이지네이션 |
|---|---|---|---|
| `/api/v1/users` (Public) | `X-N8N-API-KEY` 헤더 | id, email, firstName, lastName, createdAt, updatedAt, isPending, role | `limit` + `cursor` (base64) |
| `/rest/users` (Internal) | owner 세션 쿠키 | 위 + **`lastActiveAt`**, `settings.userActivated`, `settings.userActivatedAt`, `mfaEnabled`, `signInType`, `personalizationAnswers` | `take` + `skip` (max 50/page) |

**Public 만으로는 "마지막 접속" 류 정보를 알 수 없음.** Last Active 같은 운영 지표가 필요하면 반드시 internal 사용.

### Internal REST 로그인 (owner 세션)

```bash
curl -sS -A "Mozilla/5.0" -c /tmp/n8n_cookies.txt \
  -X POST "https://n8n.datapopcorn.win/rest/login" \
  -H "Content-Type: application/json" \
  -d '{"emailOrLdapLoginId":"<owner email>","password":"<owner password>"}'
```

이후 `-b /tmp/n8n_cookies.txt` 로 `/rest/*` 호출 가능. owner 계정은 `datapopcorn@gmail.com` (인스턴스별 owner 는 `role=global:owner` 로 식별).

owner 비밀번호는 메모리/`.env` 에 저장하지 않음 — 작업 시점에 사용자에게 받아서 사용.

### Pending 유저 일괄 삭제

1. **목록 수집** — `GET /api/v1/users?limit=250` (Public API 충분). `nextCursor` URL-인코딩해서 페이지네이션. `isPending: true` 가 가입 대기.
2. **카운트 보고 + 사용자 확인** — 되돌릴 수 없음.
3. **일괄 DELETE** — `DELETE /api/v1/users/{id}`. pending 유저는 워크플로우/크레덴셜 없으므로 transferId 불필요.
4. **검증** — 재조회 후 `isPending` 카운트 0.
5. **결과 보고** — 성공/실패 + 마스킹 샘플.

### "한 번도 접속 안 한 유저" 식별 — ⚠️ 함정 주의

UI 의 "Last Active: Never" 는 `lastActiveAt = null` 인 유저인데, 이건 **"한 번도 n8n 을 사용하지 않았다" 라는 뜻이 아님**. `lastActiveAt` 필드가 n8n 신버전에 추가되었고 과거 활동은 백필되지 않음. 즉 워크플로우를 잔뜩 만든 옛날 유저도 `lastActiveAt = null` 일 수 있음.

**유저 삭제 전 반드시 워크플로우 소유 여부 확인**:

1. `/rest/workflows?take=100&skip=0` 페이지네이션으로 전체 워크플로우 수집. 응답에 `homeProject` 있음.
2. personal project 의 `name` 필드가 `"FirstName LastName <email@domain>"` 형식 → 정규식 `<([^>]+)>` 로 이메일 추출.
3. 대상 유저 이메일 set 과 매칭하여 owned workflows 카운트.
4. 워크플로우 보유자는 별도 검토 트랙으로 분리.

### 워크플로우 소유 유저 삭제 — `transferId` 필수

`DELETE /api/v1/users/{id}` 를 그냥 호출하면 그 유저의 personal project 통째로 삭제 = **워크플로우/크레덴셜 전부 삭제됨**. 반드시 다음 중 선택:

- **이전 후 삭제**: `DELETE /api/v1/users/{id}?transferId=<다른유저id>` — 모든 자원이 transferId 유저로 이전된 뒤 원 유저 삭제. 보통 owner 로 이전.
- **수동 정리 후 삭제**: 워크플로우를 먼저 다른 곳으로 옮기거나 백업/삭제한 다음 일반 DELETE.

### 권장 단계 (대규모 정리 시)

1. **pending 유저** → 즉시 삭제 (자원 없음).
2. **lastActiveAt=null 이면서 워크플로우 0 인 유저** → 즉시 삭제 (손실 0).
3. **lastActiveAt=null 이지만 워크플로우 보유 유저** → 보유 목록(특히 active/archived 카운트) 사용자에게 보고하고 케이스별 결정.
4. 활성 유저는 절대 자동 삭제 후보로 올리지 않음.

### 알려진 함정

- **Cloudflare 1010 (403)**: Python `urllib` 기본 UA (`Python-urllib/3.x`) 가 datapopcorn.win 앞단 Cloudflare 에 차단됨. 반드시 `User-Agent: Mozilla/5.0 ...` 헤더 추가. curl 은 `-A "Mozilla/5.0"`.
- **Public API cursor 인코딩**: `nextCursor` 값에 `=` 패딩이 있어 raw 삽입 시 다음 페이지 403. `urllib.parse.quote(cursor, safe='')` 또는 `%3D` 로 인코딩.
- **Internal API 페이지 크기**: `take` 가 50 초과해도 50개만 반환. `skip` 으로 페이지네이션.
- **`userActivated` 의미 오해**: `settings.userActivated = true` 는 personalization survey 완료를 뜻함. "로그인했다" 와 다름. inactive 식별 기준으로 단독 사용 금지.
- **개인정보 마스킹**: 유저 이메일은 PII. 출력 시 항상 마스킹 (`local` 앞 2글자만 노출, 나머지 `*`, 도메인은 그대로). 예: `kdy15589@gmail.com` → `kd******@gmail.com`.
- **MCP 미지원**: `mcp__n8n-mcp__*` 에는 user 관리 도구가 없음. 계정 작업은 항상 직접 REST.

### Public API 인증

`X-N8N-API-KEY` 헤더. 키는 `~/Documents/my-project/n8n/.env`:
- 로컬: `N8N_HOST_URL` / `N8N_API_KEY`
- win: `N8N_REMOTE_HOST_URL` / `N8N_REMOTE_API_KEY`
- xyz: `N8N_REMOTE_XYZ_HOST_URL` / `N8N_REMOTE_XYZ_API_KEY`

## 안전 원칙

- 운영 작업 중 워크플로우를 수정해야 할 경우 반드시 사용자 확인을 받습니다.
- 실행 데이터에서 비밀 정보(API 키, 개인정보 등)가 보이면 출력에서 마스킹합니다.
- 삭제, 비활성화 등 영향이 큰 조치는 사용자 승인 후에만 수행합니다.
