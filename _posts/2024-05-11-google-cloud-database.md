---
title: Spring Boot (kotlin) Google Cloud Database 연결
categories:
  - spring boot
tags:
  - kotlin
  - springboot
  - jpa
  - mysql
  - google_cloud
---
# Spring Boot (Kotlin) MySQL(GCP) 연결
로컬 데이터베이스를 연결하면 다른 컴퓨터로 작업을 할 때 데이터베이스 환경을 같게 맞췆줘야 한다. 하지만 구글 클라우드 같이 외부 데이터베이스를 사용하면 서로 다른 컴퓨터를 사용해도 데이터베이스 사용에 대해서 추가적인 환경 구성을 할 필요가 없다.

이러한 장점을 고려하여 구글 클라우드를 이용한 데이터 베이스 구축 및 데이터 베이스를 spring boot 서버에 연결하고 테스트 하는 작업을 해볼려고 한다.

## 구글 클라우드 SQL (GCP) 생성하기
먼저 구글 클라우드에서 데이터 베이스를 연결하는 방법을 작성한다.

> **참고**
> 구글 클라우드는 처음 사용할 때 무료 크레딧 (90일)을 사용할 수 있다. 현재 이 글은 무료 크레딧 기준으로 설명하고 있다.

### 1. 프로젝트 생성
구글 클라우드에 로그인을 한 후 좌측 프로젝트 박스를 클릭한다.
![](/assets/images/post/Pasted%20image%2020240507005540.png)

**새 프로젝트** 버튼을 눌러 프로젝트를 새로 만든다.
![](/assets/images/post/Pasted%20image%2020240507005731.png)


**프로젝트 이름**을 입력한 후 **만들기** 버튼을 눌러 프로젝트를 생성한다.
![](/assets/images/post/스크린샷%202024-05-07%20005852.png)

기존에 사용하던 프로젝트가 있는 경우 **프로젝트 박스**를 클릭한 후 생성한 프로젝트를 클릭한다.
![select](/assets/images/post/select.png)

### 2. SQL 인스턴스 생성

왼쪽의 탐색 메뉴 창을 클릭하고 아래 메뉴 중에 **SQL** 쪽을 클릭한다.
![Pasted image 20240507010640](/assets/images/post/Pasted%20image%2020240507010640.png)

여기서 **무료 크레딧으로 인스턴스 만들기** 버튼을 클릭해서 인스턴스를 만들 수 있도록 한다.
![](/assets/images/post/스크린샷%202024-05-07%20010802.png)


데이터 베이스를 선택하라는 창이 나오는데 나는 **MySQL**을 사용하다.


![Pasted image 20240507011112](/assets/images/post/Pasted%20image%2020240507011112.png)

그다음 인스턴스 아이디와 비밀 번호를 입력한다.
![](/assets/images/post/Pasted%20image%2020240507011853.png)

데이터베이스 서버 사용 지역(리전)을 선택한 후 하고 싶은 멀티존 영역을 선택한다.
그 다음 **인스턴스 만들기 버튼**을 누른다.
![](/assets/images/post/Pasted%20image%2020240507012056.png)

좌측의 **연결** 탭으로 가서 **네트워킹**을 누른 후 **공개 IP를 추가**할 수 있다. 여기서 내 ip를 추가하도록 하자. 

