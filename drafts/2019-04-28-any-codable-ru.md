---
layout: post
title: Universal JSONDecoder
subtitle: Handling several cases in json keys simultaneously
gh-repo: Nekitosss/AnyCodingKey
tags: [swift, ios, foundation]
comments: true
---

На данный момент подавляющее большинство мобильных приложений являются клиент-серверными. Повсюду происходит подгрузка, синхронизация, отправка событий и основным способом взаимодействия с сервером является обмен данными посредством формата json.

### Key decoding

В Foundation присутствует два механизма сериализации-десереадизации данных. Старый - `NSJSonSerialization` и новый – `Codable`. Последний в списке достоинств имеет в себе такую замечательную вещь, как автогенерация ключей для json данных на основе структуры (или класса), реализующей `Codable` (`Encodable`, `Decodable`) и инициализатора для декодирования данных.
И вроде бы всё прекрасно, можно использовать и радоваться, но реальность не так проста. Довольно часто на сервере можно встретить json вида:

```json
{"topLevelObject":
  {
    "underlyingObject": 1
  },
  "Error": {
    "ErrorCode": 400,
    "ErrorDescription": "SomeDescription"
  }
}
```

Это реальный пример с одного из серверов по проекту.
Для класса `JsonDecoder` можно указать работу со snake_case ключами, но что делать, если у нас UpperCamelCase, dash-snake-case или вообще сборная солянка, а вручную ключи писать не хочется?
К счастью, Apple предоставила возможность конфигурировать преобразование ключей перед их сопоставлением с `CodingKeys` структуры с помощью `JSONDecoder.KeyDecodingStrategy`. Этим мы и воспользуемся.
Для начала создадим структуру, реализующую протокол `CodingKey`, потому что таковой нет в стандартной библиотеке

```swift
  struct AnyCodingKey: CodingKey {

    var stringValue: String
    var intValue: Int?

    init(_ base: CodingKey) {
      self.init(stringValue: base.stringValue, intValue: base.intValue)
    }

    init(stringValue: String) {
      self.stringValue = stringValue
    }

    init(intValue: Int) {
      self.stringValue = "\(intValue)"
      self.intValue = intValue
    }

    init(stringValue: String, intValue: Int?) {
      self.stringValue = stringValue
      self.intValue = intValue
    }
  }
```

Затем необходимо отдельно обработать каждый case наших ключей. Основные:
snake_case, dash-snake-case, lowerCamelCase и UpperCamelCase. Проверяем, запускаем, всё работает.
Затем сталкиваемся с довольно ожидаемой проблемой: аббревиатуры в camelCase’ах (вспомните многочисленные `id`, `Id`, `ID`). Чтобы всё заработало, необходимо правильно их преобразовать и ввести правило – *аббревиатуры преобразуются в camelCase, сохраняя большой только первую букву и myABBRKey превратится в myAbbrKey*.
Данное решение отлично работает и для сочетаний нескольких кейсов.

```swift
static func convertToProperLowerCamelCase(keys: [CodingKey]) -> CodingKey {
  guard let last = keys.last else {
    assertionFailure()
    return AnyCodingKey(stringValue: "")
  }
  if let fromUpper = convertFromUpperCamelCase(initial: last.stringValue) {
    return AnyCodingKey(stringValue: fromUpper)
  } else if let fromSnake = convertFromSnakeCase(initial: last.stringValue) {
    return AnyCodingKey(stringValue: fromSnake)
  } else {
    return AnyCodingKey(last)
  }
}
```

### Date decoding

Следующая рутинная проблема – способ передачи дат. Микросервисов на сервере много, команд чуть меньше, но тоже приличное количество и в итоге мы оказываемся перед кучей форматов дат вида «да я стандартную использую». К тому же, кто-то передаёт даты строкой, кто-то в Epoch-time. В итоге у нас снова оказывается сборная солянка из сочетаний строки-числа-таймзоны-миллисекунд-разделителей, а `DateDecoder` в iOS жалуется и требует строгий формат дат. Решение тут несложное, просто перебором ищем признаки того или иного формата и комбинируем их, получая в итоге необходимый. Данные форматы успешно и стопроцентно покрывали мои кейсы.

{: .box-note}
**Note:** This is custom DateFormatter initializer. Its just set format to created formatter.

```swift
static let onlyDate = DateFormatter(format: "yyyy-MM-dd")
static let full = DateFormatter(format: "yyyy-MM-dd'T'HH:mm:ss.SSSx")
static let noWMS = DateFormatter(format: "yyyy-MM-dd'T'HH:mm:ssZ")
static let noWTZ = DateFormatter(format: "yyyy-MM-dd'T'HH:mm:ss.SSS")
static let noWMSnoWTZ = DateFormatter(format: "yyyy-MM-dd'T'HH:mm:ss")
```

Пристёгиваем это к нашему декодеру с помощью `JSONDecoder.DateDecodingStrategy` и получаем декодер, который обрабатывает почти всё, что угодно и преобразует в удобоваримый для нас формат.

### Тесты производительности

Тесты производились для json строки размером 7944 байта.

|  | convertFromSnakeCase strategy | anyCodingKey strategy |
| :------ |:--- | :--- |
| Absolute | 0.00170 | 0.00210 |
| Relative | 81% | 100% |

Как мы видим, кастомный `Decoder` медленее примено на 20% из-за обязательной проверки каждого ключа в json на необходимость трансформирования. Однако, это является небольшой платой за отсутствие необходимости явно прописывать ключи для структур данных, реализуя `Codable`. Количество боилерплейта очень сильно сократилось в проекте с добавлением данного декодера. Стоит ли его использовать, чтобы сохранить время разработчика, но ухудшить проихводительность? Решать вам.

Полный пример кода в [библиотеке на github](https://github.com/Nekitosss/AnyCodingKey)