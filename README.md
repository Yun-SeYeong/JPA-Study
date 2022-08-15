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





# 애플리케이션 구현 준비



계층형 구조 사용

controlle, web: 웹 계층

service: 비즈니스 로직, 트랜잭션 처리

repository: JPA를 직접 사용하는 계층, 엔티티 매니저 사용

domain: 엔티티가 모여있는 계층, 모든 계층에서 사용





# 회원 레파지토리 개발



Spring Boot에서는 Repository Class에 대해 @Repository 어노테이션을 통해 구분한다.

```java
@Repository
public class MemberRepository {
    ...
}
```



JPA에서 데이터를 쓰고 읽는 등 작업을 할때 EntityManager를 이용하는데 @PersistenceContext를 이용하면 자동으로 주입해준다.

```java
@PersistenceContext
private EntityManager em;
```



다음과 같이 EntityManagerFactory를 직접 주입받을 수도 있다.

```java
@PersistenceUnit
private EntityManagerFactory emf;
```



기본적으로 @Transactional을 클래스에 지정하면 public 매서드에 대해서는 Transactinal하게 동작한다. 만약 옵션으로 readOnly를 사용할 경우 해당 함수는 읽기 전용으로 열리게 되는데 영속성 컨텍스트에서 더티체크를 안한다던지 보다 최적화 할 수 있다.

```java
@Transactional(readOnly = true)
public class MemberService {
    @Transactional
    public write() {}
    public read() {}
}
```



Test 작성시 클래스에 Transactional을 넣으면 기본적으로 insert query문이 전송되지 않는다. Rollback(value=false)를 넣어주면 insert query문이 전송된다. 또는 em.flush()를 통해 데이터베이스에 반영할 수 도 있다.

```java
@SpringBootTest
@Transactional
class MemberServiceTest {

    @Autowired
    MemberService memberService;
    @Autowired
    MemberRepository memberRepository;

    @Test
    @Rollback(value = false)
    public void 회원가입() {
        //given
        Member member = new Member();
        member.setName("kim");

        //when
        Long savedId = memberService.join(member);

        //then
        assertEquals(member, memberRepository.findOne(savedId));
    }
}
```



Test시 데이터베이스와 격리된 환경에서 테스트 하고 싶으면 메모리 모드로 테스트 해볼 수 있다.

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:test
    username: sa
    password:
    driver-class-name: org.h2.Driver
```

하지만 스프링 부트는 기본적으로 메모리 모드로 DB를 올린다. 따라서 위 설정을 제거해도 정상 동작할 수 있다.







# 상품 도메인 개발



엔티티에 필요한 비즈니스로직을 추가한다.

```java
    //==비즈니스 로직==//

    /**
     * stock 증가
     */
    public void addStock(int quantity) {
        this.stockQuantity += quantity;
    }

    /**
     * stock 감소
     */
    public void removeStock(int quantity) {
        int restStock = this.stockQuantity - quantity;
        if (restStock < 0) {
            throw new NotEnoughStockException("need more stock");
        }
        this.stockQuantity = restStock;
    }
```





### Item 등록

```java
public void save(Item item) {
        if (item.getId() == null) {
            em.persist(item);
        } else {
            em.merge(item);
        }
}
```

id의 경우 기본적으로 null이 입력되어있다. 따라서 null이 들어있을 시에는 id를 새로 생성해주어야 한다. 만약 있을경우 해당 Id에 업데이트 하게 된다.





# 주문 도메인 개발



Order 비즈니스 로직

```java
    //==생성 매소드==//
    public static Order createOrder(Member member, Delivery delivery, OrderItem... orderItems) {
        Order order = new Order();
        order.setMember(member);
        order.setDelivery(delivery);
        for (OrderItem orderItem : orderItems) {
            order.addOrderItem(orderItem);
        }
        order.setStatus(OrderStatus.ORDER);
        order.setOrderDate(LocalDateTime.now());
        return order;
    }

    //==비즈니스 로직==//
    /**
     * 주문 취소
     */
    public void cancel() {
        if (delivery.getStatus() == DeliveryStatus.COMP) {
            throw new IllegalStateException("이미 배송완료된 상품을 취소가 불가능합니다.");
        }

        this.setStatus(OrderStatus.CANCEL);
        for (OrderItem orderItem : orderItems) {
            orderItem.cancel();
        }
    }

    //==조회 로직==//
    /**
     * 전체 주문 가격 조회
     */
    public int getTotalPrice() {
        return orderItems.stream()
                .mapToInt(OrderItem::getTotalPrice)
                .sum();
    }
