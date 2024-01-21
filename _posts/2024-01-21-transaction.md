---
layout: post
date: 2024-01-21
categories: [database,JPA]
title:  "@Transactional(readOnly=true) 에서 쓰기 트랜잭션"
img_path: /img/path
---

## 들어가며
데이터베이스에 정말 읽기만 하는 경우 @Transactional(readOnly=true) 로 읽기 전용 트랜잭션을 사용하여 플러쉬를 호출하지 않음으로써 어느 정도의 성능 최적화를 어느정도 이뤄낼 수 있다. 
또는 읽기 전용 DB 에 쿼리를 보내기 위해 @Transactional(readOnly=true) 를 사용하여 라우팅 하기도 한다.

읽기 트랜잭션에서 쓰기 트랜잭션을 사용하면 서버에서 오류가 발생할 것이라고 예상했지만 어느 날 @Transactional(readOnly=true) 트랜잭션 내에서 쓰기 트랜잭션을 사용할 때 서버에서 오류가 나는 것이 아닌 DB에서 오류가 발생하는 문제를 마주했다.

## @Transactional(readOnly=true) 에서 쓰기 작업

읽기만을 위해 readOnly=true 로 설정했는데 쓰기 작업을 수행한 것은 분명히 실수임이 틀림없다. 하지만 테스트 단계에서 이 문제는 어째서 발견되지 못했을까?

예를 들어 사용자 id 로 Member 객체를 가져오고 만약 없다면 새롭게 만들어서 반환하는 로직이 있다고 해보자.

```java
@Service
public class MemberService {

    private final MemberReader memberReader;
    private final MemberCreator memberCreator;
    
    @Transactional(readOnly = true)
    public Member getMember(Long id) {
        List<Member> allMember = memberReader.getAllMember();
        return allMember.stream()
                .filter(member -> member.getId().equals(id))
                .findFirst()
                .orElseGet(memberCreator::create);
    }
}

======

@Component
public class MemberCreator {

    private final MemberRepository memberRepository;

    @Transactional
    public Member create() {
        return save(new Member("default"));
    }
}
```

이러한 로직이 나올 수 있는 상황은 정말 많다. 작업자가 다른 경우나 요구사항이 변경되어 MemberSerivce 가 하는 역할이 조금 달라진 경우  , 프로젝트가 매우 복잡한 경우 고려해야 할 부분이 많아서 readOnly=true  로 설정한 트랜잭션에서 쓰기 트랜잭션을 호출하는 실수를 할 수 있다. 

중요한 점은 개발자가 미처 캐치하지 못했다고 하더라도 테스트를 통해 충분히 발견되어 수정되었어야 했다.

그리고 테스트에서 발견되지 못한 이유는 읽기 트랜잭션에서 쓰기 트랜잭션을 호출해도 오류가 발생하지 않고 정상적으로 동작했기 때문이다.

## 오류가 발생하지 않은 이유

오류가 발생하지 않아 테스트에서 미처 발견되지 못할 수 있는 부분을 암시하는 내용을  `org.springframework.transaction.annotation` 의 javadoc 에서 확인할 수 있었다.

> A boolean flag that can be set to true if the transaction is effectively read-only, allowing for corresponding optimizations at runtime.
Defaults to false.
This just serves as a hint for the actual transaction subsystem; it will not necessarily cause failure of write access attempts. A transaction manager which cannot interpret the read-only hint will not throw an exception when asked for a read-only transaction but rather silently ignore the hint
> 

@Transactional(readOnly=true) 로 설정했더라도 반드시 쓰기에 대한 엑세스 시도가 실패하는 것이 아닐 수 있다고 한다. 즉 DB 마다 조금씩 다를 수 있음을 명시해 놓았다.

### H2

테스트에서  h2 데이터베이스를 자주 사용하는데 h2 데이터베이스를 사용하는 경우에 오류가 발생하지 않는지 확인해 보았다.

