# Idea: Introduce 'generic-case' to refine pattern matching of enums with associated values

## Introduction

**Generic-case** is a new feature that refines pattern matching of enums with associated values, especially in switch statements.

## Motivation

Currently, enums with associated values cannot be used flexibly in switch statements. Sometimes you have to write codes with such ugly repetitions.

```Swift
enum Either<First, Second>: CustomStringConvertible {
    case first(First)
    case second(Second)
  
    var description: String {
        switch self{
        case let .first(value): return "\(value)"
        case let .second(value): return "\(value)"
        }
    }   
}
```

In the switch statement, each case does actually the same operation. However, it is difficult to make this code simpler. This is a reasonable result considering type safety.

```Swift
//Error
//Pattern variable bound to type 'Second', expected type 'First'
case let .first(value), let .second(value): return "\(value)"
```

This example includes only one switch statement, but during dealing with this `Either`, you have to write such code again and again. And if you have more cases in the enum, it becomes much harder to deal with.

There are many problems here. Not only awful developer experience but also the possibility to cause bugs due to mistakes in copy/paste operation exist. Also, if the cases are many, maintenance costs large. Especially in fields that cases increase as time passes, such ugly, complex, and huge switch statements are often produced and work as the source of bugs.

Another example was found in

* https://stackoverflow.com/questions/52428073/generic-enum-with-switch-swift
* https://stackoverflow.com/questions/47659168/swift-get-associated-value-in-enums-without-switch

For this example, you can use `Any` as a temporary solution. With today's Swift, you can cast values inside the pattern, so the following code is valid. 

```Swift
    var description: String {
        switch self{
        //OK
        case let .first(value as Any), let .second(value as Any): 
            //here you can use (value: Any)
            return "\(value)"
        }
    }   
```

However, this strategy doesn't always work fine.

### Problem#1: Cannot call generic functions

You can cast `value` as a protocol existential so that `value` can be used as a variable conforming to the protocol.

```Swift
enum Things{
    case int(Int), string(String) 
}

//OK
case let .int(value as Encodable), let .string(value as Encodable): 
    //you can use (value: Encodable)
```

However, this is not enough when dealing with **generic functions**. Following code doesn't work because `container.encode` is a generic function.

```Swift
func encode(to encoder: Encoder) throws {
    var container = encoder.container(keyedBy: CodingKeys.self)
    switch thing{
    case let .int(value as Encodable), let .string(value as Encodable): 
        //Error
        //No exact matches in call to instance method 'encode'
        try container.encode(value, forKey: .value)
    }
}
```

Definition of `container.encode` is **generic**.

```Swift
mutating func encode<T>(_ value: T, forKey key: KeyedEncodingContainer<K>.Key) throws where T : Encodable
```

You can solve this problem by defining a new function using protocol extension and open the existential.

```Swift
extension Encodable{
    func containerEncode<CodingKeys: CodingKey>(container: inout KeyedEncodingContainer<CodingKeys>, key: CodingKeys) throws {
        try container.encode(self, forKey: key)
    }
}
```

Then you can use this function like the next.

```Swift
case let .int(value as Encodable), let .string(value as Encodable): 
    try value.containerEncode(container: &container, key: .value)
```

However, it is much too frustrating to extend protocols every time you want to write code like the example.

### Problem#2: Cannot cast as protocols with associated types

Also, if the target protocol has **Self or associated type requirements**, this technique cannot be applied.

```Swift
enum Integer{
    case int64(Int64)
    case int32(Int32)
    case int16(Int16)
    case int8(Int8)

    var bitWidth: Int {
        switch self{
        //Error
        //Protocol 'BinaryInteger' can only be used as a generic constraint because it has Self or associated type requirements
        case let .int64(value as BinaryInteger),
             let .int32(value as BinaryInteger),
             let .int16(value as BinaryInteger),
             let .int8(value as BinaryInteger):
            return value.bitWidth
        }
    }
}
```

You can solve this problem by defining a new protocol that does not use Self nor associated types, so that you can make the code simpler.

```Swift
protocol BitWidthServer{
    var bitWidth: Int {get}
}

extension Int64: BitWidthServer{}
extension Int32: BitWidthServer{}
extension Int16: BitWidthServer{}
extension Int8: BitWidthServer{}

case let .int64(value as BitWidthServer),
     let .int32(value as BitWidthServer),
     let .int16(value as BitWidthServer),
     let .int8(value as BitWidthServer):
    return value.bitWidth
```

