---
title: 아토믹 Kotlin 
date: 2025-04-23 17:00:00 +0900
categories: [Kotlin]
math: true
mermaid: true
---

### 1.basic 
- 왜 코틀린인가 
  - 가독성, ide
  - 명령형, 함수형, 객체지향
  - 멀티플랫폼 지원 
    - jvm, android, javascript, native binary 
- 초기화 
  - var n: Int = 1
- 수타입
  - 정수는 가독성을 위해 언더바 표시 가능 1_000_000
- in 키워드 
  - 범위내에서 해당되는 값이 있다면 true 반환 
  - 10 in 5..25 는 true 

### 2.객체    
- 명사(객체)와 동사(함수) 모두에 초점을 맞추어 작성될 수 있다 
- 객체를 출력해보면 클래스명@메모리주소 형태로 표현됨 
- 리스트
  - listOf 로 읽기전용 리스트 초기화 
  - mutableListOf 를 사용하면 가변 리스트 초기화 
  - var 로 선언된 리스트에 += 연산자를 쓰면 불변 리스트에 값을 더하여 새로운 리스트를 만들어 낸다 
- 가변인자목록 
  - vararg 를 사용하면 함수에서 가변 인자를 받을수 있다   
  - varage 를 인자로 받아서 다른 함수에 그대로 넘길수 있다 스프레드 연산자 * 를 붙혀서 전달한다 
- 집합 
  - val intSet = setOf(1,2,3,9,9,4)
  - (9 in intSet) eq false 
  - contains, union, distinct 등 가능 
  - 중복 제거됨, 순서 없고 같은 원소 있으면 같은 집합 
  - list 와 마찬가지로 +=, -= 연산자로 추가, 삭제 가능 
- 맵
  - key, value 쌍으로 Map 생성 가능 하다 mapOf() 사용  
  - 일반 맵은 읽기전용이고 수정하려면 mutableMapOf() 사용 
  - map['key값'] 으로 값이 없으면 Null 반환 
  - map.getValue('key값') 으로 값이 없으면 NoSearchElementException 발생 
- 프로퍼티 접근자 
  - getter, setter 따로 정의 안해도 기본적으로 접근 가능하나 커스텀 가능 
  - get, set 을 각각 private 하게 재정의 가능 

### 3.사용성 
- 확장함수 
  - 기존클래스에 멤버 함수 추가 가능 
```kotlin
fun String.singleQuote() = "'$this'"
fun main() {
  "hi".singleQuote() eq "'hi'"
}
```
- 다른 패키지에서 사용하려면 import extentisonFunctions.singleQuote 형태로 선언 
- 확장함수 안에서 this 사용가능 
- 이름붙은 인자와 디폴트 인자 
```kotlin
func color(red: Int = 0, green: Int = 0, blue: Int = 0) {
  println("red:$red, green:$green, blue:$blue")
}
color(blue = 255) eq "red:0, green:0, blue:255"
```
- 오버로딩 
```kotlin
class My{
  fun foo() = 0
}

fun My.foo(i: int) = i + 2

fun main() {
  val my = My()
  my.foo() eq 0
  my.foo(1) eq 3
}
```
- when 
```kotlin
fun ordinal(i: Int) = when(i) {
  1 -> "first"
  2 -> "second"
  3 -> "third"
  else -> "unknown"
}
```
- enum 
  - values, toList 사용가능
- data class 
  - toString() 자동 생성
  - equals() 자동 생성하여 동등 == 연산 
  - copy() 자동 생성
- 구조 분해 (destructuring)
  - data 클래스를 이용하여 하나 이상의 정보를 리턴하도록 정의 할 수 있다
  - data 클래스를 사용하지 않고 구조분해 하려면 Pair나 Triple 클래스를 사용해야 한다
```kotlin
data class Computation(val k: Int, val v: String)

fun evaluate(input: Int) =
  if (input > 5) {
    Computation(input, "big")
  } else {
    Computation(input, "small")
  }

fun main() {
  val (key, value) = evaluate(10)
  value eq "big"
}
```
- 널이 될 수 있는 타입 
```kotlin
val s: String? = null
s?.echo() // s 가 null 이므로 아무일도 하지 않는다 
(s ?: "---") eq "---" // 엘비스(elvis) 연산자를 이용해 s 가 null 이므로 --- 을 출력한다 
s.isNullOrEmpty() eq true 
s.isNullOrBlank() eq true 
```
  - String 과 String? 는 타입이 다르다 
  - .? 사용하면 null safe 한 호출 
  - 반면에 !! 사용하여 null 이면 NullPointerException 발생시킬 수 있다 
