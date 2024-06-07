---
title: Spring Boot (kotlin) jwt(2/2)
categories:
  - spring boot
tags:
  - springboot
  - springsecurity
  - kotlin
  - jwt
---
# Spring Boot (Kotlin) JWT 로그인(2/2)
저번 글에서는 JWT가 무엇이고 어떤 형식으로 사용되는지 알아보았다. jwt 토큰에 대해 어떻게 사용해야 하는지 알게 되었다.

이번에는 kotlin으로 직접 코드를 변환해 보면서 어떤 식으로 동작하는 지 알아가 볼려고 한다.

> 참고로 데이터베이스를 사용하는데 User 테이블안에 refresh token을 저장하고 있다.

## 1. 의존성 주입
이번 프로젝트에서는 사람들이 많이 사용하는  **jjwt 라이브러리**를 사용한다. 라이브러리를 추가하기 위해 아래와 같이 입력한다.

```gradle
// spring security
implementation("org.springframework.boot:spring-boot-starter-security")

// jwt 라이브러리
implementation("io.jsonwebtoken:jjwt-api")  
runtimeOnly("io.jsonwebtoken:jjwt-impl")  
runtimeOnly("io.jsonwebtoken:jjwt-jackson")
```

## 2. 설정 파일 작성
jwt에서 핵심적인 설정 부분은 보여 주면 안되기 때문에 별도의 YML 파일로 분리하였다.
해당 코드 안에는 jwt 암호화 키와 각 토큰의 만료시간을 주었다.

**application-jwt.yml** 파일을 resources 파일 안에 생성한다.

```yml
jwt:  
  # jwt 암호화 키  
  secretKey: 3b672bb100b6297eb3af1c41ae200e730adda560c4df3366d3bb0860087bd4e3f9ca2a1763fb9e97915d6e280326fa5fb17b62178e427a5466c559acee859fa8  
  
  # access token 설정  
  access:  
    expiration: 600000  
    header: Authorization  
  
  # refresh token 설정  
  refresh:  
    expiration: 1209600000  
    header: Authorization-refresh
```

다른 YML 파일을 사용하기 위해 application.yml 파일에 다음과 같이 작성하면 된다.

```yml
spring:   
  profiles:  
    include:  
      - jwt # application-jwt.yml 파일 임포트
```

## 3.  JWTService 코드 작성

jwt 대한 로직을 수행하는 코드이다.

전체 코드는 다음과 같다.

