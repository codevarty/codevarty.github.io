---
title: Spring Boot (kotlin) jwt(1/2)
categories:
  - spring boot
tags:
  - kotlin
  - springboot
  - springsecurity
  - jwt
---
# Spring Boot (Kotlin) JWT 로그인(1/2)
팀 프로젝트에서 JWT를 사용하였지만 그 때 당시에 Spring Boot가 처음이기도 하고 JWT를 처음 사
용하는 것이기 때문에 잘 알지 못한 상태에서 코드를 작성하여 이번에 제대로 정리를 해보자고 한다.

코드를 작성하기 이전에 먼저 JWT가 무엇인지 정리해볼려고 한다.

## JWT 인증 절차
[jwt(JsonWebToken)](https://jwt.io/)는 클라이언트와 서버 사이에 통신을 할 때 인증을 하기 위해 사용하는 토큰이다.  웹 표준인 **JSON 형태로 데이터를 주고 받기 위해 표준 규약에 따라 생성한 암호화된 토큰**이다.

>**※ 참고 ※**<br>
>Json 이란 **key, Value가 한 쌍을 이룬 객체를 의미**한다.
>표시 형식은 `{"key": "value"}`이다.

### 1. JWT 구성 요소
JWT는 헤더(Header), 페이로드(Payload), 서명(Signature) 세 파트로 나뉘어져 있으며 아래와 같다.

![](https://velopert.com/wp-content/uploads/2016/12/jwt.png)

#### Header
```json
{
  "alg": "HS256",
  "typ": "JWT"
}

```
- 토큰의 타입과 해시 알고리즘으로 구성되어 있다.
- 토큰의 타입을 지정하지 않아도 된다.
- Base64 인코딩
- `alg`: 해시 알고리즘을 지정한다.
- `type`: 이 토큰을 JWT로 지정한다.

#### Payload
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```
- **토큰의 데이터를 담고 있는 곳**이다.
- 데이터 각각의 key를 **claim**이라고 부른다.
- Claim은 등록된 클레임(registered), 공개 클레임(public), 비공개 클레임(private)으로 나눌 수 있다.
- **등록된 클레임**:  3글자로 필수는 아니지만 사용이 권장된 클레임
- **공개 클레임**: 사용자가 자유롭게 정의할 수 있는 클레임
- **비공개 클레임**: 등록된 또는 공개 클레임이 아닌 클레임이며, 정보를 공유하기 위해 만들어진 커스터마이징된 클레임

> **사용시 주의사항**<br>
> Payload는 단순하게 Base64로 인코딩된 곳이며 누구나 디코딩하여 데이터 열람이 가능하다.
> 그렇기 때문에 비밀번호 같은 **민감한 정보는 Payload 안에 넣지 않는다.**

암호화된 코드
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

다음 사이트는 Base64 디코더 사이트 주소이다.

[https://www.base64decode.org/](https://www.base64decode.org/)

아래 그림과 같이 손쉽게 디코딩이 가능하다.

![base64](/assets/images/post/Pasted%20image%2020240509001348.png)

#### Signature
```json
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```
- 가장 중요한 부분으로 헤더와 정보를 합친 후 발급해준 서버가 지정한 secret key로 암호화 시켜 토큰을 변조하기 어렵게 만들어준다.

***
### 2. JWT 동작 원리
기존 session cookie를 이용하는 인증 방식에서는 서버가 사용자의 정보를 가지고 있어야 했다. 이 경우에는 사용자가 많을 수록 서버가 저장해야 할 정보가 많아져 서버의 부하가 걸릴 수 있다. 이러한 점을 해결하기 위해 나온 것이 바로 JWT이다.

JWT는 **사용자 정보를 서버가 가지고 있지 않고 사용자에게 넘기는 인증 방식**이다.
이러한 방식을 `STATELESS`라고 한다.

이렇게 서버에 인증하는 JWT 토큰을 **Access Token** 이라고 한다.

![](https://mblogthumb-phinf.pstatic.net/MjAxOTA1MjVfNDMg/MDAxNTU4Nzk1NjQ3Nzg5.cz-5fOL_RPyifrETlD_Go9cuUmyCl8Jrl01uY_T5PgUg.FE9xhe58eOPiC_ZUucbewNUHAf35kj9cjo3qStzO5msg.PNG.shino1025/asdasd.png?type=w800)

또한 **토큰이 클라이언트에 보관된다는 점이 보안에 취약할 우려**가 있어 민감한 정보를 저장하면 안된다.

JWT는 <span style="background-color:#fff5b1"> 사용자를 식별하기 위한 토큰 </span> 으로 사용하는 것이다.

위에서 설명했듯이 JWT는 클라이언트에 보관되기 때문에 보안에 취약하다. 그리고 서버에 관리 하지 않기 때문에 JWT가 탈취 당해도 서버는 알 방법이 존재하지 않는다.

이를 방지 하기 위해 **만료시간을 짤게 설정하여 탈취되는 것을 방지**하고 있다.

그러나 이러한 방법을 사용하면 만료기간 뒤에 다시 인증을 해줘야 하는 문제가 발생한다. 이를 해결하기 위해 나온 방안이 **Refresh Token**이다.

***
### 3. Refreh Token
refresh token 재발급 토큰으로 **AccessToken이 만료가 되었을 때 새로운 토큰을  사용자에게 발급하기 위해 사용하는 토큰**이다.

![](https://velog.velcdn.com/images%2Fkshired%2Fpost%2Ffa1ca964-9203-4f84-8284-a7fd1593186b%2F99DB8C475B5CA1C936.png)

Refresh Token은 Access Token 과 함께 사용자에게 발급 되며 만료기간은 Access Token에 비해 상당히 길다.

그러나 Refreh Token 을 통해 AccessToken을  재발급하기 때문에 이 토큰을 서버에서 검증 할 수 있도록 저장하는 **Refresh Token Storage**가 필요하다.
***
### 4. Refresh Token Storage
Access Token의 장점이 서버에서 사용자 정보를 가지고 있지 않다는 점이다.
그러나 Refresh Token을 통해 Access Token이 만료 된후 토큰을 발급하기 위해 서버에서 검증을 해야 하다.

> **Refresh Token을 검증 하는 이유**<br>
> 검증을 해야 하는 이유는 Refresh Token 또한 JWT 토큰으로 서버에서 사용자에게 발급하므로 **Access Token 과 같이 탈취될 가능성이 존재하기 때문**이다.

그렇다면 Refresh Token을 어디에 저장해야 하는가? 
크게 다음과 같이 **두 가지 형태**로 볼 수 있다.

1. 서버에 저장하는 방식
2. 데이터베이스에 저장하는 방식

1번의 경우 Refresh Token을 서버에 저장하는 형태로 세션과 동일하게 **Stateful 형식**으로 사용이 된다. 그렇기 때문에 JWT에 대한 장점을 사용하지 않는 것이기 때문에 2번의 방식이 보안성이 더 높고 Stateless 형식을 사용한다.

이런 이유로 기존 프로젝트에서는 **2번 방식**을 택하게 되었다.

***
# 결론
현재 웹 서버에서 JWT 방식을 많이 사용한다고 한다. 이전에 작업했을 때는 JWT가 어떤 형식으로 동작하는 지 자세히 알지 못했기 때문에 코드를 작성하는 것이 쉽지는 않았고 에러가 났을 때 어느 부분이 문제인지 파악히 힘들었다. 이번 정리를 통해 JWT가 왜 사용되는지에 대해 알게 되었고 어떤식으로 이용이 되는지에 대해 알게 되었다. 

**다음 시간에는 코드를 통해 JWT 토큰을 발급하여 로그인하는 로직을 작성해 본다.**

