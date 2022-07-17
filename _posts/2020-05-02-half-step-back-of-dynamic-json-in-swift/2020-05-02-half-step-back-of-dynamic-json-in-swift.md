---
layout: post
title: Half Step Back of Dynamic JSON in Swift
date: 2020-05-02
description: After implemented dynamic JSON in Swift, communicating with old school rapidly becomes the foremost, and half step back will be appropriate.
tag:
- Swift
---

## Origin

The first time I found heavy needs in accessing dictionary by dot notation, was dealing with some network APIs with extremely deep JSON. Even Python is enough simple with libraries support, such as `requests`, it's still annoying to tap endless square brackets and quotation marks, while no hint to avoid enter fault is much worse.

Therefore, inheriting from built-in Dictionary `dict`, using some magic methods like `__getattr__` and `__setattr__`, a new Dictionary `Dict` was created, which solved the dynamic access problem and is now extensively used.

However, things get more difficult when it comes to Swift. Swift is a type-safe language which is strict with those "uncertain" type like `Dictionary`. During the age when a key even needed `if let` check, the dynamic access was beyond imagination. Until recent, with a new feature was introduced, this requirement came to be satisfied.

## One Step Forward

When Swift was updated to 4.2, an attribute called `dynamicMemberLookup` came into sight. As mentioned in official documentation, with an implementation of `subscript(dynamicMemberLookup:)`, it's easy to access key paths like normal members. Soon the potential was explored and the JSON parser [DynamicJSON](https://github.com/saoudrizwan/DynamicJSON) became a reality.

Briefly the parser can initialize with `Data` or other basic type, and access member dynamically. Although it needs to choose a specific type manually and return an optional type at most time due to the type-safe feature, it's good enough for such a strict language to perform that dynamic.

## Half Step Back

After calming down from the celebration of dynamic practice in Swift, some engineering level concern has to be considered. One thing is that for most exist project, writing `struct` or `class` as model is common, which will increase too much labor; the other thing is that directly using the parser need set default the optional type, which it's bad for wrapping.

For these reasons, building a "bridge" will be a natural choice.

### Preparation

First we write a protocol for after connection:

```swift
protocol JSONable {
    init()
    
    init(_ json: JSON)
}
```

The first `init()` ensure the object with default value, and the second `init(_:)` is responsible for initializing `JSON` to normal object.

But there is a lack of it. If the object is an `Array`, we can't directly use the constructor and the extension come to the stage:

```swift
extension Array: JSONable where Element: JSONable {
    init(_ json: JSON) {
        self = json.array!.map { Element($0) }
    }
}
```

### Usage

To demonstrate the usage, I choose the API `https://learnappmaking.com/ex/users.json` from the [tutorial](https://learnappmaking.com/urlsession-swift-networking-how-to/) which was used for freshman learning:

```json
[
  {
    "first_name": "Ford",
    "last_name": "Prefect",
    "age": 5000
  },
  {
    "first_name": "Zaphod",
    "last_name": "Beeblebrox",
    "age": 999
  },
  {
    "first_name": "Arthur",
    "last_name": "Dent",
    "age": 42
  },
  {
    "first_name": "Trillian",
    "last_name": "Astra",
    "age": 1234
  }
]
```

And we can briskly write down the familiar model with our new constructor.

```swift
struct User: JSONable {
    var age: Int
    var firstName: String
    var lastName: String
    
    init() {
        self.age = 0
        self.firstName = ""
        self.lastName = ""
    }
    
    init(_ json: JSON) {
        self.age = json.age.int ?? 0
        self.firstName = json.first_name.description
        self.lastName = json.last_name.description
    }
}
```

For actual network request, create a function to connect model `T` and `data` through `JSON`: 

```swift
enum NetworkError: Error {
    case badURL, requestFailed, decodeFailed
}

func fetch<T: JSONable>(
    _ type: T.Type,
    from urlString: String,
    completion: @escaping (Result<T,NetworkError>) -> Void
) {
    guard let url = URL(string: urlString) else {
        completion(.failure(.badURL))
        return
    }
    
    URLSession.shared.dataTask(with: url) { data, response, error in
        DispatchQueue.main.async {
            guard let data = data, error == nil else {
                completion(.failure(.requestFailed))
                return
            }
            
            guard let json = try? JSON(data: data) else {
                completion(.failure(.decodeFailed))
                return
            }
            
            completion(.success(T(json)))
        }
    }.resume()
}
```

Finally we can test it, and four users will display if fetch data successfully:

```swift
fetch([User].self, from: url) { result in
    switch result {
    case .success(let users):
        for user in users {
            print("Age: \(user.age)")
            print("First Name: \(user.firstName)")
            print("Last Name: \(user.lastName)")
        }
    case .failure(let e):
        print(e)
    }
}
```

### Conclusion

It seems that the sample code is massive, even deviates from the intention of simplification via dynamically accessing members. But basing on those exist code, the change is relatively small, and has achieved a delicate balance between dynamic and safety.
