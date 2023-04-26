# Kotlin 에서의 try-with-resources

JVM 에서는 메모리라는 자원을 자동으로 처리한다(GC)    
하지만 메모리 이외에도 처리해야하는 리소스들이 있다.    
(파일, 네트워크 연결, 스트림 등등..)

## Java 에서의 자원해제

Java 에서는 다음과 같이 try-with-resources 를 사용하여 자동으로 자원 해제가 가능하다.

```java
try (Resource resource = new Resource()) {
    resource.use();
} catch(Exception e) {
    // handle error
}
```

이때 try 에 명시하는 resource 객체는 AutoClosable 인터페이스를 구현해야 한다.

```java
public class Resource implements AutoClosable {
    ...
} 
```

## Kotlin 의 자원 해제 use 함수

Kotlin 에서는 Java 의 `try-with-resources` 와 같이 자원을 자동으로 관리해주는 `언어 전용 구문`이 없다.

대신 표준 라이브러리에서 제공하는 use 함수가 있다.

Java 처럼 AutoClosable 을 구현하는 모든 객체는 use 함수를 호출할 수 있다.

```kotlin
FileWriter("test.txt")
  .use { it.write("something") }
```

## Kotlin use 함수 주의사항

만약 kotlin 설정에서 jvm 1.6 이하를 사용한다면 주의해야할 점이 있다.

jvm 1.6 이하에서는 Kotlin 의 use 함수의 시그니쳐는 다음과 같다

```kotlin
public inline fun <T : Closeable?, R> T.use(block: (T) -> R): R {
```

코드를 보면 Closeable 인터페이스의 확장 함수로 정의됐다.
자바 6 이하 버전에서는 AutoCloseable 이 존재하지 않았으며 당연히 Closeable 도 AutoCloseable을 확장하지 않았다.

이 경우 아래와 같이 Kotlin 확장에 대한 종속성을 추가해줄 수 있다.

```maven
<dependency>
    <groupId>org.jetbrains.kotlin</groupId>
    <artifactId>kotlin-stdlib-jdk8</artifactId>
</dependency>
```

또는 kotlin gradle 을 사용한다면 아래와 같이 설정할 수 있다.

```groovy
tasks.withType<KotlinCompile> {
    kotlinOptions.jvmTarget = "1.6"
}
```

이렇게 JVM 버전을 올려주면 use 함수의 시그니쳐는 아래와 같이 바뀐걸 볼 수 있다.

```kotlin
public inline fun <T : AutoCloseable?, R> T.use(block: (T) -> R): R {
```
