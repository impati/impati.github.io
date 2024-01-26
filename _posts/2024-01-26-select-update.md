---
layout: post
date: 2024-01-21
categories: [database,JPA]
title:  "SELECT 만 했는데 왜 UPDATE 쿼리가"
img_path: /img/path
---

## 들어가며
다음과 같이 Entity 를 조회만 했는데 update 쿼리를 만난 이야기를 다루고자한다.

```java
@Service
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository memberRepository;

    @Transactional
    public Member getMember(Long id) {
        List<Member> allMember = memberRepository.findAll();
        return allMember.stream()
                .filter(member -> member.getId().equals(id))
                .findFirst()
                .orElseThrow(IllegalStateException::new);
    }
}
```

```java
@SpringBootTest
class MemberCommandServiceTest {
    
    @Autowired
    private MemberService memberService;

    @Test
    @DisplayName("id 로 Member 를 가져온다.")
    void getMember() {
        // given
        Long memberId = 1L;

        // when
        Member member = memberService.getMember(memberId);

        // then
        assertThat(member.getId()).isNotNull();
    }
}
```

MemberServiceTest 를  spring.jpa.show-sql=true 설정을 준 뒤 실행하면 select 쿼리 후에 update 쿼리가 전달되는 것을 확인할 수 있다.

![](/select/first.png)
분명히 로직으로 보았을 때에 Member 에 어떠한 변경을 가하지도 않았는데 update 쿼리는 왜 전달되는 것일까?
```java
@Getter
@Entity
@NoArgsConstructor
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Convert(converter = AgeConverter.class)
    private Age age;

    public Member(final String name, final Age age) {
        this.name = name;
        this.age = age;
    }
}
```
심지어 Member Entity 에는 Member 의 상태를 변경할 수 있는 어떠한 API 도 없다.  이 미스테리한 문제를 파헤쳐보기 위해 변경감지가 어떻게 이루어지는지 알아볼 필요가 있다.

## 변경 감지 동작 방식 알아보기
트랜잭션 범위 내에서 JPA Entity 객체를 가져오면 그 Entity 의 복사본을 생성하고 flush 가 호출되는 시점에 복사본  Entity 와 비교하여 변경이 되었으면 update 쿼리를 전달하는 것으로 알려져있다. 이를 변경감지(dirty checking) 이라고한다.

Entity 와 복사본 Entity 간의 비교는 AbstractEntityPersister 의 findDrity 에서 properites 마다 체크를 수행하며 isDrity 메서드를 호출하면서 시작된다.
![](/select/second.png)

properties[i] 의 Type 으로는 다음과 같이  OneToOne , ManyToOne ComponetType , AnyType 등 이 있으며 각 Type 마다 isDirty 를 구현하고 있다.
![](/select/third.png)
내부 구현을 살펴보면  `@OneToMany`, `@OneToOne`, `@ManyToOne`등으로 선언된 객체를 EntityType 이라고 하는데 이 경우에는 항상 false 를 응답한다.

자바 객체 String , Integer ,Boolean , LocalDateTime 등의  자바 타입의 경우에는 이미 구현되어 있는 JavaTypeDescriptor 가 객체 간의 동등성 비교를 수행하여 변경 여부를 판단하고 @Embeddable 타입의 경우 ComponentType 인데  ComponentType 의 변경 여부 판단은 @Embeddable 타입의 객체 내부 속성을 비교하여 여부를 판단하게 된다.

그리고 이외에 JavaTypeDescriptor 가 등록되어 있지 않은 타입이거나 @Embeddable 혹은 EntityType 이 아닌 경우에는 Objects.equals 를 호출하며 더티체킹 여부를 판단한다.

## AttributeConverter 를 사용했을 때

Entity Properties 로 사용자 정의 객체를 선언할 수 있는 방법은 크게 세가지 있는데 첫 번째는 위에서 언급한 EntityType , 두 번째는 @Embeddable 마지막으로 AttributeConverter 가 있다. 이때 AttributeConverter 를 사용해서 Entity Properties 로 사용자 정의 객체를 사용한다면 그에 맞는 JavaTypeDescriptor 를 등록해주거나 반드시 equals,hashcode 를 재정의해줘야한다.

만약 트랜잭션 범위내에서 Entity 와 복사본 Entity 간의 비교할 때 AttributeConverter 를 사용한 propertie 의  JavaTypeDescriptor 혹은 equals,hashcode 정의가 되어 있지 않다면 참조만을 비교하기 때문에 더티체킹하는 시점에 항상 true 가 응답되어 update 쿼리가 발생하기 때문이다.

앞서 살펴보았던 Member 의 Age 에 JavaTypeDescriptor 혹은 equals,hashcode 를 정의해주지 않았기 때문에 복사할 때마다 참조가 다르게 되어 플러쉬 시점에 update 쿼리가 전달되는 것이었다. 

```java
@Getter
public class Age {

    private final int value;

    public Age(int value) {
        this.value = value;
    }

    public Integer getValue() {
        return value;
    }
}
```

equals , hashcode 를 정의해주고 다시한번 테스트를 실행하면 더이상 update 쿼리는 전달되지 않는 모습을 볼 수 있을 것이다.

## 발생할 수 있는 장애 상황

AttributeConverter 를 사용한 properties 에 equals,hashcode 를 정의해주지 않는 상태로 사용하게 된다면 트랜잭션에서 N개의 Entity 를 조회했을 때 N 개의 update 가 전달되는 일이 벌어진다는 것이다. 이는 분명 예상치 못한 사이드 이펙트임이 틀림없다.

발생할 수 있는 다른 장애 상황으로는 영속성 컨텍스트가 관리되는 트랜잭션에서 **@Modifying 쿼리를 사용했을 때이다.**

```java

@Transactional
public void deleteAllMember() {
    List<Member> allMember = memberRepository.findAll();
    List<Long> ids = allMember.stream()
            .map(Member::getId)
            .collect(Collectors.toList());

    memberRepository.deleteAllByIds(ids);
}

==========

public interface MemberRepository extends JpaRepository<Member, Long> {

    @Modifying
    @Query("delete from Member m where m.id in(:ids)")
    void deleteAllByIds(@Param("ids") List<Long> ids);
}
```

만약 위와 같이 트랜잭션에서 모든 Member 를 가져와 영속성 컨텍스트에 복사본을 저장하고 

@Modifying 어노테이션을 사용해서 `deleteAllByIds` 를 호출하여 DB 에 직접 Member 를 삭제하면 현재 영속성 컨텍스트에서는 삭제된 사실을 알지 못하기 때문에 트랜잭션 커밋전 플러쉬가 호출됨에 따라 update 쿼리가 전달될 것이고 이미 DB 에는 어떠한 Member 정보가 없는데 update 를 시도했기 때문에  `Row was updated or deleted by another transaction` 가 발생한다.

같은 맥락으로 @Modifying 어노테이션을 사용해서 update 쿼리를 영속성 컨텍스트에 반영하지 않고 바로 DB 에 반영하는 경우 DB 에 업데이트한 내용이 유실되는 문제가 나타날 수 있다. 이 경우에는 에러도 발생하지 않으니 발견하는 것도 어려울 것 같다.

## 마무리

결론은 Entity 에서 AttributeConverter 를 사용한 properties 에 equals,hashcode 를 잊지말고 정의해주자.