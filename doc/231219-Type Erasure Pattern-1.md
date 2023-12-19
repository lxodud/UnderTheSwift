# Type Erasure Pattern - 1

오늘 공부해볼 주제는 Type Erasure Pattern입니다.
물론 현재는 any, some같은 문법들로 해당 아티클에서 얘기할 문제들을 해결할 수 있지만 여전히 apple의 standard library에는 Type Erasure Pattern을 적용한 객체들이 존재하기 때문에 알아두면 좋을 것 같습니다!

먼저 알아두면 좋은 개념들과 Type Erasure Pattern을 사용하는 배경에 대해서 얘기해보겠습니다.

# Generic vs Existential Type
Generic은 정의한 요구사항에 따라 모든 타입에서 동작할 수 있는 유연하고 재사용 가능한 함수와 타입을 작성할 수 있는 문법입니다.
예를 들어서 Sequence를 채택하며  내부 요소가 Int인 타입을 인자로 받는 함수를 Generic을 사용해서 아래와 같이 선언할 수 있습니다.

```swift
func f<S: Sequence>(seq: S) where S.Element == Int { ...
```

위 함수에서 S를 return 타입으로 쓸 수는 없습니다.
```swift
// error
func g<S: Sequence>() -> S where S.Element == Int { ...
```

Existential Type은 아래 코드처럼 프로토콜을 타입으로 사용하는 것입니다.

```swift
protocol SomeProtocol {
    
}

struct SomeStruct {
    let a: SomeProtocol
}

func someFunc(someParameter: SomeProtocol) -> SomeProtocol { }
```

Existential Type으로 위 함수를 작성해보면 아래와 같습니다.

```swift
// error
func g() -> Sequence { ...
```

안타깝게도 위 코드도 동작하지 않습니다.
Protocol 내부에 associated type이 있다면 해당 프로토콜을 type constraints로만 사용할 수 있기 때문입니다.
조금 더 자세하기 왜 둘다 안되는지 알아봅시다.

이걸 이해하려면 일단 Generic과 Existential Type의 추상화 레벨에 대한 개념이 필요합니다.
Generic은 type level 추상화이고 Existential Type은 value level 추상화입니다.
이게 뭔 소린지 이해안되는게 당연합니다.

아래 코드로 예시를 들어보겠습니다.

```swift
func bar<T: Collection>(x: T, y: T) -> [T] { ... }
```

위 함수에서 x, y는 서로 같은 구체 타입이라고 보장할 수 있습니다. 즉, x에는 Array 타입의 값이 할당되고 y에는 Dictionary 타입의 값이 할당될 수 없다는 뜻입니다. 이를 type level 추상화라고 합니다.
```swift
func func someFunc(x: SomeProtocol, y: SomeProtocol) -> SomeProtocol { }
```
Existential Type은 Generic과 대조적으로 해당 프로토콜을 채택하기만 하면 어떤 구체 타입의 값이든 할당할 수 있습니다. 이를 value level 추상화라고 합니다.

이제 다시 위에 error가 났던 함수를 봅시다.

```swift
// error
func g<S: Sequence>() -> S where S.Element == Int { ...
```
Generic은 type level 추상화라고 했죠?
그럼 S는 구체 타입이 보장되어 있습니다. 그리고 Generic은 호출되는 곳에서 타입이 정해집니다. 그럼 구현부에서는 S가 어떤 타입인지 알지 못하고 따라서 return 타입으로 사용이 불가능합니다.

```swift
// 이 함수가 좀 더 이해하기 명화해보임
func bar(x: Sequence, y: Sequence) { ... }
```
이번엔 Existential Type 차례입니다.
Existential Type은 value level 추상화라고 했죠?
따라서 x랑 y에는 Sequence를 채택한 어떤 타입이든 들어올 수 있습니다.
그럼 Sequencec 내부에 associatedtype도 서로 다를 수 있겠죠?

<img width="617" alt="스크린샷 2023-12-19 오후 11 22 25" src="https://github.com/lxodud/UnderTheSwift/assets/85005933/700c91a8-0c44-4ad4-94ba-509f65a47e08">

그럼 x랑 y를 내부에서 다룰 때 서로다른 타입의 요소끼리 연산을 하기라도 한다면 문제가 발생하겠죠?
따라서 associatedtype이 있는 Protocol은 Existential Type으로 사용할 수 없는 것입니다. (Swift는 타입에 엄격하기 때문!)

너무 길어져서 Type Erasure Pattern 다음 아티클에서 마저 알아봅시다!
