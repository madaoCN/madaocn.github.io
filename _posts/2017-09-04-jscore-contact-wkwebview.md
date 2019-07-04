---
layout: post
title: JavaScriptCore与JS交互笔记
date: 2017-09-04
Author: madaoCN
categories: 
tags: [源码分享]
comments: true
---

JavaScriptCore 可以允许开发者在App内执行js脚本，
框架提供在Object-c，swift中执行javascript的能力，同时，允许javascript环境中插入自定义对象

```
// Classes 主要类
JSContext
JSManagedValue
JSValue
JSVirtualMachine
// 协议
JSExport
```


![javascriptcore.png](http://upload-images.jianshu.io/upload_images/1749699-7612fb95dd580e33.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图为类之间的关系架构

* JSValue
`JSValue ` 包括一系列方法用于访问其可能的值以保证有正确的 Foundation 类型，OC和JS对象之间的转换也是通过它
类型转换表：

| Object-C        | Swift           | Javascript  |
| ------------- |:-------------:| -----:|
|nil      |nil | undefined |
| NSNull     | NSNull      |   null |
| NSString | String     |    String |
| NSNumber | NSNumber     |    Number |
| BOOL | Bool     |    Boolean |
| NSDictionary | Dictionary     |    Object |
| NSArray | Array     |    Array |
| NSData | NSData     |    Date |
| objc_object | AnyObject     |    Object |
| Class | AnyClass     |    Object |
| NSRange,CGRect,CGPoint,CGSize | 同左     |    Object |
| block  | closure     |    Object |

* JSManagedValue
主要用来保存`JSValue `对象,解决OC对象中存储js的值，导致的循环引用问题。将`JSValue ` 转为`JSManagedValue ` 类型后，可以通过`addManagedReference `方法添加到`JSVirtualMachine ` 对象中，这样能够保证你在使用过程中`JSValue ` 对象不会被释放掉，当不再需要使用该`JSValue ` 对象后，通过`removeManagedReference `移除`JSManagedValue ` 对象的引用，`JSValue `对象就会随之被释放
* JSContext
`JSContext `代表着一个js的执行环境，可以在此环境中执行js代码，也可以把native 对象，方法，函数暴露给JavaScript,`JSContext `之间可以互相传递参数

```swift
let context = JSContext.init()//初始化，默认生成JSVirtualMachine
let vtMachine = JSVirtualMachine.init()
let context1 = JSContext.init(virtualMachine: vtMachine)//使用JSVirtualMachine初始化
```

* JSVirtualMachine
`JSVirtualMachine `的实例代表着一个独立的js执行环境，一般情况下不会接触到它。主要在如下两个场景中使用它：
1、 需要支持js脚本的并发
每个`JSContext `都从属于一个`JSVirtualMachine `，一个`JSVirtualMachine `可能包含多个`JSContext `(如图：javascriptcore.png)，但是`JSVirtualMachine `是独立的，无法在`JSVirtualMachine `之间传递对象。
JavaScriptCore的API 是线程安全的，你可以在线程中创建执行脚本的`JSValue `，但是多个线程使用同一个`JSVirtualMachine `会进入等待状态，所以需要并发执行js，需要为每个线程创建一个独立的JSVirtualMachine
2、管理js和oc，swift桥接对象的内存
当你暴露Objective-C 或者 Swift对象给JavaScript时，不应该在对象中持有JavaScript 的值，这会造成循环引用。我们应该使用` JSManagedValue`



内存管理：
不要在block中直接使用context,或者使用外部`JSValue `
```swift
let context:JSContext = JSContext.init()
context.evaluateScript("var str = '안녕하새요!'")
let simplifyString: @convention(block) () -> String? = {

    // 不产生循环引用
    guard let input = JSContext.current().objectForKeyedSubscript("str")?.toString() else{
        return nil
    }
    
    //循环引用
//    guard let input = context.objectForKeyedSubscript("str")?.toString() else{
//        return nil
//    }

    var mutableString = NSMutableString(string: input) as CFMutableString
    CFStringTransform(mutableString, nil, kCFStringTransformToLatin, false)
    CFStringTransform(mutableString, nil, kCFStringTransformStripCombiningMarks, false)
    return mutableString as String
}
context.setObject(unsafeBitCast(simplifyString, to: AnyObject.self), forKeyedSubscript: "simplifyString" as NSCopying & NSObjectProtocol)
print(context.evaluateScript("simplifyString()"))
```
或者直接将参数传递给函数
```swift
let simplifyString: @convention(block) (String) -> String? = {
input in
....略
}
print(context.evaluateScript("simplifyString('传入参数')"))
```
使用内存管理辅助对象`JSManagedValue `，详情看下面 `JSExport 协议`中代码

* JSExport 协议
另一种将OC和Swift类暴露给JS的方法是定义并实现`JSExport `协议，对于协议中定义的类型，有如下转换关系：

| Object-C / Swift       |    JS       |
| ------------- |:-------------:|
|属性      |JS中的Getter 和 Setter | 
| 实例方法     | JS中的function      |
| 类方法     | JS中的global object中的function    |

映射关系示例：
```swift
// swift对象
protocol MyPointExports : JSExport {
    
    var x : Double? {get set}  // 存储属性
    var y : Double? {get set}  // 存储属性
    
    func dzscription() // 实例方法
    init(x : Double, y : Double)  // 初始化方法
    static func makePoint(x : Double, y : Double) -> MyPoint // 类方法
}

class MyPoint : MyPointExports{

    var x : Double? 
    var y : Double?
    
    func dzscription(){print("MyPoint")}
    
    required init(x : Double, y : Double){
        self.x = x
        self.y = y
    }
    
    static func makePoint(x : Double, y : Double) -> MyPoint{
        return MyPoint.init(x: x, y: y)
    }
} 
```
```javascript
// 对应的js对象调用方式
point.x;
point.x = 10;
// 实例方法 -> 函数
point.description();
// 初始化方法 -> constructor语法
var p = MyPoint(1, 2);
// 类方法 -> constructor对象上的方法
var q = MyPoint.makePointWithXY(0, 0);
```


`JSExport`协议使用示例：
```swift
// 声明一个协议
protocol JSExportProtocol : JSExport {
  
    var jsValue : JSValue {get set}
}
// 具体类遵守协议
class JSExportObject: NSObject, JSExportProtocol {
    
    var manageValue : JSManagedValue?
    // 只暴露jsValue， manageValue不暴露给js，因为没有在协议中声明
    var jsValue: JSValue{
        set{
            self.jsValue = newValue
// JSManagedValue引用，防止循环引用
            manageValue = JSManagedValue.init(value: self.jsValue)
            JSContext.current().virtualMachine.addManagedReference(manageValue, withOwner: self)
        }
        get{
            return self.jsValue
        }
    }
    
    override init() {
        super.init()
    }
    
}

class MDJSBase: NSObject {

    let context:JSContext = JSContext.init()
    var managedValue : JSManagedValue?
    var exportObject : JSExportObject = JSExportObject.init()
    
    func md_executeJS() {
        
//        let jsValue = JSValue.init(object: ["key":"value"], in: context)
//        exportObject.jsValue = jsValue
        context.evaluateScript(
                "var str = 'string';" +
                "function printDict(obj){ " +
                "this.obj = obj;" +  //js引用OC对象
                "obj.jsValue = str;" +//OC引用js对象
                "return obj.jsValue;}"
                )
        guard let funcValue = context.objectForKeyedSubscript("printDict") else{
            fatalError()
        }
        print(funcValue.call(withArguments: [exportObject]))
    }
}
```

> 以上主要内容来自
> * 官方文档的翻译和总结 [官方文档](https://kapeli.com/dash_share?docset_file=Apple_API_Reference&docset_name=Apple%20API%20Reference&path=dash-apple-api://load?request_key=ccjavascriptcore%23&platform=apple&repo=Main&source=https://developer.apple.com/reference/javascriptcore?language=objc)    
>* [[JavaScriptCore Tutorial for iOS: Getting Started](https://www.raywenderlich.com/124075/javascriptcore-tutorial)](https://www.raywenderlich.com/124075/javascriptcore-tutorial) 
>*  [[Java​Script​Core - NSHipster](http://nshipster.cn/javascriptcore/)](http://nshipster.cn/javascriptcore/)


概念讲完了，接下来做些实践，实践参考[[JavaScriptCore Tutorial for iOS: Getting Started](https://www.raywenderlich.com/124075/javascriptcore-tutorial)](https://www.raywenderlich.com/124075/javascriptcore-tutorial) ：
![Simulator Screen.png](http://upload-images.jianshu.io/upload_images/1749699-3fe1fae6c7cf6863.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

 实现一个查询IP地址归属地的功能，Native发起请求，交给JS解析，返回结果，并更新UI，代码有打注释，这里就不赘述了

示例解析代码：
```swift
// 一 JSExort协议 已经将IPInfo对象暴露给js，直接在js中对象解析
let mapFunction = context.objectForKeyedSubscript("mapToNative")
guard let ipInfo = mapFunction?.call(withArguments: [parsedJson]).toObject() as? IPInfo else {
      print("Unable to parse JSON to Native Object")
       return nil
      }
```
```swift
// 1  block 类型转换成泛型
let factoryBlock = unsafeBitCast(IPInfo.ipInfoFactory, to: AnyObject.self)
// 2
context.setObject(factoryBlock, forKeyedSubscript: "ipInfoFactory" as (NSCopying & NSObjectProtocol)!)
let factory = context.evaluateScript("ipInfoFactory")
// 3 调用native Block
 guard
     let ipInfo = factory?.call(withArguments: [parsedJson]).toObject() as? IPInfo else {
     print("Error while processing ipInfo.")
     return nil    
     }
```
> 源码Github地址 [点击这里](https://github.com/madaoCN/JSCore-IPQuery-Demo.git) 有缘人可以个赏个star