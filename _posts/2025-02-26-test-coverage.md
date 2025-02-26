---
title: 테스트 커버리지 측정 jacoco kover  
date: 2025-02-26 12:00:00 +0900
categories: [Framework]
math: true
mermaid: true
---

### 개요
- kotlin + spring boot + gradle 프로젝트에서 테스트 커버리지를 측정해보자

--- 

### 사용법 
- gradle 설정 

```groovy
//플러그인
id 'org.jetbrains.kotlinx.kover' version '0.9.1' // kover
id 'jacoco' // jacoco

//https://kotlin.github.io/kotlinx-kover/gradle-plugin/
//koverHtmlReport, koverVerify 실행하여 결과 확인
kover {
  reports {
    verify {
      rule {
        minBound(10)
      }
    }
    filters {
      includes {
        classes("groot.hexagonal.application.service.*", "groot.hexagonal.adapter.*")
      }
    }
  }
}

jacoco {
  toolVersion = "0.8.8"
}

// 필터 선언
def jacocoFileFilter = {
  fileTree(dir: it,
    includes: ['groot/hexagonal/application/service/**'], // 특정 패키지만 포함
    excludes: [
      '**/config/**',
      '**/dto/**'
    ]
  )
}

// 테스트 커버리지 리포트 설정
jacocoTestReport {
  reports {
    xml.required = true
    csv.required = false
    html.required = true
    // build/jacocoHtml 에 리포트 생성
    html.outputLocation = layout.buildDirectory.dir('jacocoHtml')
  }

  // 필터 적용
  afterEvaluate {
    classDirectories.setFrom(files(classDirectories.files.collect(jacocoFileFilter)))
  }

}

// 메소드 레벨 커버리지를 verify 위한 설정
jacocoTestCoverageVerification {
  // 필터 적용
  afterEvaluate {
    classDirectories.setFrom(files(classDirectories.files.collect(jacocoFileFilter)))
  }

  violationRules {
    rule {
      // 메소드 단위 커버리지 설정
      limit {
        counter = 'METHOD'
        value = 'COVEREDRATIO'
        minimum = 0.8 // 80% 커버리지 요구
      }
    }
  }
}

tasks.named('test') {
  useJUnitPlatform()

  // 테스트 수행 결과가 있어야 jacocoTestReport, jacocoTestCoverageVerification 실행 가능
  finalizedBy jacocoTestReport
  finalizedBy jacocoTestCoverageVerification

  // koverHtmlReport, koverVerify 실행하여 결과 확인
  finalizedBy koverHtmlReport
  finalizedBy koverVerify
}

``` 

- gradle test 를 수행하고 report 를 생성하거나 verify 를 실행하여 커버리지 조건에 맞는지 확인한다 

--- 

### 회고 
- kover 는 line 단위로 커버리지를 측정하고 jacoco 는 메소드 단위로 커버리지를 측정했다 
  - kover 로는 메소드 단위 커버리지를 측정하는 옵션을 찾지 못했다  
- 옵션은 jacoco 가 더 많아 보인다 SonarQube, Jenkins 와의 통합 측면에서도 jacoco 가 더 많은 기회를 제공한다   
- 반면 kover 는 코틀린에 최적화 되어 inline 함수, default 파라메터를 사용하는 함수에 대한 정확한 커버리지 측정이 가능하다 

