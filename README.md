# JPA API
* Entity 내에 유효성 검증하는 로직이 잇다면 어느곳에서는 필요없는 로직일수도 있기에 확장성이 떨어진다.
  * Api 스펙이 변경될수 있기에 절대로 **엔티티를 외부에 노출하거나 파라미터를 엔티티로 받으면 안된다.**
* Entity가 수정되는경우 API 스펙 자체가 변경이 되므로 디비와 바로 1:1 연결되어 잇기에 큰 장애가 일어날 확률이 높다.
* API 통신을 위한 엔티티에 경우 별도에 DTO를 받아 엔티티에 값을 넣어야된다.
* API 스펙을 보지 않는 이상 엔티티만 보고 어떤 값이 들어오는지 알 수 있는 방법이 없다.
  * 하지마 중간에 별도에 DTO가 있으면 API가 어떤값이 들어와야하는지 보기 쉽다.
~~~
    @PostMapping("/api/v2/members")
    public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request) {
        // request 별도에 DTO 만들어 매핑
        Member member = new Member();   // Entity 생성
        member.setName(request.getName());
        Long id = memberService.join(member);
        return new CreateMemberResponse(id);
    }
~~~

* API 조회시에도 엔티티로 조회를 하면안된다. 마찬가지로 API스펙 변경에 우려가 있다.
  * 조회용 DTO를 만들어 최대한 유연하게 작성하여 보여준다.
* 아래와 같이 사용하면 외부가 배열로 감싸져 있기에 확장성이 떨어진다.
  * 객체 형태로 감싸지고 안에 배열행태가 좋습니다.
~~~
  // 엔티티 직접노출
  @GetMapping("/api/v1/members")
  public List<Member> membersV1() {
      return memberService.findMembers();
  }
  // 엔티티 노출되신 Result라는 조회용 DTO 생성
  @GetMapping("/api/v2/members")
  public Result memberV2() {
      List<Member> members = memberService.findMembers();
      List<MemberDto> collect = members.stream()
              .map(e -> new MemberDto(e.getName()))
              .collect(Collectors.toList());
      return new Result(collect);
  }
  // 조회용 DTO 제네릭을 사용해 data안에 Object를 감싸 노출
  // 갯수나 API스펙이 변경되어도 유연하게 대처할수 있음
  @Data
  @AllArgsConstructor
  static class Result<T>{
      // private int size; ...
      private T data;
  }
~~~

### 지연로딩 조회 성능 최적화
* 엔티티를 직접 노출하여 양방향 연관관계일때 한쪽은 @JsonIgnore 하여 한쪽은 참조를 끊어 무한참조를 막아야한다.
* Hibernate5Module를 이용하여 엔티티를 간단하게 API반환할수도 있지만 DTO로 반환하는것이 더 좋은 방법이다.
* **주의 :** 지연로딩을 피하기위해 즉시로딩(EAGER) 설정을 하면안된다. 즉시로딩때문에 연관관계가 필요없는 경우도 데이터를 항상 조회하여<br>
성능 문제가 발생할수있다. 즉시로딩 설정시 성능 튜닝이 매우 어려우므로 기본을 지연로딩으로 하고 성능 최적화를 위해서는 fetch join을 이용한다.

* SQL 사용할때처럼 원하는 값을 선택하여 new 명령어를 사용 JPQL 결과를 DTO로 즉시 반환
* 데이터를 직접 선택하므로 DB -> 어플리케이션 네트워크 용량 최적화(생각보다 미비)
* repository에 재사용성이 떨어지게된다. 유연하지 않음
~~~
return em.createQuery(
                "select o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d",Order.class
        ).getResultList();

return em.createQuery(
                "select new jpabook.jpashop.repository.order.simplequery.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d",OrderSimpleQueryDto.class
        ).getResultList();
~~~

