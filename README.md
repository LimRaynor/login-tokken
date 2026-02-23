# 인증 흐름 정리 (Auth Flow)

1. SecurityConfig
-   클라이언트 요청 → CORS 필터 → JWT 인증 필터 → Security 인가 검사 → Controller

2. Service 비교 (로그인 핵심 로직)
-   ① 사용자 조회 → ② 비밀번호 검증 → ③ 토큰 생성 → ④ RefreshToken DB 저장
   
- 전체 로그인 시퀀스 요약

클라이언트                Controller              Service              DB
|                        |                      |                   |
|-- POST /login -------->|                      |                   |
|   {email, password}    |-- login(request) --->|                   |
|                        |                      |-- findByEmail --->|
|                        |                      |<-- User 반환 -----|
|                        |                      |                   |
|                        |                      | passwordEncoder   |
|                        |                      | .matches() 검증    |
|                        |                      |                   |
|                        |                      | AccessToken 생성   |
|                        |                      | RefreshToken 생성  |
|                        |                      |                   |
|                        |                      |-- save(token) --->|
|                        |<-- TokenResponse ----|                   |
|<-- 200 OK ------------|                      |                   |
|  body: {accessToken,   |                      |                   |
|         refreshToken}  |                      |                   |
|  cookie: refreshToken  |                      |                   |
|  (HttpOnly)            |                      |                   |

  ---

3. JWT 토큰 생성 (JwtTokenProvider)
- (앱 시작 시) Secret Key 생성(@PostConstruct, 1회)
- (요청 시) Access Token 생성 -> Refresh Token 생성 -> 토큰 검증 (validateToken) -> 토큰에서 사용자 정보 추출

4. JWT 인증 필터 (JwtAuthenticationFilter)
   ① 토큰 추출 (getJwtFromRequest) ->  ② 토큰 유효성 확인 -> ③ 이메일 추출 ->  ④ UserDetails 로드
->  ⑤ 인증 객체 생성 & 등록 -> ⑥ 다음 필터로 전달

5.  토큰 갱신 (AuthCommandService.refreshToken)
    Cookie에서 refreshToken 추출 -> 토큰 서명 검증 -> 이메일 추출 ->  DB 토큰 조회
-> 토큰 일치 확인 -> 만료 확인 -> 새 두가지 토큰  발급 -> 같은 이메일로 기존토큰 덮어서DB 갱신 -> 새로만든 토큰으로 응답 반환

6. 로그아웃 (AuthCommandService.logout)
   Cookie에서 refreshToken 추출 -> 토큰 검증 -> 이메일 추출 -> DB에서 RefreshToken 삭제
-> 쿠키 삭제 maxAge=0 (즉시 만료) -> 클라이언트(access토큰 제거후 메인페이지로 이동)

---

Spring Security는: 모든 HTTP요청 → FilterChain → Controller
JwtAuthenticationFilter extends OncePerRequestFilter == 요청한번당 한번만 실행 (중복방지)
→ 모든 요청이 이 필터를 거치지만, 토큰이 없으면 인증 없이 통과(chain.doFilter)
