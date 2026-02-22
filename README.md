# 인증 흐름 정리 (Auth Flow)

## 토큰 만료 시간

| 토큰 | 만료 시간 | 설정 위치 |
|------|----------|----------|
| Access Token | 24시간 (86400000ms) | `backend/src/main/resources/application-local.yml:14` |
| Refresh Token | 7일 (604800000ms) | `backend/src/main/resources/application-local.yml:15` |

---

## 1. 로그인

```
[사용자] 이메일 + 비밀번호 입력
       ↓
[frontend/src/views/auth/LoginView.vue:16]
  authApi.login(form.value)
       ↓
[frontend/src/api/auth.js:7]
  POST /api/auth/login
       ↓
[backend] AuthCommandController.java:33
  @PostMapping("/login") → authCommandService.login()
       ↓
[backend] AuthCommandService.java:58  login()
  1. userRepository.findByEmail()       → 없으면 USER_NOT_FOUND
  2. passwordEncoder.matches()          → 틀리면 LOGIN_FAILED
  3. jwtTokenProvider.createToken()        → Access Token 생성 (24h)
  4. jwtTokenProvider.createRefreshToken() → Refresh Token 생성 (7일)
  5. refreshTokenRepository.save()         → DB에 RefreshToken 저장
       ↓
[backend] AuthCommandController.java:84  buildTokenResponse()
  - accessToken  → Response Body
  - refreshToken → HttpOnly Cookie (maxAge=7일, SameSite=Strict)
       ↓
[frontend/src/api/index.js:24]
  응답 인터셉터 → response.data.data 반환
       ↓
[frontend/src/views/auth/LoginView.vue:17]
  authStore.setAuth(data.accessToken, { email })
  → localStorage에 accessToken 저장
```

### RefreshToken DB 구조
`backend/.../auth/command/domain/aggregate/RefreshToken.java`

```
refresh_tokens 테이블
├── userEmail (PK)  ← 이메일이 기본키 (1인 1토큰)
├── token           ← JWT 문자열
└── expiryDate      ← 만료 일시
```

> userEmail이 PK이므로 동일 계정으로 재로그인 시 기존 토큰이 **덮어쓰기(upsert)** 됨

---

## 2. JWT 토큰 생성 구조

`backend/.../security/JwtTokenProvider.java:39~66`

```
Access Token  = { subject: email, role: "USER", iat: 발급시각, exp: 발급+24h  } + HMAC-SHA 서명
Refresh Token = { subject: email, role: "USER", iat: 발급시각, exp: 발급+7일  } + HMAC-SHA 서명
```

- Secret Key: `jwt.secret` 값을 Base64 디코딩 후 HMAC 키로 사용 (`JwtTokenProvider.java:33~36`)

---

## 3. API 요청마다 토큰 검증

```
[frontend/src/api/index.js:14]
  요청 인터셉터 → Authorization: Bearer {accessToken} 헤더 자동 첨부
       ↓
[backend] SecurityConfig.java:55
  JwtAuthenticationFilter 가 UsernamePasswordAuthenticationFilter 앞에서 실행
       ↓
[backend] JwtAuthenticationFilter.java:27  doFilterInternal()
  1. getJwtFromRequest()        → "Bearer " 이후 토큰 추출
  2. validateToken()            → 서명 검증 + 만료 검사
  3. getUserEmailFromJWTToken()  → subject(email) 추출
  4. userDetailsService.loadUserByUsername(email) → DB 조회
  5. SecurityContextHolder 에 Authentication 등록
       ↓
[backend] SecurityConfig.java:50~53  접근 제어
  - POST /api/auth/** → permitAll  (비로그인 허용)
  - 그 외 모든 요청   → authenticated() 필요
```

### validateToken() 예외 처리
`JwtTokenProvider.java:82~100`

| 예외 | 의미 |
|------|------|
| `SecurityException / MalformedJwtException` | 서명 불일치 / 형식 오류 |
| `ExpiredJwtException` | 토큰 만료 |
| `UnsupportedJwtException` | 지원하지 않는 토큰 형식 |
| `IllegalArgumentException` | Claims 비어있음 |

> 모두 `BadCredentialsException`으로 변환 → Filter에서 catch 후 인증 없이 통과
> `authenticated()` 엔드포인트는 이후 Spring Security가 **401** 반환

---

## 4. Access Token 만료 시 자동 갱신

