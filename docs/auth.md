# 인증

카카오, 구글 OAuth2로 로그인합니다. 로그인에 성공하면 서버가 리프레시 토큰을 `refresh_token` 쿠키(HttpOnly)로 내려주고, 프론트는 [`POST /auth/refresh`](#post-auth-refresh)로 액세스 토큰을 받아 `Authorization: Bearer` 헤더에 실어 보냅니다.

| 토큰 | 전달 방식 | 유효 기간 |
|---|---|---|
| 액세스 토큰 (JWT) | 응답 본문 `data` → 요청 헤더 `Authorization: Bearer <토큰>` | 10분 |
| 리프레시 토큰 | `refresh_token` 쿠키 (HttpOnly, Path `/`) | 24시간, 재발급할 때마다 새 토큰으로 교체 |

## GET /oauth2/authorization/{provider}

OAuth 로그인을 시작합니다. XHR이 아니라 브라우저 이동(`window.location.href`)으로 호출합니다.

- 인증: 공개

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | `provider` | string | O | `kakao` 또는 `google` |

### 응답

`302 Found`. 제공자 로그인 화면으로 이동하고, 로그인이 끝나면 서버 콜백(`/login/oauth2/code/{provider}`)을 거쳐 프론트로 다시 이동합니다.

| 결과 | 이동 위치 |
|---|---|
| 성공 | `<프론트 출처>/`. 이때 `Set-Cookie: refresh_token=...`이 함께 내려옵니다. |
| 실패 | `<프론트 출처>/login-error` |

프론트 출처는 로그인을 시작한 페이지의 `Referer`로 정합니다. 서버에 허용된 출처가 아니면 허용 목록의 첫 주소로 이동합니다.

## POST /auth/refresh

리프레시 토큰 쿠키로 액세스 토큰을 발급합니다. 쓴 리프레시 토큰은 폐기하고 새 리프레시 토큰을 쿠키로 내려줍니다.

- 인증: 공개 (`refresh_token` 쿠키 필요)

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| cookie | `refresh_token` | string | O | 로그인이나 직전 재발급 때 받은 리프레시 토큰 |

본문은 없습니다.

### 응답

`200 OK`. `Set-Cookie`로 새 `refresh_token`이 내려옵니다.

```json
{ "data": "eyJhbGciOiJIUzI1NiJ9..." }
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `data` | string | 액세스 토큰 |

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 401 | `A001` | 쿠키가 없거나, 만료되었거나, 이미 쓴 토큰인 경우. 같은 토큰으로 동시에 요청하면 하나만 성공하고 나머지는 401입니다. |

## POST /auth/logout

리프레시 토큰을 폐기하고 쿠키를 지웁니다.

- 인증: 공개 (`refresh_token` 쿠키 필요)

### 요청

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| cookie | `refresh_token` | string | O | 폐기할 리프레시 토큰 |

본문은 없습니다.

### 응답

`204 No Content`. `Set-Cookie`로 `refresh_token`을 `Max-Age=0`으로 내려 쿠키를 지웁니다. 이미 폐기된 토큰이어도 204입니다.

### 오류

| 상태 | 코드 | 상황 |
|---|---|---|
| 401 | `A001` | 쿠키가 없거나 형식이 잘못된 경우 |
