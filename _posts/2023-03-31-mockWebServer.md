---
layout: post
date: 2023-03-31 
categories: [test]
title:  "MockWebServer 사용기"
img_path: /img/path
---


# 들어가며
---
최근에 프로젝트를 진행하면서 사용자에 대한 인증과 인가 기능을 [외부 서버](https://github.com/impati/Customer-server)로 분리하는 작업을 수행했었습니다.

덕분에 프로젝트에서 인증과 인가 기능 코드와 책임이 많이 줄었지만 **server to server 통신을 포함하는 로직테스트에 어려움이 생겼습니다.**

외부 서버와 통신하는 로직의 테스트의 어려움은 다음과 같습니다.

- 외부서버로부터 데이터를 얻어오는 경우 늘 같은 데이터를 가져온다는 보장이 없다.
- 외부서버로부터 데이터를 변경하는 경우 테스트 수행으로 외부서버의 데이터가 변경될 수 있다.
- 외부서버에 대해 테스트를 수행하기위해 인증과 인가 과정을 처리해야할 수도 있다.
- 테스트가 외부서버에 종속된다.;외부서버가 동작하지 않은 경우 작성한 테스트는 실패한다.

프로젝트에서 사용자 서버에 사용자 정보를 수정 요청을 보내고 현재 Principal 객체와 그 내용을 동기화하는 코드입니다.
```java
    public void edit(customerEditRequest customerEditRequest) {
        ...
        editCustomer(customerEditRequest); // 외부서버에 사용자 정보를 수정
        synchronizePrincipal(customerEditRequest); // 현재 인증객체와 동기화 
    }

    private void editCustomer(CustomerEditRequest customerEditRequest){
        RestTemplate restTemplate = new RestTemplate(new HttpComponentsClientHttpRequestFactory());
        RequestEntity<CustomerEditRequest> request = RequestEntity.patch(customerServer.getServer() + 사용자 정보 수정 요청 URI)
                .contentType(MediaType.APPLICATION_JSON)
                .header(HttpHeaders.AUTHORIZATION, "Bearer " + ACCESSTOKEN)
                .body(customerEditRequest);
        restTemplate.exchange(request, Void.class);
    }

    private void synchronizePrincipal(CustomerEditRequest customerEditRequest) {
        CustomerPrincipal principal = (CustomerPrincipal) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        principal.setNickname(customerEditRequest.getNickname());
        ....
    }
    
```
이때 사용자 서버에 사용자 정보를 수정하고 현재 Principal 객체가 동기화까지 테스트하는 코드를 작성하는 것은 앞서 말씀드린 이유들로 
안정적인 테스트를 작성하기 어려워 보입니다.

# 그래서 어떻게 ?
---

이러한 문제를 해결하기 위해 외부 서버를 Mocking 하여 테스트를 진행합니다.

![mockwebserver](/mockwebserver/test.png)

외부 서버를 Mocking 하여 테스트를 진행한다면 외부 서버에 의존하지 않고 테스트가 가능하다는 장점이 있습니다.

<a href="https://github.com/square/okhttp/tree/master/mockwebserver" target="_blank">MockWebServer</a> 는 외부 서버를 Mocking 하여 HTTP 및 HTTPS 호출할 시 반환할 응답 값을 지정하고 그 응답 값 이후 예상대로 진행되는지 테스트를 수행할 수 있도록 도와줍니다.



MockWebServer를 사용해서 사용자 서버에 사용자 정보가 올바르게 수정되어 정상 응답을 받았다고 Mocking하고
현재 Principal 객체 동기화가 이루어졌는지 테스트를 해보겠습니다.


우선 Gradle 종속성을 추가해보도록 하겠습니다. (Spring Boot 2.7.7 , java 11)

```
testImplementation("com.squareup.okhttp3:mockwebserver:4.10.0")
```

테스트를 위해 먼저 SecurityContext에 임의의 인증객체를 생성합니다.

```java
@BeforeEach
void setup() throws IOException {
    Authentication authentication = UsernamePasswordAuthenticationToken
            .authenticated(getCustomerPrincipal(), "Bearer AccessToken", Collections.singletonList(new SimpleGrantedAuthority("ROLE_ADMIN")));
    SecurityContext context = SecurityContextHolder.getContext();
    context.setAuthentication(authentication);

}
```

이후에는 MockMvcServer 를 생성하고 start()메서드를 사용하여 Server를 구동해줍니다.

따로 Host , Port , PATH 을 설정하지 않는다면 http://localhost와 임의의 포트를 사용합니다.

외부 서버에서 반환할 응답 값이 정상 응답인 200코드만 받으면 되기 때문에 기본값을 사용하겠습니다.



```java

@BeforeEach
void setup() throws IOException {
    mockWebServer = new MockWebServer();
    mockWebServer.start();
}

@AfterEach
void cleanup() throws IOException {
    mockWebServer.shutdown();
}

@Test
@DisplayName("customer-server 에 사용자 수정 테스트")
public void customerEditorTest() throws Exception {

    String baseUrl = mockWebServer.url("/").toString();

    // 요청 URL을 MockWebServer URL로 Mocking
    BDDMockito.given(customerServer.getUrl()).willReturn(baseUrl);

    // 기대하는 응답을 지정
    mockWebServer.enqueue(new MockResponse().setResponseCode(200));

    // 사용자 정보 수정 메서드 호출
    customerEditor.edit(new customerEditRequest(....)); 

    CustomerPrincipal principal = (CustomerPrincipal) SecurityContextHolder
                    .getContext()
                    .getAuthentication()
                    .getPrincipal();

    // principal 검증 코드
    ...
    ...

}
```

MockResponse 객체를 활용하여 보다 자세하게 기대하는 응답을 지정할 수 있습니다. 

뿐만 아니라 takeRequest() 메서드를 통해 요청이 예상대로 수행되었는지 검증할 수도 있습니다.

자세한 사항은  <a href="https://github.com/square/okhttp/tree/master/mockwebserver" target="_blank">MockWebServer</a> 참고해주세요.



# 마무리
---

MockWebServer 를 활용하여 외부 서버 기능을 이용하는 로직의 단위 테스트를 진행할 수 있었습니다.

기대되는 응답 값을 지정하고 검증 대상을 테스트하기 위해 외부 기능을 Mocking 한다는 점에서 기존에 Mocking 하여 단위 테스트를 진행하신 분들에게는 사용하기에 어렵지 않은 것 같습니다.

앞으로도 사용할 일이 많아질 것 같습니다. 그때마다 새롭게 알게 된 내용이나 다르게 포스팅한 내용이 있다면 수정하도록 하겠습니다.

감사합니다.




# Reference
---
- <a href="https://github.com/square/okhttp/tree/master/mockwebserver" target="_blank">https://github.com/square/okhttp/tree/master/mockwebserver</a>