```kotlin
//import 문 생략  

@Service 
class JwtService(private val userRepository: UserRepository) {  

    private val log = LoggerFactory.getLogger(this.javaClass)  
  
    // static final 상수 정의  
    companion object {  
        private const val ACCESS_TOKEN_SUBJECT = "Authorization"  
        private const val REFRESH_TOKEN_SUBJECT = "Authorization refresh"  
        private const val BEARER = "Bearer "  
    }  
	
    @Value("\${jwt.secretKey}")  
    private lateinit var secretKey: String  
  
    @Value("\${jwt.access.expiration}")  
    private lateinit var accessTokenExpiration: String // lateinit 은 기본형 초기화를 지원하지 않는다.  
  
    @Value("\${jwt.refresh.expiration}")  
    private lateinit var refreshTokenExpiration: String  
  
    @Value("\${jwt.access.header}")  
    private lateinit var accessTokenHeader: String  
  
    @Value("\${jwt.refresh.header}")  
    private lateinit var refreshTokenHeader: String  
  
    // jwt 토큰에 사용되는 키  
    private lateinit var key: SecretKey  
  
    @PostConstruct  
    fun init() {  
        // Base64로 인코딩  
        val encodeKey = Base64.getEncoder().encodeToString(secretKey.toByteArray())  
        key = Keys.hmacShaKeyFor(encodeKey.toByteArray())  
    }  
  
    /**  
     *  Access Token 생성 메소드     */    fun generateAccessToken(email: String): String {  
        val now = Date()  
        return Jwts.builder()  
            .subject(ACCESS_TOKEN_SUBJECT)  
            .expiration(Date(now.time + accessTokenExpiration.toLong()))  
            .claim("email", email)  
            .signWith(key)  
            .compact()  
    }  
  
    /**  
     *  Refresh Token 생성 메소드     */    fun generateRefreshToken(): String {  
        val now = Date()  
        return Jwts.builder()  
            .subject(REFRESH_TOKEN_SUBJECT)  
            .expiration(Date(now.time + refreshTokenExpiration.toLong()))  
            .signWith(key)  
            .compact()  
    }  
  
    /**  
     *  Access Token을 헤더에 담아서 보낸다.     */    fun sendAccessToken(response: HttpServletResponse, accessToken: String) {  
        response.status = HttpServletResponse.SC_OK  
        response.addHeader(accessTokenHeader, accessToken)  
    }  
  
    /**  
     * Access Token과 Refresh Token을 헤더에 담아서 보낸다.     */    fun sendAccessAndRefreshToken(response: HttpServletResponse, accessToken: String, refreshToken: String) {  
        response.status = HttpServletResponse.SC_OK  
        response.addHeader(accessTokenHeader, accessToken)  
        response.addHeader(refreshTokenHeader, refreshToken)  
    }  
  
    /**  
     * 데이터베이스에 저장된 Refresh Token 가져오기     */    fun getRefreshToken(email: String): String {  
        val user = userRepository.findByEmail(email)  
            .orElseThrow { Error("사용자를 찾을 수 없습니다.") }  
        return user.refreshToken!!  
    }  
  
    /**  
     * 데이터베이스에 저장된 Refresh Token 업데이트     */    fun updateRefreshToken(email: String, refreshToken: String) {  
        val user = userRepository.findByEmail(email)  
            .orElseThrow { Error("사용자를  찾을 수 없습니다,") }  
  
        user.updateToken(refreshToken)  
        userRepository.save(user)  
    }  
  
    /**  
     *  받은 Refresh Token 에서 Bearer 제거 시키기     */    fun extractRefreshToken(request: HttpServletRequest): Optional<String> {  
        return Optional.ofNullable(request.getHeader(refreshTokenHeader))  
            .filter { it.startsWith(BEARER) }  
            .map { it.replace(BEARER, "") }  
    }  
  
    /**  
     *  받은 Access Token 에서 Bearer 제거 시키기     */    fun extractAccessToken(request: HttpServletRequest): Optional<String> {  
        return Optional.ofNullable(request.getHeader(accessTokenHeader))  
            .filter { it.startsWith(BEARER) }  
            .map { it.replace(BEARER, "") }  
    }  
  
    fun extractEmail(token: String): Optional<String> {  
        try {  
            val claims = Jwts.parser()  
                .verifyWith(key)  
                .build()  
                .parseSignedClaims(token)  
            return Optional.ofNullable(claims.payload.get("email", String::class.java))  
        } catch (e: Exception) {  
            return Optional.empty()  
        }  
    }  
  
    /**  
     * 토큰 검증 메소드     */    fun isTokenValid(token: String): Boolean {  
        try {  
            Jwts.parser().verifyWith(key).build().parseSignedClaims(token)  
            return true  
        } catch (e: MalformedJwtException) {  
            log.error("Invalid JWT token: {}", e.message)  
        } catch (e: ExpiredJwtException) {  
            log.error("JWT token is expired: {}", e.message)  
        } catch (e: UnsupportedJwtException) {  
            log.error("JWT token is unsupported: {}", e.message)  
        } catch (e: IllegalArgumentException) {  
            log.error("JWT claims string is empty: {}", e.message)  
        }  
  
        return false  
    }  
  
    fun isRefreshTokenValid(token: String): Boolean {  
        if (isTokenValid(token)) {  
            return true  
        }  
  
        // 토큰이 같은지 검사를 한다.  
        val findByRefreshToken = userRepository.findByRefreshToken(token)  
  
        return !findByRefreshToken.isEmpty  
    }  
  
  
}
```

여기서 필드의 값들은 `@Value`를 통해 YML 파일에서 가져 오고 있다.

> `@Value`의 경우 초기화가 바로 되지 않기 때문에 `latent var`를 사용한다.

기능에 대한 설명에 앞서 각 메소드 마다 주석을 통해 설명을 적어 놓았다.

### 3-1. 토큰 생성 부분

```kotlin
fun generateAccessToken(email: String): String {  
    val now = Date()  
    return Jwts.builder()  
        .subject(ACCESS_TOKEN_SUBJECT)  
        .expiration(Date(now.time + accessTokenExpiration.toLong()))  
        .claim("email", email)  
        .signWith(key)  
        .compact()  
}  
  
/**  
 *  Refresh Token 생성 메소드 */fun generateRefreshToken(): String {  
    val now = Date()  
    return Jwts.builder()  
        .subject(REFRESH_TOKEN_SUBJECT)  
        .expiration(Date(now.time + refreshTokenExpiration.toLong()))  
        .signWith(key)  
        .compact()  
}
```

여기서 **만료 시간인 경우 발급한 시간 기준**에 하기 때문에 Date 클래스의 time을 통해 현재 시간을 구하였다.

**AccessToken인 경우** claim에 사용자를 구분하기 위한 email을 주었고 **RefreshToken인 경우** claim이 필요가 없기 때문에 추가하지 않았다.

### 3-2 토큰 헤더 추가 부분
```kotlin
/**  
 *  Access Token을 헤더에 담아서 보낸다. */fun sendAccessToken(response: HttpServletResponse, accessToken: String) {  
    response.status = HttpServletResponse.SC_OK  
    response.addHeader(accessTokenHeader, accessToken)  
}  
  
/**  
 * Access Token과 Refresh Token을 헤더에 담아서 보낸다. */fun sendAccessAndRefreshToken(response: HttpServletResponse, accessToken: String, refreshToken: String) {  
    response.status = HttpServletResponse.SC_OK  
    response.addHeader(accessTokenHeader, accessToken)  
    response.addHeader(refreshTokenHeader, refreshToken)  
}
```

