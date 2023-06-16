---
layout: post
date: 2023-04-30 
categories: [test,spring]
title:  "테스트와 문서화 한꺼번에 하기 위한 REST Docs 설정기"
img_path: /img/path
---

# 들어가며
---
프론트 개발자분들과 협업을 진행할 때면 CORS 문제와 API 스펙 문서로 인해 통신 장애가 자주 발생한 경험이 많습니다.
이러한 문제의 책임은 백엔드 개발자에게 있다고 생각하고
API 문서를 오타 없이 작성하면서 API 가 제대로 동작하는지 테스트하는 과정은 굉장히 중요하다고 생각합니다.
나아가 기능 개발이 추가되고 변경되어도 반드시 문서를 업데이트 해야하는 과정도 동반되어야합니다.
API 문서를 생성해주는 라이브러리는 많지만 테스트가 성공해야 문서가 생성되는 매력적인 기술에 대해 알아보고 프로젝트에 도입하기 위한
설정기를 소개하겠습니다

# Spring REST Docs 란?
---

> **Spring REST Docs의 목표는 RESTful 서비스에 대한 정확하고 읽기 쉬운 문서를 생성하도록 돕는 것입니다.**
> 
> Spring REST Docs 는 Spring MVC 의 테스트 프레임워크 , REST Assured  , WebTestClinet 로 작성된 테스트에서 생성된 스니펫을 사용합니다.
> 테스트 기반 접근 방식은 문서의 정확성과 신뢰성을 보장하는데 도움이 됩니다. 올바르지 않으면 실패합니다

# Spring REST Docs 를 사용하는 이유
---

- **문서의 정확성과 신뢰성을 보장하는데 도움이 된다.**
  
  테스트를 기반으로 문서를 생성하기 때문에 문서를 신뢰할 수 있다.
    
- **테스트와 문서화를 동시에**
   문서 작업을 따로 할 필요가 없다는 장점이 있다.

- **문서화를 위한 main 코드에 추가적인 코드가 필요없다.**


# 빌드 구성
---

> java 11
> 
> Spring Boot 2.7.9
> 
> Gradle 7.6.1
> 
> spring-web,Lombok

1. Asciidoctor 플러그인 적용
```gradle
plugins { 
	id "org.asciidoctor.jvm.convert" version "3.3.2"
}
```
2. Asciidoctor를 확장하는 종속성에 대한 구성을 선언합니다 
```gradle
configurations {
	asciidoctorExt 
}
```
3. 의존성 추가
```gradle
asciidoctorExt 'org.springframework.restdocs:spring-restdocs-asciidoctor'
testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc'
// spring-restdocs-webtestclientspring-restdocs-restassured  --> REST Assured 경우
```
4. 생성된 스니펫의 출력 위치를 정의하도록 속성을 구성
```gradle
ext { 
	snippetsDir = file('build/generated-snippets')
}
```
5. 테스트 후 생성된 스니펫 디렉토리를 snippetsDir 에 출력으로 구성
```gradle
test { 
	outputs.dir snippetsDir
}
```
6. 문서가 생성되기전에 테스트가 실행되도록 구성하고 스니펫 디렉토리를 입력으로 구성
```gradle
asciidoctor { 
	inputs.dir snippetsDir 
	configurations 'asciidoctorExt' 
	dependsOn test 
}
```

**전체 빌드 구성**
```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.7.11'
    id 'io.spring.dependency-management' version '1.0.15.RELEASE'
    id "org.asciidoctor.jvm.convert" version "3.3.2"
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

configurations {
    asciidoctorExt
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    asciidoctorExt 'org.springframework.restdocs:spring-restdocs-asciidoctor'
    testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

ext {
    snippetsDir = file('build/generated-snippets')
}

test {
    outputs.dir snippetsDir
}

asciidoctor {
    inputs.dir snippetsDir
    configurations 'asciidoctorExt'
    dependsOn test
}

tasks.named('test') {
    useJUnitPlatform()
}
```


# 테스트 설정
---
```java
@RestController
@RequestMapping("/api/v1/docs")
public class RestDocsController {

    @GetMapping
    public ResponseEntity<Response> exampleGetMapping(){
        return ResponseEntity.ok(Response.of("OK"));
    }
		
    @Getter
    @NoArgsConstructor
    @AllArgsConstructor
    static class Response{
        private String status;
        
        static Response of(String status){
            return new Response(status);
        }
    }
}
```
- GET 방식으로 /api/v1/docs 요청 시  `{"status": "OK"}` 을 응답하는 컨트롤러입니다.

