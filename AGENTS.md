# 작업 안내 (사람, 에이전트 공통)

이 저장소는 Fin 백엔드 API 명세입니다. 명세의 원본은 마크다운이고, `index.html`은 그 마크다운을 읽어 GitHub Pages에 보여주기만 합니다. 빌드 단계는 없습니다.

## 구조

- `README.md`: 개요, 공통 규칙(인증 수준, 응답과 오류 형식), 문서 목록
- `docs/<도메인>.md`: 도메인별 엔드포인트 명세
- `index.html`: 렌더러. `README.md`의 `## 문서 목록`에 있는 `docs/*.md` 링크를 순서대로 불러옵니다. 첫 `##` 전까지는 랜딩 상단, 문서 목록은 도메인 카드가 됩니다. 그래서 목록 항목은 `- [제목](docs/<도메인>.md): 설명` 형식을 지킵니다. 화면에는 주소 해시가 가리키는 문서 하나만 보입니다(`#search`, `#post-search-products`는 상품 검색 문서, 해시가 없으면 랜딩). 한 페이지에 다 펼치면 상품 검색 같은 긴 문서에서 위치를 잃기 쉬워서 나눴고, 그 대가로 브라우저 찾기(Ctrl+F)는 지금 보이는 문서 안에서만 됩니다.
- `assets/`: Fin-FE에서 가져온 로고와, 로고의 원형 마크만 잘라 만든 파비콘
- `.nojekyll`: Pages가 `.md`를 HTML로 바꾸지 않고 원본 그대로 서빙하게 합니다. 지우지 않습니다.

## 명세를 읽을 때

- 사이트: https://yuyeol3.github.io/fin-api-docs/
- 원문: `https://yuyeol3.github.io/fin-api-docs/README.md`, `https://yuyeol3.github.io/fin-api-docs/docs/<도메인>.md`
- `README.md`의 공통 규칙을 먼저 읽고 도메인 문서를 읽습니다. 공통 오류는 도메인 문서에 반복하지 않습니다.

## 언제 고치나

이 명세는 Fin-BE 코드와 자동으로 맞춰지지 않습니다. Fin-BE `main`에 API 계약(경로, 요청과 응답 필드, 인증 수준, 오류 코드, 동작 규칙)을 바꾸는 PR이 머지되면 해당 `docs/*.md`를 고치고 `README.md`의 `기준 커밋`을 그 머지 커밋으로 올립니다.

## 명세를 고칠 때

1. Fin-BE의 컨트롤러, DTO, 서비스, `*ErrorCode`를 근거로 씁니다. 코드에 없는 동작은 추측해서 적지 않습니다.
2. 경로, 파라미터, 필드 이름과 타입은 Swagger가 코드에서 만든 OpenAPI 문서와 대조합니다(아래 "Swagger와 대조").
3. 도메인 문서는 아래 형식을 지킵니다. 렌더러가 `## METHOD /path` 제목을 엔드포인트로 인식해 배지와 목차를 만듭니다.
4. 새 도메인 문서를 만들면 `README.md`의 `## 문서 목록`에 링크를 추가합니다. 목록에 없는 문서는 사이트에 나오지 않습니다.
5. 기준으로 삼은 Fin-BE 커밋이 바뀌면 `README.md`의 `기준 커밋`을 갱신합니다.
6. 로컬에서 확인할 때는 저장소 루트에서 `python -m http.server`를 띄우고 http://localhost:8000 을 엽니다. `file://`로 열면 마크다운을 불러오지 못합니다.

## Swagger와 대조

Fin-BE는 springdoc으로 OpenAPI 문서를 만들고, dev와 test 프로필에서만 엽니다. 컨트롤러에 Swagger 어노테이션을 달지 않으므로 OpenAPI 문서에는 경로, 파라미터, DTO 스키마만 있고 인증 수준, 오류 코드, 동작 규칙은 없습니다. 이 저장소의 마크다운이 그 부분을 맡습니다.

1. Fin-BE를 dev 프로필로 실행합니다(실행 방법은 Fin-BE `docs/implementation/development.md`).
2. `curl -s http://localhost:8080/v3/api-docs > openapi.json`으로 받습니다. 이 파일은 커밋하지 않습니다.
3. 바뀐 엔드포인트마다 다음을 맞춰 봅니다.
   - `paths`의 경로와 메서드 ↔ `## METHOD /path` 제목
   - `parameters`, `requestBody` ↔ `### 요청` 표의 이름, 위치, 타입, 필수 여부
   - `responses`의 스키마(`components.schemas`) ↔ `### 응답` 표의 필드 경로와 타입
4. 엔드포인트 전체 목록도 `paths`와 비교해 빠지거나 남은 문서가 없는지 봅니다.

스키마의 `required`는 DTO의 검증 어노테이션에서 나옵니다. 컨트롤러 파라미터에 `@Valid`가 없으면 실제로는 검증하지 않으므로, 그런 엔드포인트는 서비스 코드에서 필수 여부를 확인합니다.

## 도메인 문서 형식

```markdown
# 도메인 이름

도메인 설명 한두 문장. 여러 엔드포인트에 걸친 개념이 있으면 여기에 적는다.

## GET /path/{id}

한 줄 설명.

- 인증: 공개 | 로그인 | 약관 동의

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | `id` | number | O | ... |
| query | `page` | number | X | 기본값 0 |
| body | `name` | string | O | ... |

본문 예시가 필요하면 표 아래에 json 코드 블록을 둔다. 요청 값이 없으면 "요청 값은 없습니다."라고 쓴다.

### 응답

`200 OK`

(json 예시)

| 필드 | 타입 | 설명 |
|---|---|---|
| `data.id` | number | ... |

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 404 | `X001` | ... |
```

- 제목 수준: 문서 제목 `#`, 엔드포인트 `##`, 하위 항목 `###`. `##`는 엔드포인트에만 씁니다.
- 위치 값: `path`, `query`, `body`, `header`, `cookie`.
- 타입 값: JSON 기준 `string`, `number`, `boolean`, `object`, `array<타입>`. 날짜는 `string (yyyy-MM-dd)`처럼 형식을 붙이고, 열거형은 설명 칸에 허용 값을 모두 적습니다.
- 중첩 필드는 `data.items[].name`처럼 점 경로로 씁니다.
- 도메인에 고유한 오류가 없으면 `### 오류` 절을 생략합니다.
- 다른 문서의 엔드포인트는 `[POST /auth/refresh](auth.md#post-auth-refresh)`처럼 링크합니다. 앵커는 제목을 소문자로 바꾸고 영문, 숫자, 한글 외의 문자를 `-`로 바꾼 값입니다.
