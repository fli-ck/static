# 08. REST API

> 최초 작성: 2026-09-29 (초안 — 구현하면서 바뀌는 점은 이 문서부터 고치고 `00-decisions.md`에 남긴다)
> 근거: `02-architecture.md` §2·§3·§6, `03-agent.md`, `04-auth.md` §5.1

## 0. 공통 규약

### 0.1 경로

| 접두사 | 용도 |
| --- | --- |
| `/api/…` | JSON API. 이 문서의 대부분 |
| `/auth/…` | 브라우저 리다이렉트가 오가는 로그인 흐름 (JSON 아님) |
| `/webhook/…` | 외부 서비스가 호출하는 웹훅 수신 |
| `/ws/…` | WebSocket |
| `/health` | 헬스체크 |

버전 접두사(`/v1`)는 두지 않는다. 클라이언트가 Beacon 자신의 프론트엔드 하나뿐이다.

### 0.2 응답 형태

```json
// 성공 — 단건
{ "data": { ... } }

// 성공 — 목록 (커서 페이지)
{ "data": [ ... ], "page": { "nextCursor": "eyJpZCI6MTIzfQ" } }

// 실패
{ "error": { "code": "FORBIDDEN", "message": "Owner만 할 수 있습니다.", "details": { } } }
```

- `nextCursor`가 `null`이면 마지막 페이지다
- 본문이 없는 성공은 `204 No Content`
- `details`는 선택. 필드 검증 실패 시 `{ "field": "role", "reason": "..." }` 같은 형태

### 0.3 에러 코드

| HTTP | code | 언제 |
| --- | --- | --- |
| 400 | `INVALID_REQUEST` | 필수 값 누락, 형식 오류, 허용되지 않은 enum |
| 401 | `UNAUTHORIZED` | 로그인 세션 없음 또는 만료 |
| 403 | `FORBIDDEN` | 역할 부족, 비활성화된 계정, 자기 자신 변경 시도 |
| 404 | `NOT_FOUND` | 리소스 없음 (웹훅 slug 포함) |
| 409 | `CONFLICT` | 상태 충돌 (예: Owner를 강등, 이미 처리된 승인) |
| 409 | `ACTION_EXPIRED` | 에이전트 승인 대기 시간 만료 |
| 413 | `PAYLOAD_TOO_LARGE` | 웹훅 본문 크기 초과 |
| 502 | `UPSTREAM_UNAVAILABLE` | 원천 시스템(어댑터 대상)에 닿지 못함 |
| 500 | `INTERNAL_ERROR` | 그 외 |

`UNAUTHORIZED`와 `PAYLOAD_TOO_LARGE`는 이 문서에서 추가했다 (`02-architecture.md` §6 목록 보강).

### 0.4 값 규칙

| 항목 | 규칙 |
| --- | --- |
| 시각 | ISO-8601 UTC, `Z` 표기. 예: `2026-09-29T04:32:10Z`. 쿼리의 `from`·`to`도 동일 |
| 기본 조회 범위 | `from`·`to`를 생략하면 최근 24시간 |
| enum | 대문자 스네이크. 역할 `OWNER` `OPERATOR` `VIEWER`, 상태 `UP` `DOWN` `DEGRADED` `UNKNOWN` |
| 페이지 | `?limit=`(기본 50, 최대 200)과 `?cursor=`. 커서는 서버가 준 문자열을 그대로 돌려보낸다 |
| 문자열 입력 | 라우팅 진입점에서 NFKC 정규화 (D23). 단, 웹훅 원본 본문은 원문 그대로 저장한다 |
| 대상 식별 | `{adapterId}/{sourceId}` 두 조각. 원천 ID가 곧 Beacon ID가 되지 않게 (D43) |

### 0.5 인증과 권한

- 로그인하면 세션 쿠키(`HttpOnly`, `Secure`, `SameSite=Lax`)가 발급된다. `/api`와 `/ws`는 이 쿠키로 인증한다
- 상태를 바꾸는 요청(`POST` `PATCH` `PUT` `DELETE`)은 `Content-Type: application/json`만 받는다. `SameSite=Lax`와 함께 CSRF 방어가 된다 (HTML 폼은 JSON을 보낼 수 없음)
- 역할은 `VIEWER` < `OPERATOR` < `OWNER`. 아래 표의 "최소 역할"보다 낮으면 `403`
- 비활성화된 계정의 세션은 즉시 무효 (`04-auth.md` §6)

