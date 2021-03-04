---
layout: post
title: iOS/Swift Quick Reference
---


_This is a live document that will be updated with new topics. The aim is to create a collection of quick reference notes for review/practice._

## iOS

 - Static vs. dynamic libraries/frameworks:
 	Libraries are files that define pieces of code and data that are not a part of your Xcode target. Libraries fall into two categories based on how they are linked to the app:
	Static libraries — .a
	Dynamic libraries — .dylib
	Additionally, a special kind of library exists:
	Text Based .dylib stubs — .tbd
	Framework is a package that can contain resources such as dynamic libraries, strings, headers, images, storyboards etc. With small changes to its structure, it can even contain other frameworks. Such aggregate is known as umbrella framework.

 - Apple’s identifier for advertisers (IDFA) changes for iOS 14 deferred. Used to uniquely identify devices across apps. The change is that before the IDFA can be accessed by an app and passed to AdTech companies, users will have to opt in. AppTrackingTransparency framework will have to be used to access the IDFA.

 - With iOS 13 Apple now mandates that the VoIP notifications must be reported to CallKit framework.
 
 - 2 ways of implementing **deep linking** in IOS: URL scheme and Universal Links. Universal links allow users to intelligently follow links to content in app or to its website. 
 For URL schemes to work, Add URL types to project settings, and implement a method `func application(_ app: UIApplication, open url: URL, options: [UIApplicationOpenURLOptionsKey : Any] = [:]) -> Bool`. 
 For universal links, iOS reaches out to `site.com/apple-app-site-association` for appID(teamID.bundleID) and paths array. Enable associated domains setting in Xcode. Implement `public func application(_ application: UIApplication, continue userActivity: NSUserActivity, restorationHandler: @escaping ([Any]?) -> Void) -> Bool` in the AppDelegate.
 
 - Core Bluetooth: Applicable for BLE(low energy) devices. A Bluetooth device can be either central or peripheral: Central: the object that receives the data from a Bluetooth device, Peripheral: the Bluetooth device that publishes data to be consumed by other devices. Peripherals broadcast advertising packets, the job of the central is to scan for them, identify them and connect for more info. Peripheral’s data is organized into services(a collection of data and associated behaviors describing a specific function or feature of a peripheral) and characteristics(provides further details about a peripheral’s service). Most work done via delegate methods. The central is represented by `CBCentralManager` and its delegate is `CBCentralManagerDelegate`. `CBPeripheral` is the peripheral and its delegate is `CBPeripheralDelegate`. We can start scanning with centralManager only after state has switched to poweredOn. Check [UUID](https://www.bluetooth.com/specifications/gatt/services) for needed service and stop scanning once discovered and attempt to connect. After connection, discover services, and after that discover [characteristics](https://developer.apple.com/documentation/corebluetooth/cbcharacteristicproperties). Once this is done, you can get notified, or read values as specified for the characteristics.
 
### Concurrency

 - Operations and GCD are the 2 default options. Operations is built on top of GCD.
 - Terms: _Synchronous_ - caller waits for execution of called block, _Asynchronous_ - caller continues execution after dispatching block, _Serial_ - a queue that has a single associated thread such that execution of enqueued tasks is serial, _Concurrent_ - a queue that has multiple threads associated with it.
 - GCD is Apple’s implementation of C’s libdispatch library. All of the tasks that GCD manages for you are placed into GCD-managed first-in, first-out (FIFO) queues. Each task that you submit to a queue is then executed against a pool of threads fully managed by the system.
 - To work with Operations, subclass `Operation` to submit to an `OperationQueue`. The states possible are isReady, isExecuting, isCancelled, and isFinished which can be monitored by KVO. `BlockOperation` is a subclass given for concurrent execution of one or more closures on the default global queue. We can chain dependencies on operations and even pause/cancel/resume them. 
 
### Security

 - Secure enclave: The Secure Enclave is a secure coprocessor that includes a hardware-based key manager, which is isolated from the main processor to provide an extra layer of security. Keys can be stored in the secure enclave, but they must also have been generated there. Also, those keys cannot be directly accessed but operations can be performed on them. 
 - Keychain: It is the infrastructure and a set of APIs used by Apple operating systems and third-party apps to store and retrieve passwords, keys and other sensitive credentials. Keychain items are encrypted using two different AES-256-GCM keys: a table key (metadata) and a per-row key (secret key). Keychain metadata (all attributes other than kSecValue) is encrypted with the metadata key to speed searches while the secret value (kSecValueData) is encrypted with the secret key. The meta-data key is protected by the Secure Enclave but is cached in the application processor to allow fast queries of the keychain. The secret key always requires a round trip through the Secure Enclave. The Keychain is implemented as a SQLite database, stored on the file system. A securityd daemon manages accesses by apps and rights for it. 
 - SSL Pinning:  Transport Layer Security (TLS) protocol used to provide secure communications. SSL was an ancestor of TLS. Works in 3 phases; in the first phase, the client initiates a connection with the server, the client then sends the server a message, which lists the versions of TLS it can support along with the cipher suite it can use for encryption. The server responds with the selected cipher suite and sends one or more digital certificates back to the client. The client verifies that those digital certificates — certificates, for short — are valid. The second phase of verification begins, the client generates a pre-master secret key and encrypts it with the server’s public key — i.e., the public key included in the certificate. The server and client each generate the master secret key and session keys based on the pre-master secret key. That master secret key is then used in the last phase to decrypt and encrypt the information that the two actors exchange. SSL Certificate Pinning, or pinning for short, is the process of associating a host with its certificate or public key. Once you know a host’s certificate or public key, you pin it to that host. In other words, you configure the app to reject all but one or a few predefined certificates or public keys. App should include the digital certificate or the public key within your app’s bundle. 2 types: 1. Pin the certificate: You can download the server’s certificate and bundle it into your app. At runtime, the app compares the server’s certificate to the one you’ve embedded. 2. Pin the public key: You can retrieve the certificate’s public key and include it in your code as a string. At runtime, the app compares the certificate’s public key to the one hard-coded in your code.

 
## Swift

- struct vs class advice: Use structures by default. Use classes when you need Objective-C interoperability. Use classes when you need to control the identity of the data you're modeling. Use structures along with protocols to adopt behavior by sharing implementations.

### Swift 5.0

 - Result type
 	
 	An enum type that has _success_ and _failure_ as cases. Both use generics but failure must conform to type Error. It also has a get() method to get success value or throw the error.
 	
 - Raw Strings
 	
 	Start and end by the # symbol. ex: #"This is a "string"."#. The internal quotes don't end/start the string. Use #""" and """# for multi-line. Interpolation here has syntax \#().
 
 - String.StringInterpolation
 	
 	We can now extend StringInterpolation and add custom behaviour using methods like appendInterpolation() and appendLiteral methods.
 	Ref: [Apple proposal link](https://github.com/apple/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md)
 	Also read CustomStringConvertible, ExpressibleByStringLiteral, ExpressibleByStringInterpolation.

 - @dynamicCallable and @dynamicMemberLookup
 	
 	Marks a type as being dynamically callable. 
 	Must implement either of:
```  
 	func dynamicallyCall(withArguments args: [Int]) -> Double

	func dynamicallyCall(withKeywordArguments args: KeyValuePairs<String, Int>) -> Double
	
``` 


 - Future enum cases
 	[Swift enum proposal](https://github.com/apple/swift-evolution/blob/master/proposals/0192-non-exhaustive-enums.md). Allows a @unknown attribute for default case. Means that all future cases would go that route and enforces that the present declared cases all have valid statements. 
 	
 - Transforming and unwrapping dictionary values with compactMapValues()
 


### Swift 5.1

 - Can omit _return_ keyword from single-expression functions/
 - Default value of property would now also be implicitly generated for default, implicit initializer.
 - Universal Self in [Swift proposal](https://github.com/apple/swift-evolution/blob/master/proposals/0068-universal-self.md)
 - Opaque return types with the use of _some_ keyword in front of conforming return. ex: func returnEquatable() -> some Equatable. The benefits of this over simple protocols or generics are: Our functions decide what type of data gets returned, not the caller of those functions, We don’t need to worry about Self or associated type requirements, because the compiler knows exactly what type is inside, We don’t expose private internal types to the outside world, we can change the returned object in the future without need to change type.
 - Static and class subscripts, meaning they can now apply to  the type as well as the instance of a type
 - Matching optional enums against non-optionals
 - A new difference(from:) method that calculates the differences between two ordered collections – what items to remove and what items to insert. This can be used with any ordered collection that contains Equatable elements. 
 - A new init for creating uninitialized arrays. We can tell it the capacity we want, then provide a closure to fill in the values however we need. The closure will be given an unsafe mutable buffer pointer where we can write your values, as well as an inout second parameter that lets us report back how many values we actually used.
 

## Architecture/Design

 - Association/Composition/Aggregation: Association is a superset of the other two and simply implies a link between objects. Composition means A has an object of B, say as a property, and is responsible for its lifecycle. Aggregation is similar in that A has an object of B but it was passed in and is not responsible for its lifecycle(meaning the instance of B can live on without A). Association superset of Aggregation which is a superset of Composition.
 - Object-oriented design keywords: Encapsulation(Container), Abstraction(Hiding Complexity), Inheritance(Hierarchically extend functionality), Polymorphism(Many forms, overriding(run-time, same signature)/overloading(compile-time, different signature)).
 - SOLID: 
 

S - Single-responsiblity Principle: 
O - Open-closed Principle: Objects or entities should be open for extension but closed for modification.
L - Liskov Substitution Principle: This means that every subclass or derived class should be substitutable for their base or parent class.
I - Interface Segregation Principle: A client should never be forced to implement an interface that it doesn’t use, or clients shouldn’t be forced to depend on methods they do not use.
D - Dependency Inversion Principle: Decoupling. Entities must depend on abstractions, not on concretions. It states that the high-level module must not depend on the low-level module, but they should depend on abstractions.

 - REST vs SOAP: Representational state transfer (REST) is a set of architectural principles. Simple object access protocol (SOAP) is an official protocol. REST can leverage JSON, XML, HTML, txt, while SOAP uses XML. REST has 6 guidelines: client-server architecture, stateless communication between them, cacheable data, uniform interface between components, layered system, and code on demand(to extend functionality, optional).

 - Design patterns:
 
1. Creational design patterns:


	_Abstract Factory_: Creates an instance of several families of classes,
	_Builder_: Separates object construction from its representation,
	_Factory Method_: Creates an instance of several derived classes,
	_Object Pool_: Avoid expensive acquisition and release of resources by recycling objects that are no longer in use,
	_Prototype_: A fully initialized instance to be copied or cloned,
	_Singleton_: A class of which only a single instance can exist.

2. Structural design patterns:


	_Adapter_: Match interfaces of different classes
	_Bridge_: Separates an object’s interface from its implementation
	_Composite_: A tree structure of simple and composite objects
	_Decorator_: Add responsibilities to objects dynamically
	_Facade_: A single class that represents an entire subsystem
	_Flyweight_: A fine-grained instance used for efficient sharing
	_Private_: Class Data Restricts accessor/mutator access
	_Proxy_: An object representing another object

3. Behavioral design patterns


	Chain of responsibility: A way of passing a request between a chain of objects
	_Command_: Encapsulate a command request as an object
	_Interpreter_: A way to include language elements in a program
	_Iterator_: Sequentially access the elements of a collection
	_Mediator_: Defines simplified communication between classes
	_Memento_: Capture and restore an object's internal state
	_Null Object_: Designed to act as a default value of an object
	_Observer_: A way of notifying change to a number of classes
	_State_: Alter an object's behavior when its state changes

	_Strategy_: Encapsulates an algorithm inside a class
	_Template method_: Defer the exact steps of an algorithm to a 	subclass
	_Visitor_: Defines a new operation to a class without change

 - MVC/MVP/MVVM:
 

 	MVC: Model: Acts as the model for data
View : Deals with the view to the user which can be the UI
Controller: Controls the interaction between Model and View, where view calls the controller to update model. View can call multiple controllers if needed.
	MVP: Similar to traditional MVC but Controller is replaced by Presenter. But the Presenter, unlike Controller is responsible for changing the view as well. The view usually does not call the presenter.
	MVVM: The difference here is the presence of View Model. It is kind of an implementation of Observer Design Pattern, where changes in the model are represented in the view as well, by the VM. So there is a two-way data binding.

## Software development models

 - Agile: Agile is an iterative approach that delivers work in small, but consumable, increments. Requirements, plans, and results are evaluated continuously so teams have a natural mechanism for responding to change quickly.
 - Scrum is a framework that helps teams work together.


## References

 - [Libs vs. Frameworks](https://www.vadimbulavin.com/static-dynamic-frameworks-and-libraries/), [performance comparison](https://medium.com/@acecilia/static-vs-dynamic-frameworks-in-swift-an-in-depth-analysis-ff61a77eec65) and [details](https://stackoverflow.com/a/59216151)	
 - [GCD vs. Operations](https://www.raywenderlich.com/books/concurrency-by-tutorials/v2.0/chapters/2-gcd-vs-operations)
 - [Apple Security doc](https://support.apple.com/en-in/guide/security/welcome/web)
 - [Swift 5.1 updates](https://www.hackingwithswift.com/articles/182/whats-new-in-swift-5-1)
 - [Swift 5.0 updates](https://www.hackingwithswift.com/articles/126/whats-new-in-swift-5-0)
 - [SOLID](https://www.digitalocean.com/community/conceptual_articles/s-o-l-i-d-the-first-five-principles-of-object-oriented-design)
 - [REST vs SOAP](https://www.redhat.com/en/topics/integration/whats-the-difference-between-soap-rest)
 - [Design patterns](https://sourcemaking.com/design_patterns)
 - [Agile](https://www.atlassian.com/agile)
 - [Choosing Between Structures and Classes](https://developer.apple.com/documentation/swift/choosing_between_structures_and_classes)
 - [Deep linking](https://medium.com/wolox/ios-deep-linking-url-scheme-vs-universal-links-50abd3802f97)
 - [Core Bluetooth](https://developer.apple.com/documentation/corebluetooth) and [here](https://www.raywenderlich.com/231-core-bluetooth-tutorial-for-ios-heart-rate-monitor).