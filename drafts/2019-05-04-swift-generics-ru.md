---
layout: post
title: Whats behind swift generic system?
subtitle: Generic boxing and specialization
tags: [swift, ios, foundation]
comments: true
---

> Generic code enables you to write flexible, reusable functions and types that can work with any type, subject to requirements that you define. You can write code that avoids duplication and expresses its intent in a clear, abstracted manner. - **Swift docs**

Каждый, кто писал на Swift использовал дженерики. `Array`, `Dictionary`, `Set` - самые базовые варианты использования дженериков из стандартной библиотеке. Расмотрим, как данная основополагающая возможность языка реализована инженерами Apple.

### Исходный код

Для начала создадим *generic.swift* файл и напишем небольшую generic функцию, которую будем рассматривать. Пусть это будет очень простой метод, который сравнивает два переданных аргумента. Получится просто обёртка над методом сравнения.

```swift
func isEquals<T: Equatable>(first: T, second: T) -> Bool {
    return first == second
}
isEquals(first: 10, second: 11)
```

Теперь необходимо понять, во что это в итоге превращается компилятором.

Для реализации дженериков Swift использует два подхода:

1. Runtime-way - generic код является обёрткой
2. Compiletime-way - generic код преобразуется в код конкретного типа.

### Boxing

Компилятор swift создаёт один единственный блок кода, который будет вызываться для работы с любым `<T>`. То есть, независимо от того напишем мы `isEquals(first: 1, second: 2)` или `isEquals(first: "Hello", second: "world")`, будет вызван один и тот же код и дополнительно в метод будет передана информация о типе `<T>`, содержащая в себе необходимую информацию. Это можно наглядно посмотреть, скомпилировав наш *.swift* файл в *Swift Intermediate Language* или `SIL`.

#### Немного о SIL и процессе компиляции

`SIL` является результатом одним из нескольких этапов компиляции swift.

![Raw Sil](/img/swift-generics/compiler-pipeline.gif){: .center-image }

Исходный код *.swift передаеётся Lexer, который создаёт абстрактное синтаксическое (`AST`) дерево языка, на основе которого проводится проверка типов и семантический анализ кода. SilGen преобразует `AST` в `SIL`, называемый `raw SIL`, на основе которого происходит оптимизация кода и получается оптимизированный `canonical SIL`, который передаётся в `IRGen` для преобразования в `IR` - специальный формат, понятный `LLVM`, который будет преобразован в `*.o` файлы, собранные под конкретную архитектуру процессора. Во что превращается наш дженериковый код можно посмотреть как раз на этапе создания `SIL`.

#### И снова к дженерикам

Создадим `SIL` файл из нашего исходного кода.

```shell
swiftc generic.swift -O -emit-sil -o generic-sil.s
```

Получим новый файл с расширением `*.s`. Заглянув вовнутрь, мы увидим гораздо менее читаемый код, чем исходный, но, всё равно, относительно понятный. 

![Raw Sil](/img/swift-generics/raw-sil.png)

Найдём строку с комментарием `// isEquals<A>(first:second:)`. Это и есть начало описания нашего метода. Заканчивается он комментарием `// end sil function '$s4main8isEquals5first6secondSbx_xtSQRzlF'`. У вас название может немного отличаться. Немного разберём описание метода.

* `%0` и `%1` на 21 строке являются `first` и `second` параметрами соответственно
* На 24 строке получаем информацию о типе и  передаём в `%4`
* На 25 строке получаем указатель на метод сравнения из информации о типе
* на 26 строке Вызываем метод по указателю, передавая ему оба параметра и информацию о типе
* На 27 строке отдаём результат.

В итоге мы видим: чтобы выполнить необходимые действия в реализации дженерикового метода, нам нужно во время выполнения программы получать информацию из описания типа `<T>`, что является излишним действием, однако, это гибкий подход, позволяющий использовать generic систему, описывая реализацию метода только один раз.

### Generic specialization

Чтобы устранить излишнюю необходимость получения информации во время выполнения программы, был использован так называемый подход специализации дженериков. Он позволяет заменить дженериковую обёртку конкретным типом с конкретной реализацией. К примеру, для двух вызовов `isEquals(first: 1, second: 2)` и `isEquals(first: "Hello", second: "world")`, помимо основной "обёрточной" реализации, будут скомпилированы две дополнительные абсолютно разные версии метода для `Int` и для `String`.

В скомпилированном `SIL` файле сразу после объявления общего метода `isEquals` следует объявление специализированного для типа `Int`.

![Raw Sil](/img/swift-generics/specialized-sil.png)

На 39 строке вместо получения метода в рантайме из информации о типе сразу вызывается метод сравнения целых чисел `"cmp_eq_Int64"`.

Чтобы метод "специализировался", необходимо [включить оптимизацию](https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst#enabling-optimizations). Также, необходимо знать, что

> The optimizer can only perform specialization if the definition of the generic declaration is visible in the current Module ([Источник](https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst#advice-put-generic-declarations-in-the-same-module-where-they-are-used))

То есть, метод не может быть специализирован между разными модулями Swift (например, дженериковый метод из Cocoapods библиотеки). Исключением является стандартная библиотека Swift, в которой такие базовые типы, как `Array`, `Set` и `Dictionary`. Все дженерики базовой библиотеки специализируются в конкретные типы.

{: .box-note}
**Note:** В Swift 4.2 были реализованы аттрибуты `@inlinable` и `@usableFromInline`, которые позволяют оптимизатору видеть тела методов из других модулей и вроде как есть возможность их специализировать, но данное поведение мной не тестировалось ([Источник](https://github.com/apple/swift-evolution/blob/master/proposals/0193-cross-module-inlining-and-specialization.md))

### Ссылки

1. Описание дженериков <https://github.com/apple/swift/blob/master/docs/Generics.rst>
2. Оптимизация в Swift <https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst>
3. Более подробная и глубокая презентация по теме <https://www.youtube.com/watch?v=ctS8FzqcRug>