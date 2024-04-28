---
title: Spring boot API 명세서 만들기
categories:
  - spring boot
tags:
  - springboot
  - swagger
  - java
  - API_document
---
# 스프링 부트에서 API 명세서 만들어 보기
팀 프로젝트를 하는데 서버를 Spring Boot를 이용해서 작업을 하고 있다. 기존에는 노션에 API 명세서를 만들고 있었는데 작성하기도 귀찮고 작업한 내용에 대해 정확히 맞는지 일일이 테스트를 해야 하는 상황이 되었다. 

 API 명세서를 스프링 부트로 만들 수 있는 것을 알게 되었고 Ref Docs와 Swagger 둘 중에서 처음이다 보니 사용하기 간편한 Swagger를 선택하였다. 또한 Swagger의 장점이 문서에서 바로 테스트를 할 수 있다는 점에서 아무래도 프론트 입장에서도 사용하기 쉬울 것 같아서 채택하게 되었다.

# Swagger 의존성 추가하기
Swagger를 사용하기 위해서  **build.gradle** 파일에 아래 코드를 입력한다.

```gradle
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.2.0'
```

현재 사용하고 있는 Spring boot 의 버전이 3.2 이상이므로  2.xxx 버전을 사용하였다.

그런 다음 아래와 같은 링크를 통해 API 명세서를 확인할 수 있게 된다.