```

생성시 Order객체 생성후 Setter를 통해 초기화한뒤 반환 해준다. static 매소드로 만들어 생성 로직을 위 함수를 사용하도록 한다. 하지만 누군가 실수로 생성할 수 있음으로 생성자를 protected로 만들어준다. 이를 생략할 수 있는 어노테이션이 `@NoArgumentConstructor(access = AccessLevel.PROTECTED)` 이다.



취소시 해당 주문의 상태를 취소로 만들고 연결된 OrderItem들에도 cancel을 호출한다.



조회시 모든 Item의 count 와 price를 곱해서 응답한다.



OrderItem 비즈니스 로직



```java
     //==생성 매서드==//
    public static OrderItem createOrderItem(Item item, int orderPrice, int count) {
        OrderItem orderItem = new OrderItem();
        orderItem.setItem(item);
        orderItem.setOrderPrice(orderPrice);
        orderItem.setCount(count);

        item.removeStock(count);
        return orderItem;
    }

    //==비즈니스 로직==//
    public void cancel() {
        getItem().addStock(count);
    }

    //==조회 로직==//
    /**
     * 주문상품 전체 가격 조회
     */
    public int getTotalPrice() {
        return getOrderPrice() * getCount();
    }
```

생성시 OrderItem객체 생성 후 Setter를 통해 초기화한다. 마지막으로 item의 재고를 줄인다.



```java
    /**
     * 주문
     */
    @Transactional
    public Long order(Long memberId, Long itemId, int count) {
        //엔티티 조회
        Member member = memberRepository.findOne(memberId);
        Item item = itemRepository.findOne(itemId);

        //배송 생성
        Delivery delivery = new Delivery();
        delivery.setAddress(member.getAddress());

        //주문상품 생성
        OrderItem orderItem = OrderItem.createOrderItem(item, item.getPrice(), count);

        //주문 생성
        Order order = Order.createOrder(member, delivery, orderItem);

        //주문 저장
        orderRepository.save(order);

        return order.getId();
    }

    /**
     * 주문 취소
     */
    @Transactional
    public void cancelOrder(Long orderId) {
        //주문 엔티티 조회
        Order order = orderRepository.findOne(orderId);
        //주문 취소
        order.cancel();
    }
