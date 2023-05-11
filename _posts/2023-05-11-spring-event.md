---
layout: post
date: 2023-05-11 
categories: [spring]
title:  "스프링 이벤트를 활용하여 관심사 분리하기"
img_path: /img/path
---


# 들어가며 
---
최근에 팀원분들이랑 진행하고 있는 여행 관련 프로젝트에서 작성한 게시글에 좋아요  혹은 댓글 등의 눌리거나 달렸을 때  게시글 작성자에게 피드 형식의 알람을 구현하는 역할을 맡았다.

알람이 생성될 수 있는 곳은 여러 군데가 가능하므로 확장 가능성을 열어두어야하고 알람 기능을 구현함에 있어 기존 기능이 받는 영향을 최소로 해야한다.


# 어떻게 구현하지?
---
가장 간단하게 생각해보면

알람 Entity 를 정의하고 댓글을 작성하고 난 후 알람 Entity 를 생성해 영속 상태로 만들어버리면 된다.

```java
@Transactional
public CommentResponse createComment(Long writerId, Long articleId, String content) {
    Member member = findMember(writerId);
    Article article = findArticle(articleId);

    Comment comment = Comment.builder()
            .content(content)
            .article(article)
            .writer(member)
            .build();

    ArticleComment articleComment = commentRepository.save(comment);

	Alarm alarm = createAlarm(...); // 알람 생성.
	alarmRepository.save(alarm); // 알람을 데이터베이스에 저장.

     return CommentResponse.of(articleComment, true);
}
```

# 문제점 발견
---

댓글을 작성하고 알람 Entity를 생성하고 저장하는 위 코드의 문제점은 createComment라는

댓글을 생성하는 메서드라고 사용했더니 알람까지 생성하고 저장하고 있다는 점이다.

뿐만 아니라 댓글을 저장하는 Transaction 안에서 알람을 생성하는데 실패했다고 가정해 보자.

알람 Entity를 생성하기 위한 부가 기능 때문에 주 기능인 댓글을 생성하는데 실패한다.

이 문제를 해결하기 위해서는 댓글을 작성하는 기능과 알람 Entity 을 생성하는 기능을 각각 구현한 후 누군가 이 두 기능을 차례로 호출해 주면 된다.

![example](/event/class.png)

이름으로 그 기능을 모두 보일 수 있을려면 메서드 이름은 createCommentAndAlarm() 이라고 하는게 좋겠다.
…
하지만 아무리 보아도 댓글을 생성하는 기능이 알람 기능 때문에 어떤식으로든 영향을 받는 것 같은데 더 좋은 방법없을까?


# 이벤트를 활용한 문제 해결
---
![example](/event/event.png)
댓글을 생성한 후 특정 이벤트를 발생시키고 알람 Entity 를 생성하는 기능이 이 이벤트를 듣고 호출되는 방법으로 구현한다면 댓글을 작성하는 기능과 알람을 생성하는 기능이 분리하여 관심사를 분리할 수 있다.

```java
@Transactional
public CommentResponse createComment(Long writerId, Long articleId, String content) {
    Member member = findMember(writerId);
    Article article = findArticle(articleId);

    Comment comment = Comment.builder()
            .content(content)
            .article(article)
            .writer(member)
            .build();

    ArticleComment articleComment = commentRepository.save(comment);

    // 이벤트 생성
    eventPublisher.publishEvent(new AlarmEvent(..));

    return CommentResponse.of(articleComment, true);
}
```

하지만 그럼에도 여전히 메서드 이름만으로 댓글을 저장하면서 이벤트를 발행하는지 알 수 없으며

이벤트를 생성하는데 실패한다고 가정했을 때에도 댓글이 생성되지 못하는 문제가 남아있다.

뿐만 아니라 알람이 생성될 수 있는 기능은 게시글에 좋아요 기능이라든지 프로젝트 내에 여러 군데가 될 수 있다.


# AOP의 도움
---
댓글이 생성되고 난 후 이벤트를 발행한다면 문제를 모두 해결할 수 있지 않을까?

기존의 기능들을 최대한 변경하지 않는 선에서 알람 생성이 필요한 영역에 알람 생성 기능을 도입하기 위해서 AOP를 사용해 보자.
```java
@Slf4j
@Aspect
@Component
@RequiredArgsConstructor
public class AlarmPublisher {

    private final ApplicationEventPublisher eventPublisher;

    @AfterReturning(value = "@annotation(....Alarm)", returning = "result")
    public void doReturning(JoinPoint joinPoint, Object result) {
        AlarmArgs alarmArgs = typeConvert(result);
				
				// 이벤트 발행
        eventPublisher.publishEvent(new AlarmEvent(joinPoint.getTarget(), alarmArgs.id, alarmArgs.alarmType));
    }

    private AlarmArgs typeConvert(Object result) {
        if (result instanceof CommentResponse) {
            CommentResponse comment = (CommentResponse) result;
            return new AlarmArgs(comment.getCommentId(), AlarmType.COMMENT);
        } else if (result instanceof LikeResponse) {
            LikeResponse like = (LikeResponse) result;
            return new AlarmArgs(like.getLikeId(), AlarmType.LIKE);
        }
        throw new IllegalArgumentException("지원하지 않는 알람 기능 입니다.");
    }

    static class AlarmArgs {
        private Long id;
        private AlarmType alarmType;

        public AlarmArgs(Long id, AlarmType alarmType) {
            this.id = id;
            this.alarmType = alarmType;
        }
    }

}
```
알람 생성 기능이 사용된다는 것을 명시하기 위해 어 로테이션을 활용한 어드바이저를 구현한다.

`@AfterReturning` 을 사용해서 @Alarm 어 로테이션이 붙은 기능이 성공적으로 리턴 한 경우에 실행해 주며

결괏값을 받아서 적절한 이벤트를 생성해 준다.

하지만 AlarmEvent 스펙에 맞추기 위해 이 방법도 반환 타입을 직접 변환해 주는 작업이 동반된다는 점에서 한계가 존재한다.


```java
@Alarm
@Transactional
public CommentResponse createComment(Long writerId, Long articleId, String content) {
    Member member = findMember(writerId);
    Article article = findArticle(articleId);
    Comment comment = Comment.builder()
            .content(content)
            .article(article)
            .writer(member)
            .build();
    ArticleComment articleComment = commentRepository.save(comment);
    return CommentResponse.of(articleComment, true);
}
    
```

이제는 알람을 생성하는 기능 때문에 댓글을 생성하는 작업이 실패하는 경우는 발생하지 않는다.

createComment() 메서드만 보고 알람을 생성하는지를 알 수 없지만 @Alarm 어 로테이션을 사용해서 이 의도를 전달하고 있다. 그뿐만 아니라 이제는 다른 기능들에서 알람 생성 기능을 사용하고 싶다면

@Alarm 어 로테이션을 사용해서 간단하게 적용할 수 있다.

~~반환값을 지정해야 하는 등, TypeConvert 작업을 동반해야 하지만~~


# 마무리하며
---

AOP 와 이벤트를 활용함으로써 알람이 생성될 수 있는 기능에 대한 확장 가능성을 열어두었고

알람 기능을 구현함에 있어 기존의 댓글이나 좋아요 기능에서 받는 영향을 @Alarm 어노테이션만을 붙임으로써 최소한으로 했다.

이러한 방법이 아직 경험이 많이 없는 입장에서 득이 될지 해가 될지 모르겠지만 직접 겪어보며 깨닫는 것이 추후에 도움이 되지 않을까

# Reference

- <a href="https://www.baeldung.com/spring-events" target="_blank">https://www.baeldung.com/spring-events</a>