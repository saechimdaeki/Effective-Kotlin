# 아이템 7. 결과 부족이 발생할 경우 null과 Failure를 사용하라.

함수가 원하는 결과를 만들어 낼 수 없을 때가 있다. 몇가지 예를 들어보면 다음과 같다.
- 서버로부터 데이터를 읽어 들이려고 했는데, 인터넷 연결 문제로 읽어 들이지 못한 경우
- 조건에 맞는 첫 번째 요소를 찾으려 했는데, 조건에 맞는 요소가 없는 경우
- 텍스트를 파싱해서 객체를 만들려고 했는데, 텍스트의 형식이 맞지 않는 경우

이러한 상황을 처리하는 메커니즘은 크게 다음과 같이 두 가지가 있다.
- `null` 또는 실패를 나타내는 sealed 클래스(일반적으로 `Failure`라는 이름을 붙인다) 를 리턴한다.
- 예외를 throw 한다.

이러한 두 가지는 중요한 차이가 있다. 일단 예외는 정보를 전달하는 방법으로 사용해서는 안된다. 예외는 잘못된 특별한 상황을 나타내야 하며, 처리되어야 한다.

예외는 예외적인 상황이 발생했을 때 사용하는 것이 좋다. 이유를 정리하면 다음과 같다.
- 많은 개발자가 예외가 전파되는 과정을 제대로 추적하지 못한다
- 코틀린의 모든 예외는 `unchecked`예외이다. 따라서 사용자가 예외를 처리하지 않을 수도 있으며, 실제로 API를 사용할 때 예외와 관련된 사항을 메서드 등을 사용하면서 파악하기 힘들다
- 예외는 예외적인 상황을 처리하기 위해서 만들어졌으므로 명시적인 테스트(explicit test)만큼 빠르게 동작하지 않는다.
- try-catch 블록 내부에 코드를 배치하면, 컴파일러가 할 수 있는 최적화가 제한된다.

반면, 첫 번째로 설명했던 `null`과 `Failure`는 예상되는 오류를 표현할 때 굉장히 좋다. 이는 명시적이고, 효율적이며, 간단한 방법으로 처리할 수 있다.

따라서 충분이 예측할 수 있는 범위의 오류는 `null`과 `Failure`를 사용하고, 예측하기 어려운 예외적인 범위의 오류는 예외를 throw해서 처리하는 것이 좋다.

```kotlin
inline fun <reified T> String.readObjectOrNull(): T? {
    //...
    if(incorrectSign) {
        return null
    }
    //...
    return result
}

inline fun <reified T> String.readObject(): Result<T> {
    //...
    if(incorrectSign) {
        return Failure(JsonParsingException())
    }
    //...
    return Success(result)
}

sealed class Result<out T>
class Success<out T>(val result: T): Result<T>()
class Failure(val throwable: Throwable): Result<Nothing>()

class JsonParsingException: Exception()
```

이렇게 표시되는 오류는 다루기 쉬우며 놓치기 어렵다. null을 처리해야 한다면, 사용자는 안전호출(safe call) 또는 Elvis 연산자 같은 다양한 널 안정성 기능을 활용한다

```kotlin
val age = userText.readObjectOrNull<Person>()?.age ?: -1
```

Result와 같은 공용체(union type)를 리턴하기로 했다면, when 표현식을 사용해서 이를 처리할 수 있습니다.

```kotlin
val person = userText.readObjectOrNull<Person>()
val age = when(person) {
    is Success -> person.age
    is Failure -> -1
}
```

이러한 오류 처리 방식은 try-catch 블록보다 효율적이며, 사용하기 쉽고 더 명확하다. 예외는 놓칠 수도 있으며, 전체 애플리케이션을 중지시킬수도 있다.

null과 sealed result 클래스는 명시적으로 처리해야 하며, 애플리케이션의 흐름을 중지하지도 않는다.

null값과 sealed result 클래스의 차이점이 궁금할 수 있는데 추가적인 정보를 전달해야 한다면 sealed result를 사용하고, 

그렇지 않으면 null을 사용하는 것이 일반적이다. `Failure`는 처리할 때 필요한 정보를 가질 수 있다.

일반적으로 두 가지 형태의 함수를 사용한다. 하나는 예상할 수 있을 때, 다른 하나는 예상할 수 없을때 사용한다.

List는 두 가지를 모두 갖고 있으므로 이를 기반으로 살펴보자.

- get: 특정 위치에 있는 요소를 추출할때 사용한다. 만약 요소가 해당 위치에 없다면 IndexOutOfBoundsException을 발생시킨다.
- getOrNull: out of range 오류가 발생할 수 있는 경우에 사용하며, 발생한 경우에는 null을 리턴한다.

이외에도 `getOrDefault`와 같은 다른 선택지도 있다. 하지만 일반적으로 `getOrNull` 또는 Elvis연산자를 사용하는 것이 쉽다.

개발자는 항상 자신이 요소를 안전하게 추출할 거라 생각한다. 따라서 nullable을 리턴하면 안된다. 개발자에게 null이 발생할 수 있다는 경고를 주려면,

`getOrNull`등을 사용해서 무엇이 리턴되는지 예측할 수 있게 하는 것이 좋다.