- generics 
```kotlin
class Box<T>(val value: T) {
  fun getValue(): T = value
}
// 유니버셜 타입 (모든 타입의 부모 타입)인 any keyword 사용 
class Box(private val value: Any) {
  fun getValue(): Any = value
}
// generic function 
fun <T> List<T>.first(): T {
  if (this.isEmpty()) throw NoSuchElementException("List is empty")
  return this[0]
}
```
### 4.함수형 프로그래밍
- 람다 
```kotlin
val list = listOf(1, 2, 3, 4, 5)
val result = list.map({ n -> "[$n]"})
result eq "[1], [2], [3], [4], [5]"
val even = list.filter { it % 2 == 0 }
even eq "[2, 4]"
// 고차함수 지원, 함수를 다른 함수의 인자로 넘김 
val isPlus: (Int) -> Boolean = { it > 0 }
listOf(1, -2, 3, -4, 5).filter(isPlus) eq "[1, 3, 5]"
```
- 맵 만드는 다양한 방법 
```kotlin
data class Person(val name: String, val age: Int)
val names = listOf("Alice", "Bob", "Charlie", "David")
val ages = listOf(25, 30, 35, 25)
val people = names.zip(ages) { name, age -> Person(name, age) }
people.groupBy(Person::age) eq
  "{25=[Person(name=Alice, age=25), Person(name=David, age=25)], 30=[Person(name=Bob, age=30)], 35=[Person(name=Charlie, age=35)]}"
```
- 시퀀스 
  - java stream 과 비슷한 개념
  - 차이점은 lazy evaluation 지연 계산을 실행한다는 점이다 
  - 지연 계산은 모든 항목에 대해 체이닝된 연산을 실행하지 않고 필요한 항목에 대해서만 연산을 실행하고 조건을 충족하면 멈춘다 
  - 예를 들어 filter(), map(), any() 가 같은 연산이 연결되어 있을때 filter(), map() 에서 중간 연산 결과를 저장하여 처리하고 최종 연산 any() 에서 조건을 만족하면 멈춘다
- 지역 함수 
  - 함수 안에 함수를 정의할 수 있다 반복 코드를 줄이는데 활용하자 
  - 지역함수는 클로저로 동작하면서 변수에 선언 될 수 있다 
- 리스트 접기 
  - fold(), runningFold(), reduce(), runningReduce() 를 사용하여 리스트를 접을 수 있다
  - running 을 붙이면 중간 결과를 모두 저장한다
```kotlin
val list = listOf(1, 2, 3, 4, 5)
list.fold(1) { acc, i -> acc + i } eq 16
list.runningFold(7) { acc, i -> acc + i } eq
  "[7, 8, 10, 13, 17, 22]"
list.reduce { acc, i -> acc + i } eq 15
list.runningReduce { acc, i -> acc + i } eq
  "[1, 3, 6, 10, 15]"  
```    
- 재귀 호출 
  - tailrec 키워드를 사용하여 재귀 호출을 최적화 할 수 있다 
  - 재귀 호출이 끝나는 지점에서 return 이 발생하면 tailrec 이 적용된다 
```kotlin
private tailrec fun sum(n: Long, acc: Long): Long = 
  if (n == 0L) acc else sum(n - 1, acc + n)

fun sum(n: Long) = sum(n, 0)

fun main() {
  sum(1000000) eq 500000500000
}
```

