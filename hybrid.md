# hybrid
本文记录hybrid有关的知识和资料，主要分为3部分，JSCore，WKWebView，WebKit

## JavaScripeCore
[JavaScripeCore介绍](https://tech.meituan.com/2018/08/23/deep-understanding-of-jscore.html)

JSCore在hybrid开发中，存在两个使用场景：
1. 于WebKit中，用作js引擎，用以处理&执行前端的js代码
2. 于iOS中，作为iOS sdk提供的一个framework，供native生成一个可自定义的js运行环境，执行js代码

### JSCore于Webkit
JS引擎的功能主要由两部分组成：
1. 解释js代码为bitcode
2. 运行

js解释的过程，和python/传统java等解释语言的解释过程没啥diff，大体过程无外乎如下：
1. 词法分析：将代码拆分为token
2. 语法分析：将token组装为AST
3. bitcode生成：基于AST生成bitcode

每一份js代码，都是要先变成bitcode，才能开始被解释运行，解释运行并不是边词法分析边语法分析边运行的

当bitcode ready后，就开始进入了运行这一部分
解释器就会开始边运行边将bitcode翻译为机器码，同时运行过程中，对热点代码等进行JIT优化

另外，JSCore的其他部分，也会开始在运行时开始加入工作，比如开始进行内存管理（JSVM），GC之类的

### JSCore于iOS sdk
[applew文档](https://developer.apple.com/documentation/javascriptcore)

苹果将WebKit中的JSCore进行了封装，为app提供了使用上js的能力，并且app还可以对js环境进行自定义

JSCore的核心由5个部分组成，分别是JSVirtualMachine，JSContext，JSValue，JSExport，JSManagedValue

#### JSVirtualMachine
苹果对JSVirtualMachine的介绍：
1. A self-contained environment for JavaScript execution
2. to support concurrent JavaScript execution, and to manage memory for objects that bridge between JavaScript and Objective-C or Swift

第一点所表述的，JSVirtualMachine就是一个js运行环境，提供类似于内存管理，GC这类的底层功能
其中，每个JSVM都是一个独立的内存空间，不同的JSVM之间的内存地址是不能分享使用的，不然就会经典野指针
跨JSVM的js代码如果需要分享数据，则需要native作为中间层进行转译

第二点则想表明，JSVM可以执行并发，同时可以管理native与js之间的桥接对象
JSVM确实是可以执行并发，但是并发!=并行，如果是想要worker那种并行的效果，则需要开启两个JSVM
桥接对象指的是，native可以持有并使用js对象，js同时也可以持有并使用native对象，而这个场景下的内存管理，由JSVM作为中间人进行管理

#### JSContext
苹果对JSContext的介绍：
1. A JavaScript execution environment
2. to evaluate JavaScript scripts from Objective-C or Swift code; to access values that JavaScript defines or calculates; and to make native objects, methods, or functions accessible to JavaScript

JSContext也是一个运行环境，和JSVM有什么不同？
可以这么理解：JSContext提供的是写js代码所需要的上下文和功能，比如：string这个东西的定义，带有的方法
而JSVM提供的是更底层的东西，内存管理，GC等

于此，我们就能明白介绍中的第二点，如果没有JSContext，怎么能运行js代码呢，连string是个啥都不懂呢
通过JSContext的实例方法，native就可以在一个JSContext实例上，执行js代码，并获得结果

同时，native还可以向JSContext实例上添加native对象和方法，供js代码执行和调用，这个在hybrid场景下是十分常见的
如何添加？通过KVC

js和native是两种不同的语言，对象和方法怎么能相互使用呢？
这个问题在JSVM中其实也出现了，但是这个坑没有填，我们到这填一填

我们先来聊聊，对象，那么就要引入下一个话题，JSValue

#### JSValue
[applew文档](https://developer.apple.com/documentation/javascriptcore/jsvalue)
苹果对JSValue的介绍太长了，这里就抛个链接拉倒

JSValue对于native来说，代表着的就是一个js对象
在js中，什么都是一个对象，方法其实本质上也是个对象，因此，js方法在native看来，也是个JSValue

JSValue的核心在于，提供了一套方法，让我们可以把js对象，转换成native对象
比如，一个js string，可以通过这个JSValue实例的方法，变成一个NSString/Swift String
同样，一个native string，也可以通过JSValue的类方法，变成一个代表着js string的JSValue

通过JSValue这个转译器，native和js对象，就能相互被使用

那么这里顺带填一下跨JSVM数据传递的坑
跨JSVM的js代码如果需要分享数据，则需要native作为中间层进行转译
native转译的过程就是把JSValue变成native object，然后再变成JSValue

为什么要这样？
因为一个JSValue是绑定一个JSContext的，一个JSContext又是绑定一个JSVM，相当于一个JSValue绑定于一个JSVM
每个JSVM实例都是有自己独立的内存空间的，这个JSVM的JSValue的地址在另一个JSVM里必然是不会被识别的
所以想要跨JSVM，只能是先把JSValue变成native object，然后再在对应的JSContext实例中，生成新的JSValue

下一个问题就是方法
js的方法是个JSValue，ok，没有问题，那么native的方法呢？

#### JSExport
The protocol for exporting Objective-C objects to JavaScript
Implement this protocol to export your Objective-C classes and their instance methods, class methods, and properties to JavaScript code





#### JSManagedValue


#### 其他小细节
js对象的本质：字典。ECMA把对象定义为：无序属性的集合，其属性可以包含基本值、对象或者函数

globalObject：JSContext实际上就是globalObject的native壳，鉴于globalObject也是个对象，是个字典，因此对JSContext进行KVC，是可以将native对象绑定上js环境的