```



주문시 member와 item을 조회하여 필요한 정보를 수집한다. 수집된 정보를 바탕으로 OrderItem과 Order를 생성한다. 이때 생성자로 생성하는게 아닌 static으로 생성해둔 생성 매소드를 통해 생성한다.

마지막에 order를 저장하면 연결된 orderItem와 Item이 모두 업데이트 된다.



취소시 해당 order를 찾아 cancel매소드를 호출한다.







# 회원 등록



Form 데이터를 체크하기 위한 GetMapping을 한다.

```java
@GetMapping("/members/new")
public String createForm(Model model) {
    model.addAttribute("memberForm", new MemberForm());
    return "members/createMemberForm";
}
```



Form 데이터를 전달된 데이터를 memberSerivce를 통해 DB에 생성하고 `/`로 리다이렉트한다. 이때 @Valid를 통해 유효성 검사를 한다. 만약 데이터에 문제가 있다면 BindingResult에 에러가 들어오고 이를 응답해줄 수 있다.

```java
@PostMapping("members/new")
public String create(@Valid MemberForm form, BindingResult result) {
    if (result.hasErrors()) {
        return "members/createMemberForm";
    }

    Address address = new Address(form.getCity(), form.getStreet(), form.getZipcode());

    Member member = new Member();
    member.setName(form.getName());
    member.setAddress(address);

    memberService.join(member);
    return "redirect:/";
}
```





# 변경감지와 병합



### 준영속 컨텍스트

준영속 컨텍스트는 DB에서의 식별자를 가지고 있지만 직접 new로 생성한 객체로 영속성 컨텍스트가 변경에 대해 감지하지 않는다. 따라서 준영속컨텍스트는 변경후 commit을 하지 않으면 DB에 반영되지 않는다.



### merge

`merge()`를 사용하면 준영속 컨텍스트를 1차캐시된 엔티티를 찾아서 해당 엔티티에 set을 통해 데이터를 모두 저장한다. 그뒤 해당 영속성 컨텍스트를 return해준다. 따라서 이후 변경시에는 반환된 영속성 컨텍스트를 사용하면 된다.

주의할 점은 merge를 요청시 준영속성 컨텍스트의 모든 내용이 들어가기때문에 변경하지 않을려는 데이터도 모두 들어가고 없다면 null로 넣게된다. 따라서 실무에서는 사용하지 않는걸 권장한다.











# 실전! 스프링 부트와 JPA활용2 - API개발과 성능 최적화



## 회원 등록 API

```java
@PostMapping("/api/v1/members")
public CreateMemberResponse saveMemberV1(@RequestBody @Valid Member member) {
    Long id = memberService.join(member);
    return new CreateMemberResponse(id);
}


@Data
static class CreateMemberResponse {
    private Long id;

    public CreateMemberResponse(Long id) {
        this.id = id;
    }
}
```



Member를 생성하는 API로 Request Body에 대해서 Entity객체를 통해 bind 받게 되어있다. 이는 API 스펙과 Entity스펙을 연결 짓게 된다. 문제점은 Member의 스펙이 변경되면 모든 API 스펙에 변경이 요규된다. 따라서 이는 따로 가져가는 것이 중요하다.



```java
@PostMapping("/api/v2/members")
public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request) {

    Member member = new Member();
    member.setName(request.name);

    Long id = memberService.join(member);
    return new CreateMemberResponse(id);
}

@Data
static class CreateMemberRequest {
    private String name;
}
```

API 스펙과 Entity 스펙을 분리하였다. 따라서 Entity 변경시 API 스펙에 영향을 받지 않게 된다. 또한 API별로 @NotEmpty등 요구사항을 따로 적용할 수 있게된다.







## 회원 수정 API

```java
@PutMapping("/api/v2/members/{id}")
public UpdateMemberResponse updateMemberV2(
        @PathVariable("id") Long id,
        @RequestBody @Valid UpdateMemberRequest request
) {
    memberService.update(id, request.getName());
    Member findMember = memberService.findOne(id);
    return new UpdateMemberResponse(findMember.getId(), findMember.getName());
}
```

```java
@Transactional
public void update(Long id, String name) {
    Member member = memberRepository.findOne(id);
    member.setName(name);
}
```



`@PathVariable`을 통해 변경할 Member의 id를 받고 Body를 통해 변경될 내용을 받게 된다. update함수는 Member를 영속성 컨텍스트로 받아와 name을 변경하고 종료된다. 다시 돌아와 API에서는 Member를 조회하여 전달한다. 이때 조회를 한번더 하는건 update함수에서 조회 기능을 배제하기 위함이다. 성능적으로 문제가 없다면 유지보수 차원에서 효과적일 수 있다.





## 회원 조회 API

```java
@GetMapping("/api/v1/members")
public List<Member> membersV1() {
    return memberService.findMembers();
}
```

기본적으로 생각할 수 있는 조회 API이다. memberService를 통해 조회한 Entity를 바로 return 해주면 Json으로 변경되어 응답할 수 있게된다. 문제점은 Response와 Entity가 같다는 것이다. 만약 Entity가 변경되면 관련되 API 스펙에 영향을 끼치게 된다. 따러서 분리하는것이 좋다. 또한 응답이 List형태로 시작한다. 이는 추가적인 파라미터를 넣기에 유연하지 못하다.



```java
@GetMapping("/api/v2/members")
public Result membersV2() {
    List<Member> findMembers = memberService.findMembers();
    List<MemberDto> collect = findMembers.stream().map(m -> new MemberDto(m.getName()))
            .collect(Collectors.toList());

    return new Result(collect);
}

