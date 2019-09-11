---
layout: list
title: Whats behind swift generic system?
subtitle: Swift under the hood
tags: [swift, iOS, foundation]
comments: true
---

> Generic code enables you to write flexible, reusable functions and types that can work with any type, subject to requirements that you define. You can write code that avoids duplication and expresses its intent in a clear, abstracted manner. - **Swift docs**

Everyone who wrote on Swift used generics. `Array`, `Dictionary`, `Set` are the most basic options for using generics from the standard library. Let's look at how this fundamental language feature is implemented by Apple engineers. Generic parameters can be both bounded to protocols (like in Java or C #) and not bounded, although mostly generics are used in conjunction with protocols that describe what exactly you can do with method parameters or type fields.

Swift uses two approaches to implement generics:

1. Runtime way - generic code is a wrapper (Boxing).
2. Compiletime way - generic code is converted to a specific type of code for optimization (Specialization).

## Boxing

Let's take a look on a simple method with an unlimited protocol, the generic parameter:

```swift
func test<T>(value: T) -> T {
    let copy = value
    print(copy)
    return copy
}
```

The swift compiler creates one single block of code that will be called to work with any `<T>`. That is, regardless of whether we write `test(value: 1)` or `test(value: "Hello")`, the same code will be called and, additionally, information about the type `<T>` will be passed to the method containing everything you need.

Not much can be done with such unbounded protocol parameters, but to implement this method, you need to know how to copy a parameter, you need to know its size, how to allocate memory for it in runtime, you need to know how to destroy it when the parameter leaves the visibility. To store this information `Value Witness Table` (`VWT`) is used. `VWT` is created at the compilation stage for all types and the compiler guarantees exact the same a layout of the object that will be in runtime. Recall that the structures in Swift are passed by value, and classes by reference, so for `let copy = value` with `T == MyClass` and `T == MyStruct` different things will be done.

![Raw Sil](/img/swift-generics/value-witness-table.gif){: .center-image }

That is, the call to the `test` method with the transfer of the declared structure there will end up looking like this:

```swift
// Approximate pseudocode, metadata parameter added by the compiler
let myStruct = MyStruct()
test(value: myStruct, metadata: MyStruct.metadata)
```

Things get a little more complicated when `MyStruct` is itself a generic structure and takes the form `MyStruct<T>`. Depending on `<T>` inside `MyStruct`, the metadata and `VWT` will be different for the types `MyStruct<Int>` and `MyStruct<Bool>`. These are two different types in runtime. But creating metadata for every possible combination of `MyStruct` and `T` is extremely inefficient, so Swift goes the other way and for such cases constructs metadata in runtime on the go. The compiler creates one *metadata pattern* for the generic structure, which can be combined with a specific type and, as a result, receive complete information on the type in runtime with the correct `VWT`.

```swift
// Again, pseudo-code, the metadata parameter is added by the compiler
func test<T>(value: MyStruct<T>, tMetadata: T.Type) {
    // We combine the information and get the final metadata
    let myStructMetadata = get_generic_metadata(MyStruct.metadataPattern, tMetadata)
    ...
}

let myStruct = MyStruct<Int>()
test(value: myStruct) // Source code
test(value: myStruct, tMetadata: Int.metadata) // Something like this is compiled
```

Когда мы комбинируем информацию, мы получаем метаданные, с которыми можно работать (копировать, перемещать, уничтожать).

Всё ещё немного сложнее, когда на дженерики добавляются ограничения в виде протоколов. К примеру, ограничим `<T>` протоколом `Equatable`. Пусть это будет очень простой метод, который сравнивает два переданных аргумента. Получится просто обёртка над методом сравнения.

```swift
func isEquals<T: Equatable>(first: T, second: T) -> Bool {
    return first == second
}
```

For the program to work properly, you must have a pointer to the `static func ==(lhs: T, rhs: T)` comparison method. How to get it? Obviously, the usage of `VWT` is not enough, it does not contain this information. To solve this problem, there is `Protocol Witness Table` or `PWT`. This table is similar to `VWT` and is created at compile time for protocols and describes these protocols.

```swift
isEquals(first: 1, second: 2) // Source code

// Something like this is compiled
isEquals(first: 1, // 1
         second: 2,
         metadata: Int.metadata, // 2
         intIsEquatable: Equatable.witnessTable) // 3
```

1. Two arguments are passed.
2. Passing metadata for `Int` so that you can copy/move/destroy objects
3. Passing information and that `Int` implements `Equatable`.

If the restriction required the implementation of another protocol, for example, `T: Equatable & MyProtocol`, then information about `MyProtocol` would be added with the following parameter:

