---
layout: post
title: Universal JSONDecoder
subtitle: Handling several cases in json keys simultaneously
gh-repo: Nekitosss/AnyCodingKey
gh-badge: [follow]
bigimg: /img/2019-04-28-any-codable/encoding-decoding.png
tags: [swift, ios, foundation]
comments: true
---

At the moment, the vast majority of mobile applications are client-server applications. Loading, synchronization and sending events are everywhere and the main way of interacting with the server is the exchange of data through the json format.

### Key decoding

The Foundation framework has two mechanisms for data serialization and deserization. The old one is `NSJSonSerialization` and the new one is `Codable`. The last one provides such a wonderful things as autogenerating keys for json data based on a structure (or class) that implements `Codable` (`Encodable`, `Decodable`) and an autogenerated initializer for data decoding.
Everything seems to be fine, you can use and rejoice, but the reality is not so simple. Quite often on the server you can find json like:

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

This is a real example from one of the project servers.
For the `JsonDecoder` class, you can specify how to work with snake_case keys, but what to do if we have UpperCamelCase, dash-snake-case, or even all in one, and do not like manually writing the keys (duh, such a boilerplate)?
Fortunately, Apple has provided the ability to configure key conversion before mapping them to the `CodingKeys` structure using the `JSONDecoder.KeyDecodingStrategy`. We will take advantage of this.
To begin with, we will create a structure that implements the `CodingKey` protocol, because there is no such structure in the standard library

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

Then it is necessary to process each case of our keys separately. Most common cases are snake_case, dash-snake-case, lowerCamelCase and UpperCamelCase. Check, run, everything works. Then we encounter a rather expected problem: abbreviations in camelCases (remember the numerous `id`, `Id`,`ID`). To make it work, you need to correctly convert them and introduce a rule - *abbreviations are converted to camelCase, keeping only the first letter large and myABBRKey will turn into myAbbrKey*.

This solution works great for combinations of several cases.

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

The next routine problem is the method of passing dates. There are a lot of microservices on the server, a few less commands, but also a decent amount and as a result we find ourselves in front of a bunch of date formats like “guys, I use the standartized”. In addition, someone sends the dates in a string, someone in Epoch-time. As a result, we again have combinations of string-numbers-timezone-millisecond-delimiters, and `DateDecoder` in iOS complains and requires a strict date format. The solution here is simple, just look through the signs of one format or another and combine them, eventually obtaining the necessary. These formats successfully and completely covered my cases:

{: .box-note}
**Note:** This is custom DateFormatter initializer. Its just set format to created formatter.

```swift
static let onlyDate = DateFormatter(format: "yyyy-MM-dd")
static let full = DateFormatter(format: "yyyy-MM-dd'T'HH:mm:ss.SSSx")
static let noWMS = DateFormatter(format: "yyyy-MM-dd'T'HH:mm:ssZ")
static let noWTZ = DateFormatter(format: "yyyy-MM-dd'T'HH:mm:ss.SSS")
static let noWMSnoWTZ = DateFormatter(format: "yyyy-MM-dd'T'HH:mm:ss")
```

We attach it to our decoder using `JSONDecoder.DateDecodingStrategy` and we get a decoder that processes almost anything and converts it into a iOS desired format.

### Performance tests

Tests were performed for json strings of 7944 bytes in size.

|  | convertFromSnakeCase strategy | anyCodingKey strategy |
| :------ |:--- | :--- |
| Absolute | 0.00170 | 0.00210 |
| Relative | 81% | 100% |

As we can see, the custom `Decoder` is slower applied by 20% due to the mandatory check of each key in json for the need to transform. However, this is a small price to pay for the absence of the need to explicitly set keys for data structures, implementing `Codable`. The number of boilerplate has been greatly reduced in the project with the addition of this decoder. Should you use it to save developer time, but worsen production? You decide.

See full code in [github](https://github.com/Nekitosss/AnyCodingKey) repo.