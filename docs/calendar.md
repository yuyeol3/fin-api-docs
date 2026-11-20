# 캘린더

청년 금융 정책의 신청 일정을 보여줍니다. 두 계층의 데이터를 한 목록으로 돌려주고, 필터(신청 기간 유형, 상품 분류, 내 조건 맞춤)는 프론트가 응답 필드로 거릅니다. 두 엔드포인트 모두 `data`로 감싸지 않고 결과 객체를 그대로 반환합니다.

| 계층 (`tier`) | 대상 | 상세 이동 |
|---|---|---|
| `POLICY_PRODUCT` | Y-Fin 정책상품 회차. 가입 가능한 옵션이 하나 이상 있는 상품만 나옵니다. | `productId`로 상품 상세 |
| `YOUTH_POLICY` | 온통청년에서 수집한 청년 금융 지원 정책. 활성 정책만 나옵니다. | `externalUrl`(온통청년 상세 페이지) |

회차마다 신청 기간으로 일정 유형(`scheduleKind`)을 정합니다. 기간 일수는 시작일과 마감일을 모두 포함해 셉니다.

| `scheduleKind` | 조건 | 표시 위치 |
|---|---|---|
| `PERIOD` | 아래 둘에 해당하지 않는 회차 | 그리드의 시작, 마감 일정 |
| `YEAR_ROUND` | 신청 기간이 180일 이상이고 12월 31일에 끝나는 회차 (연중 접수) | 요약의 `yearRound`. 시작 전이면 그리드에 시작 일정만 찍힙니다. |
| `ALWAYS` | 신청 기간이 없는 온통청년 정책 (상시 신청). 회차가 없습니다. | 요약의 `always` |

"오늘"은 서버의 한국 시간(Asia/Seoul) 날짜입니다.

두 엔드포인트 모두 토큰 없이 호출할 수 있습니다. 필수 약관에 동의한 사용자(`RECOMMENDATION`, `ADMIN`)의 유효한 액세스 토큰을 보내면 회차마다 자격 판정(`eligibility`)이 채워집니다. 토큰이 없거나, 약관 동의 전 토큰이거나, 만료, 위조된 토큰이면 오류 없이 비로그인으로 처리하고 `eligibility`는 `null`입니다.

자격 판정은 나이와 거주지역만 봅니다.

- 나이: 프로필 생년월일로 오늘 기준 만 나이를 구해 `minAge`, `maxAge`와 비교합니다(양 끝 포함).
- 거주지역: 프로필에서 고른 거주지역(`REGION_*`)이 회차의 `regions` 중 하나와 맞아야 합니다. `regions`가 빈 목록이면 전국 대상이라 제한이 없습니다.
- 한 조건이라도 맞지 않으면 `INELIGIBLE`, 제한이 있는 조건의 프로필 값이 비어 판정하지 못하면 `NEEDS_INFO`, 그 외에는 `ELIGIBLE`입니다. 프로필이 없으면 모든 제한 조건을 판정하지 못한 것으로 봅니다.

## GET /calendar/events

조회 기간 안에 찍히는 신청 시작, 마감 일정을 반환합니다. 월간 그리드, 주간 보기, 리스트 보기에 같이 씁니다.

- 인증: 공개 (약관 동의 사용자의 토큰을 보내면 `eligibility`가 채워집니다)

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| query | `from` | string (yyyy-MM-dd) | O | 조회 시작일. 이 날짜를 포함합니다. |
| query | `to` | string (yyyy-MM-dd) | O | 조회 종료일. 이 날짜를 포함합니다. `from`과 같아도 됩니다. |

조회 기간의 최대 길이는 제한하지 않습니다.

일정은 아래 규칙으로 만듭니다.