```swift
isEquals(...,
        intIsEquatable: Equatable.witnessTable,
        intIsMyProtocol: MyProtocol.witnessTable)
```

Using wrappers to implement generics allows you to flexibly implement all the necessary features, but it has an overhead that can be optimized.

## Generic specialization

To eliminate the unnecessary requirement to obtain information during program execution, the so-called generics specialization approach was used. It allows you to replace the generic wrapper with a specific type with a specific implementation. For example, for two calls `isEquals(first: 1, second: 2)` and `isEquals(first: "Hello", second: "world")`, in addition to the main "wrapper" implementation, two additional completely different versions will be compiled method for `Int` and for `String`.

### Source code

To begin, create a *generic.swift* file and write a small generic function that we will consider.

```swift
func isEquals<T: Equatable>(first: T, second: T) -> Bool {
    return first == second
}
isEquals(first: 10, second: 11)
```

Now you need to understand what this ultimately turns into a compiler.
This can be clearly seen by compiling our *.swift* file into *Swift Intermediate Language* or `SIL`.

#### Few words about SIL and compilation process

`SIL` is the result of one of several swift compilation steps.

![Raw Sil](/img/swift-generics/compiler-pipeline.gif){: .center-image }

The source code *.swift is passed to Lexer, which creates an abstract syntactic (`AST`) language tree, on the basis of which type checking and semantic code analysis are carried out. SilGen converts `AST` to `SIL`, called `raw SIL`, then optimizer converts raw sil file to optimized `canonical SIL`, which is passed to `IRGen` for conversion to `IR` - a special format understandable by `LLVM` which will be converted to `* .o` files compiled for a specific processor architecture. We will review our generic code at the stage of creating a `SIL` file.

#### Back to generics

Create a `SIL` file from our source code.

```shell
swiftc generic.swift -O -emit-sil -o generic-sil.s
```

We will get a new file with the extension `*.s`. Looking inside, we will see a much less readable code than the source code, but still relatively understandable.

![Raw Sil](/img/swift-generics/raw-sil.png)

Find the line with the comment `// isEquals<A>(first: second:)`. This is the beginning of the description of our method. It ends with the comment `// end sil function '$s4main8isEquals5first6secondSbx_xtSQRzlF'`. You may have a slightly different name. Let's analyze the method a little.

* `%0` and `%1` on the 21st line are the `first` and `second` parameters respectively
* On line 24 we get information about the type and transfer to `%4`
* At line 25 we get a pointer to the comparison method from the type information
* On line 26 we call the method on the pointer, passing both parameters and type information to it
* At line 27, we give the result.

As a result, we see that in order to perform the necessary actions in the implementation of the generic method, we need to retrieve information from the type description `<T>` during the execution of the program, which is unnecessary action, however, it is a flexible approach that allows using the generic system describing the implementation of method only once.

### Generic specialization

To eliminate the unnecessary requirement to obtain information during program execution, the so-called generics specialization approach was used. It allows you to replace the generic wrapper with a specific type with a specific implementation. For example, for two calls `isEquals(first: 1, second: 2)` and `isEquals(first: "Hello", second: "world")`, in addition to the main "wrapper" implementation, two additional completely different versions will be compiled method for `Int` and for `String`.

In the compiled `SIL` file immediately after the declaration of the common `isEquals` method follows the declaration specialized for the type `Int`.

![Raw Sil](/img/swift-generics/specialized-sil.png)

On the 39th line, instead of getting the method in runtime, the method of comparing integers `"cmp_eq_Int64"` is immediately called from the information about the type.

To make the method "specialized", you must [enable optimization](https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst#enabling-optimizations). Also, you need to know that

> The optimizer can only perform specialization if the definition of the generic declaration is visible in the current Module ([Source](https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst#advice-put-generic-declarations-in-the-same-module-where-they-are-used))

That is, the method cannot be specialized between different Swift modules (for example, the generic method from the Cocoapods library). The exception is the standard Swift library, in which basic types such as `Array`, `Set` and `Dictionary`. All generic base libraries specialize in specific types.

{: .box-note}
**Note:** In Swift 4.2, the attributes `@inlinable` and `@usableFromInline` were implemented, which allow the optimizer to see the bodies of methods from other modules and it seems like there is an opportunity to specialize them, but this behavior was not tested by me. ([Source](https://github.com/apple/swift-evolution/blob/master/proposals/0193-cross-module-inlining-and-specialization.md))

### References

1. Generics description <https://github.com/apple/swift/blob/master/docs/Generics.rst>
2. Swift optimization tips <https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst>
3. More detailed and in-depth presentation <https://www.youtube.com/watch?v=ctS8FzqcRug>
