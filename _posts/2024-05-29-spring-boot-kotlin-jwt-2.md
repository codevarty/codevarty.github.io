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

## 3-3 토큰 추출 부분

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

## 3-4 토큰 검증 코드
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

