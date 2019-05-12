---
title: "Making a Promise to the Future"
date: "2019-05-12"
description: "Understanding Futures & Promises in SwiftNIO"
tags: ["swift-nio","futures","promises"]
categories: ["swift-nio"]
---

SwiftNIO is pretty much a low-level framework, where you need to understand concepts from the event-driven and network worlds, but ultimately what it is exposed in final frameworks like Vapor are its Futures and Promises events. 

[John Sundell](https://twitter.com/johnsundell) has a good definition about it:

* A Promise is something you make to someone else.
* In the Future you may choose to honor (resolve) that promise, or reject it.

Go read [his post](https://www.swiftbysundell.com/posts/under-the-hood-of-futures-and-promises-in-swift) about it for a great introduction of the concept. After that, you will better understand the examples here.

### Starting a new event loop

Every promise is created from an event loop since we need somewhere (a.k.a. a thread) where to start the operation, watch its events and return the result (wheter it succeeds or fails) in the _future_. Luckily, SwiftNIO comes out-of-the-box with a lot of helpers and classes ready to use, so we don't need to setup a lot of boilerplate all the time.

`MultiThreadedEventLoopGroup` is one of those handy classes. It does pretty much what you can understand by the name: spawns _n_ threads, each one tied to an event loop. And yes, it is easy to start a new instance:

```swift
let group = MultiThreadedEventLoopGroup(numberOfThreads: 1)
```

### Creating a new future

A basic example of asynchronous processing is doing a request to an URL: 

```swift
guard let url = URL(string: "https://httpbin.org/get") else {
    // ☹️
}

let dataTask = URLSession.shared.dataTask(with: url) { (data, response, error) in
    if let error = error {
        // ☹️
    }
    
    let statusCode = (response as! HTTPURLResponse).statusCode
    guard 200..<300 ~= statusCode else {
        // ☹️
    }
    
    guard let data = data, let result = String(data: data, encoding: .utf8) else {
        // ☹️
    }
    
    // 🎉
}

dataTask.resume()
```

In this flow, we have four possibilities of failure and one of success, all that will happen in an unknown future. To fulfill those, we need a new promise from our event loop:

```swift
let eventLoop = group.next() // get the next available event loop in the group
let promise = eventLoop.newPromise(of: String.self) // create new promise of String type
```

Every new promise must have a type, that will be the result of its success. It doesn't mean that you always need a type: `Void.self` is also a valid type in the Swift world, in case you don't need to return anything. With this new promise, now it is possible to fulfill our promises to the future:

{{< highlight swift "linenos=table" >}}
func makeRequest(eventLoop: EventLoop) -> EventLoopFuture<String> {
    let promise = eventLoop.newPromise(of: String.self)
    
    guard let url = URL(string: "https://httpbin.org/get") else {
        promise.fail(error: RequestError.urlError)
        return promise.futureResult
    }
    
    let dataTask = URLSession.shared.dataTask(with: url) { (data, response, error) in
        if let error = error {
            promise.fail(error: error)
            return
        }
        
        let statusCode = (response as! HTTPURLResponse).statusCode
        guard 200..<300 ~= statusCode else {
            promise.fail(error: RequestError.requestError(status: statusCode))
            return
        }
        
        guard let data = data, let result = String(data: data, encoding: .utf8) else {
            promise.fail(error: RequestError.responseError)
            return
        }
        
        promise.succeed(result: result)
    }

    eventLoop.execute {
        dataTask.resume()
    }
    
    return promise.futureResult
}
{{< / highlight >}}

I wrapped the logic in a new function for clarity. 

In lines 6 and 33 we are returning the *future result* of the created promise. It is an `EventLoopFuture` that will be fulfilled in the lines 5, 11, 17, 22 and 26. 

Line 29 is where we execute our task inside the event loop. Remember when I said that SwiftNIO comes with usefult helpers? The `EventLoop` protocol has a `submit(_:)` function that already creates the promise and execute the task inside a closure, although it is not useful for our example here since `URLSession` do the request asynchronous. Nevertheless, it is worth taking a look.

Now, how do we handle those results?

### Handling the future

Going back to the first examples, we now have something like that:

```swift
let group = MultiThreadedEventLoopGroup(numberOfThreads: 1)
let futureRequest = makeRequest(eventLoop: group.next())
```

Using the autocomplete you can see that the `EventLoopFuture` in `futureRequest` has a lot of options. I highly recommend you to go through all the list and check all the methods to know more of all the options that SwiftNIO provides. For example, we can use `whenSuccess` to handle the success case:

```swift
futureRequest.whenSuccess { (response) in
    print(response)
}
```

But then what happens when we run the program? A `Program ended with exit code: 0` happens. Since it is all asynchronous, Unix systems does not know when the program is supposed to finish. We have to explicity _wait_ for the future:

```swift
do {
    let response = try futureRequest.wait()
    print(response)
} catch {
    print(error)
}
```

The `wait` instruction will block the current thread until the promise is fulfilled and return the result (or throw the error). Other frameworks, like Vapor for example, handles it and wraps the event loops, so the functions usually are expected to return a `Future` with your processing.

### Conclusion

SwiftNIO is a powerful abstraction, it is pushing forward the possibilities of Swift beyond the mobile world and taking a stand in the event-driven world. Although it has a [long way](https://github.com/apple/swift-nio/issues) to go (58 issues at the time of writing), it is already been widely used with great success and is ready for the prime time.



