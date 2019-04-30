---
layout: post
title: DITranquillity tutorial
subtitle: Resolve dependencies easily with lightweight dependency framework
gh-repo: ivlevAstef/DITranquillity
gh-badge: [star, fork, follow]
tags: [swift, dependency injection]
comments: true
---

Dependency Injection - довольно популярный паттерн, позволяющий гибко конфигурировать систему и правильно выстраивать зависимости компонентов этой системы друг от друга. Благодаря типизации, Swift позволяет использовать удобные фреймворки с помощью которых можно очень коротко описать граф зависимостей. Сегодня я хочу немного рассказать об одном из таких фреймворков - `DITranquillity`.

В данном туториале будут рассмотрены следующие возможности библиотеки:

* Регистрация типов
* Внедрение с помощью инициализатора
* Внедрение в переменную
* Циклические зависимости компонентов
* Использование библиотеки с `UIStoryboard`

### Описание компонентов

Приложение будет состоять из следующих основных компонентов: `ViewController`, `Router`, `Presenter`, `Networking` - это довольно общие компоненты в любом iOS приложении.

// Тут будет рисуночек

`ViewController` и `Router` будут внедряться друг в друга циклически.

### Подготовка

Для начала создадим Single View Application в XCode, добавим DITranquillity с помощью [CocoaPods](https://github.com/ivlevAstef/DITranquillity#via-cocoapods). Создадим необходимую иерархию файлов, затем добавим на Main.storyboard второй контроллер и соединим его с помощью `StoryboardSegue`. В итоге должна получиться следующая структура файлов:

![File Structure](/img/ditranquillity-tutorial/file-structure.png){: .center-image }

Создадим зависимости в классах следующим образом:

```swift
protocol Presenter: class {
    func getCounter(completion: @escaping (Int) -> Void)
}

class MyPresenter: Presenter {

    private let networking: Networking

    init(networking: Networking) {
        self.networking = networking
    }

    func getCounter(completion: @escaping (Int) -> Void) {
        // Implementation
    }
}
```

```swift
protocol Networking: class {
    func fetchData(completion: @escaping (Result<Int, Error>) -> Void)
}

class MyNetworking: Networking {
    func fetchData(completion: @escaping (Result<Int, Error>) -> Void) {
        // Implementation
    }
}
```

```swift
protocol Router: class {
    func presentNewController()
}

class MyRouter: Router {
    unowned let viewController: ViewController

    init(viewController: ViewController) {
        self.viewController = viewController
    }

    func presentNewController() {
        // Implementation
    }
}
```

#### Ограничения

В отличие от других классов, `ViewController` создается не нами, а библиотекой UIKit внутри реализации `UIStoryboard.instantiateViewController`, поэтому, пользуясь сторибордом, мы не можем внедрять зависимости в наследников `UIViewController` с помощью инициализатора. Так же дела обстоят и с наследниками `UIView` и `UITableViewCell`.

```swift
class ViewController: UIViewController {
    var presenter: Presenter!
    var router: Router!
}
```

Заметьте, что во все классы внедряются объекты, скрытые за протоколами. В этом одна из основных задачь внедрения зависимостей - сделать зависимости не от реализаций, а от интерфейсов. Это поможет в будущем предоставить разные реализации протоколов для переиспользования или тестирования компонентов.

### Внедрение зависимостей

После того, как все компоненты системы созданы, приступим к связи объектов между собой. В DITranquillity отправной точной является `DIContainer`, который добавляет в себя регистрации с помощью метода `container.register(...)`. Для разделения зависимостей на части используются `DIFramework` и `DIPart`, которые необходимо реализовать. Для удобства создадим только один класс `ApplicationDependency`, который будет реализовывать `DIFramework` и будет служить местом регистраций всех зависимостей. Интерфейс `DIFramework` обязывает реализовать только один метод - `load(container:)`.

```swift
class ApplicationDependency: DIFramework {
    static func load(container: DIContainer) {
        // registrations will be placed here
    }
}
```

Начнём с самой простой регистрации, у которой нет своих зависимостей - `MyNetworking`

```swift
container.register(MyNetworking.init)
```

Данная регистрация использует внедрение через инициализатор. Несмотря на то, что у самого компонента нет зависимостей, ининциализатор необходимо предоставить, чтобы дать понять библиотеке, как создавать компонент.

Аналогичным образом зарегистрируем `MyPresenter` и `MyRouter`.

```swift
container.register1(MyPresenter.init)
container.register1(MyRouter.init)
```

{: .box-note}
**Note:** Заметьте, что используется не `register`, а `register1`. К сожалению, так необходимо указывать, если объект имеет в инициализаторе одну и только одну зависимость. То есть, если зависимостей 0 или две и больше, необходимо использовать просто `register`. Данное ограничение является багом Swift версии 4.0 и больше.

Пришла пора регистрировать наш `ViewController`. Он внедряет объекты не через инициализатор, а напрямую в переменную, поэтому описание регистрации получится чуть больше.

```swift
container.register(ViewController.self)
    .injection(cycle: true, \.router)
    .injection(\.presenter)
```

Синтаксис вида `\.presenter` является SwiftKeyPath, благодаря которому можно лаконично внедрить зависимость. Так как `Router` и `ViewController` циклически зависят друг от друга, необходимо явно это указать с помощью `cycle: true`. Библиотека и сама может разрешить эти зависимости без явного указания, но данное требование было введено, чтобы человек, читающий граф, сразу понимал, что в цепочке зависимостей есть циклы. Так же обратите внимание, что используется **НЕ** `ViewController.init`, **но** `ViewController.self`. Об этом писалось выше в разделе *Ограничения*.

Также необходимо зарегистрировать `UIStoryboard` с помощью специального метода.

```swift
container.registerStoryboard(name: "Main")
```

Теперь у нас описан весь граф зависимостей для одного экрана. Но доступа к этому графу пока нет. Необходимо создать `DIContainer`, позволяющий получить доступ к объектам в нём.

```swift
static let container: DIContainer = {
    let container = DIContainer() // 1
    container.append(framework: ApplicationDependency.self) // 2
    assert(container.validate(checkGraphCycles: true)) // 3
    return container
}()
```

1. Инициализируем контейнер
2. Добавляем описание графа к нему
3. Проверяем, что мы всё сделали правильно. Если допущена ошибка, приложение упадёт не во время резолва зависимостей, а сразу при создании графа

Затем необходимо сделать контейнер отправной точкой старта приложения. Для этого в `AppDelegate` реализовываем метод `didFinishLaunchingWithOptions` вместо указания `Main.storyboard` как точко запуска в настройках проекта.

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    window = UIWindow(frame: UIScreen.main.bounds)
    let storyboard: UIStoryboard = ApplicationDependency.container.resolve()
    window?.rootViewController = storyboard.instantiateInitialViewController()
    window?.makeKeyAndVisible()
    return true
}
```

### Запуск

При первом запуске произойдёт падение и валидация не пройдёт по следующим причинам:

* Контейнер не найдёт типы `Router`, `Presenter`, `Networking`, потому что мы зарегистрировали только объекты. Если мы хотим дать доступ не к реализациям, а к интерфейсам, необходимо явно указать интерфейсы
* Контейнер не понимает, как ему разрешить циклическую зависимость, потому что необходимо явно указать, какие объекты при резолве графа не должны каждый раз пересоздаваться

Исправить первую ошибку просто - есть специальный метод, позволяющий указать, под какими протоколами доступен метод в контейнере.

```swift
container.register(MyNetworking.init)
    .as(check: Networking.self) {$0}