@Data
@AllArgsConstructor
static class Result<T> {
    private T data;
}
@Data
@AllArgsConstructor
static class MemberDto {
    private String name;
}
```

Response에 대해 DTO를 정의해주고 Entity로부터 필요한 데이터만 파싱해온다. 이는 API스펙과 Entity 스펙을 분리할 수 있도록 해준다. 마지막에 Result로 감싸서 Response의 상단이 Object형태로 올 수 있도록 한다.









## 간단한 주문 조회 V1: 엔티티를 직접 노출

```java
@GetMapping("/api/v1/simple-orders")
public List<Order> ordersV1() {
    List<Order> all = orderRepository.findAllByString(new OrderSearch());
    return all;
}
```

위와 같이 Entity를 직접호출하여 Order전체를 조회한다. 이때 Order를 조회하면서 내부에 Lazy로딩이 걸리는 데이터에 대해서 모두 호출이 된다. 조회된 Entity에서는 다시 Lazy로딩을 한다. 따라서 순환참조가 일어나 무한 로딩이 걸리게 된다. 이를 막기 위해서는 한쪽에서 JsonIgnore를 통해 다시 참조하지 않도록 해야한다. 하지만 해당 엔티티는 null이 들어 갈 수 없다. 왜냐하면 프록시 객체가 들어있기 때문이다. 따라서 해당 데이터를 null로 인식되도록 아래 설정을 추가할 수 있다.

```java
@Bean
Hibernate5Module hibernate5Module() {
	return new Hibernate5Module();
}
```





## 간단한 주문 조회 V1: 엔티티를 DTO로 변환

```java
@GetMapping("/api/v2/simple-orders")
public List<SimpleOrderDto> ordersV2() {
    List<Order> orders = orderRepository.findAllByString(new OrderSearch());
    return orders.stream()
            .map(SimpleOrderDto::new)
            .collect(Collectors.toList());
}