---

## 1. 전체 목록

단계: **지금** = D47에 따라 먼저 만드는 것 / M4·M5·M6 = 해당 마일스톤 / **보류** = 원천 어댑터가 생긴 뒤에 의미가 생김 (지금 만들면 빈 응답)

| # | 메서드 | 경로 | 최소 역할 | 단계 | 설명 |
| --- | --- | --- | --- | --- | --- |
| **시스템** | | | | | |
| 1 | GET | `/health` | 공개 | 지금 | 헬스체크, 버전 |
| **인증 — tsidp** | | | | | |
| 2 | GET | `/auth/oidc/login` | 공개 | 지금 | tsidp로 리다이렉트 |
| 3 | GET | `/auth/oidc/callback` | 공개 | 지금 | 코드 교환, 계정 생성·조회, 세션 발급 |
| 4 | POST | `/api/auth/logout` | VIEWER | 지금 | 로그아웃 |
| 5 | GET | `/api/auth/me` | VIEWER | 지금 | 현재 사용자 |
| **사용자 · 역할** | | | | | |
| 6 | GET | `/api/users` | OWNER | 지금 | 사용자 목록 |
| 7 | GET | `/api/users/{userId}` | OWNER | 지금 | 사용자 단건 |
| 8 | PATCH | `/api/users/{userId}` | OWNER | 지금 | 역할 변경, 비활성화·재활성화 |
| 9 | POST | `/api/users/{userId}/transfer-ownership` | OWNER | 지금 | Owner 이전 |
| **웹훅** | | | | | |
| 10 | GET | `/api/webhook-sources` | OWNER | 지금 | 웹훅 수신처 목록 |
| 11 | POST | `/api/webhook-sources` | OWNER | 지금 | 수신처 생성 (slug 발급) |
| 12 | DELETE | `/api/webhook-sources/{sourceId}` | OWNER | 지금 | 수신처 삭제 |
| 13 | POST | `/api/webhook-sources/{sourceId}/rotate` | OWNER | 지금 | slug 재발급 |
| 14 | POST | `/webhook/{slug}` | 공개 (slug가 비밀) | 지금 | 외부 이벤트 수신 |
| 15 | GET | `/api/external-events` | VIEWER | 지금 | 수신 이벤트 검색 |
| 16 | GET | `/api/external-events/{eventId}` | VIEWER | 지금 | 이벤트 단건 (원본 포함) |
| **어댑터** | | | | | |
| 17 | GET | `/api/adapters` | VIEWER | 지금 | 등록된 원천 목록과 연결 상태 |
| 18 | PATCH | `/api/adapters/{adapterId}` | OWNER | 지금 | 활성·알림 토글 |
| **감사 로그** | | | | | |
| 19 | GET | `/api/audit-log` | OPERATOR | 지금 | 누가 언제 무엇을 했나 |
| **실시간** | | | | | |
| 20 | GET (WS) | `/ws/events` | VIEWER | 지금 | 새 외부 이벤트, 어댑터·대상 상태 변화 푸시 |
| **대상 · 상태 · 점검** | | | | | |
| 21 | GET | `/api/summary` | VIEWER | 지금 | 대시보드 머리 숫자 (원천이 없으면 0) |
| 22 | GET | `/api/targets` | VIEWER | 보류 | 대상과 현재 상태 목록 |
| 23 | GET | `/api/targets/{adapterId}/{sourceId}` | VIEWER | 보류 | 대상 단건, 가동률, 최근 이력 |
| 24 | GET | `/api/status-events` | VIEWER | 보류 | 상태 전이 이력 |
| 25 | GET | `/api/maintenances` | VIEWER | 보류 | 예정·진행 점검 |
| **에이전트** | | | | | |
| 26 | POST | `/api/agent/chat` | VIEWER | M4 | 자연어 지시 (SSE 응답) |
| 27 | POST | `/api/agent/actions/{actionId}/confirm` | OPERATOR | M5 | 쓰기 승인·거부 |
| 28 | GET | `/api/agent/conversations` | VIEWER | M4 | 내 대화 목록 (30일) |
| 29 | GET | `/api/agent/conversations/{conversationId}` | VIEWER | M4 | 내 대화 단건과 도구 호출 근거 |
| **패스키** | | | | | |
| 30 | POST | `/api/auth/passkey/register/options` | VIEWER · tsidp 세션만 | M6 | 등록 옵션 발급 |
| 31 | POST | `/api/auth/passkey/register/verify` | VIEWER · tsidp 세션만 | M6 | 등록 검증·저장 |
| 32 | POST | `/api/auth/passkey/login/options` | 공개 | M6 | 로그인 옵션 발급 |
| 33 | POST | `/api/auth/passkey/login/verify` | 공개 | M6 | 로그인 검증, 세션 발급 |
| 34 | GET | `/api/me/passkeys` | VIEWER | M6 | 내 패스키 목록 |
| 35 | DELETE | `/api/me/passkeys/{passkeyId}` | VIEWER | M6 | 내 패스키 삭제 |

