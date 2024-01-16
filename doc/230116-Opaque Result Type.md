# Opaque Result Types

이번에 알아볼 주제는 Opaque Result Type입니다.

이전에 Type Erasure Pattern을 공부할 때 나왔던 some 키워드와 연관이 있는 문법으로 나오게된 배경과 어떻게 동작하는지 알아봅시다.

# Motivation

제네릭은 호출자에 의해서 타입이 결정됩니다. (Type Erasure Pattern - 1에서 얘기했죠!)

```swift
func someFunc<T>(p: T) { 
  print(T.self)
}

someFunc(p: 1)
// prints Int

someFunc(p: "")
// prints String
```

그런데 반환값은 구현하는 쪽에서 결정하고 추상화하는게 일반적입니다.

예를 들어서 어떤 컬렉션을 반환하지만 구체적인 타입을 감추고 싶을 수 있습니다. 그 이유는 이를 사용하는 클라이언트가 구체적인 타입에 강하게 결합된다면 추후에 컬렉션을 바꿔야 하는 일이 생긴다면 변경의 영향이 모든곳에 전파되기 때문입니다. 따라서 쉽게 변하는 구체적인 것을 감추고 추상적인 것을 드러내는 것이죠!

그럼 이런코드는 어떨까요?

```swift
func someFunc<C: Collection, Output: Collection>(in collection: C) -> Output
  where C.Element == Int, Output.Element == Int
{
  return collection.filter { $0 % 2 == 0 }
}
```

위에서 말했듯이 제네릭은 호출자에 의해 결정되고 따라서 Output은 Collection을 채택하는 어떤 타입이든 리턴 할 수 있음을 의미하기 때문에 원하는 구현을 수행할 수 없습니다.

그럼 남은 선택지는 Type Erasure Pattern, Existential Type입니다.

그러나 둘은 값 수준의 추상화입니다. 그리고 Existential Type은 아래와 같은 제약사항도 존재합니다.

```swift
func someFunc<C: Collection>(in collection: C) -> Collection where C.Element == Int {
  return collection.filter { $0 % 2 == 0 }
}
```

이전에 봤듯이 이 코드는 동작하지 않습니다. Collection은 associated type이 있기 때문이죠!

당시 Swift에는 호출자의 제어와 관계없이 반환 값을 타입 수준으로 추상화하는 문법이 존재하지 않았습니다. (제네릭 x → 호출자에 의해 제어됌, Existential, TypeErasure → Value Level)

결론적으로 제네릭과 반대로 동작하면서 타입 수준의 추상화를 할 수 있는 그런 문법이 필요했던 것이죠.

그래서 Opaque Result Type 나오게 되었습니다!!

## Opaque Result Type

Opaque Result Type의 동작은 한마디로 제네릭의 반대입니다. 호출자에 의해 결정되는 제네릭과 달리 구현부에서 실제 타입이 결정되고 호출자에게는 추상화되어 보여집니다. 코드로 예를 들면 아래와 같습니다.

```swift
func someFunc() -> some Collection {
  return [1, 2, 3]
}

// a의 타입은 some Collection
let a = someFunc()
```

some 뒤에는 class, protocol, Any, AnyObject 또는 이들의 합성으로 만드는 제약 조건이 붙습니다.

동일한 함수에 대해 여러번 호출한 반환 값은 동일합니다.

```swift
func foo<T: Equatable>(x: T, y: T) -> some Equatable {
  return x == y ? 1738 : 679
}

let x = foo(x: "apples", y: "bananas")
let y = foo(x: "apples", y: "some fruit nobody's ever heard of")

print(x == y) // 가능
```

Opaque Type이 Associated Type을 노출하는 경우 해당 타입의 ID도 유지됩니다.

컴파일 타임에 정적인 값을 할당할 수는 없습니다.

```swift
var z = foo(x: "apples", y: "bananas")
z = 123 // error
```

런타임에 동적으로 타입 캐스팅은 가능합니다!!

```swift
func foo() -> some BinaryInteger { return 219 }
if let x = foo() as? Int {
  print("It's an Int, \(x)\n")
} else {
  print("Guessed wrong")
}
```

