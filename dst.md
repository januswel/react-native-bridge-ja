Bridging in React Native
========================

http://tadeuzagallo.com/blog/react-native-bridge/

この投稿では React Native の基礎を知っている方を対象に、ネイティブと JavaScript の通信時における内部の動作へ焦点をあてます。

メインスレッド
--------------

何よりも先に、 React Native では 3 つのメインスレッド (注1) があることを覚えておいてください。

- シャドウキュー: コンポーネント再配置時に使用されます
- メインスレッド: UIKit が使用します
- JavaScript スレッド: あなたの JavaScript コードが実際に走ります

さらに、すべてのネイティブモジュールは指定されないかぎりそれぞれの [GCD](https://developer.apple.com/library/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html)(訳注1) キューを持っています ( 詳細についてはこれから述べます ) 。

注1: シャドウキューは実際にはスレッドというより GCD キューですが、この名前を提案しています
訳注1: [日本語訳](https://developer.apple.com/jp/documentation/ConcurrencyProgrammingGuide.pdf)もあります

ネイティブモジュール
--------------------

_もしあなたがネイティブモジュールの作り方を知らない場合、先に[ドキュメント](http://facebook.github.io/react-native/docs/native-modules-ios.html)を確認されることをおすすめします。_

ここでは双方向、 JavaScript から呼ばれ JavaScript を呼ぶ、 Person というネイティブモジュールを例に挙げます。

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

> We are going to focus on these two macros, RCT_EXPORT_MODULE and RCT_EXPORT_METHOD, what they expand into, what are their roles and how does it work from there.

これら 2 つのマクロ、 RCT_EXPORT_MODULE と RCT_EXPORT_METHOD について焦点をあてましょう。どんなものに展開されるか、その役割とは何か、そこからどのように動くかといったことです。