[http://localhost:8080/swagger-ui/index.html]()

# Config 클래스 정의 하기

현재 프로젝트에서 JWT 토큰을 사용하기 때문에 아래와 같이 코드를 작성하였다.

```java
// SecurityConfig class 안에 생성

@Bean  
public OpenAPI openAPI() {  

// jwt 사용하는 경우 components 코드를 작성한다.
    String jwt = "JWT";  
    SecurityRequirement securityRequirement = new SecurityRequirement().addList(jwt);  
    Components components = new Components().addSecuritySchemes(jwt, new SecurityScheme()  // JWT 토큰 사용
            .name(jwt)  
            .type(SecurityScheme.Type.HTTP)  
            .scheme("bearer")  
            .bearerFormat("JWT") 
    );  
    return new OpenAPI()  
            .components(new Components())  
            .info(apiInfo())  
            .addSecurityItem(securityRequirement)  
            .components(components);  
}  
  
private Info apiInfo() {  
    return new Info()  
            .title("UNMUTE API") // API의 제목  
            .description("Project API Documentation") // API에 대한 설명  
            .version("1.0.0"); // API의 버전  
}
```

만약 Spring security를 사용한다면 Swagger URL 접근 허용을 해줘야 한다.
`securityFilterChain` 메서드 안에 접근 허용 코드를 작성한다.

```java
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
// 생략
http.authorizeHttpRequests(authorizationManagerRequestMatcherRegistry ->  
        authorizationManagerRequestMatcherRegistry  
                .requestMatchers("/v3/**", "/swagger-ui/**", "/api-docs").permitAll()  
                .anyRequest().authenticated());

// 생략
}
```

만약 api 명세서에 대한 링크를 설정하고 싶으면 yml(properties) 파일에 아래와 같은 코드를 추가한다.

```yml
springdoc:
  swagger-ui:
    path: /api-docs  # swagger-ui 접근 경로에 대한 별칭, 해당 주소로 접속해도 http://localhost:8080/swagger-ui/index.html로 리다이렉션 됨.
```

이렇게 하면 화면이 API 명세서 화면이 보일 것이다.
Swagger를 사용하는 경우 자동으로 controller 클래스를 통해 API 명세서를 작성해준다.

API 명세서 모습은 다음과 같다.

![](../assets/images/post/Pasted%20image%2020240429000105.png)
오른 쪽 하단에 **Authorize** 부분은 JWT 를 추가 했을 때 나오게 된며 토큰값을 지정할 수 있게 된다.

![](../assets/images/post/Pasted%20image%2020240429000212.png)
위 이미지 처럼 아무 값을 넣고 초록색 버튼을 누르면 각 API 옆에 자물쇠가 잠기게 된다.

![](../assets/images/post/Pasted%20image%2020240429000315.png)

### 추가사항
위에서 문제는 회원 가입하는 경우 JWT 토큰을 사용하지 않지만 API 명세서에서는 자물쇠가 추가되는데 이건 전역적으로 추가되는 것이라 따로 지정하는 방법이 없는 것 같다.

# API 설정 관련 어노 테이션

**@Tag** :  API 그룹 태그명 지정 가능하다.**
****@Optional** : 각 API 대한 이름 및 설명을 추가할 수 있다,**
****@ApiResponse** : 응답 코드에 대한 정보를 나타낼 수 있다.

회원 관리 Controller 를 예시로 설명하도록 하겠다.

```java
@RestController  
@RequiredArgsConstructor  
@RequestMapping("/api")  
// 이름을 '회원 관리 API', 설명 부분에 '회원 정보 API'라고 뜨게 된다.
@Tag(name = "회원 관리 API", description = "회원 정보 API")  
public class UserController {  
    private final UserService userService;  
  
    @PostMapping("/signup")  
    @Operation(  
            summary = "유저 회원가입",  
            description = "회원가입에는 JWT 토큰이 필요하지 않으며 아래와 같은 형식의 데이터를 받는다.")  
    public String signup(@RequestBody @Valid SignupUser user) {  
        userService.join(user);  
        return "success";  
    }  
  
    @GetMapping("/profile")  
    @Operation(summary = "회원 정보 상세조회", description = "회원의 정보를 상세 조회 API")  
    public UserResponse getUser(@AuthenticationPrincipal CustomUserDetails user) {  
        return userService.getUserProfile(user.getId());  
    }  
  
    @PutMapping("/update")  
    @Operation(summary = "회원 정보 수정", description = "회원 정보를 수정에 사용하는 API")  
    public Long update(@AuthenticationPrincipal CustomUserDetails userDetails, @RequestBody @Valid UpdateUser user) {  
        return userService.update(userDetails.getEmail(), user);  
    }  
  
    @DeleteMapping("/delete")  
    @Operation(summary = "회원 탈퇴", description = "회원 탈퇴 API")  
    public void deleteUser(@AuthenticationPrincipal CustomUserDetails userDetails) {  
        userService.deleteUser(userDetails.getEmail());  
    }  
  
}
```

위 코드에 대한 예시는 다음과 같다.

![](../assets/images/post/Pasted%20image%2020240428235624.png)

그리고 dto 클래스에서 **@Schema** 어노테이션을 통해 입력 값에 대한 설명과 예시를 줄 수 있다.

아래 코드는 회원 가입 dto를 나타낸다.

```java
@Data  
@Builder  
@NoArgsConstructor  
@AllArgsConstructor  
public class SignupUser {  
    @Email(message = "이메일 형식이 올바르지 않습니다.")  
    @Schema(description = "유저 이메일", example = "test@example.com")  
    private String email;  
    @Pattern(regexp = "^(?=.*\\d)(?=.*[!@#$%^&*]).{8,}$",  
            message = "비밀번호는 8자 이상이며, 숫자와 특수문자를 모두 포함해야 합니다.")  
    @Schema(description = "유저 비밀번호(8자리 이상, 숫자 특수문자 모두 포함)", example = "Test1234!")  
    private String password;  
    @NotBlank(message = "유저이름을 입력해주세요.")  
    @Schema(description = "유저 이름", example = "테스트")  
    private String username;  
    @Schema(description = "사용자 휴대폰 번호", example = "01012345678")  
    private String phone;  
}
```

위 코드에 대한 예시는 아래와 같다.

![](../assets/images/post/Pasted%20image%2020240428235946.png)

Example Value 를 통해 입력 값 예시를 설정할 수 있다.

위 이미지에서 **Try it out** 버튼을 통해 API를 테스트 할 수 있다.

![](../assets/images/post/Pasted%20image%2020240429000626.png)

여기서 파란 색 버튼을 누르면 테스트 할 수 있다.
그리고  Response body 부분에 응답 코드와 Response header ()

![](../assets/images/post/Pasted%20image%2020240429000650.png)