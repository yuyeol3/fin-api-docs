# 회원

로그인한 사용자의 계정 정보(`/user/me`)와 금융 프로필(`/user/me/profile`)을 다룹니다. 금융 프로필은 정보 입력 화면에서 저장하고 마이페이지에서 조회하는 값입니다.

이 문서의 엔드포인트는 응답을 `data`로 감싸지 않고 객체를 그대로 반환합니다.

## GET /user/me

내 계정 정보를 조회합니다.

- 인증: 로그인

### 요청

요청 값은 없습니다.

### 응답

`200 OK`

```json
{
  "id": 12,
  "email": "user@example.com",
  "name": "김핀",
  "userRole": "RECOMMENDATION"
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | number | 회원 ID |
| `email` | string | 이메일. 값이 없으면 `null` |
| `name` | string | 이름 |
| `userRole` | string | `BEFORE_AGREED`(필수 약관 미동의), `RECOMMENDATION`(필수 약관 동의), `ADMIN` 중 하나. DB에 저장된 역할이라, 약관 동의 직후처럼 액세스 토큰을 아직 재발급받지 않았으면 토큰의 권한과 다를 수 있습니다. |

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `U001` | 토큰의 회원이 없는 경우 (예: 탈퇴 후 남은 액세스 토큰으로 호출) |

## PATCH /user/me

이메일이나 이름을 수정합니다. 보낸 필드만 바꾸고, 빠지거나 `null`인 필드는 그대로 둡니다.

- 인증: 로그인

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| body | `email` | string | X | 새 이메일 |
| body | `name` | string | X | 새 이름 |

```json
{ "name": "김핀" }
```

형식 검사(이메일 형식, 빈 문자열)는 하지 않습니다. 이메일은 다른 회원과 겹치는지만 확인하며, 지금 내 이메일과 같은 값을 보내도 중복으로 처리합니다.

### 응답

`200 OK`. 본문은 없습니다.

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `U001` | 토큰의 회원이 없는 경우 |
| 409 | `U002` | 보낸 `email`을 이미 사용하는 회원이 있는 경우 (내 현재 이메일 포함) |

## DELETE /user/me

회원 탈퇴합니다. 회원과 함께 리프레시 토큰, 금융 프로필, 약관 동의 기록, 관심 상품이 모두 삭제됩니다.

- 인증: 로그인

### 요청

요청 값은 없습니다.

### 응답

`204 No Content`. `Set-Cookie`로 `refresh_token`을 `Max-Age=0`으로 내려 쿠키를 지웁니다.

이미 발급된 액세스 토큰은 만료될 때까지 인증을 통과하지만, 회원을 조회하는 엔드포인트는 `U001`을 반환합니다.

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `U001` | 토큰의 회원이 없는 경우 (이미 탈퇴한 경우 포함) |

## GET /user/me/profile

저장된 금융 프로필을 조회합니다. 저장한 원본 값(정보 입력 화면 프리필용)과 서버가 계산한 표시 값(`display`, 마이페이지용)을 함께 반환합니다.

- 인증: 약관 동의

### 요청

요청 값은 없습니다.

### 응답

`200 OK`

```json
{
  "hasProfile": true,
  "birthdate": "1998-05-14",
  "annualIncome": 3600,
  "householdSize": 1,
  "householdIncomePercent": 100,
  "tenureMonths": 24,
  "isFirstJob": true,
  "isHomeless": true,
  "isHouseholder": true,
  "monthlySavingsGoal": 500000,
  "neverUsedBanks": ["0011625"],
  "maturedSavingBanks": ["0010001"],
  "selectedOptionIds": [1, 32],
  "display": {
    "age": 28,
    "householdIncomeGuide": 256,
    "region": "서울",
    "preferentialConditions": ["급여이체 우대"],
    "transactionHistory": {
      "firstTransactionBanks": ["신한은행"],
      "redepositBanks": ["우리은행"]
    }
  }
}
```

`birthdate`부터 `selectedOptionIds`까지는 [PUT /user/me/profile](#put-user-me-profile)로 저장한 값을 그대로 돌려주며, 저장하지 않은 값은 `null`입니다. 저장된 프로필이 없으면 `hasProfile`만 `false`이고 나머지 필드는 모두 `null`입니다.

```json
{
  "hasProfile": false,
  "birthdate": null,
  "annualIncome": null,
  "householdSize": null,
  "householdIncomePercent": null,
  "tenureMonths": null,
  "isFirstJob": null,
  "isHomeless": null,
  "isHouseholder": null,
  "monthlySavingsGoal": null,
  "neverUsedBanks": null,
  "maturedSavingBanks": null,
  "selectedOptionIds": null,
  "display": null
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `hasProfile` | boolean | 저장된 프로필이 있는지 |
| `birthdate` | string (yyyy-MM-dd) | 생년월일 |
| `annualIncome` | number | 연소득 |
| `householdSize` | number | 가구원 수 |
| `householdIncomePercent` | number | 기준 중위소득 대비 가구소득 구간(%) |
| `tenureMonths` | number | 재직 기간(개월) |
| `isFirstJob` | boolean | 첫 직장 여부 |
| `isHomeless` | boolean | 무주택 여부 |
| `isHouseholder` | boolean | 세대주 여부 |
| `monthlySavingsGoal` | number | 월 저축 가능액(원) |
| `neverUsedBanks` | array<string> | 첫거래 대상 은행 코드 목록 |
| `maturedSavingBanks` | array<string> | 재예치 대상 은행 코드 목록 |
| `selectedOptionIds` | array<number> | 선택한 카테고리 옵션 ID 목록 |
| `display` | object | 서버 계산 표시 값 |
| `display.age` | number | 만 나이. 조회일(Asia/Seoul) 기준. `birthdate`가 없으면 `null` |
| `display.householdIncomeGuide` | number | 가구소득 금액 가이드(만원). 조회 연도의 `householdSize`인 가구 기준 중위소득의 `householdIncomePercent`% 금액. 두 값 중 하나가 없거나, `householdIncomePercent`가 60, 80, 100, 120, 150, 180이 아니거나, 해당 연도 중위소득 데이터가 없으면 `null` |
| `display.region` | string | `selectedOptionIds` 중 거주 지역 옵션의 이름. 없으면 `null` |
| `display.preferentialConditions` | array<string> | `selectedOptionIds` 중 은행 우대 조건 옵션의 이름. 없으면 빈 배열 |
| `display.transactionHistory.firstTransactionBanks` | array<string> | `neverUsedBanks`의 은행 코드를 은행명으로 바꾼 값. 이름을 찾지 못한 코드는 코드 그대로 둡니다. `neverUsedBanks`가 `null`이면 `null` |
| `display.transactionHistory.redepositBanks` | array<string> | `maturedSavingBanks`를 같은 방식으로 바꾼 값 |

## PUT /user/me/profile

금융 프로필을 저장합니다. 없으면 만들고, 있으면 전체를 덮어씁니다. 빠진 필드는 `null`로 저장되므로 화면의 모든 값을 함께 보냅니다.

- 인증: 약관 동의

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| body | `birthdate` | string (yyyy-MM-dd) | X | 생년월일 |
| body | `annualIncome` | number | X | 연소득 |
| body | `householdSize` | number | X | 가구원 수 |
| body | `householdIncomePercent` | number | X | 기준 중위소득 대비 가구소득 구간(%). 표시 값 계산은 60, 80, 100, 120, 150, 180만 지원 |
| body | `tenureMonths` | number | X | 재직 기간(개월) |
| body | `isFirstJob` | boolean | X | 첫 직장 여부 |
| body | `isHomeless` | boolean | X | 무주택 여부 |
| body | `isHouseholder` | boolean | X | 세대주 여부 |
| body | `monthlySavingsGoal` | number | X | 월 저축 가능액(원) |
| body | `neverUsedBanks` | array<string> | X | 거래한 적 없는 은행 코드 목록(첫거래 대상). 코드는 [GET /providers/banks](provider.md#get-providers-banks) 기준 |
| body | `maturedSavingBanks` | array<string> | X | 만기 예적금이 있는 은행 코드 목록(재예치 대상) |
| body | `selectedOptionIds` | array<number> | X | 선택한 카테고리 옵션 ID 목록. [GET /api/categories](category.md#get-api-categories)의 `optionId` |

```json
{
  "birthdate": "1998-05-14",
  "annualIncome": 3600,
  "householdSize": 1,
  "householdIncomePercent": 100,
  "tenureMonths": 24,
  "isFirstJob": true,
  "isHomeless": true,
  "isHouseholder": true,
  "monthlySavingsGoal": 500000,
  "neverUsedBanks": ["0011625"],
  "maturedSavingBanks": ["0010001"],
  "selectedOptionIds": [1, 32]
}
```

값 범위나 옵션 ID, 은행 코드가 실제로 있는지는 저장할 때 확인하지 않습니다.

### 응답

`200 OK`. 본문은 없습니다.

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `U001` | 프로필을 처음 만들 때 토큰의 회원이 없는 경우 |

## DELETE /user/me/profile

저장된 금융 프로필을 삭제합니다. 프로필이 없어도 같은 응답을 반환합니다.

- 인증: 약관 동의

### 요청

요청 값은 없습니다.

### 응답

`200 OK`. 본문은 없습니다.
