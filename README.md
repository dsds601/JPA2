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
  ~~~# JPA2
