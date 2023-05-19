---
layout: post
date: 2023-05-19
categories: [JAVA]
title:  "객체의 수정도 빌더와 함께"
img_path: /img/path
---

# 들어가며
---

객체의 수정에 대해서는 불변성을 보장하기위해 변경할 수 있는 포인트를 열어두지 않고 항상 새로운 객체를 반환하는 것으로 객체를 수정하는 방법이 있다.

하지만 이번 포스트에서는 JPA, 변경감지(Dirty Check)를 사용하여 객체를 수정하는 ,이미 생성된 객체를 변경할 때 보다 안전하게 변경하기 위한 방법에 대해 알아보자.

> 객체를 불변으로 유지하기 위함은 변경으로 인해 의도치 않은 사이드 이펙트가 발생할 수 있고 원본 데이터가 의도와 다르게 변경 , 훼손될 수 있기 때문이다.
JPA의 Entity 는 변경이 발생해도 식별자로 구분할 수 있으며 JPA Entity 를 변경할 때 사이드 이펙트가 일어나는 것이  JPA 설계 의도이다. 물론 의도에 맞는 변경일 경우에만.


# 수정 요구사항과 API
---
먼저 다음과 같은 Services Entity 가 있다.

```java
@Entity
public class Services{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String serviceName;
    private String logoStoreName;
    private String serviceUrl;
    private String content;
    private String title;
		...

}
```

그리고 serviceName , logoStoreName ,  serviceUrl , content , title 에 대해 변경할 수 있는 API 가 존재한다.


```java
@Getter
public class ServiceEditRequest {
    private Long serviceId;
    private String logoStoreName;
    private String serviceName;
    private String serviceUrl;
    private String title;
    private String content;

}
```


```java
@PatchMapping("/api/service)"
public ResponseBody<Void> edit(@RequestBody ServiceEditRequest request){
	// TODO : 수정
	...
	...
}
```

수정API 요구사항과 주의할 점은 클라이언트에서 모든 필드에 값을 채우지 않을 수도 있다는 점이다.

즉 , 이 수정 API 를 사용할 때 serviceUrl 값을 넣지 않고 요청할 수 있다.

이럴 경우 serviceUrl 값은 변경되지 않아야한다.

# 수정하기
---
```java
@Entity
public class Services{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String serviceName;
    private String logoStoreName;
    private String serviceUrl;
    private String content;
    private String title;

    public void edit(String logoStoreName,
                    String serviceName,
                    String serviceUrl,
                    String title,
                    String content){
        if(StringUtils.hasText(logoStoreName) this.logoStoreName = logoStoreName;
        if(StringUtils.hasText(serviceName) this.serviceName = serviceName;
        if(StringUtils.hasText(serviceUrl) this.serviceUrl = serviceUrl;
        if(StringUtils.hasText(title) this.title = title;
        if(StringUtils.hasText(content) this.content = content;
    }
}
```

Services 클래스에 값이 존재하는 경우에만 변경하는 edit() 메서드를 만든다.

```java

public void edit(ServiceEditRequest request){
	// 트랜잭션 시작.
	Long serviceId = request.getId();

	Services service = findServices(serviceId);
	
	service.edit(request.getLogoStoreName(),
                request.getServiceName(),
                request.getServiceUrl(),
                request.getTitle(),
                request.getContent());
	
	// 트랜잭션 종료.
}

```

Services Entity 를 수정하기 위해서 식별자로 service 를 가져온 후 edit 메서드를 호출하고 JPA 변경 감지 기능을 이용하여 트랜잭션이 끝난 후 데이터베이스에 변경된 내용을 반영한다.


# 문제점
---
**수정하기**에서 문제점을 찾아보자.

만약 어떤 개발자가 getTitle()과 getContent() 의 순서를 바꾸어 edit () 메서드를 호출했다고 가정해보자.

```java
service.edit(request.getLogoStoreName(),
            request.getServiceName(),
            request.getServiceUrl(),
            request.getContent(),
            request.getTitle());
```
이렇게 파라미터를 잘못 넣어도 컴파일 오류가 발생하지 않으며 직접 API 를 개발한 사람도 잘못된 점을 찾지 못할 것이다. 

이런 문제가 발생할 수 있는 이유는 content 타입과 title 이 같은 String 타입이기 때문이다.

이는 치명적인 오류로 이어질 수 있으며 디버깅조차 쉽지 않다.

또 다른 문제점은 edit() 메서드를 다른 곳에서 사용하는데 serviceName 만 변경하고 싶을 수도 있다.

```java
service.edit(null,request.getServiceName(),null,"",""));
```

이럴 경우 null값 혹은 “” 값을 호출할 때 삽입해줘야한다.

serviceName 이름만 수정하는 API 를 만들면 되지만 변경 API를 여러개 만드는 것도 문제이다 .


# 빌더를 활용한 문제 해결
---

이런 문제를 해결할 수 있는 가장 좋은 방법은 builder 패턴을 이용하는 것이다.

먼저 빌더에 대해 짧게 알아보자.

## 빌더 패턴
---

