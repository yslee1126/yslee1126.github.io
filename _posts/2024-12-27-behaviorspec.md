---
title: BDD BehaviorSpec
date: 2024-12-27 10:00:00 +0900
categories: [Library]
math: true
mermaid: true
---

### 개요 
- BDD 스타일 테스트를 작성할 수 있게 해주는 테스팅 프레임워크 kotest 를 사용해본다 

--- 
### 의존성 

```
testImplementation 'io.kotest:kotest-runner-junit5:5.9.1'
testImplementation 'io.kotest:kotest-assertions-core:5.9.1'
testImplementation 'io.mockk:mockk:1.13.14'
```

--- 
### 코드 
```kotlin
class MemberServiceTest: BehaviorSpec ({

    val memberQueryPort = mockk<MemberQueryPort>()
    val memberService = MemberService(memberQueryPort)

    Given("a request to get members") {
        val name = "John"
        val size = 10
        val page = 1
        val expectedMembers = listOf(Member(1, "John Doe"), Member(2, "Jane Doe"))

        every { memberQueryPort.getMembers(name, size, page) } returns expectedMembers

        When("the service is called to get members") {
            val result = memberService.getMembers(name, size, page)

            Then("it should return the expected list of members") {
                result shouldBe expectedMembers
            }

            Then("it should call the memberQueryPort with correct parameters") {
                verify { memberQueryPort.getMembers(name, size, page) }
            }
        }
    }

})


```

--- 
### 회고 
- junit 보다 가독성도 좋고 구조가 명확해짐 
