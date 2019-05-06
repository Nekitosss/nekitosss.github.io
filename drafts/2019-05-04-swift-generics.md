---
layout: post
title: Whats behind swift generic system?
subtitle: Generic boxing and specialization
tags: [swift, ios, foundation]
comments: true
---

> Generic code enables you to write flexible, reusable functions and types that can work with any type, subject to requirements that you define. You can write code that avoids duplication and expresses its intent in a clear, abstracted manner. - **Swift docs**

Everyone who wrote on Swift used generics. `Array`,`Dictionary`, `Set` are the most basic options for using generics from the standard library. Let's look at how this fundamental language feature is implemented by Apple engineers.

### Source code

To begin, lets create a *generic.swift* file and write a small generic function that we will consider. Let it be a very simple method that compares two passed arguments. It turns out just a wrapper over the comparison method.

```swift
func isEquals<T: Equatable>(first: T, second: T) -> Bool {
    return first == second
}
isEquals(first: 10, second: 11)
```

Now we need to understand what this ultimately turns into a compiler.

Swift uses two approaches to implement generics:

1. Runtime-way - generic code is a wrapper
2. Compiletime-way - generic code is converted to a specific type.

### Boxing

The swift compiler creates one single block of code that will be called to work with any `<T>`. That is, regardless of whether we write `isEquals(first: 1, second: 2)` or `isEquals (first: "Hello", second: "world")`, the same code will be called and additionally to the method information about the type `<T>` containing the necessary information has been transferred. This can be clearly seen by compiling our *.swift* file into *Swift Intermediate Language* or `SIL`.

#### Few words about SIL and compilation process

`SIL` is the result of one of several swift compilation steps.

![Raw Sil](/img/swift-generics/compiler-pipeline.gif){: .center-image }

The source code *.swift is passed to Lexer, which creates an abstract syntactic (`AST`) language tree, on the basis of which type checking and semantic code analysis is carried out. SilGen converts `AST` to `SIL`, called `raw SIL`, then optimizer converts raw sil file to optimized `canonical SIL`, which is passed to `IRGen` for conversion to `IR` - a special format understandable by `LLVM` which will be converted to `* .o` files compiled for a specific processor architecture. We will review our generic code at the stage of creating a `SIL` file.

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