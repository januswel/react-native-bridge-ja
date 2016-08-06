Bridging in React Native
========================

この投稿では React Native の基礎を知っている方を対象に、ネイティブと JavaScript の通信時における内部の動作へ焦点をあてます。

メインスレッド
--------------

何よりも先に、 React Native では 3 つのメインスレッド[^1]があることを覚えておいてください。

- シャドウキュー
    - コンポーネント再配置時に使用されます
- メインスレッド
    - UIKit が使用します
- JavaScript スレッド
    - あなたの JavaScript コードが実際に走ります

さらに、すべてのネイティブモジュールは指定されないかぎりそれぞれの [GCD](https://developer.apple.com/library/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html)[^2] キューを持っています ( 詳細についてはこれから述べます ) 。

[^1]: 名前を示すとおり、シャドウキューは実際にはスレッドというより GCD キューです
[^2]: 訳注 [日本語訳](https://developer.apple.com/jp/documentation/ConcurrencyProgrammingGuide.pdf)もあります

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


これら 2 つのマクロ、 RCT_EXPORT_MODULE と RCT_EXPORT_METHOD について焦点をあてましょう。どんなものに展開されるか、その役割とは何か、そこからどのように動くかといったことです。

RCT_EXPORT_MODULE([js_name])

名前が示すとおり、あなたのモジュールをエクスポートします。が、このときの「エクスポート」とはどういう意味でしょうか ? これは「ブリッジ」にあなたのモジュールを認識させることなのです。

その定義はとてもシンプルなものです。

```objc
#define RCT_EXPORT_MODULE(js_name) \
  RCT_EXTERN void RCTRegisterModule(Class); \
  + (NSString \*)moduleName { return @#js_name; } \
  + (void)load { RCTRegisterModule(self); }
```

これは次に挙げることをしています。

- まず RCTRegisterModule を extern 関数として宣言しています。これは、関数の実装がコンパイラーから見えないですが、リンク時に使用可能であることを意味しています。
- 次に、任意のマクロパラメーター js_name を返す moduleName というメソッドを宣言しています。ここではあなたのモジュールが Objective-C のクラス名ではなく、 JavaScript での名前を持って欲しいからですね。
- 最後に、「ブリッジ」にこのモジュールを認識させるため、上で定義した RCTRegisterModule 関数を呼び出す load メソッドを宣言しています。アプリはメモリー上にロードされた際、すべてのクラスに対してこの load メソッドを呼び出します。
