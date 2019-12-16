---
layout: post
title: Implement custom combine publishers
subtitle: Replace with publisher
tags: [swift, ios, combine]
comments: true
---

В данной статье рассмотрим взаимодействие Publisher, Subscriber и реализуем полезный кастомный паблишер и орператор, пока что отсутствующий в стандартной библиотеке.

Для начала мы будем два похожих случая рассматривать. Первый, когда последовательность завершается, не опубликовав ни одно значение. К примеру, мы точно ждём какой-то ввод от пользователя, а если его не произошло, нам нужно дополнительно что-то сделать. Второй, когда паблишер завершается, не опубликовав не одно значение. К примеру, вам нужно получить значение из базы данных, но его там нет. В императивном программировании в таком случае используется nil-coalescing оператор (`a ?? b`). На первый взгляд это по смысле разные вещи, но это можно реализовать с помощью простого оператора. В Combine мы хотим получить что-то вида:

```swift
// Первый случай:
userOutputPublisher
    .replaceEmpty(with: self.anotherPublisher)
    .sink(...)

// Второй случай:
fetchFromDatabasePublisher
    .compactMap({$0}) // Filter nil value
    .replaceEmpty(with: self.fetchFromAnotherStoragePublisher)
    .sink(...)
```

В стандартной реализации можно предоставить уже имеющееся значение, чтобы заменить пустое. К примеру, `fetchFromDatabasePublisher.compactMap({$0}).replaceEmpty(with: 10)`. Но возможность подставить туда ещё один Publisher и асинхронно достать данные оттуда отсутствует. Именно это мы и будем реализовывать.

## Реализация

### Как работает Combine

Предположим, что основные концепции фреймворка Combine вам известны и вкратце рассмотрим механизм подписки и как `Publisher`, `Subscriber` и `Subscription` работают между собой.

![New friends](/assets/img/2019-12-18-compine-replace-with-publisher/combine-mechanism.png){: .center-image }

1. `Subscriber` подписывается на `Publisher`
2. `Publisher` создаёт подписку (`Subscription`) и передаёт её обратно в `Subscriber` (`receive(subscription:)`)
3. `Subscriber` запрашивает значения из подписки, передавая количество значений, которые он хочет получить (`subscription.request(_:)`)
4. `Subscription` начинает работу и начинает посылать значения в `Subscriber` в метод `subscriber.receive(value:)`
5. Так происходит до момента, пока не будет передано в `Subscription`, что `Subscriber` больше не нуждается в значениях, либо наоборот, пока `Subscription` не передаст `Subscriber` информацию, что больше значений нет или проихошла ошибка (вызван метод `receive(completion:)`)

### Создаём свой Publisher

```swift
extension Publishers {
    public struct ReplaceEmptyWithPublisher<NewPublisher: Publisher, Upstream: Publisher>: Publisher where NewPublisher.Failure == Upstream.Failure, NewPublisher.Output == Upstream.Output {

        public typealias Output = Upstream.Output
        public typealias Failure = Upstream.Failure

        let upstream: Upstream
        let newPublisher: NewPublisher

        init(upstream: Upstream, newPublisher: NewPublisher) {
            self.upstream = upstream
            self.newPublisher = newPublisher
        }

        public func receive<S>(subscriber: S) where S : Subscriber, Failure == S.Failure, Output == S.Input {
            let subscription = OriginalSubscriber(upstream: upstream, downstream: newPublisher, subscriber: subscriber)
            subscriber.receive(subscription: subscription)
        }
    }
}
```

Наш созданный `Publisher` достаточно простой. Описываем, что он хранит у себя основной `UpstreamPublisher` (это наш `fetchFromDatabasePublisher`),  `NewPublisher` (это наш `fetchFromAnotherStoragePublisher`). В методе `receive` передаём нашу собственную подписку (`OriginalSubscriber`, код которой мы опишем ниже).
`subscriber`'ом здесь является кто-то, подписавшийся ниже, то есть, `sink`.
 На этом код основного паблишера заканчивается. Достаточно просто, не так ли?

### Создаём свой Subscriber

Код для `Subscriber` чуть сложнее, потому что там происходят основные действия. Необходимо принимать значения, посылать их в родительскую подписку, запоминать, были ли они получены.

Хранить в нём нужно всё: `upstream`, `newPublisher` и полученный `subscriber`. После того, как в `receive` методе паблишера мы отдали свой собственный `Subscriber`, в нашем сабскайбере вызовется метод:

```swift
func request(_ demand: Subscribers.Demand) {
    originalPublisher.receive(subscriber: self)
}
```

В нём мы сами уже подписываемся на исходным `Publisher`, чтобы инициировать его запуск и чтобы нам стали приходить значения.
После того, как мы подписались на изначальный паблишер, он отдаст нам подписку, с которой мы будем взаимодействовать (`Subscription`, см схемку на рисунке выше).

```swift
func receive(subscription: Subscription) {
    self.subscription = subscription // Сохраняем подписку
    subscription.request(.unlimited) // запрашиваем данные у основного паблишера
}
```

Если всё прошло хорошо, мы начнём получать данные из родительского паблишера. Нам необходимо запомнить, а были ли данные вообще. Если данные были, то при вызове `receive(completion:)`, мы прокидываем его дальше и завершаем работу. Если же данных ни разу не было, необходимо "разбудить" наш подменный `Publisher` и подписаться на него.

```swift
func receive(_ input: OriginalPublisher.Output) -> Subscribers.Demand {
    didReceiveAtLeastOneValue = true
    return subscriber.receive(input)
}
```

И наконец, в `receive(completion:)` мы смотрим, что нам ну/но вызвать дальше.

```swift
func receive(completion: Subscribers.Completion<OriginalPublisher.Failure>) {
    switch completion {
    case .failure,
            .finished where didReceiveAtLeastOneValue:
        subscriber.receive(completion: completion)
        subscription = nil
    default:
        let fallthroughSubscriber = FallthroughSubscriber(subscriber: subscriber)
        replacingPublisher.receive(subscriber: fallthroughSubscriber)
        subscription = nil
    }
}
```

Код `FallthroughSubscriber` достаточно прост. Он подписывается на второй, подменяющий `Publisher` и полностью проксирует результат вызова в уже имеющийся `sink subscriber`.

```swift
func receive(subscription: Subscription) {
    subscription.request(.unlimited)
}

func receive(_ input: Input) -> Subscribers.Demand {
    return subscriber.receive(input)
}

func receive(completion: Subscribers.Completion<Failure>) {
    subscriber.receive(completion: completion)
}
```

В завершение обернём всё это в удобный оператор, чтобы легко и непринуждённо вызывать всё из любого места.

``` swift
public func replaceEmpty<P: Publisher>(_ newPublisher: P) -> Publishers.ReplaceEmptyWithPublisher<P, Self> {
    return Publishers.ReplaceEmptyWithPublisher(upstream: self, newPublisher: newPublisher)
}
```

## Заключение

Чтобы разобраться, как работает Combine и как всё взаимодействует между собой, лучше сесть и попробовать реализовать что-то самому. Когда я только подошёл к этому вопросу, я уже использовал встроенные операторы Combine и представлял, как с ним работать, но имел весьма смутное представление, как это работает изнутри. Написание такого небольшого паблишера позволило немного разобраться с основными концепциями и лучше понять "внутреннюю кухню" библиотеки.

## Ссылки

<https://store.raywenderlich.com/products/combine-asynchronous-programming-with-swift>