빌더 패턴은 클라이언트가 필요한 객체를 직접 만드는 대신 필수 매개변수 만으로 생성자를 호출해 빌더 객체를 얻어 일종의 세터 메서드들로 원하는 선택 매개변수들을 설정하고 build() 메서드를 통해 필요한 객체를 생성하는 패턴이다.  점층적 생성자 패턴의 안정성과 가독성을 겸비하고 있는 장점이 있다.

일반적으로 빌더 패턴은 객체를 생성할 때 사용한다. 이를 살짝 틀어서 객체를 수정하는 Builder 를 만들고 

만든 Builder 로 객체를 수정하면 문제점들을 보안하면서 안전하게 객체를 수정할 수 있다.


## 빌더 적용
---

- ServiceEditor 정의

```java
@Getter
public class ServiceEditor {

    private String serviceName;
    private String logoStoreName;
    private String serviceUrl;
    private String content;
    private String title;

    public ServiceEditor(String logoStoreName,
                        String serviceName,
                        String serviceUrl,
                        String title,
                        String content) {
        this.logoStoreName = logoStoreName;
			  ...
        this.title = title;
        this.content = content;
    }

    public static ServiceEditorBuilder builder() {
        return new ServiceEditorBuilder();
    }

    public static class ServiceEditorBuilder {
        private String serviceName;
        private String logoStoreName;
        private String serviceUrl;
        private String content;
        private String title;

        ServiceEditorBuilder() {
        }

        public ServiceEditorBuilder serviceName(final serviceName serviceName) {
            if(StringUtils.hasText(serviceName) this.serviceName = serviceName;
            return this;
        }

        public ServiceEditorBuilder logoStoreName(final String logoStoreName) {
            if(StringUtils.hasText(logoStoreName) this.logoStoreName = logoStoreName;
            return this;
        }
		
        public ServiceEditorBuilder serviceUrl(final String serviceUrl) {
            if(StringUtils.hasText(serviceUrl) this.serviceUrl = serviceUrl;
            return this;
        }
        
        public ServiceEditorBuilder content(final String content) {
            if(StringUtils.hasText(content) this.content = content;
            return this;
        }
		
        public ServiceEditorBuilder title(final String title) {
            if(StringUtils.hasText(title) this.title = title;
            return this;
        }

        public ServiceEditor build() {
            return new ServiceEditor(this.serviceName,
                                    this.logoStoreName,
                                    this.serviceUrl,
                                    this.content,
                                    this.title);
        }
    }

}
```
ServiceEditor 라는 클래스를 정의하고 ServiceEditor를 빌더 패턴으로 생성


- Services 변경

```java
@Entity
public class Services{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String serviceName;
    private String logoStoreName;
    private String serviceUrl;
    private String content;
    private String title;
		
    // 먼저 자신의 값을 채워서 빌더를 반환
    public ServiceEditor.ServiceEditorBuilder serviceEditorBuilder(){
        return new ServiceEditor.ServiceEditorBuilder()
                                .serviceName(this.serviceName)
                                .logoStoreName(this.logoStoreName)
                                .serviceUrl(this.serviceUrl)
                                .content(this.content)
                                .title(this.title);
    }
    
    public void edit(ServiceEditor editor){}
        this.logoStoreName = editor.getLogoStoreName();
        this.serviceName = editor.getServiceName();
        this.serviceUrl = editor.getServiceUrl();
        this.title = editor.getTitle();
        this.content = getContent();
    }
}

```

- 먼저 Services 클래스에서 Services를 수정할 수 있는 빌더를 자신의 값으로 채워 반환
- ServiceEditor 로 자신의 값을 수정


```java
public void edit(ServiceEditRequest request){
    
    // 트랜잭션 시작.
    Long serviceId = request.getId();
    Services service = findServices(serviceId);
    
    ServiceEditor serviceEditor = service.serviceEditorBuilder()
                                .serviceName(request.getServiceName())
                                .logoStoreName(request.getLogoStoreName())
                                .serviceUrl(request.getServiceUrl())
                                .title(request.getTitle())
                                .content(request.getContent())
                                .build();


   service.edit(serviceEditor);
   // 트랜잭션 종료.
}

```

빌더 패턴을 적용하여 수정한 이 방법은 이제 title 과 cotent 값을 실수로 바꾸어 전달하는 실수를 사전에 예방할 수 있고 다른 곳에서 serviceName 이름만 수정하고 싶다고 해도 **serviceName(newServiceName)** 만 사용하면 안전하고 직관적으로 변경할 수 있다.


# 마무리하며
---
빌더 패턴을 사용하는 이유는 객체의 필드 값이 많고 ,점층적으로 생성하는 경우가 많으며 생성하고자 하는 필드 값의 타입이 같은 경우 생성할 때 대표적으로 앞에서 살펴 본 여러 문제들이 있기 때문이다.  

이는 비단 객체를 생성할 때 뿐만아니라 객체를 수정해야할때도 같은 문제가 발생한다는 것을 알아보았고

이를 결국 빌더 패턴으로 이 문제를 해결할 수 있다는 것도 알아보았다.