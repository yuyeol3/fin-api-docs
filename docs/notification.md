# 알림

상품 일정을 이메일로 알려줍니다. 흐름은 세 단계입니다.

1. 이메일 인증: [POST /notifications/email/send-verification](#post-notifications-email-send-verification)으로 인증 메일을 보내고, 사용자가 메일의 링크([GET /notifications/email/verify](#get-notifications-email-verify))를 누르면 인증이 끝납니다. 구글 로그인 사용자는 구글이 확인한 이메일이면 로그인할 때 인증된 것으로 처리합니다.
2. 알림 설정: [POST /notifications/settings](#post-notifications-settings)로 대상과 알림 종류를 등록하면 발송 일정(`schedules`)이 만들어집니다.
3. 발송: 매일 09:00(Asia/Seoul)에 발송일이 오늘이고 설정이 `ACTIVE`인 일정을 보냅니다. 받는 주소는 회원 이메일이고, 이메일이 없거나, 인증 전이거나, 수신거부한 주소면 보내지 않고 일정을 `SKIPPED`로 둡니다.

알림 메일 아래의 수신거부 링크는 [GET /notifications/unsubscribe](#get-notifications-unsubscribe)를 가리킵니다.

이 문서의 엔드포인트는 응답을 `data`로 감싸지 않습니다.

> Fin-BE#51(`27162ee`, 머지 전) 기준입니다.

## POST /notifications/email/send-verification

인증 메일을 보냅니다. 메일의 링크는 서버의 [GET /notifications/email/verify](#get-notifications-email-verify)를 가리키고, 24시간 동안 한 번 쓸 수 있습니다.

- 인증: 로그인

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| body | `email` | string | O | 인증할 이메일. 이메일 형식이어야 합니다. 회원 이메일과 달라도 되고, 인증이 끝나면 회원 이메일이 이 값으로 바뀝니다. |

```json
{ "email": "fin@example.com" }
```

같은 이메일로 60초 안에 다시 보내면 `N003`입니다. 다시 보내도 앞서 보낸 링크는 만료되거나 쓰이기 전까지 유효합니다.

### 응답

`200 OK`. 본문은 없습니다.

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `N001` | 회원을 찾을 수 없음 |
| 429 | `N003` | 같은 이메일로 보낸 지 60초가 지나지 않음 |
| 500 | `N004` | 메일 발송 실패 |

## GET /notifications/email/verify

인증 메일의 링크입니다. 토큰을 확인해 이메일 인증을 끝내고 프론트로 이동시킵니다. XHR이 아니라 브라우저 이동으로 열립니다.

- 인증: 공개

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| query | `token` | string | O | 인증 메일 링크에 담긴 토큰 |

### 응답

`302 Found`. `Location`은 `<프론트 출처>/verify-success`입니다. 회원은 인증된 상태가 되고 회원 이메일은 인증한 이메일로 바뀝니다.

실패하면 이동하지 않고 아래 오류를 JSON으로 반환합니다.

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 400 | `N002` | 없는 토큰, 만료된 토큰, 이미 쓴 토큰 |
| 500 | `C002` | 토큰이 8자보다 짧음 (현재 동작) |

## POST /notifications/settings

알림 설정을 등록하고 발송 일정을 만듭니다.

- 인증: 로그인

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| body | `targetId` | number | O | 알림 대상 ID. 발송 메일은 이 값을 상품 ID로 보고 상품 이름과 상세 링크를 채웁니다. 대상이 있는지는 확인하지 않습니다. |
| body | `targetType` | string | O | `ROUND`, `SUBSCRIPTION` |
| body | `notificationType` | string | O | `DEADLINE`(마감), `START`(시작), `MATURITY`(만기) |
| body | `deadlineDate` | string (yyyy-MM-dd) | X | 마감일. `DEADLINE`이면 필수이고 오늘보다 뒤여야 합니다. 다른 종류에서는 무시합니다. |

```json
{
  "targetId": 101,
  "targetType": "ROUND",
  "notificationType": "DEADLINE",
  "deadlineDate": "2026-12-12"
}
```

같은 회원이 같은 `targetId`, `targetType`, `notificationType`으로 다시 등록하면 `N009`입니다.

발송 일정은 `DEADLINE`일 때만 만듭니다. 마감 7일, 3일, 1일 전 중 발송일이 오늘보다 뒤인 것만 만들므로, 마감이 가까우면 일정이 일부만 생기거나 하나도 없을 수 있습니다. `START`, `MATURITY`는 설정만 저장하고 `schedules`는 빈 배열입니다.

### 응답

`201 Created`

```json
{
  "id": 12,
  "targetId": 101,
  "targetType": "ROUND",
  "notificationType": "DEADLINE",
  "status": "ACTIVE",
  "schedules": [
    { "id": 31, "sendDate": "2026-12-05", "offsetLabel": "D-7", "status": "SCHEDULED" },
    { "id": 32, "sendDate": "2026-12-09", "offsetLabel": "D-3", "status": "SCHEDULED" },
    { "id": 33, "sendDate": "2026-12-11", "offsetLabel": "D-1", "status": "SCHEDULED" }
  ]
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | number | 알림 설정 ID |
| `targetId` | number | 요청한 대상 ID |
| `targetType` | string | `ROUND`, `SUBSCRIPTION` |
| `notificationType` | string | `DEADLINE`, `START`, `MATURITY` |
| `status` | string | `ACTIVE`, `INACTIVE`. 등록 직후는 `ACTIVE` |
| `schedules[].id` | number | 발송 일정 ID |
| `schedules[].sendDate` | string (yyyy-MM-dd) | 발송일 |
| `schedules[].offsetLabel` | string | 마감까지 남은 날. `D-7`, `D-3`, `D-1` |
| `schedules[].status` | string | `SCHEDULED`(대기), `SENDING`(발송 중), `SENT`(발송함), `FAILED`(발송 실패), `SKIPPED`(조건이 안 맞아 건너뜀) |

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 400 | `N010` | `DEADLINE`인데 `deadlineDate`가 없거나 오늘 이전 |
| 404 | `N001` | 회원을 찾을 수 없음 |
| 409 | `N009` | 같은 대상, 같은 종류의 설정이 이미 있음 |

## GET /notifications/settings

내 알림 설정 목록을 조회합니다.

- 인증: 로그인

### 요청

요청 값은 없습니다.

### 응답

`200 OK`. 설정 배열이고, 항목은 [POST /notifications/settings](#post-notifications-settings)의 응답과 같습니다. 설정이 없으면 빈 배열입니다. 정렬 순서는 정하지 않았습니다.

```json
[
  {
    "id": 12,
    "targetId": 101,
    "targetType": "ROUND",
    "notificationType": "DEADLINE",
    "status": "ACTIVE",
    "schedules": [
      { "id": 31, "sendDate": "2026-12-05", "offsetLabel": "D-7", "status": "SENT" }
    ]
  }
]
```

## GET /notifications/settings/{settingId}

알림 설정 하나를 조회합니다.

- 인증: 로그인

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | `settingId` | number | O | 알림 설정 ID |

### 응답

`200 OK`. [POST /notifications/settings](#post-notifications-settings)의 응답과 같습니다.

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `N008` | 없는 설정이거나 다른 회원의 설정 |

## DELETE /notifications/settings/{settingId}

알림 설정을 삭제합니다. 발송 일정도 함께 지웁니다.

- 인증: 로그인

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | `settingId` | number | O | 알림 설정 ID |

### 응답

`204 No Content`

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `N008` | 없는 설정이거나 다른 회원의 설정 |

## PATCH /notifications/settings/{settingId}/toggle

알림 설정의 `status`를 `ACTIVE`와 `INACTIVE` 사이에서 바꿉니다. 발송 일정은 그대로 둡니다.

- 인증: 로그인

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | `settingId` | number | O | 알림 설정 ID |

`INACTIVE`인 동안 발송일이 지난 일정은 다시 `ACTIVE`로 바꿔도 보내지 않습니다(`SCHEDULED`로 남습니다).

### 응답

`200 OK`. 바뀐 설정입니다. 형태는 [POST /notifications/settings](#post-notifications-settings)의 응답과 같습니다.

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `N008` | 없는 설정이거나 다른 회원의 설정 |

## GET /notifications/unsubscribe

알림 메일의 수신거부 링크입니다. 이 이메일 주소로 가는 알림 메일을 모두 멈춥니다. 인증 메일은 막지 않습니다. 브라우저 이동으로 열립니다.

- 인증: 공개

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| query | `email` | string | O | 수신거부할 이메일 |
| query | `sig` | string | O | `email`에 대한 서버 서명. 서버만 만들 수 있으므로 메일 링크에 담긴 값을 그대로 씁니다. |

### 응답

`200 OK`. 본문은 JSON이 아닌 문자열입니다.

```text
수신거부가 완료되었습니다. 더 이상 알림 이메일을 받지 않습니다.
```

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 400 | `N006` | `sig`가 `email`과 맞지 않음 |

## POST /notifications/unsubscribe/cancel

수신거부를 해제합니다. 수신거부한 적이 없는 이메일이어도 성공으로 응답합니다.

- 인증: 공개

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| query | `email` | string | O | 수신거부를 해제할 이메일 |
| query | `sig` | string | O | [GET /notifications/unsubscribe](#get-notifications-unsubscribe)와 같은 서명 |

### 응답

`200 OK`. 본문은 JSON이 아닌 문자열입니다.

```text
수신거부가 해제되었습니다.
```

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 400 | `N006` | `sig`가 `email`과 맞지 않음 |
