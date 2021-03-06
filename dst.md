Bridging in React Native
========================

この投稿では React Native の基礎を知っている方を対象に、ネイティブと JavaScript の通信時における内部の動作へ焦点をあてます。

メインスレッド
--------------

何よりも先に、 React Native では 3 つの主要なスレッド[^1]があることを覚えておいてください。

- シャドウキュー
    - コンポーネント再配置時に使用されます
- メインスレッド
    - UIKit が使用します
- JavaScript スレッド
    - あなたの JavaScript コードが実際に走ります

さらに、すべてのネイティブモジュールは指定されないかぎりそれぞれの [GCD](https://developer.apple.com/library/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html)[^2] キューを持っています ( 詳細についてはこれから述べます ) 。

[^1]: 名前を示すとおり、シャドウキューは実際にはスレッドというより GCD キューです
[^2]: 訳注： [日本語訳](https://developer.apple.com/jp/documentation/ConcurrencyProgrammingGuide.pdf)もあります

ネイティブモジュール
--------------------

_もしあなたがネイティブモジュールの作り方を知らない場合、先に[ドキュメント](http://facebook.github.io/react-native/docs/native-modules-ios.html)を確認されることをおすすめします。_

ここでは双方向、 JavaScript から呼ばれ JavaScript を呼ぶ、 `Person` というネイティブモジュールを例に挙げます。

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

これら 2 つのマクロ、 `RCT_EXPORT_MODULE` と `RCT_EXPORT_METHOD` について焦点をあてましょう。どんなものに展開されるか、その役割とは何か、そこからどのように動くかといったことです。

`RCT_EXPORT_MODULE([js_name])`
------------------------------

名前が示すとおり、あなたのモジュールをエクスポートします。が、このときの「エクスポート」とはどういう意味でしょうか ? これは「ブリッジ」にあなたのモジュールを認識させることなのです。

その定義はとてもシンプルなものです。

```objc
#define RCT_EXPORT_MODULE(js_name) \
  RCT_EXTERN void RCTRegisterModule(Class); \
  + (NSString \*)moduleName { return @#js_name; } \
  + (void)load { RCTRegisterModule(self); }
```

これは次に挙げることをしています。

- まず `RCTRegisterModule` を `extern` 関数として宣言しています。これは、関数の実装がコンパイラーから見えないですが、リンク時に使用可能であることを意味しています。
- 次に、任意のマクロパラメーター `js_name` を返す `moduleName` というメソッドを宣言しています。ここではあなたのモジュールが Objective-C のクラス名ではなく、 JavaScript での名前を持ってほしいからですね。
- 最後に、「ブリッジ」にこのモジュールを認識させるため、上で定義した `RCTRegisterModule` 関数を呼び出す `load` メソッドを宣言しています。アプリはメモリー上にロードされた際、すべてのクラスに対してこの `load` メソッドを呼び出します。

`RCT_EXPORT_METHOD(method)`
---------------------------

このマクロは「より興味深い」です。あなたのメソッドには何もつけ加えませんが、指定されたメソッド名の宣言に加えて、新しいメソッドを作成します。

新しく定義されるメソッドは例のようなものになります。

```objc
+ (NSArray *)__rct_export__120
{
  return @[ @"", @"log:(NSString *)message" ];
}
```

「いったいどうなってるんだ ?」というのはとてもいい反応です。

これは次の要素を連結して生成されています。

- プリフィクス `__rct_export__`
- 任意の `js_name`
    - 上の例は `js_name` が空の場合です
- 宣言されている行の行数
- [`__COUNTER__`](https://gcc.gnu.org/onlinedocs/cpp/Common-Predefined-Macros.html) マクロの値

このメソッドの目的は任意の `js_name` とメソッドシグネチャーを含む配列を返すだけです。名前の生成は単にメソッド名の衝突を避けているだけです。

[^3]: Objective-C のカテゴリーを使えば同じ名前を持つ 2 つのメソッドを生成することは技術的に可能です。ところが、実際には起こりえないはずですが、 Xcode は期待しない動作になると警告してきます。

実行時
------

これらすべての準備は「ブリッジ」へ情報を提供するためのものです。ですのでモジュールやメソッドなど、「ブリッジ」はエクスポートされているすべてのものを探すことができます。ただしすべてロード時にエクスポートされます。では、これらが実行時にどのように使われるか見ていきましょう。

次の図は「ブリッジ」初期化時の依存関係を表しています。

![initialization](https://raw.githubusercontent.com/januswel/react-native-bridge-ja/master/images/initialisation.png)

### モジュールの初期化

新しい「ブリッジ」インスタンスが生成された際、すべての `RCTRegisterModule` 関数がすることはあとで「ブリッジ」が探せるようにそのクラス自身を配列に追加することです。「ブリッジ」はモジュールの配列を参照して、すべてのモジュールについて「ブリッジ」上の自身への参照を格納し、「ブリッジ」への参照を提供するインスタンスを生成します。だから双方向と呼べるわけですね。そしてすべての他のモジュールから切り離すために新しいキューを与えないかぎり、どのキューで走るべきかを確認します。


```objc
NSMutableDictionary *modulesByName; // = ...
for (Class moduleClass in RCTGetModuleClasses()) {
  // ...
  module = [moduleClass new];
  if ([module respondsToSelector:@selector(setBridge:)]) {
    module.bridge = self;
  }
  modulesByName[moduleName] = module;
  // ...
}
```

### モジュールの設定

一度モジュールが初期化されると、バックグラウンドスレッドではそれぞれのモジュールのすべてのメソッドを列挙し、 `__rct_export__` ではじまるメソッドを呼び出します。こうすることでメソッドシグネチャーの文字列を得ることができます。パラメーターの型も知ることができるためこれは重要です。たとえば、実行時にはパラメーターが `id` 型だと知ることはできるでしょう。ただしこの方法ではそれが `NSString *` 型であることまでわかるのです。

```objc
unsigned int methodCount;
Method *methods = class_copyMethodList(moduleClass, &methodCount);
for (unsigned int i = 0; i < methodCount; i++) {
  Method method = methods[i];
  SEL selector = method_getName(method);
  if ([NSStringFromSelector(selector) hasPrefix:@"__rct_export__"]) {
    IMP imp = method_getImplementation(method);
    NSArray *entries = ((NSArray *(*)(id, SEL))imp)(_moduleClass, selector);
    //...
    [moduleMethods addObject:/* Object representing the method */];
  }
}
```

### JavaScript の実行環境を整える

JavaScript の実行環境は、より重い処理をバックグラウンドスレッドで実行可能な `-setUp` メソッドを持っています。たとえば JavaScriptCore の初期化などです。すべての実行環境ではなく、アクティブな実行環境のみが `setUp` の呼び出しを受けるため、いくらか処理の節約になります。

```objc
JSGlobalContextRef ctx = JSGlobalContextCreate(NULL);
_context = [[RCTJavaScriptContext alloc] initWithJSContext:ctx];
```

### JSON の設定を注入する

我々のモジュールの情報のみを持つ JSON の設定は次のようになります。

```objc
{
  "remoteModuleConfig": {
    "Logger": {
      "constants": { /* If we had exported constants... */ },
      "moduleID": 1,
      "methods": {
        "requestPermissions": {
          "type": "remote",
          "methodID": 1
        }
      }
    }
  }
}
```

これは JavaScript VM 上にグローバル変数の形で定義される格納場所です。そのため「ブリッジ」の JavaScript 側が初期化される際、モジュールを生成するためにこの情報を使うことができます。

### JavaScript コードの読みこみ

これはとても直感的ですね。指定されたすべてのプロバイダーからソースコードを読みこむだけです。たいていの場合、開発時はパッケージャーからソースをダウンロードし、本番ではディスクから読みこむことになるでしょう。

### JavaScript コードの実行

一度準備が整うと、 JavaScriptCore VM 上でアプリケーションのソースコードを読みこめるようになります。 VM はソースをコピーし、パースし、実行するでしょう。最初の実行ではすべての CommonJS モジュールを登録し、エントリーポイントのファイルを require します。


```objc
JSValueRef jsError = NULL;
JSStringRef execJSString = JSStringCreateWithCFString((__bridge
      CFStringRef)script);
JSStringRef jsURL = JSStringCreateWithCFString((__bridge
      CFStringRef)sourceURL.absoluteString);
JSValueRef result = JSEvaluateScript(strongSelf->_context.ctx,
    execJSString, NULL, jsURL, 0, &jsError);
JSStringRelease(jsURL);
JSStringRelease(execJSString);
```

JavaScript モジュール
---------------------

上で示された JSON 設定から生成されたモジュールは `react-native` の `NativeModules` オブジェクトを通して JavaScript から使用可能になります。例えば、

```js
var { NativeModules } = require('react-native');
var { Person } = NativeModules;

Person.greet('Tadeu');
```

これは次の仕組みで動作します。メソッドの呼び出しはモジュール名・メソッド名・すべての引数を含めてキューに積まれます。 JavaScript 実行の最後で、このキューが呼び出しを処理するネイティブ側に渡されるのです。

呼び出しサイクル
----------------

上のコードでモジュールを呼び出した場合、次の図に示されることがおこります。

![graph](https://raw.githubusercontent.com/januswel/react-native-bridge-ja/master/images/graph.png)

呼び出しはネイティブ側からはじまらなければなりません[^3]。実行にあたって `NativeModules` のメソッドを呼ぶことで JavaScript を呼び出します。 `NativeModules` はネイティブ側で実行される呼び出しをキューに積みます。 JavaScript 側が完了すると、ネイティブ側はキューに積まれた呼び出し群を参照し、それらを実行します。 JavaScript のコールバックや呼び出しは「ブリッジ」を経由して再び JavaScript 側で実行されます。その際、 `_bridge` インスタンスを使うことでネイティブモジュールを通した `enqueueJSCall:args:` の呼び出しが可能になります。

[^3]: 図は JavaScript を実行している途中を表しています

注意： React Native プロジェクトを追っている方はかつてネイティブ側から JavaScript 側の呼び出しにおいてもキューが同じように使われていたことをご存知かもしれません。それは vSYNC のたびに実行されるため、起動時間を短縮するために削除されました。

引数の型
--------

ネイティブ側から JavaScript 側を呼び出す場合、引数は JSON に変換されただけの `NSArray` として渡されます。が、 JavaScript 側からの呼び出しではネイティブ側の型が必要となります。 int, float, char など組み込み型を明示的にチェックするためです。しかし上で言及しているように、実行系がすべてのオブジェクトや構造体に対して `NSMethodSignature` から十分な情報を渡せるとは限りません。そのため、型情報を文字列として保存するのです。

メソッドシグネチャーから型情報を得るために正規表現を使います。また、オブジェクト変換のために [`RCTConvert`](https://github.com/facebook/react-native/blob/master/React/Base/RCTConvert.m) というユーティリティークラスを使います。これは標準でサポートされているすべての型のためのメソッドを持っています。 JSON を任意の型に変換するメソッドです。

`struct` でない限り、メソッドを動的に呼び出すには [`objc_msgSend`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ObjCRuntimeRef/#//apple_ref/c/func/objc_msgSend) を使います。 [`objc_msgSend_stret`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ObjCRuntimeRef/#//apple_ref/c/func/objc_msgSend_stret) の arm64 版が存在せず、 [`NSInvocation`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSInvocation_Class/) を使わざるを得ないからです。

すべての引数を変換するとすぐに、目的のモジュールとメソッドを呼び出すためにまた別の `NSInvocation` を使います。

次は例です。

```objc
// たとえば MyModule モジュールなどで次のメソッドを定義していた場合
RCT_EXPORT_METHOD(methodWithArray:(NSArray *) size:(CGRect)size) {}
```

```js
// そして JavaScript 側で次のように呼び出します
require('NativeModules').MyModule.method(['a', 1], {
  x: 0,
  y: 0,
  width: 200,
  height: 100
});
```

```objc
// JavaScript キューは次のようにネイティブ側へ送られます
// ** 呼び出しのキューなので、すべて配列であることを忘れないで下さい
@[
  @[ @0 ], // モジュール ID 群
  @[ @1 ], // メソッド ID 群
  @[       // 引数群
    @[
      @[@"a", @1],
      @{ @"x": @0, @"y": @0, @"width": @200, @"height": @100 }
    ]
  ]
];

// これは次の擬似コードのような呼び出しに変換されます
NSInvocation call
call[args][0] = GetModuleForId(@0)
call[args][1] = GetMethodForId(@1)
call[args][2] = obj_msgSend(RCTConvert, NSArray, @[@"a", @1])
call[args][3] = NSInvocation(RCTConvert, CGRect, @{ @"x": @0, ... })
call()
```

スレッド実行
------------

上で言及しているように、すべてのモジュールはデフォルトでそれぞれの [GCD](https://developer.apple.com/library/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html) キューを持っています。モジュールが実行されるべきキューの指定は `-methodQueue` を実装するか、 `methodQueue` プロパティを有効なキューにすることで可能です。例外は `RCTViewManager` を継承している `View Managers`[^4] です。これはデフォルトでシャドウキューを使います。また、`RCTJSThread` は特別で、キューではなくスレッドを使います。ちなみに `RCTJSThread` はただのプレイスホルダーです。

[^4]: 実は `View Managers` の親クラスは対象キューとしてシャドウキューを明示的に指定しているため、例外ではなかったりします

現在のスレッド実行における「ルール群」は次です。

- `-init` と `-setBridge` はメインスレッドで呼び出されることが保証されています
- すべてのエクスポートされたメソッドは対象キュー上で呼び出されることが保証されています
- `RCTInvalidating` プロトコルを実装した場合、 `invalidate` も対象キュー上で呼び出されることが保証されています
- `-dealloc` がどのスレッドから呼び出されるかの保証はありません

JavaScript 側から連続して呼び出しを受けた場合、対象キューによってまとめられ、並行に実行されます。

```objc
// `buckets` の中の `calls` を `queue` がまとめている
for (id queue in buckets) {
  dispatch_block_t block = ^{
    NSOrderedSet *calls = [buckets objectForKey:queue];
    for (NSNumber *indexObj in calls) {
      // 実際の呼び出し
    }
  };

  if (queue == RCTJSThread) {
    [_javaScriptExecutor executeBlockOnJavaScriptQueue:block];
  } else if (queue) {
    dispatch_async(queue, block);
  }
}
```

### 最後に

「ブリッジ」がどのように動作するか、概観から少し踏み込んでみました。より複雑なモジュールを作りたい方やコアフレームワークをより有用なものにしたい方の一助になれば幸いです。

不明点やもっと簡単な説明が欲しい場合、他のハナシ聞きたいなどありましたら、お気軽にどうぞ !!
