# 실전! 스프링 부트와 JPA활용1 - 정리



## Thymeleaf 템플릿 엔진

> Thymeleaf는 spring 친화적인 템플릿 엔진이다. Spring에서도 권장하고 다양한 기능을 지원하여 JSP쓰는것 보다 편리하다.



공식사이트: https://www.thymeleaf.org

공식 가이드: [Getting Started | Serving Web Content with Spring MVC](https://spring.io/guides/gs/serving-web-content/)



- Thymeleaf gradle setting

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
}
```



- 스피링 부트 thymeleaf viewName 매핑
  
  - resources/templates/{ViewName}.html

```java
@Controller
public class HelloController {

    @GetMapping("/hello")
    public String hello(Model model) {
        model.addAttribute("data", "hello!!!");
        return "hello";
    }
}
```

```html
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>hello</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
</head>
<body>

<p th:text="${data}">안녕하세요</p>

</body>
</html>
```

위와 같이 Get /hello 호출시 ViewName을 리턴하고 Thymeleaf를 통해 `resource/template/`아래 html파일과 매핑된다. 매핑된 html에 `Model`을 통해 데이터를 전달할 수 있다. 만약 위 HTML을 static하게 호출시 `${data}`의 값이 아닌 text인 '안녕하세요'가 나오게 된다.



> :bulb:Tip
> 
> static한 파일변경은 저장해도 바로 적용되지 않는다. 아래 dev-tools를 통해 해당 파일 리컴파일시 해당 파일을 리로드 할 수 있다.
> 
> ```groovy
> dependencies {
>     implementation 'org.springframework.boot:spring-boot-devtools'
> }
> ```





# H2 Database

> H2 Database는 개발이나 테스틑용도로 편하게 사용할 수 있고 Web Console도 지원한다.



다운로드: https://h2database.com/html/main.html



1. 다운로드 후 bin/h2.sh 실행

2. `jdbc:h2:{파일경로}` 설정을 통해 데이터 저장 위치 지정

3. `{파일경로}`에 파일 생성 확인

4. 이후 접속 할 때는 `jdbc:h2:tcp://localhost/{파일경로}`를 통해 접속



# JPA와 DB 연동



- application.yml

```yaml
spring:
  # DB 접속 설정
  datasource:
    url: jdbc:h2:tcp://localhost/~/jpashop
    username: sa
    password:
    driver-class-name: org.h2.Driver

#JPA 관련 설정
  jpa:
    hibernate:
      # 데이터베이스 새로 생성
      ddl-auto: create
    properties:
      hibernate:
#        System.out.println 으로 출력됨
#        show_sql: true
        format_sql: true

# 로그관련 설정
logging:
  level:
    org.hibernate.sql: debug
```



- Entity

```java
@Entity
@Getter
@Setter
public class Member {

    @Id@GeneratedValue
    private Long id;
    private String username;

}
```



- Repository

```java
@Repository
public class MemberRepository {

    @PersistenceContext
    private EntityManager em;

    public Long save(Member member) {
        em.persist(member);
        return member.getId();
    }

    public Member find(Long id) {
        return em.find(Member.class, id);
    }
}
```

Application 실행시 `ddl-auto: create`을 통해 database가 생성됨





- Test

```java
@SpringBootTest
class MemberRepositoryTest {

    @Autowired
    MemberRepository memberRepository;

    @Test
    @Transactional
    public void testMember() throws Exception {
        // given
        Member member = new Member();
        member.setUsername("memberA");

        // when
        Long savedId = memberRepository.save(member);
        Member findMember = memberRepository.find(savedId);

        // then
        Assertions.assertThat(findMember.getId()).isEqualTo(member.getId());
        Assertions.assertThat(findMember.getUsername()).isEqualTo(member.getUsername());
        Assertions.assertThat(findMember).isEqualTo(member);
    }

}
```

Member 객체 생성 후 repository 를 통해 데이터 저장 및 조회한다. 그리고 조회된 데이터와 생성한 데이터가 같은지 체크한다.

`@Transactional`은 함수가 하나의 트렌젝션안에서 실행될 수 있도록 해준다. 또한 Test가 끝나면 데이터를 모두 롤백한다. 만약 롤백하고 싶지 않으면 `@Rollback(value = false)` 어노테이션을 추가하면 된다.

`findMember`와 `member`를 Equal로 비교했을 때 같다고 인식된다. 왜냐하면 두 데이터는 같은 트렌젝션안에서 실행되기 때문에 같은 영속성 컨텍스트를 사용하게된다. 따라서 1차 캐시된 Entity가 나오게된다. 따라서 두 객체는 같은 데이터이다. 로그를 봐도 추가적으로 query를 하지 않는다.



> :bulb:Tip
> 
> 기본적으로 `org.hibernate.type: trace`를 사용하면 sql에 들어가는 데이터를 볼 수 있다.
> 
> 아래 dependencies를 추가하고 실행시 보다 직관적으로 sql을 확인해 볼 수 있다.
> 
> ```groovy
> dependencies {
>     implementation 'com.github.gavlyukovskiv:p6spy-spring-boot-starter:1.5.6'
> }
> ```