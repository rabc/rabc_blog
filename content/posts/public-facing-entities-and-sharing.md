---
title: "Public-Facing Entities and Sharing the Protocol"
date: 2019-07-07T13:07:34+02:00
draft: false
description: "How to create an API contract with public-facing objects and protocols"
tags: ["protocol","api","rest","sharing","vapor","swift"]
categories: ["patterns"]
---

_All the examples here are mode for Vapor 3, but could be applied anywhere else._

Everytime you are creating an API you are creating a new contract that says: `I promise to give all this agreed set of informations when requested, nothing less`. It is important to adhere to this because you will give predictability to whoever is consuming the API, don't making them assuming that something could not be present, therefore breaking its application.

This contract will not always be a full reflection of the database structure. You can have an `User` table, for example, with password and some other personal information that you should not expose in an API. It is in this moment that a _public-facing entity_ come at hand.

### Public-Facing Entities

Let's consider a table structure to an `User`:

```swift
final class User: SQLiteModel {
    var id: Int?
    var email: String
    var firstName: String
    var secondName: String
    var password: String
    var token: String
    
    init(id: Int?, email: String, firstName: String, secondName: String, password: String) {
        self.id = id
        self.email = email
        self.firstName = firstName
        self.secondName = secondName
        self.password = password
        self.token = // generate token here
    }
}
```

When you create an endpoint to retrieve the user's information, you may want to leave its password out of the response. Then you will have a new object to hold only the data you need to:

```swift
extension User {
	struct Public: Content {
		let id: Int
    	let email: String
    	let firstName: String
    	let secondName: String
    	let token: String

    	init(user: User) throws {
    		self.id = try user.requireID()
    		self.email = user.email
    		self.firstName = user.firstName
    		self.secondName = user.secondName
    		self.token = user.token
    	}
	}
}
```

The `Public` struct reflects the same structure of `User` but leaves the `password` out of it. Keeping it as an struct inside `User` gives a clarity of what it is and to who it belongs.

 The initializer will get all the information from an `User` object, what can give the option to change the information. For example, you may want to have just one user name field:

```swift
extension User {
	struct Public: Content {
		let id: Int
    	let email: String
    	let userName: String
    	let token: String

    	init(user: User) throws {
    		self.id = try user.requireID()
    		self.email = user.email
    		self.userName = "\(user.firstName) \(user.secondName)"
    		self.token = user.token
    	}
	}
}
```

### Public-Facing Protocols

An entity should not hold any business logic, as its purpose is to transmit the data. Creating a protocol for it gives more flexibility when defining the actual structure that will be transformed in the request or response JSON.

```swift
protocol UserPublicInformation: Decodable {
    var id: Int { get }
    var email: String { get }
    var userName: String { get }
    var token: String { get }
}
```

Conforming the protocol to `Decodable` already gives the information that it is in use only to output (response) information and conforming the previous `Public` structure to this protocol will guarantee that it will always have this minimum needed set of properties.

### Sharing the contract

Creating a pure-Swift code, without any specific logic or without conforming to any other specific protocol, enable this approach to be shared between serve and client.

All parts would agree in the properties and the struct, then the protocols can be in a separate repository, clearly marked and named, and versioned using tags.

This approach could enable a better understand of the API needs, on both sides, and more reliability on passing the data from one side to the other. Going even further, this repository could have the same protocol in two versions: Swift and Kotlin.

### Conclusion

Every API is a contract between server and client. Public-facing entities are the reflection of the public information from a model and defining its structure in a protocol could benefit the transmission of this information, making it more reliable and enabling a sharing of the protocol code between server and client.





