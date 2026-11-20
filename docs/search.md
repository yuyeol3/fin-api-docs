# 상품 검색

추천 입력 폼, 조건 기반 상품 추천, 상품명 검색, 상품 상세를 다룹니다. 이 문서의 엔드포인트는 모두 `data`로 감싸지 않고 결과를 그대로 반환합니다.

추천 입력은 두 부분으로 나뉩니다.

- `options`: 1단계(STEP1)에서 고른 옵션입니다. [`GET /api/categories`](category.md#get-api-categories)로 받은 `categoryId`, `optionId` 쌍을 그대로 담습니다.
- `detailedOptions`: 2단계(STEP2) 상세 입력입니다. 필드 구성은 [`POST /search/products`](#post-search-products)의 요청 표에 적었습니다.

상품은 하나 이상의 옵션(`productPropertyId`)을 가집니다. 옵션은 저축기간, 금리, 가입 조건이 다른 같은 상품의 변형입니다. 목록 응답의 `productPropertyId`는 그 행이 어느 옵션 기준인지를 뜻하고, 상세 조회 때 그대로 넘깁니다.

| 값 | 의미 |
|---|---|
| `source`, `sourceCode` = `ONTONG` | 정부 상품 (온통청년) |
| `source`, `sourceCode` = `FSS` | 은행 상품 (금융감독원 공시) |
| `productType` = `DEPOSIT`, `SAVING`, `PARKING`, `POLICY`, `SUBSCRIPTION` | 예금, 적금, 파킹통장, 정책상품, 청약 |

`POST /search/products`와 상품 상세는 공개 엔드포인트지만 토큰이 있으면 개인화 지표(실수령액, 내가 받을 수 있는 금리, 기여금 등)를 더 채웁니다. 토큰이 만료되었거나 잘못되어도 401을 내지 않고 비로그인 요청으로 처리합니다. 개인화는 아래 조건을 모두 만족할 때만 켜집니다.

| 조건 | 목돈만들기, 상품 상세 | 단기예치 |
|---|---|---|
| 토큰 | 유효한 액세스 토큰, 권한 `RECOMMENDATION` 또는 `ADMIN` (약관 동의) | 같음 |
| 저축기간 옵션 | 선택 | 선택 |
| 은행 우대 조건 옵션 (`categoryId` 6) | 1개 이상 | 1개 이상 |
| 금액 | `monthlySavingsGoal` 있음 | `depositAmount`가 0보다 큼 |
| 거래 이력 | `neverUsedBanks`, `maturedSavingBanks`가 null이 아님 (빈 배열은 입력한 것으로 봄) | 같음 |
| 그 밖의 상세 입력 | `birthdate`, `annualIncome`, `householdSize`, `householdIncomePercent`가 모두 있음 | `birthdate` 있음 |

키워드 필드(`keywords`, `badgeKeywords`, `keywordCode`)에는 아래 코드가 나옵니다.

| 코드 | 의미 |
|---|---|
| `REGION_SEOUL`, `REGION_BUSAN`, `REGION_DAEGU`, `REGION_INCHEON`, `REGION_GWANGJU`, `REGION_DAEJEON`, `REGION_ULSAN`, `REGION_SEJONG`, `REGION_GYEONGGI`, `REGION_GANGWON`, `REGION_CHUNGBUK`, `REGION_CHUNGNAM`, `REGION_JEONBUK`, `REGION_JEONNAM`, `REGION_GYEONGBUK`, `REGION_GYEONGNAM`, `REGION_JEJU` | 거주 지역 |
| `STATUS_UNEMPLOYED`, `STATUS_PART_TIME`, `STATUS_SME_WORKER`, `STATUS_MILITARY` | 현재 신분: 미취업, 알바/프리랜서, 중소기업 재직, 군복무 |
| `TERM_1_MONTH`, `TERM_3_MONTH`, `TERM_6_MONTH`, `TERM_12_MONTH`, `TERM_24_MONTH`, `TERM_36_MONTH` | 저축기간 1, 3, 6, 12, 24, 36개월 |
| `TERM_AROUND_1_YEAR`, `TERM_2_TO_3_YEARS`, `TERM_OVER_3_YEARS` | 예전 범위형 저축기간 (1년 내외, 2~3년, 3년 이상). 옵션 목록에서는 숨겼지만 상품 키워드에는 남아 있습니다. |
| `BENEFIT_MAX_INTEREST`, `BENEFIT_TAX_FREE`, `BENEFIT_EASY_CONDITION`, `BENEFIT_GOV_SUBSIDY` | 핵심 혜택: 최고이율 중심, 비과세, 우대조건 간편, 정부기여금 |
| `INTEREST_SAVINGS`, `INTEREST_LOAN` | 상품 관심사: 저축, 대출 |
| `BANK_FIRST_TRANSACTION`, `BANK_SALARY_TRANSFER`, `BANK_CARD_USAGE`, `BANK_AUTO_TRANSFER`, `BANK_MARKETING`, `BANK_REDEPOSIT` | 은행 우대 조건: 첫거래, 급여이체, 카드실적, 자동이체, 마케팅 동의, 재예치 |
| `BANK_ONLINE_JOIN`, `BANK_AGE`, `BANK_ETC` | 은행 우대 조건: 비대면 가입, 연령, 기타(분류되지 않은 우대금리) |

은행 상품의 `BENEFIT_MAX_INTEREST`는 저장된 태그가 아니라 응답 시점에 계산합니다. 비교 대상 은행 옵션들의 최고금리 중 상위 30% 경계값 이상이면 붙습니다.

## POST /search/dynamic-form

1단계 선택과 가구원 수에 따라 2단계 입력 폼의 표시 조건과 기본값을 계산합니다.

- 인증: 공개

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| body | `options` | array<object> | O | 1단계에서 고른 옵션. 빈 배열도 됩니다. |
| body | `options[].categoryId` | number | O | 카테고리 ID. 검증만 하고 계산에는 쓰지 않습니다. |
| body | `options[].optionId` | number | O | 옵션 ID. 키워드와 연결되지 않은 ID는 무시합니다. |
| body | `detailedOptions` | object | O | 2단계 입력. 빈 객체 `{}`도 됩니다. 구조는 [`POST /search/products`](#post-search-products)와 같지만 이 엔드포인트는 `householdSize`만 씁니다. |
| body | `detailedOptions.householdSize` | number | X | 가구원 수. 보내면 응답에 기준 중위소득을 담습니다. |

```json
{
  "options": [
    { "categoryId": 2, "optionId": 18 }
  ],
  "detailedOptions": { "householdSize": 2 }
}
```

### 응답

`200 OK`. `data`로 감싸지 않고 그대로 반환합니다.

```json
{
  "showTenure": false,
  "ageBound": 34,
  "yearlyEarnDefault": 0,
  "medianIncomes": {
    "year": 2026,
    "householdSize": 2,
    "p60": 252,
    "p80": 336,
    "p100": 420,
    "p120": 504,
    "p150": 630,
    "p180": 756,
    "empty": false
  },
  "preferentialInterestRateOptions": []
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `showTenure` | boolean | 재직 기간 입력을 보여줄지 여부. 현재신분 `미취업`(`STATUS_UNEMPLOYED`) 옵션을 고르면 `false`, 아니면 `true` |
| `ageBound` | number | 나이 경계값. 항상 `34`입니다. |
| `yearlyEarnDefault` | number | 연소득 기본값. `미취업`을 고르면 `0`, 아니면 null |
| `medianIncomes` | object | 기준 중위소득. `householdSize`를 보냈을 때만 채우고, 아니면 null |
| `medianIncomes.year` | number | 기준 연도. 요청 시점의 올해(KST)입니다. |
| `medianIncomes.householdSize` | number | 가구원 수 |
| `medianIncomes.p60` | number | 기준 중위소득 60%의 월 소득 |
| `medianIncomes.p80` | number | 기준 중위소득 80%의 월 소득 |
| `medianIncomes.p100` | number | 기준 중위소득 100%의 월 소득 |
| `medianIncomes.p120` | number | 기준 중위소득 120%의 월 소득 |
| `medianIncomes.p150` | number | 기준 중위소득 150%의 월 소득 |
| `medianIncomes.p180` | number | 기준 중위소득 180%의 월 소득 |
| `medianIncomes.empty` | boolean | `p60`~`p180`이 모두 0이면 `true`. 해당 연도와 가구원 수의 데이터가 없으면 모든 값을 0으로 채우고 `true`가 됩니다. |
| `preferentialInterestRateOptions` | array<object> | 항상 빈 배열입니다. |

`p60`~`p180`은 DB 값을 그대로 내려주며, 마이그레이션에 넣은 데이터는 만원 단위입니다.

## POST /search/products

1단계, 2단계 입력으로 가입 가능한 상품을 찾아 추천 목록을 만듭니다. 저축기간에 따라 대분류가 정해지고, 대분류별로 채우는 필드가 다릅니다.

- 인증: 공개 (토큰이 있고 개인화 조건을 만족하면 개인화 지표를 채웁니다. 문서 첫머리의 조건 표 참고)

| 대분류 | 조건 | 채우는 필드 |
|---|---|---|
| `LONG_TERM` (목돈만들기) | 저축기간이 6개월 이상이거나 선택하지 않음 | `governmentRanked`, `bankRanked`, `governmentTop3`, `bankTop3`, `governmentRateRanked`, `bankRateRanked`, `subscriptionProducts`, `productCardSummaries` |
| `SHORT_TERM` (단기예치) | 저축기간 1개월 또는 3개월 | `parkingProducts`, `depositSavingsProducts`, `productCardSummaries`(항상 빈 배열) |

다른 대분류의 목록 필드는 null입니다.

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| body | `options` | array<object> | O | 1단계에서 고른 옵션. 빈 배열도 됩니다. |
| body | `options[].categoryId` | number | O | 옵션이 속한 카테고리 ID |
| body | `options[].optionId` | number | O | 옵션 ID |
| body | `detailedOptions` | object | O | 2단계 입력. 빈 객체 `{}`도 됩니다. |
| body | `detailedOptions.birthdate` | string (yyyy-MM-dd) | X | 생년월일. 오늘(KST) 기준 만 나이로 상품의 나이 조건을 거릅니다. |
| body | `detailedOptions.annualIncome` | number | X | 연소득(원). 상품의 소득 상한보다 많으면 제외합니다. `0`이면 소득 증빙이 어려운 것으로 보고 소득 요건(소득 상한이나 중위소득 비율)이 있는 상품을 모두 제외합니다. |
| body | `detailedOptions.householdSize` | number | X | 가구원 수. 개인화 조건 판단에만 씁니다. |
| body | `detailedOptions.householdIncomePercent` | number | X | 가구 소득의 기준 중위소득 대비 비율(%). 상품이 허용하는 비율보다 높으면 제외합니다. |
| body | `detailedOptions.tenureMonths` | number | X | 재직 개월 수. 상품의 최소 재직 기간보다 짧으면 제외합니다. 현재신분 `미취업`을 고르면 이 값 대신 0으로 봅니다. |
| body | `detailedOptions.isFirstJob` | boolean | X | 첫 직장 여부. 검색에는 쓰지 않습니다. |
| body | `detailedOptions.isHomeless` | boolean | X | 무주택 여부. `false`면 무주택 요건이 있는 상품을 제외하고, 보내지 않으면 거르지 않습니다. |
| body | `detailedOptions.isHouseholder` | boolean | X | 세대주 여부. `false`면 세대주 요건이 있는 상품을 제외하고, 보내지 않으면 거르지 않습니다. |
| body | `detailedOptions.monthlySavingsGoal` | number | X | 월 납입 희망액(원). 목돈만들기에서는 필수입니다(`S004`). |
| body | `detailedOptions.depositAmount` | number | X | 예치 희망액(원). 단기예치에서는 필수이고 0보다 커야 합니다(`S007`). |
| body | `detailedOptions.neverUsedBanks` | array<string> | X | 거래한 적 없는 은행의 금융사 코드([`GET /providers/banks`](provider.md#get-providers-banks)의 `code`). 첫거래 우대 판단에 씁니다. |
| body | `detailedOptions.maturedSavingBanks` | array<string> | X | 만기된 예적금이 있는 은행의 금융사 코드. 재예치 우대 판단에 씁니다. |
| body | `detailedOptions.selectedInterestRateOptions` | array<object> | X | 필드가 없는 객체의 배열. 받기만 하고 쓰지 않습니다. |

`options` 처리 규칙입니다.

- 키워드와 연결되지 않은 `optionId`(없는 ID 등)는 오류 없이 무시합니다.
- 키워드와 연결된 옵션인데 `categoryId`가 1~6이 아니거나 옵션의 실제 카테고리와 다르면 `S001`입니다.
- 저축기간 옵션은 하나만 반영합니다. 여러 개를 보내면 마지막 값을 씁니다.
- 상품관심사(`categoryId` 5) 옵션은 무시합니다.
- 목록에서 숨긴 옵션도 ID로 보내면 반영합니다. 범위형 저축기간 `3년 이상`(`TERM_OVER_3_YEARS`)은 60개월로 환산되어 `S009`가 납니다.

```json
{
  "options": [
    { "categoryId": 1, "optionId": 2 },
    { "categoryId": 2, "optionId": 20 },
    { "categoryId": 3, "optionId": 40 },
    { "categoryId": 4, "optionId": 28 },
    { "categoryId": 6, "optionId": 32 }
  ],
  "detailedOptions": {
    "birthdate": "2000-05-01",
    "annualIncome": 30000000,
    "householdSize": 1,
    "householdIncomePercent": 100,
    "tenureMonths": 12,
    "isFirstJob": true,
    "isHomeless": true,
    "isHouseholder": false,
    "monthlySavingsGoal": 300000,
    "neverUsedBanks": ["0010024"],
    "maturedSavingBanks": []
  }
}
```

### 응답

`200 OK`. `data`로 감싸지 않고 그대로 반환합니다.

목돈만들기 예시입니다(비로그인).

```json
{
  "category": "LONG_TERM",
  "tabs": {
    "tabAEnabled": true,
    "tabBEnabled": false,
    "tabBDisabledReason": "로그인 후 상세 정보를 입력하면 실수령액순 정렬을 확인할 수 있어요.",
    "tabCEnabled": false,
    "tabCDisabledReason": null,
    "tabDEnabled": false
  },
  "governmentRanked": [
    {
      "productId": 2,
      "productPropertyId": 3,
      "productName": "청년미래적금",
      "providerName": "서민금융진흥원",
      "source": "ONTONG",
      "totalScore": 50.0,
      "benefitScore": 30.0,
      "periodScore": 0.0,
      "identityScore": 0.0,
      "depositScore": 20.0,
      "bankCondScore": 0.0,
      "tieBreaker": 1296000
    }
  ],
  "bankRanked": [
    {
      "productId": 36,
      "productPropertyId": 97,
      "productName": "The파트너예금",
      "providerName": "경남은행",
      "source": "FSS",
      "totalScore": 100.0,
      "benefitScore": 0.0,
      "periodScore": 28.57142857142857,
      "identityScore": 0.0,
      "depositScore": 28.57142857142857,
      "bankCondScore": 42.857142857142854,
      "tieBreaker": 3712687
    }
  ],
  "governmentTop3": [
    {
      "productId": 2,
      "productPropertyId": 3,
      "productName": "청년미래적금",
      "providerName": "서민금융진흥원",
      "source": "ONTONG",
      "totalScore": 66.0,
      "benefitScore": 33.0,
      "periodScore": 0.0,
      "identityScore": 0.0,
      "depositScore": 33.0,
      "bankCondScore": 0.0,
      "tieBreaker": null
    }
  ],
  "bankTop3": [],
  "governmentRateRanked": [],
  "bankRateRanked": [],
  "subscriptionProducts": [],
  "parkingProducts": null,
  "depositSavingsProducts": null,
  "productCardSummaries": [
    {
      "productId": 102,
      "productPropertyId": 330,
      "productType": "SAVING",
      "badgeKeywords": ["BENEFIT_EASY_CONDITION", "BANK_SALARY_TRANSFER", "BENEFIT_MAX_INTEREST"],
      "saveTrms": [6, 12, 24, 36],
      "minMonthlyLimit": 1000,
      "maxMonthlyLimit": 3000000,
      "minDepositAmount": null,
      "maxDepositAmount": null,
      "matchScore": 100.0,
      "baseRate": 3.7,
      "maxRate": 4.3,
      "achievableRate": null,
      "expectedTotalContribution": null,
      "effectiveMonthlyDeposit": null,
      "contributionPeriodMonths": null
    }
  ],
  "eligibleProductCount": 95,
  "needsDepositAmount": false,
  "needsMonthlySavingsGoal": false
}
```

단기예치 예시입니다(비로그인, 저축기간 3개월, `depositAmount` 3,000,000원).

```json
{
  "category": "SHORT_TERM",
  "tabs": {
    "tabAEnabled": false,
    "tabBEnabled": false,
    "tabBDisabledReason": null,
    "tabCEnabled": true,
    "tabCDisabledReason": "로그인 후 상세 정보를 입력하면 예상 실수령액을 확인할 수 있어요.",
    "tabDEnabled": true
  },
  "governmentRanked": null,
  "bankRanked": null,
  "governmentTop3": null,
  "bankTop3": null,
  "governmentRateRanked": null,
  "bankRateRanked": null,
  "subscriptionProducts": null,
  "parkingProducts": [
    {
      "productId": 158,
      "productPropertyId": 405,
      "productName": "카카오뱅크 저금통",
      "providerName": "카카오뱅크",
      "providerCode": "0015130",
      "baseRate": 4.0,
      "maxRate": 4.0,
      "maxRateApplicableMinAmount": null,
      "maxRateApplicableMaxAmount": null,
      "maxDepositAmount": 100000,
      "interestPaymentMethod": "월지급",
      "depositProtection": true,
      "applyUrl": "https://www.kakaobank.com/products/coinbox",
      "officialChannelUrl": null,
      "officialChannelName": "카카오뱅크"
    }
  ],
  "depositSavingsProducts": [
    {
      "productId": 52,
      "productPropertyId": 164,
      "productName": "헤이(Hey)정기예금",
      "providerName": "Sh수협은행",
      "source": "FSS",
      "baseRate": 3.6,
      "achievableRate": 3.6,
      "rateComparable": true,
      "isSubscription": false,
      "subscriptionNote": null,
      "netReturn": null,
      "principal": 3000000,
      "saveTrm": 3,
      "productType": "DEPOSIT",
      "expectedTotalContribution": null
    }
  ],
  "productCardSummaries": [],
  "eligibleProductCount": 76,
  "needsDepositAmount": false,
  "needsMonthlySavingsGoal": false
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `category` | string | 대분류. `LONG_TERM` 또는 `SHORT_TERM` |
| `tabs` | object | 탭 활성화 상태 |
| `tabs.tabAEnabled` | boolean | 목돈만들기 적합도순 탭. 목돈만들기면 `true`, 단기예치면 `false` |
| `tabs.tabBEnabled` | boolean | 목돈만들기 실수령액순 탭. 목돈만들기이고 개인화 조건을 만족하면 `true` |
| `tabs.tabBDisabledReason` | string | 목돈만들기에서 `tabBEnabled`가 `false`일 때 안내 문구. 그 밖에는 null |
| `tabs.tabCEnabled` | boolean | 단기예치 예적금 탭. 단기예치면 항상 `true`, 목돈만들기면 `false` |
| `tabs.tabCDisabledReason` | string | 단기예치에서 개인화 조건을 만족하지 못해 실수령액을 가릴 때 안내 문구. 그 밖에는 null |
| `tabs.tabDEnabled` | boolean | 단기예치 파킹통장 탭. 단기예치면 `true`, 목돈만들기면 `false` |
| `governmentRanked` | array<object> | 정부 상품 적합도순 목록 (목돈만들기) |
| `governmentRanked[].productId` | number | 상품 ID |
| `governmentRanked[].productPropertyId` | number | 점수를 매긴 옵션 ID. 상품마다 점수가 가장 높은 옵션 하나만 나옵니다. |
| `governmentRanked[].productName` | string | 상품 이름. 공시 원문 끝의 괄호 표기(예: `(정액적립식)`)는 뗍니다. |
| `governmentRanked[].providerName` | string | 금융사(기관) 이름 |
| `governmentRanked[].source` | string | `ONTONG` 또는 `FSS` |
| `governmentRanked[].totalScore` | number | 적합도 총점 (0~100). 아래 축별 점수의 합 |
| `governmentRanked[].benefitScore` | number | 핵심 혜택 축 점수 |
| `governmentRanked[].periodScore` | number | 저축기간 축 점수 |
| `governmentRanked[].identityScore` | number | 현재 신분 축 점수. 은행 상품과 TOP3 목록에서는 항상 0 |
| `governmentRanked[].depositScore` | number | 납입 한도 축 점수 |
| `governmentRanked[].bankCondScore` | number | 은행 우대 조건 축 점수. 정부 상품과 TOP3 목록에서는 항상 0 |
| `governmentRanked[].tieBreaker` | number | 동점 정렬 기준값. 정부 상품은 예상 만기 기여금 총액(원), 은행 상품은 세후 실수령액(원). 계산할 수 없거나 TOP3 목록이면 null |
| `bankRanked` | array<object> | 은행 상품 적합도순 목록 (목돈만들기). 구조는 `governmentRanked`와 같습니다. |
| `governmentTop3` | array<object> | 정부 상품 TOP3 (목돈만들기). 구조는 `governmentRanked`와 같습니다. |
| `bankTop3` | array<object> | 은행 상품 TOP3 (목돈만들기). 구조는 `governmentRanked`와 같습니다. |
| `governmentRateRanked` | array<object> | 정부 상품 실수령액순 목록 (목돈만들기). `tabBEnabled`가 `false`면 빈 배열 |
| `governmentRateRanked[].productId` | number | 상품 ID |
| `governmentRateRanked[].productPropertyId` | number | 계산에 쓴 옵션 ID. 청약 상품이면 null |
| `governmentRateRanked[].productName` | string | 상품 이름 (괄호 표기 제거) |
| `governmentRateRanked[].providerName` | string | 금융사(기관) 이름. 청약 상품이면 null |
| `governmentRateRanked[].source` | string | `ONTONG` 또는 `FSS` |
| `governmentRateRanked[].baseRate` | number | 은행 상품은 기본금리(%). 정부 상품과 청약 상품은 0 |
| `governmentRateRanked[].achievableRate` | number | 은행 상품은 기본금리에 충족한 우대금리를 더한 금리(%). 정부 상품은 기여금 연 환산 수익률(%)이고 계산할 수 없으면 0. 청약 상품은 0 |
| `governmentRateRanked[].rateComparable` | boolean | 금리 비교 대상 여부. 은행 상품은 `true`, 정부 상품은 수익률을 계산할 수 있을 때 `true`, 청약 상품은 `false` |
| `governmentRateRanked[].isSubscription` | boolean | 청약 상품 여부 |
| `governmentRateRanked[].subscriptionNote` | string | 청약 상품이면 `"청약: 금리 비교 대상 아님"`, 아니면 null |
| `governmentRateRanked[].netReturn` | number | 세후 실수령액(원). 은행 상품은 원금 + 이자 - 이자소득세(15.4%), 정부 상품은 본인 납입 원금 + 예상 기여금(비과세). 계산할 수 없으면 null |
| `governmentRateRanked[].principal` | number | 원금(원). 예금은 `depositAmount`, 적금과 정부 상품은 월 납입액(`monthlySavingsGoal`과 상품 월 한도 중 작은 값) × 기간. 계산할 수 없으면 null |
| `governmentRateRanked[].saveTrm` | number | 기간(개월). 은행 상품은 옵션의 저축기간, 정부 상품은 기여 기간. 청약 상품이면 null |
| `governmentRateRanked[].productType` | string | 상품 유형. `DEPOSIT`, `SAVING`, `PARKING`, `POLICY`, `SUBSCRIPTION` 중 하나. 청약 상품이면 null |
| `governmentRateRanked[].expectedTotalContribution` | number | 예상 만기 기여금 총액(원). 정부 상품만 채우고, 계산할 수 없거나 은행 상품이면 null |
| `bankRateRanked` | array<object> | 은행 상품 실수령액순 목록 (목돈만들기). 구조는 `governmentRateRanked`와 같습니다. `tabBEnabled`가 `false`면 빈 배열 |
| `subscriptionProducts` | array<object> | 청약 상품 목록 (목돈만들기). 구조는 `governmentRateRanked`와 같습니다. `tabBEnabled`가 `false`면 빈 배열 |
| `parkingProducts` | array<object> | 파킹통장 목록 (단기예치) |
| `parkingProducts[].productId` | number | 상품 ID |
| `parkingProducts[].productPropertyId` | number | 옵션 ID. 상품이 아니라 옵션 단위로 나오므로 같은 상품이 여러 번 나올 수 있습니다. |
| `parkingProducts[].productName` | string | 상품 이름. 다른 목록과 달리 공시 원문 이름을 그대로 씁니다(끝 괄호 표기를 떼지 않음). |
| `parkingProducts[].providerName` | string | 금융사 이름 |
| `parkingProducts[].providerCode` | string | 금융사 코드 |
| `parkingProducts[].baseRate` | number | 기본금리(%) |
| `parkingProducts[].maxRate` | number | 최고금리(%) |
| `parkingProducts[].maxRateApplicableMinAmount` | number | 최고금리가 적용되는 잔액 하한(원, 이 값 초과). 정보가 없으면 null |
| `parkingProducts[].maxRateApplicableMaxAmount` | number | 최고금리가 적용되는 잔액 상한(원, 이 값 이하). 정보가 없으면 null |
| `parkingProducts[].maxDepositAmount` | number | 최대 예치 한도(원). 없으면 null |
| `parkingProducts[].interestPaymentMethod` | string | 이자 지급 방식 공시 원문 (예: `월지급`) |
| `parkingProducts[].depositProtection` | boolean | 예금자보호 여부. 항상 `true` |
| `parkingProducts[].applyUrl` | string | 상품 신청 URL. 없으면 null |
| `parkingProducts[].officialChannelUrl` | string | 항상 null |
| `parkingProducts[].officialChannelName` | string | 금융사 이름 (`providerName`과 같음) |
| `depositSavingsProducts` | array<object> | 예적금 실수령액순 목록 (단기예치). 구조는 `governmentRateRanked`와 같습니다. 개인화 조건을 만족하지 못하면 `netReturn`을 null로 가립니다. |
| `productCardSummaries` | array<object> | 목록 카드 표시용 요약. 단기예치에서는 항상 빈 배열 |
| `productCardSummaries[].productId` | number | 상품 ID |
| `productCardSummaries[].productPropertyId` | number | 옵션 ID. 목록 행의 `productPropertyId`와 짝을 맞출 때 씁니다. |
| `productCardSummaries[].productType` | string | 상품 유형. `DEPOSIT`, `SAVING`, `PARKING`, `POLICY`, `SUBSCRIPTION` 중 하나 |
| `productCardSummaries[].badgeKeywords` | array<string> | 카드 배지 키워드. 이 옵션의 키워드에서 `REGION_*`을 뺀 값 |
| `productCardSummaries[].saveTrms` | array<number> | 이 상품에서 가입 가능한 옵션들의 저축기간(개월), 오름차순 |
| `productCardSummaries[].minMonthlyLimit` | number | 월 최소 납입액(원). 없으면 null |
| `productCardSummaries[].maxMonthlyLimit` | number | 월 최대 납입액(원). 없으면 null |
| `productCardSummaries[].minDepositAmount` | number | 최소 예치액(원). 없으면 null |
| `productCardSummaries[].maxDepositAmount` | number | 최대 예치액(원). 없으면 null |
| `productCardSummaries[].matchScore` | number | 적합도 총점 (적합도순 목록의 `totalScore`와 같은 값) |
| `productCardSummaries[].baseRate` | number | 기본금리(%). 공시하지 않으면 null |
| `productCardSummaries[].maxRate` | number | 최고금리(%). 공시하지 않으면 null |
| `productCardSummaries[].achievableRate` | number | 내가 받을 수 있는 금리 또는 기여금 연 환산 수익률(%). `tabBEnabled`가 `true`이고 비교 가능한 상품일 때만 채우고, 아니면 null |
| `productCardSummaries[].expectedTotalContribution` | number | 예상 만기 기여금 총액(원). 정부 상품(청약 제외)이고 `tabBEnabled`가 `true`일 때만 채웁니다. |
| `productCardSummaries[].effectiveMonthlyDeposit` | number | 실제 반영되는 본인 월 납입액(원). `monthlySavingsGoal`과 상품 월 한도 중 작은 값. 채우는 조건은 위와 같습니다. |
| `productCardSummaries[].contributionPeriodMonths` | number | 기여 기간(개월). 채우는 조건은 위와 같습니다. |
| `eligibleProductCount` | number | 가입 가능한 상품 수. 목돈만들기는 정부와 은행 상품 수, 단기예치는 저축기간이 맞는 은행 예적금 상품 수와 파킹통장 상품 수의 합 |
| `needsDepositAmount` | boolean | 항상 `false` |
| `needsMonthlySavingsGoal` | boolean | 항상 `false` |

목돈만들기 동작입니다.

- 나이, 소득, 무주택, 세대주, 재직 기간, 최소 납입액, 현재 신분 조건으로 가입 가능한 옵션만 남깁니다. 거주지역을 고르면 지역 제한이 없거나 고른 지역과 맞는 정부 상품만 남기고, 은행 상품은 지역으로 거르지 않습니다.
- 적합도 배점은 정부 상품이 핵심 혜택 30, 현재 신분 30, 납입 한도 20, 저축기간 20이고 은행 상품이 은행 우대 조건 30, 핵심 혜택 30, 납입 한도 20, 저축기간 20입니다. 입력하지 않은 축의 배점은 나머지 축에 비율대로 나눕니다.
- `governmentRanked`, `bankRanked`는 `totalScore` 내림차순, `tieBreaker` 내림차순, 상품 이름 오름차순으로 정렬합니다.
- `governmentTop3`, `bankTop3`는 핵심 혜택 33, 납입 한도 33, 저축기간 34로 따로 점수를 매긴 상위 3개입니다.
- `governmentRateRanked`는 비교 가능한 정부 상품(청약 제외)을 예상 기여금 총액, 수익률, 이름 순으로 정렬합니다. `bankRateRanked`는 세후 실수령액, 금리, 이름 순입니다. 둘 다 상품마다 옵션 하나만 남깁니다.
- 개인화가 켜지면 은행 우대 조건 점수에 첫거래, 재예치 우대(거래 이력 기반)도 반영합니다.

실수령액 계산의 원금은 이렇게 잡습니다. 은행 예금은 단기예치면 `depositAmount`, 목돈만들기면 `monthlySavingsGoal` × 고른 저축기간입니다. 은행 적금은 월 납입액(단기예치면 `depositAmount` ÷ 고른 저축기간, 목돈만들기면 `monthlySavingsGoal`)과 상품 월 한도 중 작은 값을 옵션의 저축기간만큼 넣는 것으로 봅니다. 이자는 옵션의 단리, 복리 구분에 따라 계산합니다.

단기예치 동작입니다.

- `parkingProducts`는 가입 가능한 파킹통장 옵션 전체에서 만 17세 미만(생년월일을 보낸 경우), 상품 나이 조건, 최소 예치액(`depositAmount` 기준)으로 거른 뒤 최고금리, 기본금리, 이름 순으로 정렬합니다. 저축기간과 은행 우대 조건은 보지 않습니다.
- `depositSavingsProducts`는 가입 가능한 은행 옵션 중 저축기간이 고른 기간과 같고 실수령액을 계산할 수 있는 것만 남겨, 상품마다 실수령액이 가장 큰 옵션 하나를 실수령액, 금리, 이름 순으로 정렬합니다. 비로그인이어도 정렬은 실수령액 기준이고 값만 가립니다.

### 오류

검증은 키워드 해석, 대분류별 필수값 순서로 합니다.

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `S001` | 옵션의 `categoryId`가 1~6이 아니거나 옵션의 실제 카테고리와 다른 경우 |
| 400 | `S004` | 목돈만들기인데 `monthlySavingsGoal`이 없는 경우 |
| 400 | `S005` | 목돈만들기인데 저축기간 옵션을 고르지 않은 경우 |
| 400 | `S009` | 저축기간이 1, 3, 6, 12, 24, 36개월이 아닌 경우 (숨긴 범위형 옵션 `3년 이상`을 보낸 경우) |
| 400 | `S006` | 목돈만들기인데 은행 우대 조건 옵션을 하나도 고르지 않은 경우 |
| 400 | `S007` | 단기예치인데 `depositAmount`가 없거나 0 이하인 경우 |

## GET /search/products

상품 이름으로 검색합니다.

- 인증: 공개

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| query | `searchInput` | string | X | 검색어. 기본값 빈 문자열. 비어 있거나 공백뿐이면 검색하지 않고 빈 배열을 돌려줍니다. |

상품 이름(공시 원문)에 검색어가 들어 있는지 대소문자 구분 없이 찾습니다. 가입 가능한 옵션이 하나 이상 있는 상품만 나옵니다. 검색어의 `%`, `_`는 와일드카드로 동작합니다.

### 응답

`200 OK`. `data`로 감싸지 않고 그대로 반환합니다.

```json
[
  {
    "productId": 2,
    "productName": "청년미래적금",
    "source": "ONTONG",
    "providerName": "서민금융진흥원",
    "baseRate": null,
    "maxRate": null
  },
  {
    "productId": 76,
    "productName": "BNK 위더스자유적금",
    "source": "FSS",
    "providerName": "경남은행",
    "baseRate": 2.3,
    "maxRate": 4.8
  }
]
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `[].productId` | number | 상품 ID |
| `[].productName` | string | 상품 이름 (괄호 표기 제거) |
| `[].source` | string | `ONTONG` 또는 `FSS` |
| `[].providerName` | string | 대표 옵션의 금융사(기관) 이름 |
| `[].baseRate` | number | 대표 옵션의 기본금리(%). 없으면 null |
| `[].maxRate` | number | 대표 옵션의 최고금리(%). 없으면 null |

대표 옵션은 가입 가능한 옵션 중 최고금리가 가장 높은 옵션입니다(최고금리가 없으면 0으로 비교). 결과 순서는 정해져 있지 않습니다.

## POST /search/products/{productId}/detail

상품 상세를 조회합니다. 공개 정보는 항상 돌려주고, 개인화 지표는 조건을 만족할 때만 채웁니다.

- 인증: 공개 (토큰이 있고 개인화 조건을 만족하면 개인화 지표를 채웁니다. 문서 첫머리의 조건 표 참고)

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | `productId` | number | O | 상품 ID |
| body | `productPropertyId` | number | X | 목록에서 누른 행의 옵션 ID. 없으면 대표 옵션을 고르고 개인화 지표를 잠급니다. |
| body | `options` | array<object> | X | 1단계 옵션. 구조와 처리 규칙은 [`POST /search/products`](#post-search-products)와 같지만 필드 검증은 하지 않습니다. |
| body | `options[].categoryId` | number | X | 카테고리 ID |
| body | `options[].optionId` | number | X | 옵션 ID |
| body | `detailedOptions` | object | X | 2단계 입력. 필드는 [`POST /search/products`](#post-search-products)와 같고 모두 선택입니다. |

본문 전체를 생략해도 됩니다. 목록에서 들어올 때는 검색에 보낸 `options`, `detailedOptions`를 그대로 같이 보내야 목록과 같은 점수와 지표를 받습니다.

```json
{
  "productPropertyId": 246,
  "options": [
    { "categoryId": 3, "optionId": 40 },
    { "categoryId": 6, "optionId": 32 }
  ],
  "detailedOptions": {
    "birthdate": "2000-05-01",
    "annualIncome": 30000000,
    "householdSize": 1,
    "householdIncomePercent": 100,
    "monthlySavingsGoal": 300000,
    "neverUsedBanks": ["0010024"],
    "maturedSavingBanks": []
  }
}
```

기준 옵션(`productPropertyId`)과 금리표에 넣는 옵션은 아래처럼 정합니다.

- `productPropertyId`를 보내면 그 옵션이 기준입니다. 이 상품의 옵션이 아니면 `S003`입니다. 모집이 끝난 옵션이어도 그대로 쓰고, 이때 금리표에는 그 옵션만 넣습니다.
- 보내지 않으면 가입 가능한 옵션 중 최고금리, 기본금리가 높은 순(같으면 ID가 작은 순)으로 첫 옵션을 고릅니다. 가입 가능한 옵션이 없으면 전체 옵션에서 같은 방식으로 고릅니다.
- 금리표와 `saveTrms`, `keywords`는 가입 가능한 옵션 전체(없으면 전체 옵션)를 기준으로 만듭니다.
- `keywords`의 `BENEFIT_MAX_INTEREST`(은행 상품)는 `productPropertyId`와 `options`를 보냈으면 그 입력으로 가입 가능한 은행 옵션들, 아니면 가입 가능한 은행 옵션 전체의 최고금리 상위 30%를 기준으로 판단합니다.

### 응답

`200 OK`. `data`로 감싸지 않고 그대로 반환합니다.

비로그인으로 은행 적금을 조회한 예시입니다.

```json
{
  "productId": 76,
  "productPropertyId": 246,
  "productType": "SAVING",
  "sourceCode": "FSS",
  "productName": "BNK 위더스자유적금",
  "providerName": "경남은행",
  "keywords": ["REGION_GYEONGNAM", "BENEFIT_EASY_CONDITION", "BANK_FIRST_TRANSACTION", "BANK_MARKETING", "BANK_ETC", "BENEFIT_MAX_INTEREST"],
  "content": "영업점,인터넷,스마트폰,기타\n\n만기후 1개월 이내: 일반정기예금 기본이율의 50%",
  "contentSummary": "영업점, 인터넷, 스마트폰, 기타 채널을 통해 가입 가능하며, 1개월 이상 36개월 이하 월단위로 가입할 수 있습니다.",
  "saveTrms": [1, 6, 12, 24, 36],
  "matchScore": 57.14285714285714,
  "minAge": null,
  "maxAge": null,
  "minMonthlyLimit": 10000,
  "maxMonthlyLimit": 1000000,
  "minDepositAmount": null,
  "maxDepositAmount": null,
  "requiresHomeless": false,
  "requiresHouseholder": false,
  "joinMethod": "영업점,인터넷,스마트폰,기타",
  "eligibilityText": "실명의 개인 및 개인사업자",
  "cautionText": "1.계약기간은 1개월 이상 36개월 이하 월단위로 한다.",
  "recruitmentPeriod": null,
  "reserveType": "FREE",
  "reserveTypeName": "자유적립식",
  "applyStatus": "AVAILABLE",
  "applyUrl": "https://www.knbank.co.kr/ib20/mnu/FPMCOM990000000?FNC_PRD_NO=0000202492&DUP_CHK=N",
  "officialChannelUrl": null,
  "metricsLocked": true,
  "lockMessage": "내가 받을 수 있는 금리·기여금 환산 수익률·예상 만기 기여금 총액은 로그인 후 1·2단계 필수 정보를 입력하면 확인할 수 있어요.",
  "government": null,
  "bank": {
    "baseRate": 2.3,
    "maxRate": 4.8,
    "achievableRate": null,
    "metConditions": [],
    "unmetConditions": []
  },
  "rateTable": [
    {
      "saveTrm": 12,
      "baseRate": 2.3,
      "maxRate": 4.8,
      "intrRateType": "SINGLE_INTEREST",
      "preferentialRates": [
        { "keywordCode": "BANK_FIRST_TRANSACTION", "rate": 1.0, "description": "신규고객 우대금리 (당행 1년 이내 예적금 신규해지 이력 미보유)" },
        { "keywordCode": "BANK_MARKETING", "rate": 0.5, "description": "마케팅 동의 우대금리" }
      ]
    }
  ]
}
```

개인화가 풀린 정부 상품이면 `bank`는 null이고 `government`를 아래처럼 채웁니다.

```json
{
  "metricsLocked": false,
  "lockMessage": null,
  "government": {
    "annualizedYield": 4.0,
    "expectedTotalContribution": 2160000,
    "contributionType": "RATIO",
    "effectiveMonthlyDeposit": 500000,
    "contributionPeriodMonths": 36
  },
  "bank": null
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `productId` | number | 상품 ID |
| `productPropertyId` | number | 이 상세의 기준 옵션 ID. 옵션이 없는 상품이면 null |
| `productType` | string | 상품 유형. `DEPOSIT`, `SAVING`, `PARKING`, `POLICY`, `SUBSCRIPTION` 중 하나 |
| `sourceCode` | string | `ONTONG` 또는 `FSS` |
| `productName` | string | 상품 이름 (괄호 표기 제거) |
| `providerName` | string | 기준 옵션의 금융사(기관) 이름. 신청 버튼 문구에도 이 값을 씁니다. |
| `keywords` | array<string> | 상품 키워드. 금리표 대상 옵션들의 키워드를 합친 값 |
| `content` | string | 상품 설명 원문 |
| `contentSummary` | string | 상품 설명 요약. 없으면 null |
| `saveTrms` | array<number> | 금리표 대상 옵션들의 저축기간(개월), 오름차순 |
| `matchScore` | number | 적합도 총점 (목록의 `totalScore`와 같은 기준). `productPropertyId`와 빈 배열이 아닌 `options`를 모두 보냈을 때만 계산하고, 아니면 null. 잠금 여부와 관계없이 채웁니다. |
| `minAge` | number | 가입 최소 나이. 없으면 null |
| `maxAge` | number | 가입 최대 나이. 없으면 null |
| `minMonthlyLimit` | number | 월 최소 납입액(원). 없으면 null |
| `maxMonthlyLimit` | number | 월 최대 납입액(원). 없으면 null |
| `minDepositAmount` | number | 최소 예치액(원). 없으면 null |
| `maxDepositAmount` | number | 최대 예치액(원). 없으면 null |
| `requiresHomeless` | boolean | 무주택 요건 여부 |
| `requiresHouseholder` | boolean | 세대주 요건 여부 |
| `joinMethod` | string | 가입 방법. 없으면 null |
| `eligibilityText` | string | 가입 대상 안내. 없으면 null |
| `cautionText` | string | 유의 사항. 없으면 null |
| `recruitmentPeriod` | string | 모집 기간. 없으면 null |
| `reserveType` | string | 적립 방식. `FIXED`(정액적립식) 또는 `FREE`(자유적립식). 없으면 null |
| `reserveTypeName` | string | 적립 방식 표시 이름. `정액적립식` 또는 `자유적립식`. 없으면 null |
| `applyStatus` | string | 신청 가능 여부. 기준 옵션이 가입 가능하면 `AVAILABLE`, 아니면 `RECRUIT_CLOSED` |
| `applyUrl` | string | 상품 자체 신청 URL. 모집이 끝났거나 URL이 없으면 null |
| `officialChannelUrl` | string | `applyUrl`이 없을 때 대신 안내할 금융사(기관) 공식 채널 URL. `applyUrl`이 있거나 모집이 끝났으면 null |
| `metricsLocked` | boolean | 개인화 지표 잠금 여부. `productPropertyId`가 없거나 개인화 조건을 만족하지 못하면 `true` |
| `lockMessage` | string | 잠겼을 때 안내 문구. 잠기지 않으면 null |
| `government` | object | 정부 상품 개인화 지표. 정부 상품(청약 제외)이고 잠기지 않았을 때만 채우고, 아니면 null |
| `government.annualizedYield` | number | 기여금 연 환산 수익률(%). 계산할 수 없으면 null |
| `government.expectedTotalContribution` | number | 예상 만기 기여금 총액(원). 계산할 수 없으면 null |
| `government.contributionType` | string | 기여금 방식. `NONE`, `RATIO`(본인 납입액 비례), `FIXED_AMOUNT`(월 정액) 중 하나. 없으면 null |
| `government.effectiveMonthlyDeposit` | number | 실제 반영되는 본인 월 납입액(원). `monthlySavingsGoal`과 상품 월 한도 중 작은 값. `monthlySavingsGoal`이 없으면 null |
| `government.contributionPeriodMonths` | number | 기여 기간(개월). 없으면 null |
| `bank` | object | 은행 상품 금리 정보. 정부 상품, 청약 상품이거나 기준 옵션이 없으면 null |
| `bank.baseRate` | number | 기준 옵션의 기본금리(%) |
| `bank.maxRate` | number | 기준 옵션의 최고금리(%) |
| `bank.achievableRate` | number | 내가 받을 수 있는 금리(%). 기본금리에 충족한 우대금리를 더한 값. 잠겼으면 null |
| `bank.metConditions` | array<object> | 충족한 우대 조건. 잠겼으면 빈 배열. 원소 구조는 `rateTable[].preferentialRates[]`와 같습니다. |
| `bank.unmetConditions` | array<object> | 충족하지 못한 우대 조건. 잠겼으면 빈 배열. 원소 구조는 위와 같습니다. |
| `rateTable` | array<object> | 금리표. 옵션 하나가 한 행입니다. 정부 상품은 기본금리나 최고금리가 있는 옵션만 넣습니다. 청약 상품이면 null |
| `rateTable[].saveTrm` | number | 저축기간(개월) |
| `rateTable[].baseRate` | number | 기본금리(%). 없으면 null |
| `rateTable[].maxRate` | number | 최고금리(%). 없으면 null |
| `rateTable[].intrRateType` | string | 이자 계산 방식. `SINGLE_INTEREST`(단리) 또는 `COMPOUND_INTEREST`(복리). 없으면 null |
| `rateTable[].preferentialRates` | array<object> | 우대금리 조건 목록 |
| `rateTable[].preferentialRates[].keywordCode` | string | 우대 조건 키워드 코드 (문서 첫머리의 키워드 표 참고, 주로 `BANK_*`) |
| `rateTable[].preferentialRates[].rate` | number | 우대금리(%p) |
| `rateTable[].preferentialRates[].description` | string | 우대 조건 설명 |

은행 상품의 우대 조건 충족 판단은 이렇습니다. 사용자가 고른 은행 우대 조건 옵션과 비대면 가입(`BANK_ONLINE_JOIN`)은 충족으로 봅니다. 연령 우대(`BANK_AGE`)는 우대 연령 구간이 19~34세와 겹칠 때 충족으로 봅니다. 첫거래(`BANK_FIRST_TRANSACTION`)는 기준 옵션의 금융사가 `neverUsedBanks`에, 재예치(`BANK_REDEPOSIT`)는 `maturedSavingBanks`에 있을 때 충족입니다. `BANK_ETC`는 충족으로 보지 않습니다.

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `S003` | 상품이 없거나, `productPropertyId`가 이 상품의 옵션이 아닌 경우 |
| 404 | `S001` | 옵션의 `categoryId`가 1~6이 아니거나 옵션의 실제 카테고리와 다른 경우 |