### 5.객체 지향 프로그래밍
- 인터페이스 
```kotlin
interface Computer {
  val name: String
  fun prompt(): String
  fun calc(): Int
}

class Desktop : Computer {
  override val name = "desktop"
  override fun prompt() = "desktop>"
  override fun calc() = 1 + 2
}

// fun interface 라고 정의 하면 컴파일러는 멤버함수가 한개인지 확인한다 
fun interface OneArg {
  fun g(n: Int): Int
}

// 인터페이스 구현하는 클래스 만들 필요없이 람다식을 넘겨서 구현체를 만든다 
val sam = OneArg { it + 1 }

```
- 상속 
```kotlin
// 기반 클래스는 open 키워드로 선언해야 한다
// open 을 명시하는 이유는 코틀린의 방향성이 상속과 다형성을 적극적으로 막는 것 이기 때문이다 
open class House( val addr: String, val zip) {
  constructor(addr: String) : this(addr, 0) 

  val fullAddr: String
    get() = "$addr, $zip"
}

// 상속받는 클래스는 부모클래스의 생성자를 호출할 수 도 있다 
class Apartment(val name: String) : House("donga") {
  override fun toString(): String {
    return "Apartment(name='$name', addr='$addr', zip=$zip)"
  }
}

// 확장을 사용하면 상속을 하지 않고도 기능을 추가할 수 있다
// 다만 확장 함수는 오버라이드 될 수 없는 한계가 있다 멤버함수로 만들지 확장함수로 만들지 설계상 선택하면 된다 
fun House.toString(): String {
  return "House(addr='$addr', zip=$zip)"
}
```
- 업캐스트 
```kotlin
interface Shape {
  fun draw()  
}
class Circle : Shape {
  override fun draw() {
    println("draw circle")
  }
}
class Square : Shape {
  override fun draw() {
    println("draw square")
  }
}
// 기반 클래스인 Shape 타입으로 매개변수를 받는다 업캐스트가 가능하다 
// 업캐스트를 사용하지 않는 상속 관계는 상속이 필요없는 경우이므로 다시 생각해보자   
fun drawShape(shape: Shape) {
  shape.draw()
}
fun main() {
  val circle = Circle()
  val square = Square()
  drawShape(circle)
  drawShape(square)
}
// 다운캐스트
fun wichShape(shape: Shape) {
  when (shape) {
    is Circle -> println("circle")
    is Square -> println("square")
    else -> println("unknown")
    // 만약 Shape 을 sealed class 로 선언하면 하위 클래스가 반드시 기반 클래스와 같은 패키지 모듈안에 있어야 한다는 봉인된 제약조건이 생겨서 else 를 생략할 수 있다
  }
}
// 강제 변환 
fun drawCircle(shape: Shape) {
  val circle = shape as Circle
  // as? 를 사용하면 실패할 경우 exception 아닌 null 이 반환된다
  circle.draw()
}
// sealed 클래스인 경우 하위 클래스 이터레이션 가능 
sealed class Top
class Middle : Top()
class Bottom : Top()
fun main() {
  Top::class.sealedSubclasses.map { it.simpleName } eq 
  "[Middle, Bottom]"
  // sealedSubClasses 는 리플렉션을 사용한다 편리한 도구이지만 성능에는..
}
```
- 합성 
  - 객체지향 언어를 사용하는 가장 큰 이유는 코드 중복을 줄이는 것 이기 때문에..
```kotlin
// 상속 
interface Building
interface Kitchen 
interface House: Building {
  val kitchen1: Kitchen
  val kitchen2: Kitchen
}
// 합성 
interface Building
interface Kitchen 
interface House: Building {
  var kitchens: List<Kitchen>
}
```
- Object
```kotlin
// 싱글톤 객체를 만들때 사용한다
object JustOne {
  val name = "JustOne"
  fun echo() = println("echo $name")
}
fun main() {
  JustOne.echo()
  JustOne.name eq "JustOne"
  val x = JustOne() // 이건 안됨 싱글톤이라 
  // ojbect 는 다른 클래스나 인터페이스를 상속 받을 수 없다 
}
```
- companion object 
```kotlin
class With {
  companion object {
    val name = "With" // 동반객체의 프로퍼티는 메모리상에 한개만 생성되어 연관된 모든 클래스가 공유한다
    fun echo() = println("echo $name")
  }
  func g() = "my name is " + name 
}
fun main() {
  val w = With()
  w.g() eq "my name is With"
  w.name eq "With" // 컴패니언 오브젝트는 클래스의 멤버로 접근 가능하다
  w.echo() eq "echo With"
  w.companion.echo() eq "echo With" object 에 이름이 없을때 디폴트 이름이 compainion 이다
}
```

### 6.실패 방지 
- Exception 을 상속하여 새 예외 타입을 정의할 수 있다 
- finally 를 적절히 활용하여 자원해제가 필요한 부분을 처리하자 
- require(), requireNotNull(), check() 를 사용하여 조건을 검사하고 예외를 발생시킬 수 있다
```kotlin
data class Month(val month: Int) {
  init {
    require(month in 1..12) { "month must be between 1 and 12" }
  }
}
fun createFile(name: String) {
  requireNotNull(name) { "name must not be null" }
  // ...
  check(file.exists()) { "file does not exist" }
}
// assert() 를 사용하면 -ea 라는 플리그를 사용하여 비활성화 할 수 있다
```
- Nothing
  - 항상 예외를 발생시키는 경우에 대한 리턴 타입 
