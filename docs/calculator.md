# 계산기

예금, 적금 상품 옵션에 금액과 기간을 넣어 만기 수령액을 계산합니다. 계산 결과는 저장하지 않습니다.

## POST /calculator

예금 또는 적금의 세전 이자, 이자 과세, 세후 수령액을 계산합니다.

- 인증: 약관 동의

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| body | `productPropertyId` | number | O | 상품 옵션 ID. 금리와 금액 상한을 검증하는 기준입니다. |
| body | `productType` | string | O | `DEPOSIT`(예금), `SAVING`(적금). 열거형에는 `PARKING`, `POLICY`, `SUBSCRIPTION`도 있지만 계산기는 지원하지 않아 400을 반환합니다. |
| body | `interestRateType` | string | O | `SINGLE_INTEREST`(단리), `COMPOUND_INTEREST`(복리) |
| body | `reserveType` | string | X | `FIXED`(정액적립식), `FREE`(자유적립식). 받기만 하고 쓰지 않습니다. 적금은 적립 방식과 관계없이 매월 초 같은 금액을 넣는 것으로 계산합니다. |
| body | `appliedRate` | number | O | 적용 연이율. 소수로 보냅니다(3.5%는 `0.035`). 0 이상이고, 옵션의 최고금리를 넘을 수 없습니다. |
| body | `amount` | number | O | 예금은 예치금, 적금은 월 납입액(원). 10,000 이상이고, 옵션의 한도를 넘을 수 없습니다. |
| body | `saveTrm` | number | O | 저축 기간(개월). 1 이상 |
| body | `taxType` | string | O | `GENERAL`(일반과세 15.4%), `NON_TAX`(비과세 0%) |

```json
{
  "productPropertyId": 1203,
  "productType": "SAVING",
  "interestRateType": "SINGLE_INTEREST",
  "reserveType": "FIXED",
  "appliedRate": 0.04,
  "amount": 300000,
  "saveTrm": 12,
  "taxType": "GENERAL"
}
```

옵션 기준 검증은 아래 두 가지만 합니다. 옵션에 값이 없으면 해당 검증을 건너뜁니다. 저축 기간, 상품 종류, 이자 계산 방식은 옵션과 대조하지 않으므로 옵션에 맞는 값을 보내야 합니다.

- 금리: `appliedRate`가 옵션 최고금리(% 단위)를 100으로 나눈 값보다 크면 400입니다.
- 금액: `productType`이 `DEPOSIT`이면 옵션의 최대 예치금, 그 외에는 월 최대 납입액보다 `amount`가 크면 400입니다.

### 계산 방식

`r`은 `appliedRate`, `n`은 `saveTrm`, 월 이율 `i = r / 12`입니다.

| 상품 | 이자 방식 | 원금 | 세전 이자 |
|---|---|---|---|
| 예금 | 단리 | `amount` | `amount × r × n / 12` |
| 예금 | 복리 (월복리) | `amount` | `amount × (1 + i)^n − amount` |
| 적금 | 단리 | `amount × n` | `amount × i × n(n+1) / 2` |
| 적금 | 복리 (월복리) | `amount × n` | `amount × (1 + i) × ((1 + i)^n − 1) / i − 원금`. 금리가 0이면 이자는 0입니다. |

- 적금은 매월 초에 같은 금액을 낸다고 가정합니다. 이 가정은 응답의 `assumptionNote`에 담깁니다.
- 이자 과세 = 세전 이자 × 세율, 세후 수령액 = 원금 + 세전 이자 − 이자 과세입니다.
- 세전 이자, 만기 금액, 이자 과세를 각각 소수 둘째 자리에서 반올림(HALF_UP)한 뒤 세후 수령액을 구합니다. 원 단위 절사는 하지 않습니다.

### 응답

`200 OK`. `data`로 감싸지 않고 결과 객체를 그대로 반환합니다.

```json
{
  "principal": 3600000.00,
  "maturityAmount": 3678000.00,
  "preTaxInterest": 78000.00,
  "taxRate": 0.154,
  "interestTax": 12012.00,
  "afterTaxAmount": 3665988.00,
  "assumptionNote": "매월 초 동일액을 납입하는 것으로 가정한 계산이며, 실제 납입 시점·금액에 따라 수령액이 달라질 수 있어요."
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `principal` | number | 원금. 예금은 예치금, 적금은 월 납입액 × 개월 수. 소수 둘째 자리 |
| `maturityAmount` | number | 세전 만기 금액(원금 + 세전 이자). 소수 둘째 자리 |
| `preTaxInterest` | number | 세전 이자. 소수 둘째 자리 |
| `taxRate` | number | 적용 세율. `GENERAL`이면 `0.154`, `NON_TAX`이면 `0` |
| `interestTax` | number | 이자 과세. 소수 둘째 자리 |
| `afterTaxAmount` | number | 세후 수령액. 소수 둘째 자리 |
| `assumptionNote` | string | 계산 가정 안내 문구. 적금만 있고 예금은 `null`입니다. |

### 오류

모두 `IllegalArgumentException`이라 공통 코드 `C003`으로 나오고, `message`로 구분합니다. 위에서부터 순서대로 검사합니다.

| 상태 | 코드 | 상황 |
|---|---|---|
| 400 | `C003` | `productPropertyId`에 해당하는 옵션이 없는 경우. `존재하지 않는 상품 옵션입니다: {id}` |
| 400 | `C003` | 적용 금리가 옵션 최고금리를 넘는 경우. `적용 금리가 상품 최고금리를 초과합니다. 최고금리: {최고금리}%` |
| 400 | `C003` | 금액이 옵션 한도를 넘는 경우. `금액이 상품 최대 한도를 초과합니다. 최대 한도: {한도}원` |
| 400 | `C003` | `productType`이 `DEPOSIT`, `SAVING`이 아닌 경우. `지원하지 않는 상품 타입입니다: {productType}` |
