---
title: 코틀린 Entity 클래스 정의 하는 방법
categories:
  - spring boot
tags:
  - springboot
  - kotlin
  - jpa
---
# Entity 클래스 작성
현재 개인 프로젝트를 kotlin을 사용하고 있다. 자바는 익숙하지만 코틀린은 사용한지 얼마 되지 않아 익숙치 않다. 

dto 클래스를 작성할 때 data class로 작성하면 아주 편하게 작성이 되었기 때문에 entity 클래스를 만들 때도 data class를 사용하면 된다고 처음에는 생각했다. 하지만 이는 잘못된 방법으로 왜 data class로는 작성하면 안돼고 entity 클래스를 작성하는 방법에 대해 공부해볼려고 한다.

***
## Data class
먼저 data class 는 **불변 객체**로 정의가 되며 `equals()`, `hashCode()`, `toString()`, `copy()`등을 자동으로 생성해준다.

하지만 entity 클래스 같은 경우 실제 작업하는 데 수정해야 할 필요성이 많다.

또한 `equals()`를 사용했을 때 예기치 못한 경우를 접할 수 있다. data class 의 경우 내부에 정의된 값만 비교하기 때문이다.
```java
data class Person(val name: String) { 
var age: Int = 0 // equals 로 비교 되지 않음.
} val person1 = Person("John") 
val person2 = Person("John") 
person1.age = 10 person2.age = 20 
person1 == person2 // true
```

data class를 사용하는 경우 모든 필드를 생성자 매개변수로 받아야 하는데 이는 좋은 방법이 아니다. 

entity는 필수 적인 값만 매개 변수로 받아야 하며 아닌 경우 내부에서 기본적인 값을 지정하도록 해야 한다. 즉 data class를 사용하면 불필요하게 생성자를 만들게 된다.

**Lazy Loading**을 위해서 data class를 사용하면 안되는데 그 이유는 초기화를 뒤로 미룰 수 있다는 장점이 있다. 이를 통해 메모리 사용을 최소화 할 수 있다.

코틀린은 기본적으로 final class이기 때문에 proxy를 통한 lazy loading을 할 수 가 없다. 이를 해결하기 위해 **allopen** 플러그인을 사용한다.

하지만 data class의 경우 위 방식으로 open이 되지 않는다.

위 문제를 통해 data class를 통해 entity 클래스를 사용하면 안된다.

그렇다면 entity 클래스는 어떻게 만들어야 좋은가? 이를 알아보자.


## Entity 클래스 작성
위에서 보았듯이 entity 클래스는 data 클래스를 사용하지 않는다. 대신 일반 클래스로 작성을 해야 한다.

### @Id
[공식문서](https://spring.io/guides/tutorials/spring-boot-kotlin)를 보면 id를 var로 지정을 해준다. 하지만 실제로 id의 경우 한번 생성되면 변경되지 않기 때문에 val로 지정을 해줘도 된다.

> 또한 식별자에 `@GeneratedValue`를 사용하게 되면 database에 값을 넣기 전에는 null이 들어간다.
> 그렇기 때문에 nullable을 해줘야 한다.

```kotlin
@Entity
class User {
	@Id
	val id: Long? = null
}
```

***
### 1:N, M:N 관계
데이터베이스의 경우 1:N, 혹은 M:N의 관게를 가진 경우 **Collection**을 사용한다.
값이 변경 가능하다면 **mutableCollection**을 사용한다.

변경 가능성을 줄이기 위해 `toList`를 사용하여 Collection을 복제하여 사용한다.
그리고 mutableCollection의 값을 private로 주어 되부에서 변경할 수 없도록 한다.

```kotlin
@OneToMany(fetch = FetchType.LAZY) 
private val mutableBoards: MutableList<Board> = mutableListOf() 
val boards: List<Board> get() = mutableBoards.toList()
```

위 클래스는 게시판 클래스의 일부로  내부적으로는 mutableBoards를 사용하지만 외부에서 사용할 때는 변경할 수 없도록 `toList()`를 통해 복사하였다.

***
### private Setter
entity 클래스의 경우는 외부에서 변경하면 안된다. 그렇기 때문에 자바에서도 private 필드를 사용한다. 하지만 코틀린에서는 private로 필드를 지정하면 getter 또한 private가 되기 때문에 이를 방지 하기 위해 `private set` 키워드를 사용한다.
또한 `protected set`을 이용하여 접근 범위를 조절할 수 있다.

```kotlin
@Entity
class User(username: String) {
    @Id
    @GeneratedValue
    var id: Long? = null
    var username: String = username
        private set // 외부에서 값을 수정할 수 없다.
}
```

***
# 결론
코틀린을 사용하면 기존 자바의 코드 보다 아주 간결해지고 사용이 편해진다. 하지만 이 편한 상황에 너무 익숙하지 않는 것도 중요한 것 같다. 편의성 보다는 효율과 안정성을 고려하여 작성할 수 있도록 하자. 이번에 배운 entity 작성 방법을 통해 앞에 적은 글처럼 많은 것을 생각하게 되었다.

아직 코트린을 공부한지 얼마 되지 않아 정확하지 않은 내용이 있을 수 있다. 좀더 공부할 수 있도록 한다!

# 참고 자료
- https://spring.io/guides/tutorials/spring-boot-kotlin
- https://spoqa.github.io/2022/08/16/kotlin-jpa-entity.html
- https://velog.io/@eastperson/Kotlin-Spring-%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B03-Entity