Opaque Type을 반환하는 함수는 각 return 문에서 동일한 구체 타입을 반환해야 합니다.

```swift
protocol P { }
extension Int : P { }
extension String : P { }

// error 동일한 구체 타입을 반환해야 함
func f2(flip: Bool) -> some P {
  if flip { return 17 }
  return "a string"
}

// error P 타입은 반환할 수 없음 -> 구체 타입을 반환해야함
func f4() -> some P {
  let p: P = "hello"
  return p
}
```

Opaque Result Type을 가진 함수는 fatalError를 호출하더라도 return 문이 있어야 합니다.

```swift
func f9() -> some P {
  fatalError("not implemented")

  // error: no return statement to get opaque type
}
```

아래처럼 해결할 수 있습니다.

```swift
extension Never: P {}
func f9b() -> some P {
  return fatalError("not implemented") // OK, explicitly binds return type to Never
}
```

## **Properties and subscripts**

Return Type이외에도 Property, Subscript에서 Opaque Type을 사용할 수 있습니다.

사용법은 같습니다. some 키워드 뒤에 제약 사항을 걸어주는 것입니다.

```swift
let strings: some Collection = ["hello", "world"]

public struct Vendor {
  private var storage: [Impl] = [/* ... */]

  public var count: Int {
    return storage.count
  }

  public subscript(index: Int) -> some P {
    get {
      return storage[index]
    }
    set (newValue) {
      storage[index] = newValue
    }
  }
}
```

마지막으로 Swift 5.7에서 업데이트된 문법에 대해서 알아봅시다.

## **Opaque Parameter Declarations**

Parameter에 Opaque를 쓸 수 있습니다.

```swift
func f<_T: P>(_ p: _T)
```

위와 같은 제네릭 구문을 아래와 같이 바꿀 수 있습니다.

```swift
func f(_ p: some P) { }
```

Opaque Return Type과 달리 호출자에 의해 타입이 결정됩니다.

즉, 기존의 제네릭을 파라미터로 사용하는 동작과 동일하고 조금 더 간결하게 작성하기 위해 만들어진 것으로 보입니다.

근런데 Opaque Parameter Declarations은 한계가 뚜렷해 보입니다.

```swift
func eagerConcatenate<Sequence1: Sequence, Sequence2: Sequence>(
    _ sequence1: Sequence1, _ sequence2: Sequence2
) -> [Sequence1.Element] where Sequence1.Element == Sequence2.Element {
  let a = sequence1.map { $0 }
  let b = sequence2.map { $0 }
  return a + b
}
```

위 함수는 두 개의 Sequence를 채택하는 타입의 인스턴스를 받아서 배열로 리턴하는 함수입니다.

이걸 Opaque Parameter Declarations로 리팩토링해봅시다.

```swift
func eagerConcatenate(
  _ sequence1: some Sequence, _ sequence2: some Sequence
) -> [some Sequence] {
  let a = sequence1.map { $0 }
  let b = sequence2.map { $0 }
  return a + b // error
}
```

Sequence의 Element가 뭔지 보장할 수 없습니다.

[실제로 해당 문법의 제안서에도 향후 방향성에 다음 문제들이 있습니다.](https://github.com/apple/swift-evolution/blob/main/proposals/0341-opaque-parameters.md#:~:text=in%20such%20cases.-,Future%20Directions,-Constraining%20the%20associated)

```swift
func eagerConcatenate<T>(
    _ sequence1: some Sequence<T>, _ sequence2: some Sequence<T>
) -> [T]
```

이런 느낌으로 해볼 수 있다고 하는데 아직은 구현되지 않은 듯 합니다!

끝~

## Refrence
[Swift Language Guide - Opaque Type](https://bbiguduk.gitbook.io/swift/language-guide-1/opaque-types)  
[Swift Forums - Improving the UI of generics](https://forums.swift.org/t/improving-the-ui-of-generics/22814)  
[Swift Evolution - Opaque Result Types](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md)  
[Swift Evolution - Opaque Parameter Declarations](https://github.com/apple/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)  
