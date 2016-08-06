Bridging in React Native
========================

On this post I assume you know the basics of React Native, and will focus on how the internals work when managing the communication between native and JavaScript.

Main Threads
------------

Before anything else, keep in mind that there are 3 "main" threads[^1] in React Native:

- The shadow queue: where the layout happens
- The main thread: where UIKit does its thing
- The JavaScript thread: where your JS code is actually running

Plus every native module has its own [GCD](https://developer.apple.com/library/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html) Queue unless it specifies otherwise (a more detailed explanation is coming).

[^1] The "shadow queue" is actually a GCD Queue rather than a thread, as the name suggests.

Native Modules
--------------

_if you don't know how to create a Native Module yet, I'd recommend you check the documentation before._

Here's an example `Person` native module, that both, receives calls from JavaScript and calls into JS.

```objc
@interface Person : NSObject <RCTBridgeModule>
@end

@implementation Logger

RCT_EXPORT_MODULE()

RCT_EXPORT_METHOD(greet:(NSString *)name)
{
  NSLog(@"Hi, %@!", name);
  [_bridge.eventDispatcher sendAppEventWithName:@"greeted"
                                           body:@{ @"name": name }];
}

@end
```

We are going to focus on these two macros, `RCT_EXPORT_MODULE` and `RCT_EXPORT_METHOD`, what they expand into, what are their roles and how does it work from there.

`RCT_EXPORT_MODULE([js_name])`
------------------------------

As the name suggests, it exports your modules, but what does export mean in this specific context? It means making the bridge aware of your module.

Its definition is actually pretty simple:

```objc
#define RCT_EXPORT_MODULE(js_name) \
  RCT_EXTERN void RCTRegisterModule(Class); \
  + (NSString \*)moduleName { return @#js_name; } \
  + (void)load { RCTRegisterModule(self); }
```

What does it do:

- It first declares `RCTRegisterModule` as an `extern` function, which means that the implementation of the function is not visible to the compiler but will be available at link time, than
- declares a method `moduleName`, that returns the optional macro parameter `js_name`, in case you want your module to have a name in JS other than the Objective-C class name, and last
- declares a `load` method (When the app is loaded into memory it'll call the `load` method for every class) that calls the above declared `RCTRegisterModule` function to actually make the bridge aware of this module.