@Data
static class SimpleOrderDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;

    public SimpleOrderDto(Order order) {
        orderId = order.getId();
        name = order.getMember().getName();
        orderDate = order.getOrderDate();
        orderStatus = order.getStatus();
        address = order.getDelivery().getAddress();
    }
}
```

API Response에 대해 미리 상의후 정의 한다. 이는 Entity와 API 스펙을 분리하는데 목적이 있다. 하지만 아직 문제점이 남아 있는데 Order조회후 getMember()호출시 Member가 Lazy로딩이 되면서 Query를 날리게 된다. 이를 N + 1 문제라고 한다.



## 간단한 주문 조회 V1: 페치 조인 최적화

```java
@GetMapping("/api/v3/simple-orders")
public List<SimpleOrderDto> ordersV3() {
    List<Order> orders = orderRepository.findAllwithMemberDelivery();
    return orders.stream()
            .map(SimpleOrderDto::new)
            .collect(Collectors.toList());
}
```



```java
public List<Order> findAllwithMemberDelivery() {
    return em.createQuery(
            "select o from Order o" +
                    " join fetch o.member m" +
                    " join fetch o.delivery d", Order.class
    ).getResultList();
}
```



위에서 생겼던 N+1 문제를 해결하기 위해 서는 Query를 조인을 통해 한번에 해야 한다. 이는 JPQL을 이용하면 한번에 관련 조인 데이터를 가져오는 쿼리를 만들 수 있다.





## 간단한 주문 조회 V1: JPA에서 DTO 바로 조회

```java
@GetMapping("/api/v4/simple-orders")
public List<OrderSimpleQueryDto> ordersV4() {
    return orderRepository.findOrderDtos();
}
```

```java
public List<OrderSimpleQueryDto> findOrderDtos() {
    return em.createQuery(
            "select new jpabook.jpashop.repository.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address) " +
                    "from Order o" +
                    " join o.member m" +
                    " join o.delivery d", OrderSimpleQueryDto.class
    ).getResultList();
}
```

실제 SQL에서 조회할 컬럼을 선택하듯이 OrderSimpleQueryDto를 생성하여 원하는 컬럼만 조회하는 방법이다. 모든 컬럼이 아닌 필요한 컬럼만 가져올 수 있다. 하지만 생각보다 성능 향상의 효과가 미비하다. 또한 API스펙에 대한 내용이 Repository 안에 들어오게 된다. 따라서 사용해야한다면 별도로 Repository를 생성하여 분리하는 것이 좋다.







## 주문 조회 V1: 엔티티 직접 노출

```java
@GetMapping("/api/v1/orders")
public List<Order> ordersV1() {
    List<Order> all = orderRepository.findAllByString(new OrderSearch());
    for (Order order : all) {
        order.getMember().getName();
        order.getDelivery().getAddress();
        List<OrderItem> orderItems = order.getOrderItems();
        orderItems.stream().forEach(orderItem -> orderItem.getItem().getName());
    }
    return all;
}
```

기본 적으로 생각할 수 있는 일대다 관계의 엔티티를 조회하는 API이다. getOrderItems() 한 뒤 Item을 조회하여 Lazy로딩을 한다. 이렇게 하면 필요한 데이터가 모두 조회되어 한번에 응답할 수 있게된다. 하지만 API스펙에 Entity가 적용되어 Entity 변경이 어려워진다.



## 주문 조회 V2: 엔티티를 DTO로 변환

```java
@GetMapping("/api/v2/orders")
public List<OrderDto> ordersV2() {
    List<Order> orders = orderRepository.findAllByString(new OrderSearch());
    List<OrderDto> collect = orders.stream()
            .map(o -> new OrderDto(o))
            .collect(Collectors.toList());

    return collect;
}
```

```java
@Getter
static class OrderDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;
    private List<OrderItemDto> orderItems;
    public OrderDto(Order order) {
        orderId = order.getId();
        name = order.getMember().getName();
        orderDate = order.getOrderDate();
        orderStatus = order.getStatus();
        address = order.getDelivery().getAddress();
        orderItems = order.getOrderItems().stream()
                .map(OrderItemDto::new)
                .collect(Collectors.toList());
    }
}

@Getter
static class OrderItemDto {
    private String itemName;
    private int orderPrice;
    private int count;