```java
@SpringBootTest
class MemberServiceTest {

    @Autowired
    private MemberService memberService;

    @Test
    @DisplayName("id 로 멤버를 가져오는데 없으면 default Member 를 생성해서 가져온다.")
    void getMember() {
        Long id = 10003L;

        Member member = memberService.getMember(id);

        assertThat(member.getId()).isNotNull();
    }
}
```

위 테스트에서 `memberService.getMember(id);` 를 호출하면 예외가 발생하면서 실패하길 기대하지만 실제로 이 테스트는 성공한다. @Transactional(readOnly=true) 로 설정했지만 쓰기에 성공한 모습이다.

### MySQL

MySQL 을 설정하고 위 테스트를 동일하게 실행하면 이런 문구와 함께 이 테스트는 실패한다. 

```java
could not execute statement
....
Connection is read-only. Queries leading to data modification are not allowed
```

테스트 DB 로 MySQL  를 사용하면 읽기 트랜잭션에서 쓰기 트랜잭션을 호출하는 경우 쿼리를 컴파일 하는 과정인 PrepareStatement 에서 예외를 던진다. MySQL 를 테스트 DB 로 사용했다면 테스트 단계에서 위 문제를 발견하고 조치할 수 있었을 것이다.

### MariaDB

에러가 발생한 프로젝트에서는  MariaDB 를 사용하고 있었는데 이 문제를 마주한 것으로 봐서는 H2 와 동일하게 위 테스트를 통과할 것으로 예상했고 실제로 통과하였다. 쿼리를 executeUpdate() 과정에서 readOnly 를 체크하고 예외를 던지는 Driver 가 있는 반면 그렇지 않은 Driver 도 있었기에 이런 차이가 발생했다.

MySQL 의 `ClientPreparedStatement` 에서는 readOnly 를 체크하고 예외를 던지는 모습이다.

![](/tx/first.png)

반면 MariaDB 의 ClientSidePreparedStatement 에서는 그런 과정을 찾아볼 수 없다.

![](/tx/second.png)

읽기 트랜잭션에서 쓰기 트랜잭션이 잘 돌아간다면 무엇이 문제인가 ? 읽기를 위한 목적과 쓰기를 하기 위한 목적을 최종적으로 모두 달성한 것이 아닌가 라는 생각이 들 수도 있을 것 같다.

문제는 Read-Write DB 를 별도로 구축되어 있고 읽기 트랜잭션에서 쿼리를 Read DB 로 보내게 되었을 때 발생한다.

Read-Write DB 를 별도로 구축되어 있다면 Read DB 에 쓰기 동작을 수행할 수 없어 쿼리가 전달되었을 때 에러가 발생하게 되고 결국 쓰기 동작의 목적은 달성하지 못하게 되어 장애로 이어질 수 있다.

## 문제 해결하기

이 문제에 대한 명확하고 좋은 해결 방법은 잘 모르겠다. DB 마다 readOnly 동작이 다를 수 있다는 것을 알고 있는 것이 중요할 것 같다. 동시성 문제처럼

그럼에도 문제 해결방법을 찾아보면 첫 번째는 MySQL 과 같은 readOnly 옵션이 true 일 때 쓰기 동작을 하는 경우 예외를 던지는 DB 를 테스트 DB 로 사용하는 것이다. 

두 번째는 이미 운영에서 Read-Write DB 로 구축되어 있다고 한다면 , 테스트 단계에서 Read-Write DB 를 간단히 구축하고 readOnly 옵션에 따라 라우팅을 하여 Read DB 에 Command 쿼리가 전달되는지 테스트하는 것이 해결책이 될 수 있을 것 같다.

## 마무리 하며

그냥 “@Transactional(readOnly=true) 로 시작한 트랜잭션에서 @Transactional 를 사용하지 않으면 되는 것 아닌가” 라는 자세보다는 시스템적으로 가능하다면 언제든 이런 에러는 발생할 수 있다고 보고 시스템적으로 불가능하게 만드는 것이 중요하다고 생각한다.