But obviously, it is too a tiresome solution. You only want to use `value.bitWidth` which must exist in each pattern.

### Summary

As above, currently, we have problems of ugly repetition due to inflexibility in switch statements with enums with associated values. We can avoid them by using cast to the protocol existentials or super types. However, this way is much too weak.

* We cannot call generic functions when we cast values as protocol existentials.
* We cannot cast type as protocols with Self or associated types.

We have to make up some smart ways to deal with such situations.

## Proposed solution

This proposal suggests a new feature named **generic-case**. This works like this.

```Swift
switch either{
//declare generic type parameter `T` after `case` and cast value as `T`
case <T> let .first(value as T), let .second(value as T):
    //you can use (value: T) inside the case
    //you can call generic function
    genericFunc(value)	//call genericFunc<T>
}
```

This example works like the next code as if there are two `case`.

```Swift
switch either{
case let .first(value):
    genericFunc(value)	//call genericFunc<First>
case let .second(value):
    genericFunc(value)	//call genericFunc<Second>
}
```

Then the examples can be written really simply.

```Swift
enum Either<First, Second>: CustomStringConvertible {
    case first(First)
    case second(Second)
  
    var description: String {
        switch self{
        case <T> 
        let .first(value as T), 
        let .second(value as T): 
            return "\(value)"
        }
    }   
}

func encode(to encoder: Encoder) throws {
    var container = encoder.container(keyedBy: CodingKeys.self)
    switch self{
    case <T: Encodable> 
    let .int(value as T), 
    let .string(value as T): 
        try container.encode(value, forKey: .value)
    }
}

enum Integer{
    case int64(Int64)
    case int32(Int32)
    case int16(Int16)
    case int8(Int8)

    var bitWidth: Int {
        switch self{
        case <T: BinaryInteger>
        let .int64(value as T),
        let .int32(value as T),
        let .int16(value as T),
        let .int8(value as T):
            return value.bitWidth
        }
    }
}
```

## Detailed design

Grammatically, this can be enabled by the following addition.

```
//current Swift
case-label → attributes(opt) 'case' case-item-list

//with generic case
case-label → attributes(opt) 'case' generic-parameter-clause(opt) case-item-list
```

To explain how it works in detail, consider the following enum.

```Swift
enum Things{
    case void
    case int(Int)
    case string(String)
    case doubles(Double, Double)
    case tuple(Int, String)
}
```

With generic-case, you can call generic functions with generic values.

```Swift
//OK
//you can call generic functions with `value: T`
switch thing{
case .void:
    print("never")
case <T> let .int(value as T), let .string(value as T):
    genericFunc(value)      //call genericFunc<T>
case <T, S> let .doubles(a as T, b as S), let .tuple(a as T, b as S):
    genericFunc(a)          //call genericFunc<T>
    genericFunc(a, b)       //call genericFunc<T,S>
}

//after here, without explicit declaration, the target value of pattern matching is `thing`

//OK
//of course this format is also allowed
case <T> .int(let value as T), .string(let value as T):

//Error
//you cannot call non-generic function with `value: T`
case <T> let .int(value as T), let .string(value as T):
    nongenericFunc(value)
```

You can add type constraints to generic-case. You can use patterns that don't satisfy the constraints, but it causes warning.

```Swift
//OK
//generic constraint works like in other generic contexts
case <T: Encodable> let .int(value as T), let .string(value as T): 
    //This operation is possible because T is generic type and not protocol existential
    try container.encode(value, forKey: .value)

//OK (Warning)
//causes warning that says 'type `Int` does not satisfy type constraints' and 'never match pattern `.int`'
case <T: StringProtocol> let .int(value as T), let .string(value as T): 
```

Type of target mast be `struct/enum/class` and not protocol existentials. This is from the same reason that protocol existentials cannot be applied to generic functions. 

```Swift
enum protocols{
    case encodable(Encodable) 
}

//Error
//target value must be struct/enum/class
case <T: Encodable> let .encodable(value as T):
```

You have to make bound variables have a unique type for type safety.

```Swift
//Error
//type of variables must be unique but `b` has both types `T` and `S`
case <T, S> let .double(a as T, b as T), let .tuple(a as T, b as S):
```

