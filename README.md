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
  * 바로 DTO 반환은 권장하지 않습니다.
~~~
// 권장
return em.createQuery(
                "select o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d",Order.class
        ).getResultList();

// 권장하지 않음
return em.createQuery(
                "select new jpabook.jpashop.repository.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d",OrderSimpleQueryDto.class
        ).getResultList();
~~~