```kotlin
// 이렇게 정의해두면 fail() 로 간결하게 예외 발생 가능하다
fun fail(message: String): Nothing {
  throw IllegalArgumentException(message)
}
```
- use() 
  - AutoCloseable 인터페이스를 구현한 객체에 대해 사용이 끝나면 자동으로 close() 를 호출한다
```kotlin
fun main() {
  DataFile("Result.txt")
    .bufferedWriter()
    .use { it.readLines().first()}
}
```
- 단위 테스트 
  - 테스팅 프레임워크 별도 공부 필요 

### 7. 파워 툴 
- 확장 람다 
```kotlin
// 다음 두 함수는 같은 결과를 반환한다 
val va (String, Int) -> String = {
  str, n -> str.repeat(n) + str.repeat(n)
}
val vb: String.(Int) -> String = {
  repeat(it) + repeat(it)
}
// 아래처럼 확장 람다에 여러 파라메터를 받을 수도 있다 
val two: Int.(Int, Int) -> Boolean = {
  arg1, arg2 -> this % (arg1 + arg2) == 0
}
// it, arg 대신에 명시적인 이름을 사용하는것이 낫다 
// 아래처럼 val 말고 fun 에 대한 확장 람다를 사용하는 경우가 더 많다 
class A() {
  fun af() = 1
}
class B {
  fun bf() = 2
}
fun f1(lambda: (A, B) -> Int) = lambda(A(), B())
fun f2(lambda: A.(B) -> Int) = lambda(A(), B())

// 확장 람다를 이용한 빌더 패턴 구현 
class HtmlBuilder {
    private val elements = mutableListOf<String>()

    fun body(block: BodyBuilder.() -> Unit) {
        val bodyBuilder = BodyBuilder()
        bodyBuilder.block()
        elements.add("<body>\n${bodyBuilder.build()}</body>")
    }

    fun build(): String {
        return elements.joinToString("\n")
    }
}

class BodyBuilder {
    private val elements = mutableListOf<String>()

    fun h1(text: String) {
        elements.add("<h1>$text</h1>")
    }

    fun p(text: String) {
        elements.add("<p>$text</p>")
    }

    fun div(block: DivBuilder.() -> Unit) {
        val divBuilder = DivBuilder()
        divBuilder.block()
        elements.add("<div>\n${divBuilder.build()}\n</div>")
    }

    fun build(): String {
        return elements.joinToString("\n")
    }
}

class DivBuilder {
    private val elements = mutableListOf<String>()

    fun span(text: String) {
        elements.add("<span>$text</span>")
    }

    fun build(): String {
        return elements.joinToString("\n")
    }
}

fun html(block: HtmlBuilder.() -> Unit): String {
    val builder = HtmlBuilder()
    builder.block()
    return "<html>\n${builder.build()}\n</html>"
}
fun main() {
    val htmlString = html {
        body {
            h1("Hello, World!")
            p("This is a paragraph.")
            div {
                span("This is a span inside a div.")
            }
        }
    }
    println(htmlString)
}
```
- 영역함수 
  - with(), apply(), run(), let(), also() 을 사용하여 객체를 다룰 수 있다 
  - 영역함수는 실행시점에 인라인되어 성능상 이점이 있다 반면 보통의 람다는 람다 코드를 외부 객체에 넣기때문에 실행시점에 약간의 부가 비용이 있다 
```kotlin
val name: String? = "Kotlin"

name?.let {
    println("이름은 $it")
}

val list = mutableListOf("A", "B", "C")

list
    .also { println("초기 상태: $it") }
    .add("D")  // 반환은 원래 객체

val user = User().apply {
    name = "Alice"
    age = 30
}   

val length = "Hello".run {
    println("문자열: $this")
    length
}
println("길이: $length")

val sb = StringBuilder()

val result = with(sb) {
    append("Hello, ")
    append("World!")
    toString()
}
println(result)

```
- 제네릭스 
```kotlin
// any를 사용하면 모든 타입을 받을 수 있다
val all: List<Any> = listOf(1, "a", 3.14)

class Box<T>(private var contents: T) {
  fun put(item: T) { contents = item }
  fun get(): T = contents
}
// T 객체를 넣을 수 만 있다 
class InBox<in T>(private var contents: T) {
  fun put(item: T) { contents = item }
}
// T 객체를 꺼낼 수 만 있다
class OutBox<out T>(private var contents: T) {
  fun get(): T = contents
}
// 클래스간 상하위 관계를 명확히 하고 싶을 때 in, out 을 활용한다 
```
- 오버로딩 
```kotlin 
data class Num(val n : Int)

operator fun Num.plus(other: Num): Num {
  return Num(this.n + other.n)
}

fun main() {
  Num(4).plus(Num(5)) eq Num(9)
}
```
- 동등성 
  - 값이 같은지를 비교 
  - == 연산자는 equals() 를 호출한다 
  - === 연산자는 참조를 비교한다 (동일성) 
  - equals() 를 오버라이드 하면 hashCode() 도 오버라이드 해야 한다 
