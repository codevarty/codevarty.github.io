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

# API 설정 관련 어노 테이션

**@Tag** :  API 그룹 태그명 지정 가능하다.**
****@Optional** : 

회원 관리 Controller 를 예시로 설명하도록 하겠다.
