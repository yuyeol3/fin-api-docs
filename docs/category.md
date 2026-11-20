# 카테고리

추천 1단계(STEP1)에서 고르는 카테고리와 옵션입니다. 프론트는 여기서 받은 `categoryId`, `optionId` 쌍을 [`POST /search/products`](search.md#post-search-products) 같은 검색 요청의 `options`에 그대로 담아 보냅니다. 서버 로직은 옵션의 표시 문구가 아니라 `optionId`(와 옵션에 연결된 키워드 코드)를 봅니다.

## GET /api/categories

화면에 표시할 카테고리와 옵션 목록을 조회합니다.

- 인증: 공개

### 요청

요청 값은 없습니다.

### 응답

`200 OK`. `data`로 감싸지 않고 배열을 그대로 반환합니다.

```json
[
  {
    "categoryId": 3,
    "categoryName": "저축기간",
    "options": [
      { "optionId": 37, "optionValue": "1개월" },
      { "optionId": 40, "optionValue": "1년" }
    ]
  },
  {
    "categoryId": 6,
    "categoryName": "은행 우대 조건",
    "options": [
      { "optionId": 32, "optionValue": "급여이체 우대" }
    ]
  }
]
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `[].categoryId` | number | 카테고리 ID |
| `[].categoryName` | string | 카테고리 표시 이름 |
| `[].options` | array<object> | 카테고리에 속한 옵션 목록 |
| `[].options[].optionId` | number | 옵션 ID. 검색 요청의 `options[].optionId`로 보냅니다. |
| `[].options[].optionValue` | string | 옵션 표시 문구 |

카테고리는 `categoryId` 오름차순, 옵션은 카테고리 안에서 `optionId` 오름차순입니다. 옵션이 하나도 없는 카테고리는 목록에 나오지 않습니다.

아래 항목은 DB에 남아 있지만 응답에서 뺍니다. 이미 저장된 선택값과 기존 매칭 로직 때문에 지우지 않은 것입니다.

| 숨김 대상 | 이유 |
|---|---|
| 카테고리 5 `상품관심사` 전체 | 다시 쓸 수 있어 행만 남겨 둡니다. |
| 범위형 저축기간 옵션 (`TERM_OVER_3_YEARS`, `TERM_2_TO_3_YEARS`, `TERM_AROUND_1_YEAR`) | 정확한 개월 수 옵션(1개월~3년)으로 대체되었습니다. |
| `BANK_FIRST_TRANSACTION`(첫거래 고객), `BANK_REDEPOSIT`(재예치) | 검색 요청의 거래 이력(`neverUsedBanks`, `maturedSavingBanks`)으로 자동 판단합니다. |

현재 응답에 나오는 옵션 구성은 아래와 같습니다. 옵션 ID는 마이그레이션(V2, V9) 삽입 순서로 정해진 값이고, 표시 문구는 V11 기준입니다. 개발 서버 응답과 일치합니다.

| `categoryId` | 카테고리 | `optionId` | 표시 문구 | 키워드 코드 |
|---|---|---|---|---|
| 1 | 거주지역 | 1~17 | 서울, 부산, 대구, 인천, 광주, 대전, 울산, 세종, 경기, 강원, 충북, 충남, 전북, 전남, 경북, 경남, 제주 | `REGION_SEOUL` ~ `REGION_JEJU` (같은 순서) |
| 2 | 현재신분 | 18 | 미취업 | `STATUS_UNEMPLOYED` |
| 2 | 현재신분 | 19 | 알바/프리랜서 | `STATUS_PART_TIME` |
| 2 | 현재신분 | 20 | 중소기업 재직 | `STATUS_SME_WORKER` |
| 2 | 현재신분 | 21 | 군복무 | `STATUS_MILITARY` |
| 3 | 저축기간 | 37 | 1개월 | `TERM_1_MONTH` |
| 3 | 저축기간 | 38 | 3개월 | `TERM_3_MONTH` |
| 3 | 저축기간 | 39 | 6개월 | `TERM_6_MONTH` |
| 3 | 저축기간 | 40 | 1년 | `TERM_12_MONTH` |
| 3 | 저축기간 | 41 | 2년 | `TERM_24_MONTH` |
| 3 | 저축기간 | 42 | 3년 | `TERM_36_MONTH` |
| 4 | 핵심 혜택 | 25 | 최고이율 중심 | `BENEFIT_MAX_INTEREST` |
| 4 | 핵심 혜택 | 26 | 비과세 | `BENEFIT_TAX_FREE` |
| 4 | 핵심 혜택 | 27 | 우대조건 간편 | `BENEFIT_EASY_CONDITION` |
| 4 | 핵심 혜택 | 28 | 정부기여금 | `BENEFIT_GOV_SUBSIDY` |
| 6 | 은행 우대 조건 | 32 | 급여이체 우대 | `BANK_SALARY_TRANSFER` |
| 6 | 은행 우대 조건 | 33 | 카드실적 우대 | `BANK_CARD_USAGE` |
| 6 | 은행 우대 조건 | 34 | 자동이체 우대 | `BANK_AUTO_TRANSFER` |
| 6 | 은행 우대 조건 | 35 | 마케팅동의 OK | `BANK_MARKETING` |

키워드 코드는 응답에 나오지 않습니다. 검색 결과의 키워드 필드(`keywords`, `badgeKeywords` 등)를 옵션과 맞춰 볼 때 참고합니다.
