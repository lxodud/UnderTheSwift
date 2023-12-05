# #selector란?

## Selector

Objective-C에서 Selector는 두 가지 의미를 가집니다.
객체에 대한 소스코드 메시지에 사용될 때 단순히 메서드 이름을 참조하는 데 사용할 수 있습니다. (객체에 메시지를 보낼 때 참조할 메서드의 이름을  나타낼 때 사용)
소스 코드가 컴파일될 때 이름을 대체하는 고유 식별자를 의미하기도 한다고합니다.
Selector를 사용하면 객체에 대한 메서드를 호출할 수 있다. 이는 Cocoa에서 target-action 디자인 패턴을 구현하기 위한 기반을 제공합니다.

## Selector Expression
Selector Expression을 사용하면 Objective-C에 메서드 또는 프로퍼티의 getter 또는 setter를 참조하는데 사용되는 Selector에 접근할 수 있습니다.

```swift
#selector(<#method name#>)
#selector(getter: <#property name#>)
#selector(setter: <#property name#>)
```

메서드 이름 (method name) 과 프로퍼티 이름 (property name) 은 Objective-C 런타임에 사용할 수 있는 메서드 또는 프로퍼티에 대한 참조여야 합니다. Selector 표현식의 값은 `Selector` 타입의 인스턴스입니다.

```swift
class SomeClass: NSObject {
    @objc let property: String

    @objc(doSomethingWithInt:)
    func doSomething(_ x: Int) { }

    init(property: String) {
        self.property = property
    }
}
let selectorForMethod = #selector(SomeClass.doSomething(_:))
let selectorForPropertyGetter = #selector(getter: SomeClass.property)
```

프로퍼티의 getter 에 대한 `Selector`를 생성할 때 프로퍼티 이름 (property name) 은 변수 또는 상수 프로퍼티에 참조될 수 있습니다. 반대로 프로퍼티의 setter 에 대한 `Selector`를 생성할 때 프로퍼티 이름 (property name) 은 변수 프로퍼티에만 참조될 수 있습니다.
메서드 이름 (method name) 은 그룹화 (grouping) 을 위해 소괄호와 이름을 공유하지만 타입 서명이 다른 메서드를 명확하기 하기위해 `as` 연산자도 포함할 수 있습니다. 예를 들어:
`Selector`는 런타임이 아닌 컴파일 시 생성되기 때문에 컴파일러는 메서드 또는 프로퍼티가 존재하고 Objective-C 런타임에 노출되는지 확인할 수 있습니다.

## 요약
Selector를 사용하면 객체에 대한 메서드를 호출할 수 있고 이는 Cocoa에서 target-action 디자인 패턴을 구현하기 위한 기반을 제공합니다.
Swift에서는 Selector Expression을 통해서 Selector에 접근할 수 있으므로 target-action 패턴을 수행하기 위해서 #selector 같은 문법을 사용합니다.

# Reference
[Selectors](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocSelectors.html)  
[Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/expressions#Selector-Expression)  
[Selector](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/Selector.html)  
