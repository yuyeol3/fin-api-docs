# 작업 안내 (사람, 에이전트 공통)

이 저장소는 Fin 백엔드 API 명세입니다. 명세의 원본은 마크다운이고, `index.html`은 그 마크다운을 읽어 GitHub Pages에 보여주기만 합니다. 빌드 단계는 없습니다.

## 구조

- `README.md`: 개요, 공통 규칙(인증 수준, 응답과 오류 형식), 문서 목록
- `docs/<도메인>.md`: 도메인별 엔드포인트 명세
- `index.html`: 렌더러. `README.md`의 `## 문서 목록`에 있는 `docs/*.md` 링크를 순서대로 불러옵니다.
- `.nojekyll`: Pages가 `.md`를 HTML로 바꾸지 않고 원본 그대로 서빙하게 합니다. 지우지 않습니다.

## 명세를 읽을 때

- 사이트: https://yuyeol3.github.io/fin-api-docs/
- 원문: `https://yuyeol3.github.io/fin-api-docs/README.md`, `https://yuyeol3.github.io/fin-api-docs/docs/<도메인>.md`
- `README.md`의 공통 규칙을 먼저 읽고 도메인 문서를 읽습니다. 공통 오류는 도메인 문서에 반복하지 않습니다.

## 명세를 고칠 때

1. Fin-BE의 컨트롤러, DTO, 서비스, `*ErrorCode`를 근거로 씁니다. 코드에 없는 동작은 추측해서 적지 않습니다.
2. 도메인 문서는 아래 형식을 지킵니다. 렌더러가 `## METHOD /path` 제목을 엔드포인트로 인식해 배지와 목차를 만듭니다.
3. 새 도메인 문서를 만들면 `README.md`의 `## 문서 목록`에 링크를 추가합니다. 목록에 없는 문서는 사이트에 나오지 않습니다.
4. 기준으로 삼은 Fin-BE 커밋이 바뀌면 `README.md`의 `기준 커밋`을 갱신합니다.
5. 로컬에서 확인할 때는 저장소 루트에서 `python -m http.server`를 띄우고 http://localhost:8000 을 엽니다. `file://`로 열면 마크다운을 불러오지 못합니다.

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
