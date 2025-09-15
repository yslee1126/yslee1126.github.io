---
title: Spring Rest Docs 사용   
date: 2025-09-15 17:00:00 +0900
categories: [library]
math: true
mermaid: true
---

### 개요 
- 내부 개발자 공유 문서로 스웨거를 사용중이였으나 외부 업체로 일부 API 스펙을 공유하는 케이스가 생김
- 외부용 API 프로젝트를 분리하기는 어려운 상황이라고 했을때 Spring Rest Docs 로 공유할 문서 생성 가능 

### 설정 및 문서 생성 

- 빌드 설정 

```groovy

plugins {
  id "org.asciidoctor.jvm.convert" version "3.3.2"
}

configurations {
  asciidoctorExt
}

dependencies {
  asciidoctorExt 'org.springframework.restdocs:spring-restdocs-asciidoctor'
  testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc'
}

ext {
  snippetsDir = file('build/generated-snippets')
}

test {
  outputs.dir snippetsDir
  useJUnitPlatform() 
}

asciidoctor {
  inputs.dir snippetsDir
  configurations 'asciidoctorExt'
  dependsOn test
  outputDir file('build/docs/asciidoc')
  sourceDir file('src/docs/asciidoc')
}

// 웹 노출은 스웨거로 하므로 아래 설정은 필요없음 
//tasks.register('copyAsciidoc', Copy) {
//    dependsOn asciidoctor
//    from "${asciidoctor.outputDir}"
//    into "build/resources/main/static/docs"
//}
//tasks.named('bootJar') {
//    dependsOn copyAsciidoc
//}


```

- 컨트롤러 테스트 코드에 document() 사용 
- 요청/응답에 대한 구성 방법은 레퍼런스를 참조 
  - https://docs.spring.io/spring-restdocs/docs/current/reference/htmlsingle
- 테스트 코드를 실행하면 문서 경로 아래에 API 메타 정보가 생성됨 
  - build/generated-snippets/문서경로 

```java


@AutoConfigureRestDocs
@WebMvcTest(Controller.class)
@ExtendWith(RestDocumentationExtension.class)
class ControllerTest {

  @Autowired
  private MockMvc mockMvc;

  @Test
  void test() throws Exception {
    mockMvc.perform(MockMvcRequestBuilders.get(url)
        .contentType(MediaType.APPLICATION_JSON)
        .accept(MediaType.APPLICATION_JSON)
        .header("x-api-key", apiToken))
      .andExpect(status().isOk())
      .andDo(print())
      .andDo(document("문서경로", // 문서 경로는 유니크 해야한다. 이 API 에 대한 메타정보가 생성되는 경로이다.
        preprocessRequest(modifyUris()
          .scheme("https")
          .host("API 서버 호스트 주소")
          .port(-1)
        ),
        requestHeaders(
          headerWithName("x-api-key").description("API 인증을 위한 JWT 토큰")
        ),
        queryParameters(
          parameterWithName("param1").description("파라메터 1 설명"),
          parameterWithName("param2").description("파라메터 2 설명")
        ),
        responseFields(
          fieldWithPath("count").description("전체 카운트"),
          fieldWithPath("data").description("데이터 목록"),
          fieldWithPath("data[].id").description("ID"),
          fieldWithPath("data[].title").description("제목")
        )
      ));

  }
  
}

```

- 생성된 API 메타 정보를 바탕으로 API Doc 의 내용을 구성하기 위한 adoc 파일 제작한다  
  - adoc 파일은 src/docs/asciidoc 아래 생성합니다.

```

= API 문서
:revnumber: 1.0.0
:doctype: book
:icons: font
:favicon: images/favicon.ico
:source-highlighter: highlightjs
:toc: left
:toclevels: 2
:sectlinks:
:operation-curl-request-title: 요청 예시
:operation-http-response-title: 응답 예시

== 소개

소개글 

== API 이름 

=== API 제목 

==== 요청

include::{snippets}/문서경로/http-request.adoc[]

==== 요청 헤더

include::{snippets}/문서경로/request-headers.adoc[]

==== 요청 파라미터

include::{snippets}/문서경로/query-parameters.adoc[]

==== 응답

include::{snippets}/문서경로/http-response.adoc[]

==== 응답 필드

include::{snippets}/문서경로/response-fields.adoc[]

==== CURL 요청 예시

include::{snippets}/문서경로/curl-request.adoc[]


```

- 이제 설정은 완료되어 문서 생성 방법은 아래와 같다 
- 문서는 build/docs/asciidoc 아래 생성된다 

```shell
./gradlew asciidoctor
```

### 회고 
- 문서가 테스트 코드에 integration 되니 문서 현행화가 자연스럽게 될 수 있을 것 같다
- 스웨거의 try it 기능이 없는게 약간 아쉽다 
- Swagger 는 내부용, Rest Docs 는 외부용으로 유지 할 생각이다