> **내 ip 주소 확인 방법**
> 1. 구글에 'ip' 검색
> 2. [https://whatismyipaddress.com/](https://whatismyipaddress.com/)사이트로 이동

**네트워크 추가**를 누르고 이름과 내 ip 주소를 입력한다.
![](/assets/images/post/Pasted%20image%2020240507015142.png)
### 3. 데이터베이스 생성
좌측 탭에 **데이터베이스**를 선택한 후 **데이터베이스 만들기** 버튼을 통해 새로운 데이터베이스를 만들 수 있다.
![](/assets/images/post/Pasted%20image%2020240507013123.png)

데이터베이스 이름을 입력하고 만들기 버튼을 통해 데이터베이스가 생성이 된다.
![](/assets/images/post/Pasted%20image%2020240507013227.png)

'friendbot' 이라는 데이터베이스가 추가된 것을 알 수 있다.
![](/assets/images/post/Pasted%20image%2020240507013301.png)

데이터 베이스에 접속하기 위해서는 **공개 IP 주소**를 사용하며 **개요** 탭에서 확인이 가능하다. (주소는 공개하지 않았다.)
![public-ip](/assets/images/post/public-ip.png)

> 추가적으로 인스턴스는 데이터베이스 사용이 끝난 뒤에는 종료하도록 하자. 
> **종료하지 않으면 돈이 추가적으로 계속 청구**가 된다.

## Spring Boot (kotlin) 데이터 베이스 연결
데이터베이스가 준비 되었으니 Spring 서버에 연결 해보자.

### 1. Build.gradle 파일 설정
JPA 를 사용하기 위해 `build.gradle.kts` 파일 안에 의존성을 주입한다.
```gradle
dependencies {
	implementation("org.springframework.boot:spring-boot-starter-data-jpa")
	runtimeOnly("com.mysql:mysql-connector-j") // MySQL 드라이버
}
// 생략
```

추가적인 설정을 위해 `plugins` 안에 다음 코드를 추가한다.
```gradle
plugins {  
    id("org.springframework.boot") version "3.2.4"  
    id("io.spring.dependency-management") version "1.1.4"  
    kotlin("jvm") version "1.9.23"  
    kotlin("plugin.spring") version "1.9.23"  
    kotlin("plugin.jpa") version "1.9.23"  
    kotlin("plugin.noarg") version "1.9.23"  // 매개 변수 없는 생성자 생성
    kotlin("plugin.allopen") version "1.9.23"  // plugin 외 open 추가
}
```

아래와 같은 코드를 추가적으로 작성한다.
```gradle
//...
// spring boot plugin 외에 open 추가  
allOpen {  
    annotation("jakarta.persistence.Entity")  
}  
  
// 매개변수가 없는 생성자를 자동으로 추가해줌  
noArg {  
    annotation("jakarta.persistence.Entity")  
}

```

### 2. 데이터베이스 연결하기
의존성을 주입한 후 Properties 파일 또는 YAML 파일에 사용하는 데이터베이스를 연결할 수 있게 작성하자.

> **※ 참고 ※**
> 해당 프로젝트에서는 보안을 위해 yml 파일을 분리 하였다.

먼저 `application-mysql.yml`을 생성한다. 그다음 아래와 같이 코드를 입력한다.
아래 대괄호([]) 안에 코드를 자신의 환경에 맞게 작성한다.
```yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: "jdbc:mysql://[DB서버IP]:[DB포트]/[DB 스키마]?autoReconnect=true&useUnicode=true&serverTimezone=Asia/Seoul"
    username: [DB사용자]
    password: [패스워드]
  jpa:
    database: mysql
    database-platform: org.hibernate.dialect.MySQL8Dialect
    hibernate:
      ddl-auto: create # 설정해야 테이블 자동생성
    open-in-view: false
    show_sql: true
```

application.yml 파일에 위 yml 파일을 팜조할 수 있도록 아래 코드를 추가한다.
```yml
spring:  
  profiles:  
    include:  
      - mysql # application-mysql.yml
```

datasources 옆에 **데이터베이스 아이콘**을 눌러 데이터베이스 서버 연결 테스트를 할 수 있다.
![](/assets/images/post/Pasted%20image%2020240507113727.png)

아래에 **Test Connection** 을 누른 다음 **success**가 나오면 연결이 된 것이다.
이 때 데이터베이스가 실행 되어 있어야 한다.
![connection](/assets/images/post/connection.png)

성공화면이 잘 나오면 Ok 버튼을 누르면 된다.

### 3. 기본 CRUD 작성
회원 가입 로직을 간단하게 사용한다. 먼저 entity 클래스를 작성한다.

사용자 정보를 저장하는 User 클래스를 만든다.

#### User.kt

```java
@Entity  
@Table(name = "user")  
class User(  
    @Column(length = 50, unique = true, nullable = false)  
    val email: String,  
    name: String,  
    password: String  
) {  
    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    val id: Long? = null  
  
    @Column(length = 20, nullable = false)  
    var name: String = name  
        protected set  

    @Column(nullable = false)  
    var password: String = password  
        protected set  
  
    val createdAt: LocalDateTime = LocalDateTime.now()  
  
    fun updateName(name: String) {  
        this.name = name  
    }  
  
    fun updatePassword(password: String) {  
        this.password = password  
    }  
}
```

실제 데이터베이스에 저장하고 업데이트 하는 JpaRepository 인터페이스를 만든다.

#### UserRepositoy.kt

```kotlin
interface UserRepository : JpaRepository<User, String> {  
    override fun findAll(): List<User>  
    override fun findById(id: String): Optional<User>  
    fun findByEmail(email: String): Optional<User>  
}
```

회원 가입 request dto와 응답 데이터인 response dto 클래스를 만들어 준다.

> **※ 참고 ※**
> dto 파일을 만드는 이유는 엔티티는 데이터베이스와 연관이 있어 각 클래스 마다 책임을 분리하기 위함이다.
> 또한 dto 클래스를 통해 자신이 원하는 값만 넘길 수 있다.

#### Request.kt

```kotlin
data class SignUpUserRequest(  
    val id: String,  
    val username: String,  
    val password: String,  
    val email: String,  
)
```

#### UserResponse.kt

```kotlin
data class UserResponse(  
    val username: String,  
    val email: String,  
)
```

실제 회원 가입이 이루어지는 service 객체를 생성한다. 업데이트 및 삭제 로직을 추가할 예정이다.

#### UserService.kt

```kotlin
@Service  
class UserService(  
    private val userRepository: UserRepository,  
    private val passwordEncoder: BCryptPasswordEncoder  
) {  
  
    @Transactional  
    fun addUser(signUpUserRequest: SignUpUserRequest): UserResponse {  
        // 비밀번호 암호화  
        val password = passwordEncoder.encode(signUpUserRequest.password)  
        val user = User(signUpUserRequest.email, signUpUserRequest.username, password)  
  
        userRepository.save(user)  
  
        return UserResponse(user.name, user.email)  
    }  
}
```

리엑트와 서버를 연결하기 위해 controller 를 생성한다. dto 클래스를 반환하기 때문에 `RestController`를 사용한다.

#### UserController.kt

```kotlin
@RestController  
class UserController(val userService: UserService) {  
  
    @GetMapping("/signup")  
    fun signup(@RequestBody signUpUserRequest: SignUpUserRequest): UserResponse {  
        return userService.join(signUpUserRequest)  
    }  
}
```


### 4. 테스트 하기

이제 서버를 킨 다음 데이터가 잘 들어가는 지 테스트 한다.
바로 테스트를 하기 위해 **postman**을 사용하였다.

현재 서버 포트는 8090으로 되어 있다. 즉 회워 가입 링크는 다음과 같다.

[http://localhost:8090/signup](http://localhost:8090/signup)

postman 으로 테스트한 결과는 다음과 같다.
![](/assets/images/post/Pasted%20image%2020240508002951.png)

데이터베이스에 저장이 되었는지 확인한다. 
![](/assets/images/post/Pasted%20image%2020240508003230.png)

데이터베이스에 성공적으로 저장되었다.
***

아래는 현재 작업중인 프로젝트 **깃허브 주소**이다.

[https://github.com/codevarty/friendbot](https://github.com/codevarty/friendbot)

# 결론
원래 프로젝트 데이터베이스는 간단하게 h2를 사용할려고 했다. 개인 프로젝트를 하면서 한번 클라우드 데이터베이스를 직접 연결하고 싶어서 무료로 사용이 가능한 구글 클라우드를 사용하였다. 구글 클라우드에서 데이터베이스를 생성하는 것은 쉬었지만 코틀린 문법에 익숙치 않아서 엔티티 클래스 작성 부분에서 애를먹었다. 

코틀린에서는 entity 클래스를 생성할 때 data class를 사용하면 안된다는 사실을 알게 되었다. 
