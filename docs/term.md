# 약관

약관은 버전별로 관리하고, 동의도 버전 단위로 기록합니다. 조회하면 약관마다 현재 버전 하나가 나오며, 같은 약관의 같은 주 버전(major)에 동의한 기록이 있으면 동의한 것으로 봅니다. 부 버전(minor)만 바뀐 개정은 다시 동의하지 않아도 됩니다.

개인정보 수집·이용 동의(`PRIVACY_POLICY`)는 Fin-BE V12 마이그레이션에서 v2.0으로 개정되었습니다. 주 버전이 올라갔으므로 v1.0에 동의한 회원도 `agreed`가 `false`가 되고, 다시 동의하기 전에 재발급받은 액세스 토큰으로는 `약관 동의` 엔드포인트를 쓸 수 없습니다. 새 버전의 `effectiveFrom`은 마이그레이션이 적용된 시각입니다.

필수 약관에 모두 동의하면 회원 역할이 `RECOMMENDATION`으로 바뀝니다. 새 권한은 [POST /auth/refresh](auth.md#post-auth-refresh)로 액세스 토큰을 다시 받아야 적용됩니다.

이 문서의 엔드포인트는 응답을 `data`로 감싸지 않습니다.

## GET /term

현재 버전 약관 목록과 내 동의 여부를 조회합니다.

- 인증: 로그인

### 요청

요청 값은 없습니다.

### 응답

`200 OK`. 배열을 그대로 반환합니다. 정렬 순서는 정해져 있지 않습니다.

```json
[
  {
    "id": 1,
    "versionId": 1,
    "code": "SERVICE_TERMS",
    "title": "[필수] Y-Fin. 서비스 이용약관 전문",
    "content": "**제1조 (목적)**\n본 약관은 ...",
    "effectiveFrom": "2026-03-11T15:00:00Z",
    "isRequired": true,
    "agreed": false
  },
  {
    "id": 4,
    "versionId": 4,
    "code": "MARKETING_TERMS",
    "title": "[선택] 마케팅 정보 수신 및 프로파일링 동의 전문",
    "content": "**1. 수집 및 이용 목적**\n...",
    "effectiveFrom": "2026-03-11T15:00:00Z",
    "isRequired": false,
    "agreed": false
  }
]
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `[].id` | number | 약관 ID |
| `[].versionId` | number | 현재 버전 ID. 동의를 보낼 때 이 값을 씁니다. 개정된 약관은 `id`와 값이 다릅니다(마이그레이션을 처음부터 적용한 DB에서 개인정보 v2.0은 `id` 2, `versionId` 5). |
| `[].code` | string | 약관 코드. 현재 데이터는 `SERVICE_TERMS`, `PRIVACY_POLICY`, `LOCATION_SERVICE_TERMS`, `MARKETING_TERMS` |
| `[].title` | string | 약관 제목 |
| `[].content` | string | 약관 본문. 현재 데이터는 마크다운 문법(굵게, 목록, 표)으로 쓰여 있습니다. |
| `[].effectiveFrom` | string (ISO-8601, UTC) | 시행 시각 |
| `[].isRequired` | boolean | 필수 약관 여부 |
| `[].agreed` | boolean | 이 약관의 같은 주 버전에 동의했는지 |

## POST /term

약관 동의 결과를 저장하고, 저장 후 회원 역할을 반환합니다.

- 인증: 로그인

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| body | `agreements` | array<object> | O | 동의 결과 목록. 빈 배열도 받지만 필드를 빼거나 `null`이면 `C003`입니다. |
| body | `agreements[].versionId` | number | O | 약관 버전 ID. [GET /term](#get-term)의 `versionId` |
| body | `agreements[].agreed` | boolean | O | `true`면 동의, `false`면 동의 철회 |

```json
{
  "agreements": [
    { "versionId": 1, "agreed": true },
    { "versionId": 5, "agreed": true },
    { "versionId": 3, "agreed": false },
    { "versionId": 4, "agreed": true }
  ]
}
```

보낸 버전만 바꾸고, 목록에 없는 약관의 기존 동의는 그대로 둡니다. 같은 값을 다시 보내도 오류 없이 저장합니다(동의 시각은 갱신).

저장 뒤 필수 약관에 모두 동의한 상태이고 역할이 `BEFORE_AGREED`이면 `RECOMMENDATION`으로 올립니다. 필수 약관 동의를 철회해도 역할은 내려가지 않습니다. 다만 액세스 토큰의 `약관 동의` 권한은 토큰을 발급할 때 동의 기록으로 다시 판단하므로, 철회 후 재발급받은 토큰으로는 `약관 동의` 엔드포인트를 쓸 수 없습니다.

### 응답

`200 OK`

```json
{ "userRole": "RECOMMENDATION" }
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `userRole` | string | 저장 후 회원 역할. `BEFORE_AGREED`, `RECOMMENDATION`, `ADMIN` 중 하나 |

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 400 | `T005` | `agreements`에 같은 `versionId`가 두 번 이상 있는 경우 |
| 400 | `T004` | 없는 `versionId`가 있는 경우 |
| 404 | `T001` | 현재 버전이 아닌(지난) 약관 버전이 있는 경우 |
| 404 | `U001` | 토큰의 회원이 없는 경우 |