By writing `value as T`, the actual type of `T` is inferred as type of `value` . All type parameters must be uniquely inferred in each pattern. About this expression `value as T`, another discussion is held in [Type parameters specifiers](#Type-parameters-specifiers).

```Swift
//OK
//`T` is `Double`
case <T>
let .double(a as T, _ as T):

//OK (Warning)
//causes warning that says 'never match pattern `.tuple` because type of `a` is always not equal to type of `b`' 
case <T> let .double(a as T, b as T), let .tuple(a as T, b as T):

//Error
//type parameters must be inferred in each pattern
//this is not allowed because the type `T` is not always decidable in patterns
case <T> let .int(a): 

//Error
//type parameters must be inferred in each pattern
//this is not allowed because the type `S` and `U` are not always decidable in patterns
//also because type of variables must be unique but `b` has both types `S` and `U`
case <T, S, U> 
let .double(a as T, b as S), 
let .tuple(a as T, b as U):
```

Generic-case can be used for more than only with enums with associated values.

```Swift
switch 42{
//OK
case <T: Encodable> let value as T: 
    //`value` can be used as type `T` conforming to `Encodable`
default: 
}
```

Type checking is not allowed because type parameters cannot be inferred.

```Swift
//Error
//type parameters must be inferred in each pattern
//especially, it cannot be used as the alternative to `case is Encodable`
case <T: Encodable> is T:
```

Because multiple pattern matching with `,` is only allowed in switch statements, this generic-case cannot be used effectively with `if case` or `for case`. However, it should be allowed for consistency of syntax.

```Swift
//of course there are no needs to write like this!

//OK
if case <T> let .int(value as T) = thing{
    //use (value: T)
}

//OK
for case <T> let .int(value as T) in things{
    //use (value: T)
}
```

## Source compatibility

Generic-case is an additive feature that doesn't affect source compatibility.

## Effect on ABI stability

TBD

## Effect on API resilience

Generic-case is an additive feature that doesn't affect API resilience.

## Relating discussion

* [SE-0043 Declare variables in 'case' labels with multiple patterns](https://github.com/apple/swift-evolution/blob/main/proposals/0043-declare-variables-in-case-labels-with-multiple-patterns.md)

## Controversial points

### Type parameters specifiers

In generic-case, we use `value as T` as the type parameter specifier. Without some specifications, compiler cannot find which value should be used as which type.

However, this is not normal use of `as` in today's Swift. The next code works fine. Here, `as` is used as the type specifier. Writing this, we can tell compiler that 'the type of `3` is `Double`.'

```Swift
//specify type of integer literal as `Double`
let value = 3 as Double
```

However, in generic-case, the order is reversed. Writing this, we are telling compiler that '`T` is the type of `value`'.

```Swift
case <T> let .foo(value as T)
```

There are two reasons why I selected `as`.

1. Consistency with today's valid syntax.

   As above, following syntax is valid.

   ```Swift
   case let .int(value as Encodable)
   ```

   Therefore, it feels natural to me to inherit this syntax.

   ```Swift
   case <T: Encodable> let .int(value as T)
   ```

2. There are no other symbols suitable for this usage.

   At first I considered `:` as type specifier. 

   ```Swift
   case <T: Encodable> let .int(value: T)
   ```

   But If an enum has label, this fails soon.

   ```Swift
   enum Integer{
       case int(number: Int64)
   }
   //awful!
   case <T: Encodable> let .int(number: value: T)
   ```

   Also, with today's Swift, even though enum doesn't have label, developer can set label in pattern. Therefore, the next example works strangely.

   ```Swift
   //Swift5.3
   enum Integer{
       case int64(Int64)
   }
   case let .int64(value: Encodable):
       //here, not `value`, but `Encodable` works as the variable of type `Int64`
   ```

   Therefore, I didn't select `:` as the type specifier.

   Because there are no other ways to specify type parameters in Swift, I couldn't select other symbols.

I'm now thinking `as` is suitable for this use, but it would be a controversial point. As an alternative to the proposed syntax, another syntax was considered. This is discussed in the section [Variable Declarations](#variable-declaration).

### Allow explicit type parameter declaration

In some cases, it makes difference whether you explicitly declare type parameters or not.

```Swift
protocol Animal{
    func foo()
}
class Mammal: Animal {
    func foo(){
        print("mammal")
    }
}
class Dog: Mammal {
    override func foo(){
        print("dog")
    } 
}
class Cat: Mammal {
    override func foo(){
        print("cat")
    } 
}
enum Pet{
    case mammal(Mammal)
}

case <T: Animal> let <Dog> .mammal(animal as T):
    //here when match `.mammal`, `animal.foo()` says "dog"

case <T: Animal> let <Mammal> .mammal(animal as T):
    //here when match `.mammal`, it is not clear what `animal.foo()` says
```

Here, without explicit specification, you cannot cast `Mammal` to `Dog`. Therefore, it is valuable when you want to do such things. However, maybe the next two points are controversial.

#### Needs

The example is only an example, and I don't know whether there are such cases in practical situation or not. If there are no needs, it is simply wasteful.

#### Order of component

This style is widely used. Therefore, it is preferred to also allow this style in generic-case. 

```Swift
//this is allowed today
case .double(let a, let b):
    //operation when `thing` is `.double`
case .tuple(let a, let b):  
    //operation when `thing` is `.tuple`
```

To achieve this style, all of type parameters should be declared at once. However, it becomes ugly when you remove `\n`.

```Swift
//it is possible
case <T, S>
<Double, Double> .double(let a as T, let b as S),
<Int, String> .tuple(let a as T, let b as S):

//it is hard to read 
//especially, the part '<T, S> <Double, Double>' is awful
case <T, S> <Double, Double> .double(let a as T, let b as S), <Int, String> .tuple(let a as T, let b as S):
```

Also, it is confusing with this form.

```Swift
//it is wrong
case <T, S>
<Double, Double> let .double(a as T, b as S),
<Int, String> let .tuple(a as T, b as S):

//it is correct
case <T, S>
let <Double, Double> .double(a as T, b as S),
let <Int, String> .tuple(a as T, b as S):
```

It allowes you to specify 'dummy parameters' that isn't used in the case.

```Swift
case <A, B, C, D, E, F, G, H>
<Double, Double, Int, Int, Int, Int, Int, Int> let .double(a as T, b as S),
<Int, String, String, Int, Int, Int, Int, Int> let .tuple(a as T, b as S):
```

On type check, this should be allowed because type parameter `T` is declared. But no matter where to put the type parameters, it seems weird.

```Swift
case <T: Encodable> is T <String>:
case <T: Encodable> is <String> T:
case <T: Encodable> <String> is T:
```

I don't know what is the best order of the components `let`, `<Type parameters>`, `pattern`.

## Alternative considered

### Variable declaration

As the alternative to proposed syntax of generic-case, following syntax is considered.

```Swift
switch either{
case <T> (value: T) let .first(value), let .second(value):
    //here `value` has type `T`
    genericFunc(value)
}

//OK
//`value` is casted and then bound, so in the case `value` has type `T` and actual type is `Any`
case <T> (value: T) let .first(value as Any), let .second(value as Any):
```

Here, a new variable `value: T` is declared at the begining after type parameters, and then variable named `value` is bound in each pattern. If `value` is successfully bound and its type satisfies constraints of the declaration at the begining, `value: T` can be used inside the case.

The good point of this syntax is that it doesn't require `as`. As written in the controversial points section, **the use of `as` is somewhere misunderstanding**. In addition, the repetition of `value as T` seems redundant. Here, the variables's type is declared only once. Also, **explicit type parameter declaration** is not required. Because you can cast value, you can specify the type of value by simply writing `as Type` explicitly. Because this casting cannot be used for specifying dummy parameters, it would be well balanced solution to allow type casting and disallow specifying dummy parameters.

The bad point of this syntax is its strong expression. Because it has 'arguments-like' form, it is rational to assume that it can be used like others that use 'arguments-like' form, for example, functions, string interpolations and enums with associated values. Therefore, the next doubtful example should be alllowd.

```Swift
//is it allowed?
case <T> (a: T, b: Double? = nil) let .int(a), let .double(a, b):
```

Also, this syntax impacts pattern matching without generic-case. The next example should be allowed for the consistency of grammar because it is not natural to allow this syntax only with generic-case.

```Swift
//here `value` has type `Int` even if `.void` matches
case (value: Int = 0) .void, let .int(value), let .tuple(value, _):
```

Because this syntax is much larger addition than the proposed syntax, this idea wasn't selected.