**만들지 않는 것**

- 점검 생성·취소 REST 엔드포인트. 쓰기는 에이전트의 WriteFlow와 승인(#27)만 거친다 (D16, D17)
- 모니터 등록 엔드포인트 (D12)
- 알림 목적지 설정 엔드포인트. Discord 웹훅 URL은 환경변수 `BEACON_NOTIFY_DISCORD_URL`로 받는다
- 감사 로그 삭제

**만드는 순서 (권장)**

```
1 → 2·3·5·4 → 6·7·8·9 → 10~16 → 17·18 → 19 → 21 → 20
```

`/health`와 공통 응답·에러 처리를 먼저 두면, 이후 엔드포인트는 전부 같은 틀을 탄다.

---

## 2. 시스템

### #1 `GET /health`

```json
{ "data": { "status": "ok", "version": "0.1.0", "uptimeSeconds": 38210 } }
```

`version`은 이미지 빌드 인자 `VERSION`에서 온다 (D36). 값이 없으면 `"dev"`.

---

## 3. 인증 — tsidp

### #2 `GET /auth/oidc/login`

- 쿼리: `returnTo` (선택, 로그인 후 돌아갈 Beacon 내부 경로. 외부 URL이면 무시)
- 동작: `state`와 PKCE `code_verifier`를 만들어 짧은 수명의 쿠키에 담고, tsidp 인가 엔드포인트로 `302`

### #3 `GET /auth/oidc/callback`

- 쿼리: `code`, `state`
- 동작
  1. `state` 쿠키와 비교, 다르면 `400`
  2. 코드 → 토큰 교환 (PKCE)
  3. 사용자 정보에서 로그인 이름을 얻어 계정 조회
  4. 없으면 생성: 로그인 이름이 `BEACON_OWNER`와 같으면 `OWNER`, 아니면 `VIEWER` (D31, D32)
  5. 비활성화된 계정이면 세션을 만들지 않고 안내 페이지로
  6. 세션 쿠키 발급(로그인 수단 `TSIDP` 기록), `lastLoginAt` 갱신, `returnTo`나 `/`로 `302`

> 구현 메모: 첫 로그인 때 받은 클레임을 로그로 한 번 찍어 보고 어떤 값을 "로그인 이름"으로 쓸지 정한다. `BEACON_OWNER`도 그 값으로 넣는다.

### #4 `POST /api/auth/logout`

세션 삭제, 쿠키 만료. `204`.

### #5 `GET /api/auth/me`

```json
{
  "data": {
    "id": 3,
    "login": "nx@github",
    "displayName": "nx",
    "avatarUrl": "https://…",
    "role": "OWNER",
    "loginMethod": "TSIDP",
    "canRegisterPasskey": true
  }
}
```

- 세션이 없으면 `401` — 프론트엔드는 이걸 보고 로그인 화면으로 보낸다
- `canRegisterPasskey`는 `loginMethod == TSIDP`일 때만 `true` (D28)

---

## 4. 사용자 · 역할

### 사용자 객체

```json
{
  "id": 7,
  "login": "teammate@github",
  "displayName": "팀원",
  "avatarUrl": null,
  "role": "VIEWER",
  "disabled": false,
  "createdAt": "2026-10-03T01:00:00Z",
  "lastLoginAt": "2026-10-05T12:30:00Z"
}
```

### #6 `GET /api/users`

- 쿼리: `role`, `disabled` (선택 필터)
- 페이지 없음 (팀 규모). 배열 그대로

### #7 `GET /api/users/{userId}`

없으면 `404`.

### #8 `PATCH /api/users/{userId}`

```json
{ "role": "OPERATOR" }
{ "disabled": true }
```

| 규칙 | 위반 시 |
| --- | --- |
| `role`은 `OPERATOR` 또는 `VIEWER`만. `OWNER`는 #9로만 | `400` |
| 자기 자신은 바꿀 수 없음 | `403` |
| 대상이 Owner면 강등·비활성화 불가 | `409` |

- 성공하면 대상의 기존 세션을 전부 무효화한다 (`04-auth.md` §6)
- 감사 로그에 `USER_ROLE_CHANGED` 또는 `USER_DISABLED` / `USER_ENABLED`
- 응답: 바뀐 사용자 객체

### #9 `POST /api/users/{userId}/transfer-ownership`

- 본문 없음 (JSON `{}`)
- 대상이 새 Owner가 되고, 호출자는 `OPERATOR`가 된다. 한 트랜잭션
- 대상이 비활성화 상태면 `409`
- 감사 로그 `OWNERSHIP_TRANSFERRED`
- 응답: `{ "data": { "newOwner": {사용자}, "previousOwner": {사용자} } }`

> 서버에 접근할 수 있으면 CLI(`beacon admin transfer-owner <login>`)로도 같은 일을 한다. 둘 다 같은 서비스 함수를 부른다.

---

## 5. 웹훅

### 수신처 객체

```json
{
  "id": 2,
  "name": "Cloudflare 상태",
  "slug": "k3v9x2mq8w",
  "url": "https://<Beacon 주소>/webhook/k3v9x2mq8w",
  "createdAt": "2026-10-10T03:00:00Z",
  "lastReceivedAt": "2026-10-11T08:12:44Z"
}
```

`slug`는 추측할 수 없는 무작위 문자열(최소 20자 권장)이다. 외부 서비스에는 인증 수단이 없는 경우가 많아서, URL 자체가 비밀 역할을 한다.

### #10 `GET /api/webhook-sources`

배열 그대로.

### #11 `POST /api/webhook-sources`

```json
{ "name": "Cloudflare 상태" }
```

`201`, 수신처 객체. 감사 로그 `WEBHOOK_SOURCE_CREATED`.

### #12 `DELETE /api/webhook-sources/{sourceId}`

`204`. 이미 받은 이벤트는 남긴다 (출처 이름은 이벤트에 복사해 두므로 삭제 후에도 보인다). 감사 로그 `WEBHOOK_SOURCE_DELETED`.

### #13 `POST /api/webhook-sources/{sourceId}/rotate`

새 slug 발급, 옛 slug는 즉시 `404`. 응답: 수신처 객체. 감사 로그 `WEBHOOK_SOURCE_ROTATED`.

### #14 `POST /webhook/{slug}`

- 본문: 아무 형식. JSON이면 파싱, 아니면 원문 문자열로 저장
- 크기 상한 1 MB, 넘으면 `413`
- 없는 slug면 `404`
- **파싱에 실패해도 `202`.** 외부 서비스가 재시도를 쏟아내지 않게 한다
- 저장: 원본 그대로 + 얕은 정규화 (`title`, `severity`, `region`을 흔한 필드 이름 후보에서 추출. 못 찾으면 `null`)
- 저장 후 `/ws/events`로 `external_event` 푸시

```json
{ "data": { "eventId": 481 } }
```

| 정규화 필드 | 후보 키 (위에서부터 먼저 찾은 것) |
| --- | --- |
| `title` | `title`, `subject`, `summary`, `name`, `incident.name` |
| `severity` | `severity`, `level`, `priority`, `status` |
| `region` | `region`, `zone`, `location`, `datacenter` |

### 이벤트 객체

```json
{
  "id": 481,
  "sourceId": 2,
  "sourceName": "Cloudflare 상태",
  "title": "Elevated error rates",
  "severity": "major",
  "region": "ap-northeast-2",
  "receivedAt": "2026-10-11T08:12:44Z",
  "rawPayload": { "...": "원본 그대로 (단건 조회에서만)" }
}
```

### #15 `GET /api/external-events`

- 쿼리: `sourceId`, `severity`, `region`, `q`(제목·원본 부분 일치), `from`, `to`, `limit`, `cursor`
- 최신순. 목록에서는 `rawPayload`를 빼서 가볍게

### #16 `GET /api/external-events/{eventId}`

`rawPayload` 포함.

---

## 6. 어댑터

어댑터는 코드에 등록된다. API로 새로 만들 수 없다 (D12). 지금 등록된 것은 `webhook` 하나다.

### 어댑터 객체

```json
{
  "id": "webhook",
  "displayName": "외부 웹훅",
  "enabled": true,
  "notify": false,
  "connection": "OK",
  "lastSuccessAt": "2026-10-11T08:12:44Z",
  "capabilities": ["EXTERNAL_EVENTS"]
}
```

| 필드 | 값 |
| --- | --- |
| `connection` | `OK` · `STALE` · `DISCONNECTED` · `DISABLED` |
| `capabilities` | `TARGET_STATUS` · `STATUS_EVENTS` · `MAINTENANCE_READ` · `MAINTENANCE_WRITE` · `EXTERNAL_EVENTS` 중 선언한 것 |
| `notify` | 알림 토글. 기본 `false` (D14) |

### #17 `GET /api/adapters`

배열 그대로.

### #18 `PATCH /api/adapters/{adapterId}`

```json
{ "enabled": false }
{ "notify": true }
```

응답: 어댑터 객체. 감사 로그 `ADAPTER_UPDATED`.

---

## 7. 감사 로그

### 항목 객체

```json
{
  "id": 1024,
  "at": "2026-10-12T05:00:00Z",
  "actor": { "id": 3, "login": "nx@github", "role": "OWNER", "loginMethod": "TSIDP" },
  "action": "USER_ROLE_CHANGED",
  "target": { "type": "USER", "id": "7" },
  "details": { "from": "VIEWER", "to": "OPERATOR" }
}
```

`action` 값: `USER_ROLE_CHANGED` `USER_DISABLED` `USER_ENABLED` `OWNERSHIP_TRANSFERRED` `WEBHOOK_SOURCE_CREATED` `WEBHOOK_SOURCE_DELETED` `WEBHOOK_SOURCE_ROTATED` `ADAPTER_UPDATED` `LOGIN` · (M4부터) `AGENT_TOOL_CALLED` · (M5부터) `AGENT_WRITE_REQUESTED` `AGENT_WRITE_CONFIRMED` `AGENT_WRITE_REJECTED`

보존 무기한 (D19). 수정·삭제 API 없음.

### #19 `GET /api/audit-log`

쿼리: `actorId`, `action`, `from`, `to`, `limit`, `cursor`. 최신순.

---

## 8. 실시간

### #20 `GET /ws/events` (WebSocket)

- 세션 쿠키로 인증. 없으면 업그레이드 거부
- `Origin`이 Beacon 자신이 아니면 거부
- 접속 직후 `snapshot` 한 번, 이후 변화만

```json
{ "type": "snapshot",       "payload": { "summary": { ... }, "adapters": [ ... ] } }
{ "type": "external_event", "payload": { 이벤트 객체, rawPayload 제외 } }
{ "type": "adapter_status", "payload": { 어댑터 객체 } }
{ "type": "target_status",  "payload": { 대상 객체 } }
```

`target_status`는 상태를 선언한 어댑터가 생긴 뒤에야 나온다.

---

## 9. 대상 · 상태 · 점검

원천 어댑터가 없는 지금(D46)은 빈 응답이다. 엔드포인트와 모양만 먼저 정해 둔다 — 자체 모니터 어댑터가 붙으면 코드 변경 없이 채워진다 (D43).

### #21 `GET /api/summary`

```json
{
  "data": {
    "targets": { "total": 0, "up": 0, "down": 0, "degraded": 0, "unknown": 0 },
    "adapters": { "ok": 1, "stale": 0, "disconnected": 0, "disabled": 0 },
    "externalEvents24h": 12,
    "activeMaintenances": 0
  }
}
```

지금 만들 가치가 있는 유일한 이 그룹 엔드포인트다. 대시보드 첫 화면이 이걸 쓴다.

### 대상 객체

```json
{
  "adapterId": "selfmon",
  "sourceId": "42",
  "name": "forgejo",
  "aliases": ["git", "코드 관리 시스템"],
  "status": "UP",
  "responseTimeMs": 143,
  "observedAt": "2026-12-01T03:00:00Z",
  "inMaintenance": false
}
```

### #22 `GET /api/targets`

쿼리: `adapterId`, `status`, `tag`. 배열.

### #23 `GET /api/targets/{adapterId}/{sourceId}`

대상 객체 + `uptime: { "24h": 99.4, "7d": 99.9, "30d": 99.7 }` + `recentEvents`(최근 10건).

### #24 `GET /api/status-events`

쿼리: `adapterId`, `sourceId`, `from`, `to`, `limit`, `cursor`. 항목: `{ adapterId, sourceId, fromStatus, toStatus, occurredAt, resolvedAt }`.

### #25 `GET /api/maintenances`

쿼리: `from`, `to`, `status`(`SCHEDULED` `ACTIVE` `ENDED`). 항목: `{ adapterId, sourceId, title, startsAt, endsAt, targets: [{adapterId, sourceId, name}] }`.

---

## 10. 에이전트 (M4 · M5)

### #26 `POST /api/agent/chat`

```json
{ "conversationId": null, "message": "최근 웹훅 중에 ap-northeast 리전 것만 요약해줘" }
```

`conversationId`가 `null`이면 새 대화. 응답은 `text/event-stream`.

```
event: tool_start
data: {"tool":"search_external_events","arguments":{"region":"ap-northeast"}}

event: tool_end
data: {"tool":"search_external_events","rowCount":4}

event: token
data: {"text":"최근 24시간 동안 "}

event: confirmation_required          ← 쓰기 의도 + OPERATOR 이상일 때만 (M5)
data: {"actionId":"act_9f2c","summary":{...},"expiresAt":"…"}

event: done
data: {"conversationId":"cnv_31a","toolCalls":1}

event: error
data: {"code":"UPSTREAM_UNAVAILABLE","message":"LLM 응답이 없습니다."}
```

- VIEWER의 쓰기 의도 → `confirmation_required` 없이 권한 안내 문장으로 끝
- 점검 쓰기를 선언한 어댑터가 없으면 → "지금은 점검을 만들 수 있는 원천이 없다"는 안내로 끝 (D46)
- LLM 장애는 `error` 이벤트로. 다른 API와 WebSocket에는 영향 없음

> 구현 메모: 브라우저의 `EventSource`는 GET만 된다. POST로 보내고 `fetch` 응답 스트림을 읽는다.

### #27 `POST /api/agent/actions/{actionId}/confirm` (M5)

```json
{ "approved": true }
```

| 경우 | 응답 |
| --- | --- |
| 승인, 실행 성공 | SSE로 결과 이어서 (`token` … `done`) |
| 거부 | `204` |
| 만료 (10분) | `409 ACTION_EXPIRED` |
| 이미 처리됨 | `409 CONFLICT` |
| 원천에 닿지 못함 | `502 UPSTREAM_UNAVAILABLE`, 세션 종료 (재시도는 새 요청) |

요청자 본인만 승인할 수 있다. 다른 사람의 `actionId`면 `404`.

### #28 `GET /api/agent/conversations`

내 대화만. 쿼리 `limit`, `cursor`. 30일 지난 것은 없다 (D19).

### #29 `GET /api/agent/conversations/{conversationId}`

메시지 목록 + 각 답변에 쓰인 도구 호출(도구 이름, 인자, 결과 행 수). 화면의 "근거 보기"가 쓴다. 남의 대화면 `404`.

---

## 11. 패스키 (M6)

WebAuthn 표준 흐름을 따른다. 옵션 발급 → 브라우저 `navigator.credentials` → 검증.

| # | 요청 본문 | 응답 |
| --- | --- | --- |
| 30 | `{ "nickname": "내 휴대폰" }` | `PublicKeyCredentialCreationOptions` (JSON) |
| 31 | 브라우저가 만든 attestation | `201`, 패스키 객체 |
| 32 | `{}` | `PublicKeyCredentialRequestOptions` |
| 33 | 브라우저가 만든 assertion | `204` + 세션 쿠키 (로그인 수단 `PASSKEY`) |

- #30·#31은 `loginMethod == TSIDP` 세션에서만. 패스키 세션이면 `403` (D28)
- #33에서 계정이 비활성화 상태면 `403` (`04-auth.md` §4)
- RP ID는 `04-auth.md` §2의 값으로 고정

패스키 객체: `{ id, nickname, createdAt, lastUsedAt }`.

---

## 12. 질문 (구현하며 확인)

| # | 질문 | 현재 가정 |
| --- | --- | --- |
| Q1 | 사용자 목록을 OPERATOR에게도 보여줄까? | 아니오. OWNER만 |
| Q2 | `LOGIN`을 감사 로그에 남길까? | 남긴다. 누가 언제 어떤 수단으로 들어왔는지 |
| Q3 | 웹훅 수신에 서명 검증(HMAC)을 붙일까? | 지금은 slug만. 서명을 보내는 서비스가 생기면 수신처별 옵션으로 |