* xToOne 이 아닌 oneToMany 같은 일대 다 조회를 통해 최적화하는 방법
* fetch join 이용하여 sql 1번으로 조회
* distinct를 사용하여 데이터베이스에 row에 중복조회를 막아준다.
* 단점 일대다를 페치조인하는 순간 부터는 페이징처리가 불가능하다.
* 메모리에 데이터를 모두 다 올리고 페이징 처리를 진행하는데 많은 양을 메모리에 올려서 하는경우 error가 발생하여 어플리케이션이 죽을수 있다.
  * 너무 위험하므로 사용하면 안된다.
* 컬렉션 페치조인은 1개만 사용할 수 있다. n대n대n이라면 엄청난 뻥튀기가 되어 데이터 자체 정합성이 이상해진다.
~~~
public List<Order> findAllWithItem() {
        return em.createQuery(
                "select distinct o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d" +
                        " join fetch o.orderItems oi" +
                        " join fetch oi.item i",Order.class)
                .getResultList();
    }
~~~

### 페이징 한계 돌파
* 컬렉션을 조인하게 되면 페이징이 불가능하게된다.
  * 일 기준으로 페이징을 해야되는데 데이터를 다(N)개 기준으로 row가 생성되어서 문제가 된다.
  * 페치조인을 페이징 시도하는 경우 모든 데이터를 메모리에 올려 페이징을 시도하여 최악의 경우 OutofMemoryError 발생하여 애플리케이션을 뻗을 수 있음

* 먼저 xToOne 관계를 모두 페치 조인을 한다.
  * row수를 증가시키지 않기 때문에 페이징 조인 쿼리에 영향을 안주게된다.
* 컬렉션은 지연 로딩으로 조인한다.
* 지연 로딩 성능을 최적화 위해 hibernate.default.batch_size , @BatchSize를 적용한다.
  * 미리 정해진 사이즈만큼 메모리에 조회하는것 -> OutofMemoryError 방지
  * 1:N:M을 -> 1:1:1 로 만들어주는 엄청난 최적화 기술이다.
* batchsize 최대 사이즈는 1000으로 제한하여 사용하자


### spring.jpa.open-in-view
* 애플리케이션 시작 시점에 warn 로그를 남기는데 이유가 있다. 영속성 컨텍스트는 화면에 다 그려질때까지 끝까지 데이터베이스 커넥션을 유지하고 있다.
  * 위 덕분에 지연로딩이 가능했다.
* 그런데 이 전략은 오랜시간 데이터베이스 커넥션을 오랜시간 유지하여 많은 리소스를 사용하기에 실시간 트래픽이 중요한 애플리케이션에는 커넥션이 모자랄 수 있다.
* **단점 :** 예를 들어 컨트롤러 외부에서 api호출하며 외부 api대기 시간만큼 커넥션 리소스를 반환 못하고 유지하고 있어야한다.
* spring.jpa.open-in-view : false -> 데이터베이스 커넥션 유지시간을 짧게 가져감
  * 서비스 계층에 트랜잭션이 끝나면 데이터베이스 커넥션을 반환함 -> 커넥션 리소를 낭비하지않음
  * **단점 :* * 지연로딩을 하려면 영속성 컨텍스트가 살아 있어야하는데 모든 지연로딩을 트랜잭션 안에서 처리해야한다. 컨트롤러나 화면에서 지연로딩을 처리하는 로직은 작동하지 않는다.

* **해결방안 중 1 :** : 쿼리용 서비스를 만들어 @transaction 안에 지연로딩이 되는 로직 자체를 안에 넣어 동작 시킨다.
* 다 안에 넣고 컨틀롤러에서 orderqueryService.service1(): <- 이런식으로 동작시킴
  ~~~
  @Transaction(readonly: true)
  public class OrderQueryService {
  List<Order> orders = orderRepository.findAllWithItem();
        return orders.stream().map(OrderDto::new)
                .collect(Collectors.toList());
  }
  ~~~
  
* 참고 : 고객서비스 기반의 실시간 API는 OSIV(open session in view) 에서는 끄고 , ADMIN 처럼 커넥션을 많이 사용하지 않는곳에는 OSIV를 켠다.