```kotlin
class A(val i: Int)

data class D(val i: Int) //data class 는 equals() 와 hashCode() 를 자동으로 오버라이드 한다

fun main() {
  val a = A(1)
  val b = A(1)
  val c = a
  (a == b) eq false // equals() 를 오버라이드 하지 않았으므로 false
  (a == c) eq true // 참조가 같으므로 true

  val d = D(1)
  val e = D(1)
  (d == e) eq true // equals() 를 오버라이드 했으므로 true
}
```
- 범위와 컨테이너 
```kotlin
data class R(val r: IntRange) {
  ovverride fun toString = "[$r]"
}

operator fun E.rangeTo(e: E) = R(v..e.v)

operator fun R.contains(e: E) Boolean = e.v in r

fun main() {
  val a = E(2)
  val b = E(3)
  val r = a..b
  (a in r) eq true // a 가 r 의 범위에 포함되어 있으므로 true
  r eq R(2..3) // toString() 을 오버라이드 했으므로 [2..3] 으로 출력된다
}
```
- 함수형 객체 연삱자 invoke 
```kotlin
class Calculator {
    operator fun invoke(a: Int, b: Int): Int {
        return a + b
    }
}

fun main() {
    val calc = Calculator()
    
    // 두 가지 방식으로 호출 가능
    val result1 = calc.invoke(3, 4)  // 일반적인 메소드 호출
    val result2 = calc(3, 4)         // invoke() 연산자 사용
    
    println("result1: $result1")  // 출력: result1: 7
    println("result2: $result2")  // 출력: result2: 7
}
```
- 연산자 사용하기 
```kotlin
// list 연산자는 오버로드한 연삱자를 호출한다 
val list = MutableListOf(1, 2, 3)
list[1] eq 2 // get() 을 사용하여 인덱스 1의 값을 가져온다
list[1] = 4 // set() 을 사용하여 인덱스 1의 값을 4로 변경한다
list += 5 // plusAssign() 을 사용하여 5를 추가한다
2 in list eq true // contains() 를 사용하여 2가 리스트에 포함되어 있는지 확인한다

class F(val i: Int): Comparable<F> {
  override fun compareTo(other: F) = i.compareTo(other.i)
}

val range = F(1)..F(10)
(F(5) in range) eq true // contains() 를 사용하여 F(5)가 범위에 포함되어 있는지 확인한다
// Comparable 인터페이스를 구현하면 정렬이 가능하고 범위연산도 가능하다 compareTo() 가 operator 로 이미 정의되어 있다 그래서 operator 라고 명시하지 않아도 된다 

class Duo(val a: Int, val b: Int) {
  operator fun component1() = a
  operator fun component2() = b
}
fun main() {
  val (a, b) = Duo(1, 2) // component1(), component2() 를 사용하여 분해한다 
  a eq 1
  b eq 2
}
``` 

- 프로퍼티 위임 
```kotlin
class Readable(val i: Int) {
  val value: String by BasicRead()
}

class BasicRead {
  operator fun getValue(r: Readable, property: KProperty<*>) = 
    "getValue: ${r.i}"
}
```
- 지연
  - lazy() 를 사용하여 초기화 시점에 값을 계산할 수 있다 
```kotlin
val lazyValue: String by lazy {
  println("lazyValue initialized")
  "Hello, World!"
}
fun main() {
  println(lazyValue) // lazyValue initialized
  println(lazyValue) // 두번째 호출에서는 초기화 되지 않는다 
}

class Bag {
  lateinit var items: String // 초기화가 안되었지만 null 이 아닌 상태로 선언할 수 있다 
  fun setUp() {
    items = "1, 2, 3"
  }
  fun status() {
    if (::items.isInitialized) { // 초기화 되었는지 확인
      println("items: $items")
    } else {
      println("items not initialized")
    }
  }
}



```