# ****ExpressibleByArrayLiteral****

Alamofire를 HTTPHeaders를 사용하던 도중 이해가 안되는 문법을 발견했습니다.

```swift
var headers: HTTPHeaders {
    switch self {
    case .fetchAllProducts:
      return []
    case .searchProducts:
      return []
    case .addProduct:
      return [HTTPHeader.contentType("application/json")]
    }
  }
```

위 코드에서 headers의 타입은 HTTPHeaders(구조체)입니다. 그런데 addProduct case에서 HTTPHeader 배열을 리턴해줬는데 컴파일이 성공했습니다.
어떻게 위와같은 상황이 가능한지 알아봅시다!
HTTPHeaders의 구현을 열어보면 의심가는 프로토콜이 하나 나옵니다.

<img width="694" alt="스크린샷 2023-12-13 오전 12 56 58" src="https://github.com/lxodud/UnderTheSwift/assets/85005933/887f5ae0-9784-4c70-a650-241990f3aca2">  

ExpressibleByArrayLiteral 이름만 봐도 딱 느낌이오는 네이밍의 중요성 👍

이 친구에 대해서 더 자세하게 알아봅시다.

## ExpressibleByArrayLiteral

```swift
public protocol ExpressibleByArrayLiteral {
  /// The type of the elements of an array literal.
  associatedtype ArrayLiteralElement
  /// Creates an instance initialized with the given elements.
  init(arrayLiteral elements: ArrayLiteralElement...)
}
```

공식문서의 설명을 보면 Array Literal을 사용해서 initialize할 수 있는 타입이라고 합니다.
Array, Set 또한 ExpressibleByArrayLiteral을 준수하기 때문에 array literal을 사용해서 아래와 같이 초기화할 수 있습니다.

```swift
let employeesSet: Set<String> = ["Amir", "Jihye", "Dave", "Alessia", "Dave"]
print(employeesSet)
// Prints "["Amir", "Dave", "Jihye", "Alessia"]"

let employeesArray: [String] = ["Amir", "Jihye", "Dave", "Alessia", "Dave"]
print(employeesArray)
// Prints "["Amir", "Jihye", "Dave", "Alessia", "Dave"]"
```

Swift의 내부 구현을 보면 Array의 경우 아래와 같이 준수하고 있습니다. (Set은 [여기](https://github.com/apple/swift/blob/main/stdlib/public/core/Set.swift))

```swift
extension Array: ExpressibleByArrayLiteral {
  // Optimized implementation for Array
  /// Creates an array from the given array literal.
  ///
  /// Do not call this initializer directly. It is used by the compiler
  /// when you use an array literal. Instead, create a new array by using an
  /// array literal as its value. To do this, enclose a comma-separated list of
  /// values in square brackets.
  ///
  /// Here, an array of strings is created from an array literal holding
  /// only strings.
  ///
  ///     let ingredients = ["cocoa beans", "sugar", "cocoa butter", "salt"]
  ///
  /// - Parameter elements: A variadic list of elements of the new array.
  @inlinable
  public init(arrayLiteral elements: Element...) {
    self = elements
  }
}
```

Array나 Set을 Array Literal `[a, b, c]` 형태로 초기화할 수 있었던 이유가 ExpressibleByArrayLiteral 때문이었습니다.

```swift
struct Alphabet {
  var list: [Character]
}
```

마지막으로 위 Alphabet 타입을 Array Literal로 초기화할 수 있게끔 만들어봅시다.

```swift
extension Alphabet: ExpressibleByArrayLiteral {
  init(arrayLiteral elements: Character...) {
    self.list = elements
  }
}

let alphabet: Alphabet = ["a", "b"]

print(alphabet.list)
// prints ["a", "b"]
```

추가적으로 Array 이외에도 Int, Float, String 등등 ExpressibleBy 여러가지 프로토콜이 존재하기 때문에 응용해봐도 좋을 것 같습니다.

진짜 마지막으로 Array Literal과 Array는 다르기 때문에 아래와 같은 코드는 동작하지 않습니다.

```swift
let alphabet: Alphabet = Array<Character>(["a"])
```

끝~~

# Reference
[ExpressibleByArrayLiteral | Apple Developer Documentation](https://developer.apple.com/documentation/swift/expressiblebyarrayliteral)  
[CompilerProtocols.swift](https://github.com/apple/swift/blob/main/stdlib/public/core/CompilerProtocols.swift)  
[Array.swift](https://github.com/apple/swift/blob/main/stdlib/public/core/Array.swift)  
[Set.swift](https://github.com/apple/swift/blob/main/stdlib/public/core/Set.swift)  
