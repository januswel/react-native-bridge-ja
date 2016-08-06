Bridging in React Native
========================

On this post I assume you know the basics of React Native, and will focus on how the internals work when managing the communication between native and JavaScript.

Main Threads
------------

Before anything else, keep in mind that there are 3 "main" threads※ in React Native:

- The shadow queue: where the layout happens
- The main thread: where UIKit does its thing
- The JavaScript thread: where your JS code is actually running

Plus every native module has its own GCD Queue unless it specifies otherwise (a more detailed explanation is coming).

※ The"shadow queue" is actually a GCD Queue rather than a thread, as the name suggests.

Native Modules
--------------

_if you don’t know how to create a Native Module yet, I’d recommend you check the documentation before._

Here’s an example Person native module, that both, receives calls from JavaScript and calls into JS.

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

We are going to focus on these two macros, RCT_EXPORT_MODULE and RCT_EXPORT_METHOD, what they expand into, what are their roles and how does it work from there.