```

Описывая регистрацию так, мы говорим: объект `MyNetworking` доступен по протоколу `Networking`. Так нужно сделать для всех объектов, спрятанных под протоколами. `{$0}` добавляем для правильной проверки типов компилятором.

Со второй ошибкой чуть сложнее. Необходимо использовать так называемые `scope`, которые описывают, как часто создаётся и сколько живеёт объект. Для каждой регистрации, участвующей в циклической зависимости, необходимо указать `scope` равный `objectGraph`. Это даст понять контейнеру, что во время резолва необходимо переиспользовать одни и те же созданные объекты, а не создавать каждый раз заного. Таким образом, получится:

```swift
container.register(ViewController.self)
    .injection(cycle: true, \.router)
    .injection(\.presenter)
    .lifetime(.objectGraph)

container.register1(MyRouter.init)
    .as(check: Router.self) {$0}
    .lifetime(.objectGraph)
```

После повторного запуска контейнер успешно проходит валидацию и откроется наш ViewController с созданными зависимостями. Можете поставить брейкпоинт во `viewDidLoad` и удостовериться.

### Переход между экранами

Далее создадим два небольших класса `SecondViewController` и `SecondPresenter`, добавим `SecondViewController` на сториборд и создадим между ними `Segue` с идентификатором `"RouteToSecond"`, позволяющий открыть второй контроллер из первого.

Добавим в наш `ApplicationDependency` ещё две регистрации для каждого из новых классов:

```swift
container.register(SecondViewController.self)
    .injection(\.secondPresenter)

container.register(SecondPresenter.init)
```

Указывать `.as` нет необходимости, потому что мы не прятали `SecondPresenter` за протоколом, а пользуемся непосредственно реализацией. Затем в методе `viewDidAppear` первого контроллера вызываем `performSegue(withIdentifier: "RouteToSecond", sender: self)`, запускаем, открывается второй контроллер, в котором котором должна быть проставлена зависимость `secondPresenter`. Как видно, контейнер увидел создание второго контроллера из `UIStoryboard` и успешно проставил зависимости.

### Заключение

Данная библиотека позволяет удобно работать с циклическими зависимостями, сторибордом и полностью пользуется автовыводом типов в Swift, что даеёт очень короткий и гибкий синтаксис описания графа зависимостей.

#### Ссылки

Полный пример кода в [библиотеке на github](https://github.com/Nekitosss/DITranquillityTutorial)

DITranquillity [на github](https://github.com/ivlevAstef/DITranquillity)