```
[frontend/src/api/index.js:29]
  응답 인터셉터 - 401 감지
  조건: status === 401 && !_retry  (무한루프 방지 플래그)
       ↓
  POST /api/auth/refresh
  쿠키의 refreshToken 자동 전송 (HttpOnly이므로 브라우저가 자동 첨부)
       ↓
[backend] AuthCommandController.java:59  @PostMapping("/refresh")
  - 쿠키에 refreshToken 없으면 → 401 즉시 반환
       ↓
[backend] AuthCommandService.java:83  refreshToken()
  1. validateToken()              → 토큰 서명/형식 검증
  2. getUserEmailFromJWTToken()   → email 추출
  3. refreshTokenRepository.findByUserEmail() → DB 조회
  4. 요청 토큰 == DB 토큰 비교    → 불일치면 INVALID_TOKEN
  5. storedToken.expiryDate < now → 만료면 EXPIRED_TOKEN
  6. 새 accessToken + refreshToken 발급
  7. refreshTokenRepository.save() → DB UPDATE (userEmail PK라 upsert)
       ↓
[backend] AuthCommandController.java:84  buildTokenResponse()
  - 새 accessToken  → Body
  - 새 refreshToken → 쿠키 갱신
       ↓
[frontend/src/api/index.js:34~36]
  authStore.setAuth(newToken, authStore.user)  → localStorage 갱신
  originalRequest 재시도                        → 원래 요청 자동 재실행
       ↓
  갱신 실패 시
  → authStore.logout() + router.push('/login')
```

---

## 5. 로그아웃

```
[frontend/src/components/layout/AppHeader.vue:11]
  handleLogout()
       ↓
[frontend/src/api/auth.js:10]
  POST /api/auth/logout
  (요청 인터셉터가 Authorization 헤더 자동 첨부)
       ↓
[backend] AuthCommandController.java:40  @PostMapping("/logout")
  @CookieValue("refreshToken") 쿠키에서 refreshToken 추출
       ↓
[backend] AuthCommandService.java:128  logout()
  1. validateToken()                          → 토큰 검증
  2. getUserEmailFromJWTToken()               → email 추출
  3. refreshTokenRepository.deleteByUserEmail() → DB에서 삭제
       ↓
[backend] AuthCommandController.java:103  createDeleteRefreshTokenCookie()
  maxAge=0 쿠키 응답 → 브라우저 쿠키 삭제
       ↓
[frontend/src/components/layout/AppHeader.vue:19]
  authStore.logout()
  → localStorage token 삭제
  → router.push('/login')
```

---

## 전체 흐름 다이어그램

```
┌─────────────────────────────────────────────────────────────────┐
│ 로그인                                                           │
│  비밀번호 BCrypt 검증 → Access(24h) + Refresh(7일) 발급         │
│  Refresh → DB 저장 + HttpOnly 쿠키 (JS 접근 불가)               │
│  Access  → Response Body → localStorage                          │
├─────────────────────────────────────────────────────────────────┤
│ API 요청 (정상)                                                  │
│  요청마다 Authorization: Bearer {Access} 헤더 자동 첨부          │
│  JwtAuthenticationFilter 에서 서명 + 만료 검증                   │
├─────────────────────────────────────────────────────────────────┤
│ Access 만료 (401)                                                │
│  쿠키의 Refresh 로 POST /auth/refresh 요청                       │
│  DB 토큰 일치 + 만료일 확인 → 새 토큰 2개 재발급                 │
│  원래 실패했던 요청 자동 재시도                                   │
│  Refresh 도 만료/불일치 → 강제 로그아웃                           │
├─────────────────────────────────────────────────────────────────┤
│ 로그아웃                                                         │
│  DB RefreshToken 삭제 + 쿠키 maxAge=0 + localStorage 삭제       │
└─────────────────────────────────────────────────────────────────┘
```
🔐 JWT 기반 인증 아키텍처 정리
1. Security 레이어 구성 (수업자료 ↔ 프로젝트 매핑)
역할	수업자료	프로젝트	비고
토큰 생성/검증	JwtTokenProvider	JwtTokenProvider	동일
사용자 조회	CustomUserDetailsService	CustomUserDetailsService	동일
요청 필터	JwtAuthenticationFilter	JwtAuthenticationFilter (@Component)	등록 방식만 다름
401 처리	RestAuthenticationEntryPoint	RestAuthenticationEntryPoint	동일
403 처리	RestAccessDeniedHandler	RestAccessDeniedHandler	동일
보안 설정	SecurityConfig	SecurityConfig	permitAll 범위 차이 있음
Refresh 저장소	AuthRepository	RefreshTokenRepository	이름만 변경
인증 서비스	AuthService	AuthCommandService	signup 추가
인증 컨트롤러	AuthController	AuthCommandController	signup 추가
permitAll 차이

