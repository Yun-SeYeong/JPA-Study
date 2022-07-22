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







# 도메인 분석 설계



### 기능

- 회원기능
  
  - 회원 등록
  
  - 회원 조회

- 상품기능
  
  - 상품 등록
  
  - 상품 수정
  
  - 상품 조회

- 주문기능
  
  - 상품 주문
  
  - 주문 내역 조회
  
  - 주문 취소

- 기타 요구사항
  
  - 상품은 재고 관리가 필요
  
  - 상품의 종류는 도서, 음반, 영화
  
  - 상품을 카테고리로 구분
  
  - 상품 주문시 정보를 입력





- Member

```java
@Entity
@Getter
@Setter
public class Member {

    @Id
    @GeneratedValue
    @Column(name = "lmember_id")
    private Long id;

    private String name;

    @Embedded
    private Address address;

    @OneToMany(mappedBy = "member")
    private List<Order> orders = new ArrayList<>();
}
```



- Order

```java
@Entity
@Table(name = "Orders")
public class Order {

    @Id
    @GeneratedValue
    @Column(name = "order_id")
    private Long id;

    @ManyToOne
    @JoinColumn(name = "member_id")
    private Member member;

    @OneToMany(mappedBy = "order")
    private List<OrderItem> orderItemList = new ArrayList<>();

    @OneToOne
    @JoinColumn(name = "delivery_id")
    private Delivery delivery;

    private LocalDateTime orderDate;

    private OrderStatus status;
}
```



Member는 Order와 일대다 관계이다. 따라서 Member에는 `List<Order>`가 들어가게 되고 Order에는 `Member` 객체가 들어가게 된다. 또한 Order테이블에서 Join을 하여 수정시 Order를 통해 수정할 수 있도록한다. Member의 List는 읽기만 가능한 상태가 된다.



- Delivery

```java
@Entity
@Getter
@Setter
public class Delivery {

    @Id
    @GeneratedValue
    @Column(name = "delivery_id")
    private Long id;

    @OneToOne(mappedBy = "delivery")
    private Order order;

    @Embedded
    private Address address;

    @Enumerated(EnumType.STRING)
    private DeliveryStatus status;
}
```



Order에서 Delivery는 일대일 관계이다. 따라서 양쪽에 Order와 Delivery 객체가 존재하게된다. 이때 Join은 양쪽 어떤 테이블에 해도 관계없으나 주로 많이 사용되는 Order에 지정하였다.

Address는 Embedded를 통해 포함 시킨다. Address에 Embeddable과 Embedded 중 하나만 넣어도 되지만 확인이 비교적 수월하도록 양쪽에 넣는다.

DeliveryStatus는 Enum을 통해 넣어주고 EnumType.STRING으로 지정한다. ORDINAL로 지정하면 숫자로 들어가게 되는데 만약 중간에 새로운 값이 추가되면 기존 데이터와 혼동이 생기게 됨으로 STRING으로 지정한다.



- OrderItem

```java
@Entity
@Getter
@Setter
public class OrderItem {

    @Id
    @GeneratedValue
    @Column(name = "order_item_id")
    private Long id;

    @ManyToOne
    @JoinColumn(name = "item_id")
    private Item item;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;

    private int orderPrice;
    private int count;

}
```



OrderItem은 Order와 다대일 Item과 다대일로 두객체의 관계성을 나타내게 된다. 따라서 두객체 모두 ManyToOne으로 설정하고 JoinColumn을 통해 객체를 수정할 수 있도록 한다.



- Item

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "dtype")
@Getter
@Setter
public class Item {

    @Id
    @GeneratedValue
    @Column(name = "item_id")
    private Long id;

    private String name;
    private int price;
    private int stockQuantity;
}
```

Item은 상속관계로 SINGLE_TABLE로 성정한다. SINGLE_TABLE은 하나의 테이블에 상속 테이블 내용이 모두 들어간다. DiscriminatorColumn을 통해 어떤 타입인지 알 수 있는 컬럼을 설정한다.



- Album

```java
@Entity
@DiscriminatorValue("A")
@Getter
@Setter
public class Album extends Item {
    private String artist;
    private String etc;
}
```

- Book

```java
@Entity
@DiscriminatorValue("B")
@Getter
@Setter
public class Book extends Item {
    private String author;
    private String isbn;
}
```

- Movie

```java
@Entity
@DiscriminatorValue("M")
@Getter
@Setter
public class Movie extends Item {
    private String director;
    private String actor;
}
```

위 3개의 객체는 Item을 상속받고 DiscriminatorValue를 통해 해당 타입을 표현할 문자를 지정한다. 기본적으로 클래스이름으로 들어간다.



- Category

```java
@Entity
@Getter
@Setter
public class Category {

    @Id
    @GeneratedValue
    @Column(name = "category_id")
    private Long id;

    private String name;

    @ManyToMany
    @JoinTable(name = "category_item",
            joinColumns = @JoinColumn(name = "category_id"),
            inverseJoinColumns = @JoinColumn(name = "item_id"))
    private List<Item> items = new ArrayList<>();

    @ManyToOne
    @JoinColumn(name = "parent_id")
    private Category parent;

    @OneToMany(mappedBy = "parent")
    private List<Category> child = new ArrayList<>();

}
```



- Item (수정)

```java
public class Item {

    --- 중략 ---

    @ManyToMany(mappedBy = "items")
    private List<Category> categories = new ArrayList<>();
}
```



Category와 Item은 다대다 관계이다. 이때 중간 테이블이 필요한데  `@JoinTable`을 통해 중간 테이블을 자동으로 생성해 줄 수 있다. joinColumns는 join할 테이블의 id, inverseJoinColumns는 반대 테이블에서 join할 id를 지정한다.

Category는 Parent와 child가 필요하다 이때 대상이 자신인 Category 테이블임으로 parent는 일대다 관계로 child를 다대일 관계로 설정한다.



### Entity 설계시 주의점



- Setter를 사용하지 말자
  
  Setter를 모두 열면 언제 변경이 되었는지 찾기 어려움



- 모든 연관관계는 지연로딩으로 설정
  
  즉시로딩(EAGER)는 예측이 어렵고 SQL을 추적하기 어려움 따라서 지연로딩(LAZY)를 설정해야한다. 특히 JPQL을 사용시 연관 테이블의 데이터를 모두 같이 조회되는 n+1문제가 발생된다. XToOne 설정시 기본 fetch는 EAGER이다. 



- cascade = CascadeType.ALL
  
  Cascade 설정 시 관련 테이블을 각각 모두 생성하지 않아도 자동으로 모두 생성해준다.



- 연관관계 매서드

- 