    public OrderItemDto(OrderItem orderItem) {
        itemName = orderItem.getItem().getName();
        orderPrice = orderItem.getOrderPrice();
        count = orderItem.getCount();
    }
}
```

DTO를 생성하여 해당 조회된 Order로 채우게 된다. 이렇게 하면 API 스펙과 Entity를 분리하였지만 Query수는 굉장히 많게 된다. 주의할 점은 OrderDto안에 OrderItem을 사용하는 것이 아닌 OrderItemDto를 만들어 넣어주어야 한다는 점이다.





## 주문 조회 V3: 엔티티를 DTO로 변환 - 패치 조인 최적화

```java
@GetMapping("/api/v3/orders")
public List<OrderDto> ordersV3() {
    List<Order> orders = orderRepository.findAllWithItem();
    List<OrderDto> collect = orders.stream()
            .map(o -> new OrderDto(o))
            .collect(Collectors.toList());

    return collect;
}
```

```java
public List<Order> findAllWithItem() {
    return em.createQuery(
            "select distinct o from Order o" +
                    " join fetch o.member m" +
                    " join fetch o.delivery d" +
                    " join fetch o.orderItems oi" +
                    " join fetch oi.item i", Order.class).getResultList();

}
```

Fetch join을 통해 쿼리를 줄일 수 있다. 하지만 그냥 페치 조인만 사용하게되면 조인된 데이터 수 만큼 늘어나게 된다. 따라서 중복을 제거해야 하는데 distinct를 사용하면 중복을 제거할 수 있다. 이때 distinct는 SQL에서의 distinct도 붙여주지만 조회된 Entity에 대해서 id로 중복을 한번더 제거해 준다. 



하지만 위 방법은 치명적인 단점이 있다. 페이징이 불가능 하다는 점이다. 만약 setFirstResult 또는 setMaxResults를 통해 페이징을 하면 SQL에 페이징이 들어가는게 아니라 Entity를 메모리에 올리고 페이징을 하게된다.









## 주문 조회 V3.1: 엔티티를 DTO로 변환 - 페치 조인 최적화

```java
@GetMapping("/api/v3.1/orders")
public List<OrderDto> ordersV3_page(
        @RequestParam(value = "offset", defaultValue = "0") int offset,
        @RequestParam(value = "limit", defaultValue = "100") int limit
) {
    List<Order> orders = orderRepository.findAllwithMemberDelivery(offset, limit);

    List<OrderDto> collect = orders.stream()
            .map(o -> new OrderDto(o))
            .collect(Collectors.toList());

    return collect;
}
```

```java
public List<Order> findAllwithMemberDelivery(int offset, int limit) {
    return em.createQuery(
            "select new jpabook.jpashop.repository.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address) " +
                    "from Order o" +
                    " join o.member m" +
                    " join o.delivery d", Order.class
            ).setFirstResult(offset)
            .setMaxResults(limit)
            .getResultList();
}
```

```yaml
  jpa:
    properties:
      hibernate:
          default_batch_fetch_size: 100
```



기존 일대 다관계에서 Lazy로 로딩하는 경우 n+1 문제 때문에 데이터들이 메모리에 한번에 올라가는 문제가 있었다. 이는 1* n * m 번의 쿼리가 전송되게 된다. 이를 해결하려면 JPA의 `default_batch_fetch_size`를 사용하면 된다. 앞서 조회한 쿼리에 대한 내용을 in으로 묶어서 한번에 조회하게된다. 따라서 1 + 1 + 1 의 쿼리가 전송되게 되는것이다. 따라서 다대일 Lazy로딩과 같은 효과를 보게되어 페이징이 가능해진다.









## 주문 조회 V4: JPA에서 DTO 직접조회

```java
@GetMapping("/api/v4/orders")
public List<OrderQueryDto> ordersV4() {
    return orderQueryRepository.findOrderQueryDtos();
}
```

```java
public List<OrderQueryDto> findOrderQueryDtos() {
    List<OrderQueryDto> result = findOrders();
    result.forEach(o -> {
        List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId());
        o.setOrderItems(orderItems);
    });
    return result;
}

private List<OrderItemQueryDto> findOrderItems(Long orderId) {
    return em.createQuery(
            "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count) from OrderItem oi" +
                    " join oi.item i" +
                    " where oi.order.id = :orderId", OrderItemQueryDto.class)
            .setParameter("orderId", orderId)
            .getResultList();
}