수업자료:
/api/v1/auth/login, /api/v1/auth/refresh 개별 허용

프로젝트:
POST /api/auth/** 전체 허용 (범위가 더 넓음)

2. 전체 아키텍처 흐름

인증 흐름은 크게 3가지로 나뉜다.

로그인 (토큰 발급)

인증 요청 (보호된 API 접근)

토큰 재발급 (Refresh)

3. 로그인 흐름 (Token 발급)
Client
  ↓
POST /api/auth/login
  ↓
AuthCommandController.login()
  ↓
AuthCommandService.login()
  ↓
UserRepository.findByEmail()
  ↓
PasswordEncoder.matches()
  ↓
JwtTokenProvider.createToken()
JwtTokenProvider.createRefreshToken()
  ↓
RefreshTokenRepository.save()
  ↓
Response
  - accessToken (Body)
  - refreshToken (HttpOnly Cookie + Body)
핵심 포인트

Access Token → 짧은 수명

Refresh Token → 긴 수명 (DB 저장)

서버는 세션을 사용하지 않음 (Stateless)

4. 보호된 API 요청 흐름
Client
  ↓
GET /api/accounts
Header: Authorization: Bearer accessToken
  ↓
[Security Filter Chain]
  ↓
JwtAuthenticationFilter
  - 토큰 추출
  - 토큰 검증
  - 이메일 추출
  - DB 사용자 조회
  - SecurityContextHolder 인증 세팅
  ↓
Security 인가 판단
  - 인증 필요한 API인가?
      YES → 인증 정보 존재?
          NO  → 401 (AuthenticationEntryPoint)
          YES → Controller 진입
중요 개념

필터는 Controller보다 먼저 실행됨

Controller는 토큰을 직접 검증하지 않음

인증 여부는 SecurityContextHolder를 통해 판단됨

5. Access 만료 → Refresh 흐름
Client
  ↓
POST /api/auth/refresh
Cookie: refreshToken
  ↓
AuthCommandController.refresh()
  ↓
AuthCommandService.refreshToken()
  ↓
JwtTokenProvider.validateToken()
  ↓
RefreshTokenRepository.findByUserEmail()
  ↓
DB 토큰 비교
  ↓
만료일 검사
  ↓
새 access / refresh 발급
  ↓
RefreshTokenRepository.save() (갱신)
  ↓
Response
  - 새 accessToken
  - 새 refreshToken (쿠키 재설정)
왜 Refresh를 DB에 저장하는가?

서버가 발급한 토큰인지 검증하기 위함

로그아웃 시 강제 만료 가능

탈취 대응 가능

한 사용자당 하나의 refresh 유지 가능

6. 예외 처리 흐름
인증 실패 (401)

발생 조건:

토큰 없음

토큰 위조

토큰 만료

인증 정보 없음

처리:

RestAuthenticationEntryPoint → 401 반환
권한 부족 (403)

발생 조건:

인증은 되었으나 ROLE 불일치

처리:

RestAccessDeniedHandler → 403 반환
7. 레이어 구조 정리
Security Layer

SecurityConfig

JwtAuthenticationFilter

JwtTokenProvider

CustomUserDetailsService

RestAuthenticationEntryPoint

RestAccessDeniedHandler

Auth Layer

AuthCommandController

AuthCommandService

RefreshTokenRepository

UserRepository

Persistence Layer

MariaDB

User

RefreshToken

8. 핵심 개념 요약

서버는 세션을 저장하지 않는다.

Access Token이 곧 인증 정보다.

Refresh Token은 재발급 티켓이다.

인증은 필터에서 처리된다.

Controller는 인증이 완료된 요청만 받는다.

SecurityContextHolder는 현재 요청의 인증 정보를 저장한다.
---

## 핵심 설계 포인트

| 항목 | 설명 |
|------|------|
| Refresh Token DB 저장 | 서버에서 강제 만료(탈취 대응, 로그아웃) 가능 |
| HttpOnly 쿠키 | JavaScript 접근 불가 → XSS 공격으로 Refresh Token 탈취 불가 |
| SameSite=Strict | CSRF 공격 방어 |
| userEmail = PK | 계정당 RefreshToken 1개 유지, 재로그인 시 자동 교체 |
| STATELESS 세션 | 서버에 세션 저장 없음, JWT로만 인증 |
| _retry 플래그 | 갱신 실패 시 무한 재시도 루프 방지 |
