# Type Erasure Pattern - 2

[1편](https://github.com/lxodud/UnderTheSwift/blob/main/doc/231219-Type%20Erasure%20Pattern-1.md)에 이어서 본격적으로 Type Erasure Pattern에 대해서 얘기해봅시다.
시작하기에 앞서 한번 더 언급하자면 현재는 any, some같은 문법들로 해당 아티클에서 얘기할 문제들을 해결할 수 있지만 여전히 apple의 standard library에는 Type Erasure Pattern을 적용한 타입들이 존재하기 때문에 알아두면 좋을 것 같습니다!

# Type Erasure Pattern

Type Erasure Pattern을 구현하는 방법은 여러가지입니다.
일단 클래스 상속을 이용해서 구현해봅시다. (Swift standard library에서 사용하는 방법)
먼저 Type Erasure Pattern을 적용할 프로토콜입니다.

```swift
protocol Sendable {
  associatedtype Target
  
  func send(_ target: Target)
}
```

Type Erasure Pattern을 구현하기 위해서는 아래와 같이 3가지 타입을 구현해야 합니다.

<img width="293" alt="스크린샷 2023-12-31 오전 1 20 11" src="https://github.com/lxodud/UnderTheSwift/assets/85005933/95aad8a7-0b09-4041-9b81-e64ed183d3a0">

하나씩 알아봅시다.

## Base
기본 추상 클래스가 될 친구이고 두 가지 요구사항이 있습니다.
1. 적용할 프로토콜을 준수합니다.
2. 해당 프로토콜의 associated type 역할을 수행할 제네릭 타입 파라미터를 정의합니다.
Base는 직접 사용되지 않고 서브클래싱 될 것이기 때문에 private으로 구현합니다.

```swift
private class _AnySendableBase<Element>: Sendable {
  func send(to target: Element) {
    fatalError("Must override send()")
  }
}
```
Swift에서는 추상 클래스를 구현하기 위한 문법이 존재하지 않기 때문에 fatalError로 대체합니다.

## Box
패턴을 적용할 프로토콜을 준수하는 구체 인스턴스를 저장하는 타입입니다.
1. Base를 상속받습니다.
2. 해당 프로토콜을 준수하는 제네릭 타입 파라미터를 정의합니다.
3. 위에서 정의한 제네릭 타입의 프로퍼티에 구체 타입의 인스턴스를 저장합니다.
4. 각 프로토콜 메서드에서 저장된 구체 타입의 인스턴스를 호출합니다.

```swift
private final class _AnySendableBox<S: Sendable>: _AnySendableBase<S.Target> {
  var s: S
  
  init(s: S) {
    self.s = s
  }
  
  override func send(to target: S.Target) {
    s.send(to: target)
  }
}
```

## ****Public Wrapper****
Type이 소거된 래퍼의 공개 인터페이스 역할을 수행하는 타입입니다.
1. 패턴을 적용할 프로토콜을 준수합니다.
2. Base와 같이 제네릭 타입 파라미터를 정의합니다.
3. initializer에서 해당 프로토콜을 준수하는 구체 타입을 받습니다.
4. 해당 구체 타입을 Box에 할당합니다. (내부에 Base 타입의 프로퍼티가 존재합)
5. 각 프로토콜 메서드에서 Box의 메서드를 호출합니다. (포워딩하기)

```swift
final class AnySendable<Target>: Sendable {
  private var box: _AnySendableBase<Target>
  
  init<S: Sendable>(sendable: S) where S.Target == Target {
    self.box = _AnySendableBox(s: sendable)
  }
  
  func send(to target: Target) {
    box.send(to: target)
  }
}
```

이제 Sendable을 채택하는 구체 타입을 만들고 사용해봅시다.
```swift
final class Phone: Sendable {
  func send(to target: Int) {
    print("\(target) 신호 가는중")
  }
}

let temp: AnySendable<Int> = AnySendable(sendable: Phone())
temp.send(to: 1234)
```

다음은 조금 더 간단한 방법으로 Type Erasure를 구현해봅시다.
바로 함수를 저장하는 방식입니다.

```swift
final class Phone: Sendable {
  func send(to target: Int) {
    print("\(target) 신호 가는중")
  }
}

struct AnySendable<Target>: Sendable {
  let _send: (Target) -> Void
  
  init<S: Sendable>(_ s: S) where S.Target == Target {
    _send = s.send(to:)
  }
  
  func send(to target: Target) {
    return _send(target)
  }
}

let temp: AnySendable<Int> = AnySendable(Phone())
temp.send(to: 1234)
```

끝입니다.

마지막으로 Swift 내부에서는 어떻게 구현되어 있는지 확인해봅시다!
먼저 위에서 설명한 Base입니다.

```swift
internal class _AnySequenceBox<Element> {
  @inlinable // FIXME(sil-serialize-all)
  internal init() { }

  @inlinable
  internal func _makeIterator() -> AnyIterator<Element> { _abstract() }

  @inlinable
  internal var _underestimatedCount: Int { _abstract() }

  @inlinable
  internal func _map<T>(
    _ transform: (Element) throws -> T
  ) rethrows -> [T] {
    _abstract()
  }
	...
}
```

네이밍이 조금 다르지만 map 메서드 내부에서 _abstract를 호출하는 것으로 추상 클래스라는 것을 확인할 수 있습니다.
```swift
internal func _abstract(
  file: StaticString = #file,
  line: UInt = #line
) -> Never {
  fatalError("Method must be overridden", file: file, line: line)
}
```

abstract함수의 구현은 다음과 같습니다!
다음으로 위에서 설명한 Box의 역할을 수행하는 타입을 봅시다.
```swift
internal final class _SequenceBox<S: Sequence>: _AnySequenceBox<S.Element> {
  @usableFromInline
  internal typealias Element = S.Element
```

바로 _SequenceBox입니다. _AnySequenceBox를 상속받고 있는 것으로 확인할 수 있습니다.
```swift
@inlinable
  internal override var _underestimatedCount: Int {
    return _base.underestimatedCount
  }
  @inlinable
  internal override func _map<T>(
    _ transform: (Element) throws -> T
  ) rethrows -> [T] {
    return try _base.map(transform)
  }
```

내부 메서드들을 보면 _base 프로퍼티의 메서드를 포워딩하는 것을 볼 수 있습니다.
따라서 _base 프로퍼티에 구체 타입이 들어감을 알 수 있습니다!
```swift
@inlinable
  internal init(_base: S) {
    self._base = _base
  }

  @usableFromInline
  internal var _base: S
```

요렇게 말이죠!
마지막으로 외부에 드러낼 Public Wrapper를 봅시다.
```swift
@frozen
public struct AnySequence<Element> {
  @usableFromInline
  internal let _box: _AnySequenceBox<Element>
```

내부에 _AnySequenceBox<Element>에 대한 참조를 가지고 있습니다.
```swift
@inline(__always)
  @inlinable
  public __consuming func makeIterator() -> Iterator {
    return _box._makeIterator()
  }
  @inlinable
  public __consuming func dropLast(_ n: Int = 1) -> [Element] {
    return _box._dropLast(n)
  }
```
추가적으로 _box의 메서드로 포워딩하는 것으로 Public Wrapper임을 알 수 있습니다!
전체적인 구조는 아래 사진과 같습니다.

<img width="293" alt="스크린샷 2023-12-31 오전 1 21 51" src="https://github.com/lxodud/UnderTheSwift/assets/85005933/164626b2-a01b-466e-ae26-bedb5c7dc496">


지금까지 Type Erasure Pattern에 대해서 알아보았습니다~
또 한번 얘기하지만 현재는 any, some같은 신규 문법들로 이러한 문제들을 해결할 수 있기 때문에 해당 내용들도 공부해보면 좋을 것 같습니다!!

# Reference
[mikeash.com: Type Erasure in Swift](https://mikeash.com/pyblog/friday-qa-2017-12-08-type-erasure-in-swift.html)  
[Sequence](https://github.com/apple/swift/blob/main/stdlib/public/core/ExistentialCollection.swift#L165)  
[AnySequence](https://github.com/apple/swift/blob/main/stdlib/public/core/Sequence.swift)  
[Breaking Down Type Erasure in Swift - Big Nerd Ranch](https://bignerdranch.com/blog/breaking-down-type-erasure-in-swift/)  
 