두 메소드의 공통점은 Response 헤더의 Status 를 OK(200)으로 설정한 것이다.

헤더를 통해 클라이언트로 보내는 것이다.

> Access Token은 **Authorization** 헤더가 되고, <br>
> RefreshToken은 **Authorization-refresh** 헤더가 된다.

### 3-3 토큰 추출 부분

```kotlin
/**  
 *  받은 Refresh Token 에서 Bearer 제거 시키기 
 */
 fun extractRefreshToken(request: HttpServletRequest): Optional<String> {  
    return Optional.ofNullable(request.getHeader(refreshTokenHeader))  
        .filter { it.startsWith(BEARER) }  
        .map { it.replace(BEARER, "") }  
}  
  
/**  
 *  받은 Access Token 에서 Bearer 제거 시키기 
 */
 fun extractAccessToken(request: HttpServletRequest): Optional<String> {  
    return Optional.ofNullable(request.getHeader(accessTokenHeader))  
        .filter { it.startsWith(BEARER) }  
        .map { it.replace(BEARER, "") }  
}
```

클라이언트에서 헤더 로 보낸  토큰에서 "Bearer " 부분을 제거 한 다음 해당 토큰을 추출하는 코드이다.

> `Optional`을 통해 null 값을 허용하고 있다.

아래는 추출한 refresh token을 통해 데이터베이스에 저장된 토큰을 업데이트 하거나 해당 토큰에 저장된 이메일을 추출하는 코드이다.

```kotlin
/**  
 * 데이터베이스에 저장된 Refresh Token 가져오기 
 */
 fun getRefreshToken(email: String): String {  
    val user = userRepository.findByEmail(email)  
        .orElseThrow { Error("사용자를 찾을 수 없습니다.") }  
    return user.refreshToken!!  
}  
  
/**  
 * 데이터베이스에 저장된 Refresh Token 업데이트 
 */
 fun updateRefreshToken(email: String, refreshToken: String) {  
    val user = userRepository.findByEmail(email)  
        .orElseThrow { Error("사용자를  찾을 수 없습니다,") }  
  
    user.updateToken(refreshToken)  
    userRepository.save(user)  
}
```

### 3-4 토큰 검증 코드
```kotlin
/**  
 * 토큰 검증 메소드 
 */
 fun isTokenValid(token: String): Boolean {  
    try {  
        Jwts.parser().verifyWith(key).build().parseSignedClaims(token)  
        return true  
	    // 잘못된 형식의 토큰일 경우 발생
    } catch (e: MalformedJwtException) {  
        log.error("Invalid JWT token: {}", e.message)  
        // 토큰이 만료 된 경우 발생
    } catch (e: ExpiredJwtException) {  
        log.error("JWT token is expired: {}", e.message)
        
        // 지원 하지 않는 형식의 토큰인 경우 발생
    } catch (e: UnsupportedJwtException) {  
        log.error("JWT token is unsupported: {}", e.message)  
        
        //JWT 토큰의 클레임 문자열이 비어있는 경우 발생
    } catch (e: IllegalArgumentException) {  
        log.error("JWT claims string is empty: {}", e.message)  
    }  
  
    return false  
}
```

해당 메서드는 받은 토큰을 검증하는 코드이다. 

`verifyWith` 를 통해 암호화된 코드를 다시 복호화를 통해 claim을 추출한다.

> 현재는 에러 로그를 출력하지만 예외 처리를 할 수 있도록 코드를 수정할 예정이다.

## 4. jwt 인증 필터 설명
jwt를 발급하고 추출하는 서비스 부분을 작성하였으니 이제 실제로 적용을 해야 한다.

만들어 놓은 JWT 필터는 특정 요청(회원 가입, 로그인 등)을 제외하고는 거의 모든 요청에  토큰을 확인하고 검증하는 작업이 있다. 

JWT 필터는 `OncePerRequestFilter` 를 상속 받아 구현을 한다.

> **OncePerRequestFilter**는 Spring에서 제공하는 추상클래스로 HTTP 요청에 대해 필터가 한 번만 실행된다.

해당 필터에 대해 간단하게 먼저 설명을 하자면 "/login" 이외에 요청을 보냈을 시, **토큰들을 받아서 유효성 검사를 실시하며 인증 처리/인증 실패/토큰 재발급 등을 수행하는 역할을 하는 필터**이다.

JWT 토큰 발급에 대한 설명은 이전 페이지를 참고하자.

