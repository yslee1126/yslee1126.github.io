---
title: 코루틴 맛보기 
date: 2025-02-21 10:00:00 +0900
categories: [Library]
math: true
mermaid: true
---

### 개요
- spring boot 에서 코루틴을 사용해보자
- 프로젝트 구조는 hexagonal architecture 를 사용한다
- 비동기로 동작하는 REST API 를 구현하는데 다른 API 를 호출하고 결과를 합쳐서 반환하는 예제를 만들어본다

### 특징 
- 쓰레드보다 가벼운 비동기 처리 방식 제공 
- 가독성 향상 
- 대규모 동시요청 데이터 스트리밍하고 할때는 이벤트루프 기반의 WebFlux 가 성능에 유리할 수 있다 
  - 많은 쓰레드가 필요한 경우 컨텍스트 스위칭 비용은 Coroutine 이 더 많이 필요할 수 있다 

---

### 사용방법 

- 의존성 추가 
```groovy
    // WebFlux => 리액티스 스트림 기반 비동기 동작, I/O 작업이 극단적으로 많은 경우 유리
    implementation 'org.springframework.boot:spring-boot-starter-webflux'

    // 코루틴 => 경량화된 스레드, 낮은 컨텍스트 스위칭, 간결한 코드, CPU 작업이 많은 경우 유리
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.6.3'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-reactor:1.6.3'
```

- 컨트롤러

```kotlin
    @GetMapping("/articles")
    suspend fun getArticles() : ResponseEntity<ArticleListResponse> {
        val articles = articleUseCase.getArticles()
        val articleResponses = articles.map { article -> ArticleResponse(article.title, article.content) }
        return ResponseEntity.ok(ArticleListResponse(articleResponses))
    }
```

- 서비스 
  - 두가지 정보를 조회하여 List<Article> 형태로 합친다

```kotlin
@Service
class ArticleService(private val articleApiPort: ArticleApiPort): ArticleUseCase {

    private val log = LoggerFactory.getLogger(this.javaClass)!!

    override suspend fun getArticles() : List<Article> {
        return coroutineScope {
            val startTime = System.currentTimeMillis()
            log.debug("Starting parallel article fetching")

            val moArticlesDeferred = async {
                log.debug("Starting Mo articles fetch")
                val moStartTime = System.currentTimeMillis()
                articleApiPort.getMoArticles().also { articles ->
                    val duration = System.currentTimeMillis() - moStartTime
                    log.debug("Completed Mo articles fetch in {}ms, got {} articles", duration, articles.size)
                }
            }

            val liArticlesDeferred = async {
                log.debug("Starting Li articles fetch")
                val liStartTime = System.currentTimeMillis()
                articleApiPort.getLiArticles().also { articles ->
                    val duration = System.currentTimeMillis() - liStartTime
                    log.debug("Completed Li articles fetch in {}ms, got {} articles", duration, articles.size)
                }
            }

            val moArticles = moArticlesDeferred.await()
            val liArticles = liArticlesDeferred.await()

            val totalDuration = System.currentTimeMillis() - startTime
            log.debug("Completed all article fetching in {}ms, total articles: {}",
                totalDuration, moArticles.size + liArticles.size)

            moArticles + liArticles
        }
    }
}
```

- 어댑터 
  - WebClient 를 사용하여 비동기로 외부 API 를 호출한다 

```kotlin
@Component
class ArticleAdapter(private val webClient: WebClient): ArticleApiPort {
    override suspend fun getMoArticles(): List<Article> {
        val moUrl = "test api 1"

        return webClient.get()
            .uri(moUrl)
            .retrieve()
            .awaitBody<MoResponse>()
            ._DATA
            .map { moArticle ->
                Article(
                    title = moArticle.ART_TITLE,
                    content = moArticle.ART_SUMMARY
                )
            }
    }

    override suspend fun getLiArticles(): List<Article> {
        val liUrl = "test api 2"

        return webClient.get()
            .uri(liUrl)
            .retrieve()
            .awaitBody<LiResponse>()
            .data
            .map { liArticle ->
                Article(
                    title = liArticle.artTitle,
                    content = liArticle.artSummary
                )
            }
    }
}
```

---

### 회고
- WebFlux 를 사용했을때와 비교하여 콜백체인을 사용할 일이 없고 코드가 절차적으로 작성되어 가독성이 많이 좋아졌다 
  - 예외 처리를 할때도 이같은 장점이 더 두드러질 수 있다고 생각된다
- 반면 백프레셔를 처리할때 추가 작업이 필요하여 WebFlux 보다 번거로움이 생긴다 
- 예제에서 외부 API 를 호출하는 부분을 코루틴으로 처리하니 코드가 깔끔해졌다 하지만... 
  - 이처럼 i/o-bound 한 작업에는 WebFlux 가 성능에서 유리하고 
  - cpu-bound 한 작업에는 코루틴이 성능에서 유리하다