```java
import static org.springframework.restdocs.mockmvc.MockMvcRestDocumentation.document;
import static org.springframework.restdocs.mockmvc.RestDocumentationRequestBuilders.get;
import static org.springframework.restdocs.operation.preprocess.Preprocessors.preprocessRequest;
import static org.springframework.restdocs.operation.preprocess.Preprocessors.preprocessResponse;
import static org.springframework.restdocs.payload.PayloadDocumentation.fieldWithPath;
import static org.springframework.restdocs.payload.PayloadDocumentation.responseFields;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(RestDocsController.class) // 테스트 대상 컨트롤러
@AutoConfigureRestDocs(uriHost = "api.test.com", uriPort = 80) // <1> RestDocs 자동 구성
@ExtendWith(RestDocumentationExtension.class) //  <2> 추가.
class RestDocsControllerTest {

    @Autowired
    private MockMvc mockMvc; // mockMvc 자동 주입

    @Test
    @DisplayName("[GET] [/api/v1/docs] GET Mapping 테스트")
    public void exampleGetMappingTest() throws Exception{

        mockMvc.perform(get("/api/v1/docs")) // 요청 /api/v1/docs
                .andExpect(status().isOk()) // 상태코드
                .andExpect(handler().methodName("exampleGetMapping")) // 핸들러 메서드이름
                .andExpect(jsonPath("$.status").value("OK")) // 기대 응답
                .andDo(document(
                        "test/", // <3> build/generated-snippets 하위 디렉토리에 스니펫 생성
                        preprocessRequest(Preprocessors.prettyPrint()),//<4> json 예쁘게 출력
                        preprocessResponse(Preprocessors.prettyPrint()), //<5> json 예쁘게 출력
                        responseFields(
                                fieldWithPath("status").description("응답 상태") //  <6>key 값과 fieldWithPath 에 대한 설명을 명시합니다
                        )
                ));
    }
}

```
- <1> @AutoConfigureRestDocs를 활용한 Rest Docs 자동 구성을 활성화합니다. \
  옵션으로 URI , 스키마 , 포트 등 구성할 수 있으며 추가 구성이 필요한 경우 `RestDocsMockMvcConfigurationCustomizer` 클래스를 사용하면 됩니다.

- <2> RestDocumentationContext 를 자동으로 관리하는데 사용되는 Extenstion을 추가합니다.

- <3> build/generated-snippets/test 하위 디렉토리에 스니펫 생성합니다. 
  
- <4> requestBody 의 json 예쁘게 출력
  
- <5> responseBody 의 json 예쁘게 출력
  
- <6> responseBody 에서 fieldWithPath()에 키명과 이에 대한 설명을 명시합니다.

- MockMvc 사용하면서 요청마다 스펙에 맞는 andDo(document(…..)) 에 설정해주는 것으로 테스트는 완료됩니다.
  
- 테스트가 성공하면 build/generated-snippets/test 에 스니펫이 생성됩니다.
  
    ![snippets](/restdocs/adoc.png)

    - curl-request.adoc : curl 명령어
    - `http-request.adoc : http-request 스펙 문서`
    - `http-response.adoc  : http-response 스펙 문서`
    - httpie-request.adoc  : httpie 명령어
    - request-body.adoc : 요청 바디
    - response-body.adoc  : 응답 바디
    - `response-fields.adoc : 응답 필드문서`

-  andDo(document(…..)) 에 따라 생성된 스니펫이 추가될 수 있습니다.



# 스니펫 사용
---
생성된 스니펫을 사용하기 전에 소스파일을 먼저 생성해야합니다.

Gradle 의 경우 src/docs/asciidoc/ 하위에 `.adoc` 소스파일을 생성하고 생성된 스니펫을 포함합니다.


```
[/src/docs/asciidoc/test.adoc]

=== 요청

include::{snippets}/test/http-request.adoc[]


=== 응답

include::{snippets}/test/http-response.adoc[]
include::{snippets}/test/response-fields.adoc[]

```

asciidoctor 작업 후 생성된 HTML 파일을  src/main/resources/static/docs 복사하는 구성을 추가해줍니다.
```gradle
bootJar {
    dependsOn asciidoctor
    copy {
        from asciidoctor.outputDir
        into "src/main/resources/static/docs"
    }
}
```

- ./gradlew bootJar 를 실행하여 /build/asciidoc/test.html 파일이 생성되었는지 확인합니다.
- 다시 ./gradlew bootJar 실행하면 src/main/resources/static/docs 하위에 test.html 파일이 생성되었음을 확인할 수 있습니다.

- 어플리케이션을 실행하고 ${HOST}/docs/test.html 에 접속해줍니다.

![example](/restdocs/example.png)

# 마무리하며
---
이렇게 해서 무사히 설정을 완료했고 간단한 예제를 통해 응답 화면을 볼 수 있었습니다. \
기존에 노션을 활용해서 문서화를 했을 때 단순한 오타나 시간에 따라 반영되지 않은 요소들이 많았는데 \
Spring REST Docs 를 잘 활용하면 이러한 문제는 해결될 것이라고 생각합니다.

다음에는 협업 프로젝트에 Spring REST Docs 를 적용한 예제를 작성해보겠습니다.


# Reference
---

- <a href="https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/#documenting-your-api" target="_blank">https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/#documenting-your-api</a>