- 시작일이나 마감일이 조회 기간 안에 있는 회차만 대상입니다. 조회 기간 전에 시작해 조회 기간 뒤에 끝나는 회차는 나오지 않습니다.
- `PERIOD` 회차는 시작일에 `START`, 마감일에 `END` 일정을 만듭니다. 시작일과 마감일이 같으면 `END` 하나만 만듭니다.
- `YEAR_ROUND` 회차는 마감 일정이 없습니다. 시작일이 오늘 이후일 때만 `START` 일정 하나를 만듭니다.
- `ALWAYS` 정책은 일정이 없어 나오지 않습니다. [GET /calendar/summary](#get-calendar-summary)의 `always`로 받습니다.
- 만든 일정 중 날짜가 조회 기간 안에 있는 것만 반환합니다. 그래서 한 회차의 `START`와 `END` 중 하나만 나올 수 있습니다.

정렬은 날짜 오름차순이고, 같은 날짜에서는 `END`가 `START`보다 먼저, 그다음 `POLICY_PRODUCT`가 `YOUTH_POLICY`보다 먼저, 그다음 이름순입니다.

### 응답

`200 OK`

```json
{
  "asOf": "2026-11-20T01:50:14.734690Z",
  "events": [
    {
      "date": "2026-12-01",
      "type": "START",
      "round": {
        "tier": "POLICY_PRODUCT",
        "productId": 101,
        "policyId": null,
        "roundId": 4,
        "name": "청년도약계좌",
        "institution": "서민금융진흥원",
        "summary": "매월 납입하면 정부기여금과 비과세 혜택을 받는 청년 자산형성 상품",
        "scheduleKind": "PERIOD",
        "applyStartDate": "2026-12-01",
        "applyEndDate": "2026-12-12",
        "minAge": 19,
        "maxAge": 34,
        "regions": [],
        "externalUrl": null,
        "scheduleChanged": false,
        "eligibility": null
      }
    },
    {
      "date": "2026-12-05",
      "type": "END",
      "round": {
        "tier": "YOUTH_POLICY",
        "productId": null,
        "policyId": 6,
        "roundId": 3,
        "name": "2026 청년층 신용회복 지원사업",
        "institution": "서울특별시 미래청년기획관 청년사업담당관",
        "summary": "신용유의자 분할상환 약정 시 초입금 지원",
        "scheduleKind": "PERIOD",
        "applyStartDate": "2026-02-06",
        "applyEndDate": "2026-12-05",
        "minAge": 19,
        "maxAge": 39,
        "regions": ["REGION_SEOUL"],
        "externalUrl": "https://www.youthcenter.go.kr/youthPolicy/ythPlcyTotalSearch/ythPlcyDetail/20261021005400214201",
        "scheduleChanged": true,
        "eligibility": null
      }
    }
  ]
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `asOf` | string | 온통청년 데이터를 마지막으로 수집한 시각(ISO-8601, UTC). 수집한 데이터가 없으면 `null` |
| `events` | array<object> | 일정 목록. 없으면 빈 배열 |
| `events[].date` | string (yyyy-MM-dd) | 일정 날짜 |
| `events[].type` | string | `START`(신청 시작), `END`(신청 마감) |
| `events[].round` | object | 일정이 속한 회차. 한 회차의 `START`, `END` 일정에 같은 내용이 반복됩니다. |
| `events[].round.tier` | string | `POLICY_PRODUCT`(Y-Fin 정책상품), `YOUTH_POLICY`(온통청년 정책) |
| `events[].round.productId` | number | 정책상품 ID. `POLICY_PRODUCT`에만 있고 `YOUTH_POLICY`는 `null` |
| `events[].round.policyId` | number | 온통청년 정책 ID. `YOUTH_POLICY`에만 있고 `POLICY_PRODUCT`는 `null` |
| `events[].round.roundId` | number | 회차 ID. 계층마다 따로 매기므로 `tier`와 함께 구분합니다. 상시 신청 정책은 `null` |
| `events[].round.name` | string | 정책상품은 상품명(끝의 괄호 부분을 뺀 이름), 온통청년은 정책명 |
| `events[].round.institution` | string | 정책상품은 가입 가능한 첫 옵션의 제공기관 이름(없으면 `null`), 온통청년은 운영 기관 |
| `events[].round.summary` | string | 상품 요약 또는 정책 요약 |
| `events[].round.scheduleKind` | string | `PERIOD`, `YEAR_ROUND`, `ALWAYS`. 문서 앞의 표를 봅니다. |
| `events[].round.applyStartDate` | string (yyyy-MM-dd) | 신청 시작일. 상시 신청 정책은 `null` |
| `events[].round.applyEndDate` | string (yyyy-MM-dd) | 신청 마감일. 상시 신청 정책은 `null` |
| `events[].round.minAge` | number | 최소 나이. `null`이면 하한이 없습니다. 정책상품은 가입 가능한 옵션 중 가장 낮은 값이고, 한 옵션이라도 제한이 없으면 `null`입니다. |
| `events[].round.maxAge` | number | 최대 나이. `null`이면 상한이 없습니다. 정책상품은 가입 가능한 옵션 중 가장 높은 값이고, 한 옵션이라도 제한이 없으면 `null`입니다. |
| `events[].round.regions` | array<string> | 대상 거주지역 키. 전국 대상이면 빈 배열. 값은 `REGION_SEOUL`, `REGION_BUSAN`, `REGION_DAEGU`, `REGION_INCHEON`, `REGION_GWANGJU`, `REGION_DAEJEON`, `REGION_ULSAN`, `REGION_SEJONG`, `REGION_GYEONGGI`, `REGION_GANGWON`, `REGION_CHUNGBUK`, `REGION_CHUNGNAM`, `REGION_JEONBUK`, `REGION_JEONNAM`, `REGION_GYEONGBUK`, `REGION_GYEONGNAM`, `REGION_JEJU` |
| `events[].round.externalUrl` | string | 온통청년 정책 상세 페이지 주소. `POLICY_PRODUCT`는 `null` |
| `events[].round.scheduleChanged` | boolean | 최근 7일 안에 신청 기간이 바뀐 회차이면 `true` |
| `events[].round.eligibility` | string | `ELIGIBLE`(자격 충족), `INELIGIBLE`(자격 미충족), `NEEDS_INFO`(프로필 정보가 없어 판정 못 함). 비로그인이면 `null` |

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 400 | `CA001` | `from`이 `to`보다 늦은 경우 |

## GET /calendar/summary

보고 있는 달과 상관없이 오늘 기준으로 지금 신청 가능한 회차, 연중 접수 회차, 상시 신청 정책을 반환합니다.

- 인증: 공개 (약관 동의 사용자의 토큰을 보내면 `eligibility`가 채워집니다)

### 요청

요청 값은 없습니다.

### 응답

`200 OK`. 아래 예시는 서울에 사는 약관 동의 사용자의 토큰으로 호출한 경우입니다.

```json
{
  "asOf": "2026-11-20T01:50:14.734690Z",
  "openNow": [
    {
      "tier": "YOUTH_POLICY",
      "productId": null,
      "policyId": 6,
      "roundId": 3,
      "name": "2026 청년층 신용회복 지원사업",
      "institution": "서울특별시 미래청년기획관 청년사업담당관",
      "summary": "신용유의자 분할상환 약정 시 초입금 지원",
      "scheduleKind": "PERIOD",
      "applyStartDate": "2026-02-06",
      "applyEndDate": "2026-11-20",
      "minAge": 19,
      "maxAge": 39,
      "regions": ["REGION_SEOUL"],
      "externalUrl": "https://www.youthcenter.go.kr/youthPolicy/ythPlcyTotalSearch/ythPlcyDetail/20261021005400214201",
      "scheduleChanged": false,
      "eligibility": "ELIGIBLE"
    }
  ],
  "yearRound": [],
  "always": [
    {
      "tier": "YOUTH_POLICY",
      "productId": null,
      "policyId": 1,
      "roundId": null,
      "name": "대구형 청년 그린 내일채움공제",
      "institution": "대구광역시 경제국 고용노동정책과",
      "summary": "중소기업 재직 청년에게 목돈마련의 기회를 제공하여 장기재직 및 지역정착 유도",
      "scheduleKind": "ALWAYS",
      "applyStartDate": null,
      "applyEndDate": null,
      "minAge": 19,
      "maxAge": 34,
      "regions": ["REGION_DAEGU"],
      "externalUrl": "https://www.youthcenter.go.kr/youthPolicy/ythPlcyTotalSearch/ythPlcyDetail/20261101005400214666",
      "scheduleChanged": false,
      "eligibility": "INELIGIBLE"
    }
  ]
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `asOf` | string | 온통청년 데이터를 마지막으로 수집한 시각(ISO-8601, UTC). 수집한 데이터가 없으면 `null` |
| `openNow` | array<object> | 오늘 신청 가능한(시작일 ≤ 오늘 ≤ 마감일) `PERIOD` 회차. 두 계층을 모두 담고, 마감일 오름차순, 같은 날이면 `POLICY_PRODUCT` 먼저, 그다음 이름순입니다. |
| `yearRound` | array<object> | 오늘 접수 중인 `YEAR_ROUND` 회차. 두 계층을 모두 담고, 시작일 내림차순, 같은 날이면 `POLICY_PRODUCT` 먼저, 그다음 이름순입니다. |
| `always` | array<object> | 활성 상태인 온통청년 상시 신청 정책. 정책상품은 들어가지 않습니다. 이름순이고, `roundId`, `applyStartDate`, `applyEndDate`는 `null`, `scheduleChanged`는 `false`입니다. |

세 목록의 원소는 [GET /calendar/events](#get-calendar-events)의 `events[].round`와 같은 형식입니다. 목록이 비어 있으면 빈 배열이고, 페이지 나눔은 없습니다.