private List<OrderQueryDto> findOrders() {
    return em.createQuery(
            "select new jpabook.jpashop.repository.order.query.OrderQueryDto(o.id, m.name, o.orderDate, o.status, d.address) from Order o" +
                    " join o.member m" +
                    " join o.delivery d", OrderQueryDto.class
    ).getResultList();
}
```



DTO로 분리하기 위해 먼저 Order를 조회하여 DTO에 초기화한다. 이때 join을 사용하여 Order만 영속화 하여 가져온다. 이때 orderItems는 컬렉션이라 바로 초기화가 안된다. 따라서 가져온 OrderQueryDto에서 orderid를 통해 OrderItems를 다시 조회하여 OrderItemQueryDto를 태운다. 이렇게 하면 DTO와 API 스펙을 분리하게 된다. 하지만 처음 Order가 하나 조회될때 OrderItem이 N번 조회되어 n+1 문제가 발생한다.







## 

## 주문 조회 V5: JPA에서 DTO 직접 조회 - 컬렉션 조회 최적화

```java
@GetMapping("/api/v5/orders")
public List<OrderQueryDto> ordersV5() {
    return orderQueryRepository.findAllByDto_optimization();
}
```

```java
public List<OrderQueryDto> findAllByDto_optimization() {
    List<OrderQueryDto> result = findOrders();

    Map<Long, List<OrderItemQueryDto>> orderItemMap = findOrderItemMap(toOrderItems(result));

    result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

    return result;
}

private Map<Long, List<OrderItemQueryDto>> findOrderItemMap(List<Long> orderIds) {
    List<OrderItemQueryDto> orderItems = em.createQuery(
                    "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count) from OrderItem oi" +
                            " join oi.item i" +
                            " where oi.order.id in :orderIds"
                    , OrderItemQueryDto.class)
            .setParameter("orderIds", orderIds)
            .getResultList();

    Map<Long, List<OrderItemQueryDto>> orderItemMap = orderItems.stream()
            .collect(Collectors.groupingBy(OrderItemQueryDto::getOrderId));
    return orderItemMap;
}

private List<Long> toOrderItems(List<OrderQueryDto> result) {
    List<Long> orderIds = result.stream()
            .map(o -> o.getOrderId())
            .collect(Collectors.toList());
    return orderIds;
}
```



Order는 똑같이 조회한 뒤 id를 for문으로 반복해서 쿼리를 전송하는게 아닌 하나의 Collection으로 만들고 `=`조건이 아닌 `in`을 통해 한번에 쿼리를 한다. 받은 데이터를 ID별로 Map을 만든다. 마지막으로 OrderQueryDto에 id 별 OrderItemQueryDto를 찾아 초기화 해준다. 위 방법을 통해 n+1 문제를 해결 할 수 있게된다.





## 주문 조회 V6: JPA에서 DTO로 직접 조회, 플렛 데이터 최적화

```java
@GetMapping("/api/v6/orders")
public List<OrderFlatDto> ordersV6() {
    return orderQueryRepository.findAllByDto_flat();
}
```

```java
public List<OrderFlatDto> findAllByDto_flat() {
    return em.createQuery(
            "select new jpabook.jpashop.repository.order.query.OrderFlatDto(o.id, m.name, o.orderDate, o.status, d.address, i.name, oi.orderPrice, oi.count)" +
                    " from Order o" +
                    " join o.member m" +
                    " join o.delivery d" +
                    " join o.orderItems oi" +
                    " join oi. item i", OrderFlatDto.class).getResultList();
}
```

```java
@Data
public class OrderFlatDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;
    private String itemName;
    private int orderPrice;
    private int count;

    public OrderFlatDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, String itemName, int orderPrice, int count) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
        this.itemName = itemName;
        this.orderPrice = orderPrice;
        this.count = count;
    }
}
```



기존에 일대다 관계에서는 해당 데이터를 모아서 다시 쿼리를 전송했는데 위와 같이하면 한번에 데이터를 조인해서 가져올 수 있다. 하지만 그룹핑 하지 않아 후작업으로 어플리케이션 레벨에서 그룹핑을 해야한다. 또한 페이징도 할 수 없게 된다.





## OSIV의 성는 최적화

open session in view는 데이터베이스의 session을 사용하는 라이프사이클을 클라이언트한테 전달 될때까지 유지한다. 따라서 session을 반환하는데 오랜 시간이 걸릴다. 만약 이걸 끄게 되면 service 까지만 session이 유지되고 이후 호출되는 session은 Exception을 발생시킨다.

```yaml
  jpa:
    open-in-view: false
```