[Spring Boot (Kotlin) JWT 로그인(1/2)](https://codevarty.github.io/spring%20boot/spring-boot-kotlin-jwt-1/)

## 5. JwtAuthenticationProcessingFilter 부분 작성

jwt 필터의 전체 코드는 다음과 같다.

```kotlin
// import 문 생략..
  
class JwtAuthenticationProcessingFilter(  
    private val jwtService: JwtService,  
    private val userRepository: UserRepository,  
    private val customUserDetailService: CustomUserDetailService  
) : OncePerRequestFilter() {  
  
    companion object {
        private const val NO_CHECK_URL = "/api/user/login"  
    }  
  
    private val authoritiesMapper = NullAuthoritiesMapper()  
  
    @Throws(ServletException::class, IOException::class)  
    override fun doFilterInternal(  
        request: HttpServletRequest,  
        response: HttpServletResponse,  
        filterChain: FilterChain  
    ) {  
        if (request.requestURI == NO_CHECK_URL) {  
            filterChain.doFilter(request, response) // /login 호출이 들어오면 다음 필터 호출  
            return  
        }  
        // 사용자 요청에서 refresh token 추출  
        val refreshToken = jwtService.extractRefreshToken(request)  
            .filter(jwtService::isTokenValid)  
            .orElse(null)  
  
        // 1. refresh token 이 있는 경우  
        if (refreshToken != null) {  
            checkRefreshTokenAndReIssueAccessToken(response, refreshToken)  
        }  
  
        // 2. refresh token 이 없는 경우  
        checkAccessTokenAndAuthentication(request, response, filterChain)  
    }  
  
    private fun checkRefreshTokenAndReIssueAccessToken(response: HttpServletResponse, refreshToken: String) {  
        userRepository.findByRefreshToken(refreshToken)  
            .ifPresent { user ->  
                run {  
                    val reIssuedRefreshToken = reIssueRefreshToken(user)  
                    jwtService.sendAccessAndRefreshToken(  
                        response,  
                        jwtService.generateAccessToken(user.email),  
                        reIssuedRefreshToken  
                    )  
                }  
            }    }  
  
    private fun reIssueRefreshToken(user: User): String {  
        val generateRefreshToken = jwtService.generateRefreshToken()  
        user.updateToken(generateRefreshToken)  
        userRepository.save(user)  
        return generateRefreshToken  
    }  
  
    @Throws(ServletException::class, IOException::class)  
    fun checkAccessTokenAndAuthentication(  
        request: HttpServletRequest?, response: HttpServletResponse?,  
        filterChain: FilterChain  
    ) {  
        jwtService.extractAccessToken(request!!)  
            .filter(jwtService::isTokenValid)  
            .ifPresent { accessToken ->  
                jwtService.extractEmail(accessToken)  
                    .ifPresent { email ->  
                        userRepository.findByEmail(email)  
                            .ifPresent { myUser: User? -> this.saveAuthentication(myUser!!) }  
                    }            }        filterChain.doFilter(request, response)  
    }  
  
    private fun saveAuthentication(user: User) {  
        val customUserDetails = customUserDetailService.loadUserByUsername(user.email)  
        val authentication = UsernamePasswordAuthenticationToken(  
            customUserDetails,  
            null,            authoritiesMapper.mapAuthorities(customUserDetails.authorities)  
        )  
  
        SecurityContextHolder.getContext().authentication = authentication  
    }  
}
```

클라이언트에서 로그인 요청시 다음 필터로 건너 뛰며 로그인 외에는 다음과 같다. 

클라이언트 요청 헤더에서 RefreshToken의 유무에 따라 두 가지 분기 작용
1. 토큰이 있을 경우 AccessToken 및 Refresh 토큰재발급
2. 토큰이 없을 경우 AccessToken을 검증 한 후 다음 필터 적용

### 5-1. Refresh Token 확인 및 토큰 재발급

```kotlin
// Refresh 토큰을 통한 유저 확인 및 재발급
private fun checkRefreshTokenAndReIssueAccessToken(response: HttpServletResponse, refreshToken: String) {  
    userRepository.findByRefreshToken(refreshToken)  
        .ifPresent { user ->  
            run {  
                val reIssuedRefreshToken = reIssueRefreshToken(user)  
                jwtService.sendAccessAndRefreshToken(  
                    response,  
                    jwtService.generateAccessToken(user.email),  
                    reIssuedRefreshToken  
                )  
            }  
        }}  

// RefreshToken 재발급 및 DB 업데이트 메소드
private fun reIssueRefreshToken(user: User): String {  
    val generateRefreshToken = jwtService.generateRefreshToken()  
    user.updateToken(generateRefreshToken)  
    userRepository.save(user)  
    return generateRefreshToken  
}
```

RefreshToken을 확인 하고 재발급을 하는 로직으로 각 메서드의 기능은 다음과 같다.

- **checkRefreshTokenAndReIssueAccessToken** : 추출된 refreshToken을 통해 데이터베이스에 저장된 유저 정보를 얻는 다. 그후 `reIssueRefreshToken` 메소드를 통해 RefreshToken을 클라이언트에 재발급한다.
- **reIssueRefreshToken** : 사용자 클래스를 받아서 RefreshToken을 새로 생성한 다음 변경된  RefreshToken으로 데이터베이스 업데이트를 한다.

### 5-2 AccessToken 검증 및 인증 처리

```kotlin
// java throws와 같음.
@Throws(ServletException::class, IOException::class)  
fun checkAccessTokenAndAuthentication(  
    request: HttpServletRequest?, response: HttpServletResponse?,  
    filterChain: FilterChain  
) {  
    jwtService.extractAccessToken(request!!)  
        .filter(jwtService::isTokenValid)  
        .ifPresent { accessToken ->  
            jwtService.extractEmail(accessToken)  
                .ifPresent { email ->  
                    userRepository.findByEmail(email)  
                        .ifPresent { myUser: User? -> this.saveAuthentication(myUser!!) }  
                }        }    filterChain.doFilter(request, response)  
}  

// 인증 처리 메서드
private fun saveAuthentication(user: User) {  
    val customUserDetails = customUserDetailService.loadUserByUsername(user.email)  
    val authentication = UsernamePasswordAuthenticationToken(  
        customUserDetails,  
        null,
        // 사용자의 권한 목록을 매핑하여 설정한다.
        authoritiesMapper.mapAuthorities(customUserDetails.authorities)  
    )  

	// 생성된 인증 객체를 현재 보안 컨텍스트에 설정.
	// 이를 통해 실행되는 모든 코드가 이 인증 정보를 참조할 수 있게된다.
    SecurityContextHolder.getContext().authentication = authentication  
}
```

`jwtService.extractAccessToken`를 통해 AccessToken을 추출한 다음 이메일을 추출해서 해당 이메일의 유저를 찾아 `saveAuthentication`메서드를 실행 시켜 인증 처리를 한다.

`customUserDetailService.loadUserByUsername(user.email)`를 통해 유저의 이메일로 `CustomUserDetails` 클래스를 생성하여 `SecurityContextHolder.getContext().authentication`에 담아 인증 처리를 한다.

>CustomUserDetails와 CustomUserDetailService 코드는 마지막 부분에 추가 코드 작성 부분을 참고하면 된다. (글을 다 작성한 이후 알게 되었다...)
## 6. JSON 로그인 필터 적용

로그인을 할 때 JSON 통신을 하기 때문에 RequestBody 형식으로 한다.

요청 형식은 다음과 같다.

```json
{

"email" : "test@naver.com",
"password" : "test1234!"

}
```

email과 password로 로그인 방식을 구현한다.

JSON 로그인 필터는 `AbstractAuthenticationProcessingFilter` 추상 클래스를 상속받아 작성한다. 

> 기본적으로 **attemptAuthentication()** 은 인증 처리 메소드이다.

전체 코드는 다음과 같다.

```kotlin
//import 문 생략...

class CustomJsonLoginFilter(private val objectMapper: ObjectMapper) :  
    AbstractAuthenticationProcessingFilter(AntPathRequestMatcher("/api/user/login", "POST")) {  
  
  
    private val log = LoggerFactory.getLogger(CustomJsonLoginFilter::class.java)  
  
  
    override fun attemptAuthentication(request: HttpServletRequest?, response: HttpServletResponse?): Authentication {  
        if (request === null) {  
            throw AuthenticationServiceException("AuthenticationService")  
        }  
  
  
        if (request.contentType === null || request.contentType != "application/json") {  
            throw AuthenticationServiceException("Authentication Content-Type not supported: " + request.contentType)  
        }  
  
        val messageBody = StreamUtils.copyToString(request.inputStream, StandardCharsets.UTF_8)  
        val usernamePasswordMap: MutableMap<String, String>? = objectMapper.readValue(  
            messageBody,  
            MutableMap::class.java  
        ) as MutableMap<String, String>?  
  
        val email = usernamePasswordMap?.get("email")  
        val password = usernamePasswordMap?.get("password")  
  
  
        log.info("email=$email, password=$password")  
  
        val authenticationToken = UsernamePasswordAuthenticationToken(email, password)  
        return authenticationManager.authenticate(authenticationToken)  
    }  
}
```

login URL 요청시 해당 필터가 작동을 한다.

### 6-1. 인증 처리 부분

```kotlin
val messageBody = StreamUtils.copyToString(request.inputStream, StandardCharsets.UTF_8)  
val usernamePasswordMap: MutableMap<String, String>? = objectMapper.readValue(  
    messageBody,  
    MutableMap::class.java  
) as MutableMap<String, String>?

val email = usernamePasswordMap?.get("email")  
val password = usernamePasswordMap?.get("password")

// email과 password 출력
log.info("email=$email, password=$password")

// 사용자 이메일과 비밀번호를 통해 인증 토큰 발행
val authenticationToken = UsernamePasswordAuthenticationToken(email, password)  
return authenticationManager.authenticate(authenticationToken)
```

objectMapper을 통해 request를 통해 받은 JSON 부분에서 email과 password 부분을 받는다.

`UsernamePasswordAuthenticationToken` 클래스는 Spring Security에서 제공하는 인증 객체로, 사용자의 자격 증명을 나타낸다. 사용자의 이메일과 비밀번호를 받아 인증 토큰을 생성하다.

그다음 `authenticationManager.authenticate(authenticationToken)`을 통해 인증을 시도한다.
인증에 성공하면 인증 정보가 반환되며, 실패할 때 인증 실패 예외가 발생한다.

## 7. 로그인 성공 & 실패 부분

이번 코드에서는 로그인 필터에서 성공 혹은 실패 했을 때 Handler 라는 클래스를 통해 처리를 한다.

이 핸들러는 컴포넌트로 로그인 필터 후에 실행이 된다. 

각각의 핸들러 코드는 다음과 같다.

### 7-1. 로그인 성공 핸들러

해당 핸들러는 `SimpleUrlAuthenticationSuccessHandler` 클래스를 상속바다 사용된다.

전체 코드는 다음과 같다.

```kotlin
//import문 생략..

@Component  
class LoginSuccessHandler( // 로그인 성공시 실행 되는 핸들러  
    private val jwtService: JwtService,  
    // userService 를 사용하면 filter 순환 에러가 발생하므로 userRepository 를 사용  
    private val userRepository: UserRepository,  
    private val objectMapper: ObjectMapper  
) : SimpleUrlAuthenticationSuccessHandler() {  
  
  
    private val log = LoggerFactory.getLogger(LoginSuccessHandler::class.java)  
  
    @Value("\${jwt.access.expiration}")  
    private lateinit var accessTokenExpiration: String  
  
    private fun extractEmail(authentication: Authentication): String {  
        val userDetails: CustomUserDetails = authentication.principal as CustomUserDetails  
        return userDetails.getEmail()  
    }  
  
    @Throws(IOException::class, ServletException::class)  
    override fun onAuthenticationSuccess(  
        request: HttpServletRequest?,  
        response: HttpServletResponse,  
        authentication: Authentication? // 사용자 인증 정보
    ) {  
        if (authentication == null) return  
        val email: String = extractEmail(authentication)  
        val accessToken: String = jwtService.generateAccessToken(email)  
        val refreshToken: String = jwtService.generateRefreshToken()  
  
        // 사용자가 로그인 할 때 마다  새로운 토큰을 생성하여 발급한다.  
        jwtService.sendAccessAndRefreshToken(response, accessToken, refreshToken)  
        
        val user: User = userRepository.findByEmail(email)  
            .orElseThrow { Error("유저를 찾을 수 없습니다.") }  
  
        jwtService.updateRefreshToken(user.email, refreshToken)  
  
        log.info("Login Success. email={}", user.email)  
        log.info("Login Success. AccessToken={}", accessToken)  
        log.info("accessToken Expiration={}", accessTokenExpiration)  
  
        response.status = HttpServletResponse.SC_OK // response.status == 200 (ok)
        response.contentType = "application/json"  
        response.characterEncoding = "UTF-8"  

		// 클라이언트에 반환할 response dto
        val userResponse = UserResponse(  
            username = user.name,  
            email = user.email,  
            birthDate = user.birthdate  
        )  
  
        objectMapper.writeValue(response.writer, userResponse)  
    }
```

로그인에 성공을 했을 때 `JwtService`를 통해 클라이언트에 헤더로 AccessToken과 RefreshToken을 을 발급한다.

로그인 정보로 이메일을 확인하여 DB에 사용자가 없으면 에러 메시지를 출력한다.

그리고 클라이언트에 UserResponse dto 클래스를 사용하여 사용자 이름, 이메일, 생일을 주게 된다.


### 7-2. 로그인 실패 핸들러

해당 핸들러는  `SimpleUrlAuthenticationFailureHandler` 클래스를 상속 받아 실행 된다.

전체 코드는 다음과 같다.

```kotlin
//import 문 생략..

@Component // 로그인 실패시 실행 되는 핸들러.  
class LoginFailureHandler(private val objectMapper: ObjectMapper) :  
    SimpleUrlAuthenticationFailureHandler() {  
  
    companion object {  
        private const val ERROR_MESSAGE = "이메일이나 비밀번호를 확인해주세요."  
    }  
  
    override fun onAuthenticationFailure(  
        request: HttpServletRequest?,  
        response: HttpServletResponse?,  
        exception: AuthenticationException?  
    ) {  
        if (response == null || !response.isCommitted) {  
            throw AuthenticationServiceException("Authentication failed")  
        }  
  
        response.status = HttpServletResponse.SC_BAD_REQUEST  
        response.contentType = MediaType.APPLICATION_JSON_VALUE  
        objectMapper.writeValue(response.outputStream, ERROR_MESSAGE)  
  
    }  
}
```

로그인이 실패 했을 때 에러 메시지를 클라이언트에 보내준다.

이 때 HTTP status 값은 400 (BadRequest)가 된다.

이제 기본적으로 작성해야 할 클래스 부분을 만들었다. 이 부분을 Spring Security에 적용하기 위해 Security Config 클래스를 생성하여 필터를 적용해야 한다.

## 9. Security Config 클래스 작성

해당 클래스에서 이 때 까지 만들어 놓은 필터와 핸들러 클래스를 사용하게 된다.

전체 코드는 다음과 같다.

```kotlin
//import 생략..

@Configuration  
@EnableWebSecurity  
class SecurityConfig(  
    private val jwtService: JwtService,  
    private val userRepository: UserRepository,  
    private val customUserDetailService: CustomUserDetailService,  
    private val objectMapper: ObjectMapper,  
    private val loginSuccessHandler: LoginSuccessHandler,  
    private val loginFailureHandler: LoginFailureHandler,  
  
    ) {  
  
    @Bean  
    fun passwordEncoder(): BCryptPasswordEncoder = BCryptPasswordEncoder()  

	// 허용 URL
    private val allowPatterns = arrayOf("/", "/api/user/signup")  
  
    @Bean  
    fun authenticationManager(): AuthenticationManager {  
        val provider = DaoAuthenticationProvider()  
        provider.setPasswordEncoder(passwordEncoder())  
        provider.setUserDetailsService(customUserDetailService)  
        return ProviderManager(provider)  
    }  
  
    @Bean  
    fun customJsonLoginFilter(): CustomJsonLoginFilter {  
        val customJsonLoginFilter = CustomJsonLoginFilter(objectMapper)  
        customJsonLoginFilter.setAuthenticationManager(authenticationManager())  
        customJsonLoginFilter.setAuthenticationSuccessHandler(loginSuccessHandler)  
        customJsonLoginFilter.setAuthenticationFailureHandler(loginFailureHandler)  
        return customJsonLoginFilter  
  
    }  
  
    @Bean  
    fun jwtAuthenticationProcessingFilter(): JwtAuthenticationProcessingFilter {  
        return JwtAuthenticationProcessingFilter(jwtService, userRepository, customUserDetailService)  
    }  
  
    @Bean  
    open fun filterChain(http: HttpSecurity): SecurityFilterChain {  
  
        http.csrf { it.disable() }  
        http.httpBasic { it.disable() } // form login, redirect 비활성화 => rest api 통신을 하기 때문  
  
        http  
            .authorizeHttpRequests { authorizeRequests ->  
                authorizeRequests  
                    .requestMatchers(*allowPatterns).permitAll()  
                    .anyRequest().authenticated()// 테스트를 위해 모두 허용  
            }  
  
        // 세션을 사용하지 않으므로 STATELESS 설정  
        http.sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }  
  
        // filter 적용  
        http.addFilterAfter(customJsonLoginFilter(), LogoutFilter::class.java)  
        http.addFilterBefore(jwtAuthenticationProcessingFilter(), CustomJsonLoginFilter::class.java)  
  
        return http.build()  
    }  
}
```


### 9-1 빈 등록 메소드 부분
먼저 가장 중요한 `filterChain`을 제외한 다른 메소드들을 설명할려고 한다.

```kotlin
@Bean  
fun passwordEncoder(): BCryptPasswordEncoder = BCryptPasswordEncoder()   
  
@Bean  
fun authenticationManager(): AuthenticationManager {  
    val provider = DaoAuthenticationProvider()  
    provider.setPasswordEncoder(passwordEncoder())  
    provider.setUserDetailsService(customUserDetailService)  
    return ProviderManager(provider)  
}  
  
@Bean  
fun customJsonLoginFilter(): CustomJsonLoginFilter {  
    val customJsonLoginFilter = CustomJsonLoginFilter(objectMapper)  
    customJsonLoginFilter.setAuthenticationManager(authenticationManager())  
    customJsonLoginFilter.setAuthenticationSuccessHandler(loginSuccessHandler)  
    customJsonLoginFilter.setAuthenticationFailureHandler(loginFailureHandler)  
    return customJsonLoginFilter  
  
}  
  
@Bean  
fun jwtAuthenticationProcessingFilter(): JwtAuthenticationProcessingFilter {  
    return JwtAuthenticationProcessingFilter(jwtService, userRepository, customUserDetailService)  
}
```

**passwordEncoder** : 비밀번호 암호화에 사용되는 메소드
- DB에 비밀번호를 저장할 때 암호화된 상태로 저장하기 위해 사용된다.


**authenticationManager** : 인증 처리를 담당하는 메소드
- 비밀번호 암호화를 위해 `passwordEncoder`를 사용했다.
- `customUserDetailService` 서비스를 사용한다. -> 추가 코드 부분에 설명하겠다.

**customJsonLoginFilter** : 로그인 필터를  빈 등록을 한 메소드
- 인증 처리를 담당하는 부분을 `authenticationManager`를 사용함
- 그라고 각 성공 & 실패 핸들러를 등록하였다.

**jwtAuthenticationProcessingFilter**  : jwt filter를 빈 등록을 한 메소드이다.

빈 등록을 통해 싱글톤으로 사용이 가능하다.

### 7-2 Filter Chain에 대한 부분

```kotlin
private val allowPatterns = arrayOf("/", "/api/user/signup")

@Bean  
open fun filterChain(http: HttpSecurity): SecurityFilterChain {  
  
    http.csrf { it.disable() }  // script 변조 공격 막기 위해 비활성화
    http.httpBasic { it.disable() } // form login, redirect 비활성화 => rest api 통신을 하기 때문  
  
    http  
        .authorizeHttpRequests { authorizeRequests ->  
            authorizeRequests  
                .requestMatchers(*allowPatterns).permitAll()  
                .anyRequest().authenticated()  
        }  
  
    // 세션을 사용하지 않으므로 STATELESS 설정  
    http.sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }  
  
    // filter 적용  
    http.addFilterAfter(customJsonLoginFilter(), LogoutFilter::class.java)  
    http.addFilterBefore(jwtAuthenticationProcessingFilter(), CustomJsonLoginFilter::class.java)  
  
    return http.build()  
}
```

주석을 확인해도 되겠지만 여기서 `authorizeHttpRequest`부분에 대해 작성한다.

```kotlin
    http  
        .authorizeHttpRequests { authorizeRequests ->  
            authorizeRequests  
                .requestMatchers(*allowPatterns).permitAll() // 접근 허용
                .anyRequest().authenticated()  
        } 
```

해당 부분은 요청에 대한 권한을 설정하는 부분으로 `requestMachers`를 통해 allowPatterns의 URL에 대한 접근을 허용하고 있다.

```kotlin
private val allowPatterns = arrayOf("/", "/api/user/signup")
```

`.anyRequest().authenticated()`는 위 허용된 URL을 제외하고 모든 접속에는 인증이 되어야 한다. 즉 JWT 토큰을 사용해야 하는 것이다.

다음은 커스텀 필터를 적용하는 것으로 적용 순서는 다음과 같다.

```kotlin
    // filter 적용  
    http.addFilterAfter(customJsonLoginFilter(), LogoutFilter::class.java)  
    http.addFilterBefore(jwtAuthenticationProcessingFilter(), 
	    CustomJsonLoginFilter::class.java)
```

로그아웃 필터 이후에 커스텀 필터가 동작한다.

**순서: LogoutFilter -> JwtAuthenticationProcessingFilter -> CustomJsonUsernamePasswordAuthenticationFilter**

## 10. 추가 코드 설명
위에서 CustomUserDetails와 CustomUserDetailService에 대한 글을 적는 부분을 깜박하여 아래에 추가할려고 한다. 

### 10-1. CustomUserDetails

UserDetails 인터페이스를 상속 받는다.

기본적으로 `getAuthorities` 사용자 권한(role)을 받아야 하는데 현재 테이블에 권한을 따로 부여하지 않았기 때문에 null을 반환 하도록 하였다.

```kotlin
class CustomUserDetails(private val user: User) : UserDetails {  
    // null을 반환하도록 한다.  
    override fun getAuthorities(): MutableCollection<out GrantedAuthority>? {  
        return null  
    }  
  
    fun getEmail(): String = user.email // 이메일을 반환하도록 한다.
  
    override fun getPassword(): String = user.password  
  
    override fun getUsername(): String = user.email  
  
    override fun isAccountNonExpired(): Boolean = true  
  
    override fun isAccountNonLocked(): Boolean = true  
  
    override fun isCredentialsNonExpired(): Boolean = true  
  
    override fun isEnabled(): Boolean = true  
}
```

인증된 사용자를 Spring Seucirty에서 저장을 한다. 그래서 Controller에서 따로 @Authentication 어노테이션을 통해 인증 페이지에서 사용자 정보를 가져 올 수 있다.

원하는 값을 반환할 수 있도록 메소드를 정의할 수 있다.

### 10-2. CustomUserDetailService

UserDetailsService 인터페이스를 상속받는다.

```kotlin
//import 문 생략..

@Service  
class CustomUserDetailService(private val userRepository: UserRepository) : UserDetailsService {  
    override fun loadUserByUsername(email: String): UserDetails {  
        val user: User = userRepository.findByEmail(email)  
            .orElseThrow { Error("사용자를 찾을 수 없습니다.") }  
  
        return CustomUserDetails(user)  
    }  
}
```

이메일을 통해 DB에 저장된 User 정보를 가져와서 `CustomUserDetails` 객체로 반환하고 있다.

## 결론
기존에 자바로 작성한 코드를 다시 한번 코틀린으로 작성을 해보았다. 그 당시에는 해당 코드가 어떤 형식으로 작동이 되는지 이해가 잘 가지 않아 에러가 났을 때 어디 부분에 문제가 있었는지 잘 알지 못했다.

이번에 다시 한번 정리해서 JWT 토큰을 서버에서 발급하는 작동 원리에 대해 자세히 알게 된 것 같다.

**다음시간에는 리엑트에서 JWT 토큰으로 통신하는 방법에 대해 알아볼려고 한다.**