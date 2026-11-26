# 관심 상품

사용자가 찜한 상품을 관리합니다. 찜은 상품 옵션(`productPropertyId`) 단위이며, 한 사용자당 최대 20개까지 등록할 수 있습니다.

이 문서의 엔드포인트는 응답을 `data`로 감싸지 않습니다.

## GET /favorites

찜 목록을 조회합니다. 사용자 조건 없이 조회하므로 적합도(`fitScore`)와 지표(`metrics`)는 `null`입니다. 조건을 반영한 값이 필요하면 [POST /favorites/list](#post-favorites-list)를 씁니다.

- 인증: 약관 동의

### 요청

요청 값은 없습니다.

### 응답

`200 OK`. `items`는 찜한 시각 기준 최신순입니다.

```json
{
  "items": [
    {
      "MyfinId": 31,
      "productPropertyId": 1024,
      "sourceCode": "FSS",
      "productCode": "WR0001F",
      "productName": "우리 SUPER주거래 적금",
      "saveTrm": "12",
      "fitScore": null,
      "keywords": ["TERM_12_MONTH", "BENEFIT_MAX_INTEREST", "BANK_SALARY_TRANSFER"],
      "summaryLine": "우리은행 · 1년 만기 · 월 1만원~50만원",
      "metrics": null,
      "calcBasisCaption": "내 조건 기준 달성 가능 금리",
      "excludeFromRateComparison": false,
      "applyStatus": "AVAILABLE",
      "applyUrl": null,
      "officialChannelUrl": "https://spot.wooribank.com/",
      "officialChannelName": "우리은행"
    },
    {
      "MyfinId": 28,
      "productPropertyId": 87,
      "sourceCode": "ONTONG",
      "productCode": "R2024010112345",
      "productName": "청년도약계좌",
      "saveTrm": "60",
      "fitScore": null,
      "keywords": ["BENEFIT_GOV_SUBSIDY", "BENEFIT_TAX_FREE"],
      "summaryLine": "정부 지원 · 5년 만기 · 월 1천원~70만원",
      "metrics": null,
      "calcBasisCaption": "매칭 방식 · 월 납입액 · 만기 기준 산출",
      "excludeFromRateComparison": false,
      "applyStatus": "AVAILABLE",
      "applyUrl": "https://example.go.kr/apply",
      "officialChannelUrl": null,
      "officialChannelName": null
    }
  ],
  "showComparisonNotice": true
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<object> | 찜한 상품 목록. 찜이 없으면 빈 배열 |
| `items[].MyfinId` | number | 찜 ID. 필드 이름이 대문자 `M`으로 시작합니다. |
| `items[].productPropertyId` | number | 상품 옵션 ID. 등록, 삭제, 찜 여부 확인에 쓰는 값 |
| `items[].sourceCode` | string | `ONTONG`(정부 상품) 또는 `FSS`(은행 상품) |
| `items[].productCode` | string | 상품 코드 |
| `items[].productName` | string | 상품명. 끝에 붙은 괄호 표기 하나를 뺀 이름 (예: `헤이(Hey)적금 (자유적립식)` → `헤이(Hey)적금`) |
| `items[].saveTrm` | string | 저축 기간(개월)을 문자열로 담은 값. 없으면 `null` |
| `items[].fitScore` | number | 적합도 점수(정수). 이 엔드포인트에서는 항상 `null` |
| `items[].keywords` | array<string> | 상품 키워드 코드. 아래 키워드 값 목록 참고 |
| `items[].summaryLine` | string | 카드 요약 문구. 아래 요약 문구 형식 참고 |
| `items[].metrics` | object | 수익 지표. 이 엔드포인트에서는 항상 `null` |
| `items[].calcBasisCaption` | string | 지표 계산 기준 문구. 은행 상품은 `내 조건 기준 달성 가능 금리`, 정부 상품은 `매칭 방식 · 월 납입액 · 만기 기준 산출` |
| `items[].excludeFromRateComparison` | boolean | 금리 비교에서 빼야 하는 상품인지 |
| `items[].applyStatus` | string | `AVAILABLE`(가입 가능) 또는 `RECRUIT_CLOSED`(모집 마감) |
| `items[].applyUrl` | string | 상품 자체 신청 URL. 없으면 `null` |
| `items[].officialChannelUrl` | string | `applyUrl`이 없을 때 대신 안내할 기관 공식 채널 URL. 없으면 `null` |
| `items[].officialChannelName` | string | 공식 채널 버튼에 쓸 기관명. `officialChannelUrl`과 짝 |
| `showComparisonNotice` | boolean | 정부 상품과 은행 상품이 섞여 있으면 `true` |

신청 링크는 셋 중 하나만 채워집니다. 모집 마감이면 셋 다 `null`, 상품 자체 신청 URL이 있으면 `applyUrl`만, 없으면 기관 공식 채널(`officialChannelUrl`, `officialChannelName`)을, 그것도 없으면 셋 다 `null`입니다.

**키워드 값**

`BANK_ETC`를 뺀 아래 값이 나올 수 있으며, 순서는 이 목록 순서를 따릅니다.

- 거주 지역: `REGION_SEOUL`, `REGION_BUSAN`, `REGION_DAEGU`, `REGION_INCHEON`, `REGION_GWANGJU`, `REGION_DAEJEON`, `REGION_ULSAN`, `REGION_SEJONG`, `REGION_GYEONGGI`, `REGION_GANGWON`, `REGION_CHUNGBUK`, `REGION_CHUNGNAM`, `REGION_JEONBUK`, `REGION_JEONNAM`, `REGION_GYEONGBUK`, `REGION_GYEONGNAM`, `REGION_JEJU`
- 신분 (가입 필수 조건): `STATUS_UNEMPLOYED`, `STATUS_PART_TIME`, `STATUS_SME_WORKER`, `STATUS_MILITARY`
- 저축 기간: `TERM_1_MONTH`, `TERM_3_MONTH`, `TERM_6_MONTH`, `TERM_12_MONTH`, `TERM_24_MONTH`, `TERM_36_MONTH`, `TERM_OVER_3_YEARS`, `TERM_2_TO_3_YEARS`, `TERM_AROUND_1_YEAR` (뒤의 셋은 이전 형식)
- 핵심 혜택: `BENEFIT_MAX_INTEREST`, `BENEFIT_TAX_FREE`, `BENEFIT_EASY_CONDITION`, `BENEFIT_GOV_SUBSIDY`
- 상품 관심사: `INTEREST_SAVINGS`, `INTEREST_LOAN`
- 은행 우대 조건: `BANK_FIRST_TRANSACTION`, `BANK_SALARY_TRANSFER`, `BANK_CARD_USAGE`, `BANK_AUTO_TRANSFER`, `BANK_MARKETING`, `BANK_REDEPOSIT`, `BANK_ONLINE_JOIN`, `BANK_AGE`

**요약 문구**

`<앞머리>[ · <만기>][ · <금액 범위>]` 형식입니다.

- 앞머리: 정부 상품은 `정부 지원`, 은행 상품은 은행명
- 만기: `saveTrm`이 12의 배수면 `N년 만기`, 아니면 `N개월 만기`. `saveTrm`이 없으면 생략
- 금액 범위: 예금은 `예치 `, 그 밖의 상품은 `월 `로 시작하고 `최소~최대`를 붙입니다. 한쪽만 있으면 그쪽만 씁니다(예: `월 ~50만원`). 둘 다 없으면 생략
- 금액 표기: 1만원 이상은 만원 단위(`50만원`), 1천원 이상은 천원 단위(`5천원`), 그 밖은 원 단위. 아래 자리는 버립니다.

## POST /favorites/list

사용자 조건을 반영해 찜 목록을 조회합니다. 상품 검색과 같은 방식으로 적합도와 수익 지표를 다시 계산합니다.

- 인증: 약관 동의

### 요청

본문은 [POST /search/products](search.md#post-search-products)와 같은 형식입니다. 본문을 생략할 수 있고, 생략하면 [GET /favorites](#get-favorites)와 같은 결과를 반환합니다. 이 엔드포인트는 입력 검증을 하지 않으므로 필드는 모두 생략할 수 있습니다.

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| body | `options` | array<object> | X | 선택한 카테고리 옵션 목록 |
| body | `options[].categoryId` | number | X | 카테고리 ID. [GET /api/categories](category.md#get-api-categories)의 `categoryId` |
| body | `options[].optionId` | number | X | 옵션 ID. 같은 응답의 `optionId` |
| body | `detailedOptions` | object | X | 상세 조건 |
| body | `detailedOptions.birthdate` | string (yyyy-MM-dd) | X | 생년월일 |
| body | `detailedOptions.annualIncome` | number | X | 연소득 |
| body | `detailedOptions.householdSize` | number | X | 가구원 수 |
| body | `detailedOptions.householdIncomePercent` | number | X | 기준 중위소득 대비 가구소득 구간(%) |
| body | `detailedOptions.tenureMonths` | number | X | 재직 기간(개월) |
| body | `detailedOptions.isFirstJob` | boolean | X | 첫 직장 여부 |
| body | `detailedOptions.isHomeless` | boolean | X | 무주택 여부 |
| body | `detailedOptions.isHouseholder` | boolean | X | 세대주 여부 |
| body | `detailedOptions.monthlySavingsGoal` | number | X | 월 저축 가능액(원). 목돈 만들기용 |
| body | `detailedOptions.depositAmount` | number | X | 예치 희망액(원). 단기 예치용 |
| body | `detailedOptions.neverUsedBanks` | array<string> | X | 거래한 적 없는 은행 코드 목록(첫거래 대상) |
| body | `detailedOptions.maturedSavingBanks` | array<string> | X | 만기 예적금이 있는 은행 코드 목록(재예치 대상) |
| body | `detailedOptions.selectedInterestRateOptions` | array<object> | X | 현재 쓰지 않는 필드 |

```json
{
  "options": [
    { "categoryId": 1, "optionId": 1 },
    { "categoryId": 3, "optionId": 40 },
    { "categoryId": 6, "optionId": 32 }
  ],
  "detailedOptions": {
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
    "maturedSavingBanks": ["0010001"]
  }
}
```

은행 상품의 적합도에 거래 이력(첫거래, 재예치)을 반영하는 조건은 상품 검색과 같습니다. `monthlySavingsGoal`, 저축 기간 옵션, 은행 우대 조건 옵션 1개 이상, `birthdate`, `annualIncome`, `householdSize`, `householdIncomePercent`, `neverUsedBanks`, `maturedSavingBanks`가 모두 있어야 반영합니다. 정부 상품은 거래 이력을 반영하지 않습니다.

없는 `optionId`는 오류 없이 무시합니다.

### 응답

`200 OK`. 형식은 [GET /favorites](#get-favorites)와 같고, 본문을 보내면 아래 필드가 채워집니다.

```json
{
  "items": [
    {
      "MyfinId": 31,
      "productPropertyId": 1024,
      "sourceCode": "FSS",
      "productCode": "WR0001F",
      "productName": "우리 SUPER주거래 적금",
      "saveTrm": "12",
      "fitScore": 82,
      "keywords": ["TERM_12_MONTH", "BENEFIT_MAX_INTEREST", "BANK_SALARY_TRANSFER"],
      "summaryLine": "우리은행 · 1년 만기 · 월 1만원~50만원",
      "metrics": {
        "contributionYieldRate": null,
        "expectedMaturityAmount": null,
        "baseRate": 2.8,
        "maxRate": 4.5,
        "achievableRate": 3.6
      },
      "calcBasisCaption": "내 조건 기준 달성 가능 금리",
      "excludeFromRateComparison": false,
      "applyStatus": "AVAILABLE",
      "applyUrl": null,
      "officialChannelUrl": "https://spot.wooribank.com/",
      "officialChannelName": "우리은행"
    },
    {
      "MyfinId": 28,
      "productPropertyId": 87,
      "sourceCode": "ONTONG",
      "productCode": "R2024010112345",
      "productName": "청년도약계좌",
      "saveTrm": "60",
      "fitScore": 74,
      "keywords": ["BENEFIT_GOV_SUBSIDY", "BENEFIT_TAX_FREE"],
      "summaryLine": "정부 지원 · 5년 만기 · 월 1천원~70만원",
      "metrics": {
        "contributionYieldRate": 3.1,
        "expectedMaturityAmount": 1440000,
        "baseRate": null,
        "maxRate": null,
        "achievableRate": null
      },
      "calcBasisCaption": "월 50만원 · 60개월 기준",
      "excludeFromRateComparison": false,
      "applyStatus": "AVAILABLE",
      "applyUrl": "https://example.go.kr/apply",
      "officialChannelUrl": null,
      "officialChannelName": null
    }
  ],
  "showComparisonNotice": true
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `items[].fitScore` | number | 적합도 점수. 계산값의 소수점 아래를 버린 정수 |
| `items[].metrics` | object | 수익 지표. 정부 상품은 앞의 두 필드만, 은행 상품은 뒤의 세 필드만 채우고 나머지는 `null` |
| `items[].metrics.contributionYieldRate` | number | 정부 상품: 기여금 연 환산 수익률(%) |
| `items[].metrics.expectedMaturityAmount` | number | 정부 상품: 예상 만기 기여금 총액(원) |
| `items[].metrics.baseRate` | number | 은행 상품: 기본금리 |
| `items[].metrics.maxRate` | number | 은행 상품: 최고금리 |
| `items[].metrics.achievableRate` | number | 은행 상품: 내 조건으로 받을 수 있는 금리 |
| `items[].calcBasisCaption` | string | 정부 상품이고 `detailedOptions.monthlySavingsGoal`이 있으면 `월 <금액> · <saveTrm>개월 기준`(`saveTrm`이 없으면 `월 <금액> · 만기 기준`). 그 밖은 GET /favorites와 같습니다. |

나머지 필드는 GET /favorites와 같습니다.

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `S001` | `options[].categoryId`가 없는 카테고리이거나 `optionId`가 속한 카테고리와 다른 경우 |

## POST /favorites

상품을 찜합니다.

- 인증: 약관 동의

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| body | `productPropertyId` | number | O | 찜할 상품 옵션 ID |

```json
{ "productPropertyId": 1024 }
```

### 응답

`201 Created`. 본문은 없습니다.

### 오류

아래 순서로 확인합니다. 이미 찜한 상품을 다시 보내면 찜 개수와 관계없이 `MF002`입니다.

| 상태 | 코드 | 상황 |
|---|---|---|
| 409 | `MF002` | 이미 찜한 상품인 경우 |
| 409 | `MF003` | 이미 20개를 찜한 경우 |
| 404 | `MF004` | 없는 상품이거나 모집이 끝나 가입할 수 없는 상품인 경우 |
| 404 | `U001` | 토큰의 회원이 없는 경우 |

## DELETE /favorites/{productPropertyId}

찜을 해제합니다.

- 인증: 약관 동의

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | `productPropertyId` | number | O | 찜을 해제할 상품 옵션 ID. 찜 ID(`MyfinId`)가 아닙니다. |

### 응답

`204 No Content`

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `MF001` | 찜하지 않은 상품인 경우 |

## GET /favorites/{productPropertyId}/status

상품을 찜했는지 확인합니다.

- 인증: 약관 동의

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | `productPropertyId` | number | O | 확인할 상품 옵션 ID |

### 응답

`200 OK`. 객체가 아니라 boolean 값 하나를 반환합니다. 없는 상품 ID여도 오류 없이 `false`입니다.

```json
true
```

## GET /favorites/count

찜한 상품 개수를 조회합니다.

- 인증: 약관 동의

### 요청

요청 값은 없습니다.

### 응답

`200 OK`. 객체가 아니라 숫자 하나를 반환합니다.

```json
3
```
