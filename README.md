# Fin API 명세

Fin 백엔드([ApptiveDev/Fin-BE](https://github.com/ApptiveDev/Fin-BE))가 제공하는 HTTP API 명세입니다.

- 기준 커밋: `d21a882` (Fin-BE `main`, 2026-11-26)
- 개발 서버: `https://test-fin.duckdns.org`

## 문서 목록

- [인증](docs/auth.md): OAuth 로그인, 토큰 재발급, 로그아웃
- [회원](docs/user.md): 내 정보, 금융 프로필, 탈퇴
- [약관](docs/term.md): 약관 조회와 동의
- [상품 검색](docs/search.md): 추천 입력 폼, 상품 검색, 상품 상세
- [카테고리](docs/category.md): 카테고리 목록
- [금융사](docs/provider.md): 은행 목록
- [관심 상품](docs/favorites.md): 관심 상품 등록, 조회, 삭제
- [계산기](docs/calculator.md): 예적금 이자 계산
- [캘린더](docs/calendar.md): 정책 일정, 월별 요약
- [알림](docs/notification.md): 이메일 인증, 알림 설정, 수신거부

## 공통 규칙

### 요청

- 본문은 `Content-Type: application/json`으로 보냅니다.
- 날짜 파라미터는 `yyyy-MM-dd` 형식입니다.
- 브라우저에서는 리프레시 토큰 쿠키를 주고받기 위해 `withCredentials: true`(fetch는 `credentials: "include"`)로 호출합니다.

### 인증 수준

각 엔드포인트의 `인증` 항목은 아래 셋 중 하나입니다.

| 수준 | 조건 |
|---|---|
| 공개 | 토큰 없이 호출할 수 있습니다. |
| 로그인 | `Authorization: Bearer <액세스 토큰>` 헤더가 필요합니다. |
| 약관 동의 | 로그인에 더해 필수 약관에 모두 동의한 사용자(`RECOMMENDATION`, `ADMIN` 권한)만 호출할 수 있습니다. |

약관 동의 여부는 액세스 토큰에 담깁니다. 필수 약관에 동의한 뒤에는 [`POST /auth/refresh`](docs/auth.md#post-auth-refresh)로 토큰을 새로 받아야 `약관 동의` 엔드포인트를 쓸 수 있습니다.

### 성공 응답

응답 본문을 `data` 필드로 감싸는지는 엔드포인트마다 다릅니다. 각 엔드포인트의 응답 예시가 실제 형태입니다.

```json
{ "data": { } }
```

감싸는 경우의 형태입니다. 감싸지 않는 엔드포인트는 DTO나 값(`true`, `3` 등)을 그대로 반환합니다.

### 오류 응답

오류는 HTTP 상태 코드와 함께 아래 형식으로 반환합니다.

```json
{ "code": "C003", "message": "올바르지 않은 입력입니다." }
```

모든 엔드포인트에 공통으로 나올 수 있는 코드입니다. 도메인별 코드는 각 문서의 `오류` 표에 적습니다.

| 상태 | 코드 | 상황 |
|---|---|---|
| 400 | `C003` | 본문 JSON 파싱 실패, 검증 실패, 파라미터 누락이나 형식 오류. `message`에 실패 사유가 담깁니다. |
| 401 | `A001` | 토큰이 없거나 만료, 위조된 경우 |
| 403 | `A002` | 권한 부족 (예: 약관 미동의 사용자가 `약관 동의` 엔드포인트 호출) |
| 404 | `C001` | 없는 경로, 허용하지 않는 메서드, 없는 리소스 |
| 500 | `C002` | 서버 내부 오류 |
