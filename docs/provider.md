# 금융사

상품을 제공하는 금융사(은행, 기관) 정보입니다. 은행 목록의 `code`는 검색 요청의 거래 이력 필드(`detailedOptions.neverUsedBanks`, `detailedOptions.maturedSavingBanks`)에 넣는 값입니다. [`POST /search/products`](search.md#post-search-products)를 참고합니다.

## GET /providers/banks

은행 목록을 조회합니다. 금융감독원 공시(`FSS`) 출처로 등록된 금융사만 돌려줍니다.

- 인증: 공개

### 요청

요청 값은 없습니다.

### 응답

`200 OK`. `data`로 감싸지 않고 배열을 그대로 반환합니다.

```json
[
  { "code": "0010024", "name": "경남은행", "category": "지방" },
  { "code": "0015130", "name": "카카오뱅크", "category": "인터넷" }
]
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `[].code` | string | 금융사 코드 (금융감독원 금융회사 코드, 예: `0010001`) |
| `[].name` | string | 금융사 이름 |
| `[].category` | string | 은행 분류. `시중`, `지방`, `특수`, `인터넷`, `기타` 중 하나입니다. 서버에 고정된 코드별 분류표로 정하고, 표에 없는 코드는 `기타`입니다. |

정렬은 DB에서 `name` 오름차순으로 하며 DB의 문자열 정렬 규칙(collation)을 따릅니다. 가나다순과 다를 수 있으니 화면에서 특정 순서가 필요하면 프론트에서 다시 정렬합니다.
