## WebView与JavaScriptcore实践
>记得一个月前一位前端的朋友问我关于JavaScript如何调用iOS原生方法的问题。我当时我也不知道如何做，就推荐了kitten同学的博客文章[UIWebView与JS的深度交互](http://kittenyang.com/webview-javascript-bridge /).不过最近我也开始学习前端开发了，回头看了一下这个问题，也把这个问题解决并总结一下。希望可以让你得到一定帮助。

先上源码:

[Object-C](https://github.com/zhaiyjgithub/JavaScriptCore-Object-C.git)

[swift](https://github.com/zhaiyjgithub/JavaScriptCore-swift.git)

### native端调用JS端
创建工程，添加JavaScriptCore.framework这个依赖库，并添加`@import JavaScriptCore;`包，或者`#import <JavaScriptCore/JavaScriptCore.h>`也可以。
在工程中先添加一个按钮`callJSFunctioinBtn`,并为这个按钮添加事件`clickCallJSFunctionBtn`。然后添加一个webView和一个HTML文件`index.html`,最后使用webView加载这个添加到工程中的HTML文件。
```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    UIButton * callJSFunctioinBtn = [[UIButton alloc] initWithFrame:CGRectMake(10, 40, 80, 44)];
    callJSFunctioinBtn.backgroundColor = [UIColor redColor];
    callJSFunctioinBtn.titleLabel.font = [UIFont systemFontOfSize:12.0f];
    [callJSFunctioinBtn setTitle:@"调用JS方法" forState:(UIControlStateNormal)];
    [callJSFunctioinBtn addTarget:self action:@selector(clickCallJSFunctionBtn:) forControlEvents:(UIControlEventTouchUpInside)];
    [self.view addSubview:callJSFunctioinBtn];
    
    CGRect webViewFrame = CGRectMake(0, 100, kWidth, kHeight - 100);
    UIWebView *webView = [[UIWebView alloc] initWithFrame:webViewFrame];
    webView.delegate = self;
    
    NSString* path = [[NSBundle mainBundle] pathForResource:@"index" ofType:@"html"];
    NSURL* url = [NSURL fileURLWithPath:path];
    NSURLRequest* request = [NSURLRequest requestWithURL:url] ;
    [webView loadRequest:request];
    [self.view addSubview:webView];
    self.webView = webView;
}
```
`callJSFunctioinBtn`按钮事件的方法：

```
- (void)clickCallJSFunctionBtn:(UIButton *)btn{
    [self.webView stringByEvaluatingJavaScriptFromString:@"callJSFunction()"];
}
```

关于本地HTML文件的内容。当前HTML文件中使用JavaScript标签谢了一个JavaScript的方法`callJSFunction`，当该方法被调用，就会在当前的webView弹出一个弹框，内容是：`原生调用JS方法成功！！`

```
	<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>hello</title>
</head>
<body>
	hello,I am webView!you can alert or write something here!
    <script type="text/javascript">
    	function callJSFunction () {
            <!--    如果调用成功就输出弹框        -->
    		 alert("原生调用JS方法成功！！");
    	}
	</script>
</body>
</html>
```

接下来，我们再回头看看`clickCallJSFunctionBtn`这个方法。在这个方法里面，我们可以通过`[self.webView stringByEvaluatingJavaScriptFromString:@"callJSFunction()"];`调用了JS的方法了。
	OK，我们来run一下当前的工程。
	![](https://github.com/zhaiyjgithub/article/raw/master/titlePic/js1.png)
	
然后我们再点击一下红色的按钮，并留意模拟器的输出。
	
![](https://github.com/zhaiyjgithub/article/raw/master/titlePic/js2.png)
	
从第二张图片可以知道，的确弹出一个弹框，并且弹框的内容跟index.html的`callJSFunction`方法内容是一致的。因此，原生调用JavaScript方法成功了。
	
在这里，你很可能会问，如果我要在native端向JS端传递一个参数呢？OK，我们接下来就继续解决这个问题。
	首先，先回到index.html文件中，在脚本中添加一个方法

```
function callJSFunctionWithParam (param) {
            <!--  如果调用成功就会输出传递的参数内容          -->
            alert("your parma is: " + param)
    	}
 
```

上面的方法就是将传递过啦的参数通过弹窗的方式输出。
	继续再修改一下原生按钮的事件方法，传递一个`mary`这个字符串过去.
	
```
 [self.webView stringByEvaluatingJavaScriptFromString:@"callJSFunctionWithParam(\"mary\")"];
```

继续run一下工程，你会发现，的确输出了弹框并且成功把参数传递过去了。

![](https://github.com/zhaiyjgithub/article/raw/master/titlePic/js3.png)

到这里，你会继续问，如果我想传递一个数组或者字典过去呢？如果按照之前的方法，将参数变成一个数组或者字典地址过去，run工程之后发现是出错的。你不可能将数组中的每个值拿出来再拼接成为一个字符串过去吧？OK，到这里我们应该使用今天文章标题提到的JavaScriptcore了。
	我们重新修改`clickCallJSFunctionBtn`方法的内容
	
```
	 NSArray * params = @[@"tony",@"zack",@"kson"];
    JSContext *context = [self.webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
    [context setExceptionHandler:^(JSContext *context, JSValue *value) {
        NSLog(@"JS exception: %@", value);
    }];
    JSValue *jsFunction = context[@"callJSFunctionWithParam"];
    [jsFunction callWithArguments:@[params]];
```

上面代码的一些解释：

* 定义一个简单的数组，用于被传递的参数
* 获取当前webView的JavaScriptContext。是的，只可以通过KVC这个黑魔法来获取`key` = `documentView.webView.mainFrame.javaScriptContext`这个环境包含了当前webView定义的JavaScript方法。
* 为这个JScontext设置语法执行异常结果回调。如果JS语法错误，那么就会执行这个block回调，并提示一些语法错误信息让我们参考。
* 定义一个类型为JSValue的jsFunction，并在context[@"callJSFunctionWithParam"]为其赋值。`callJSFunctionWithParam`这个就是在当前webView环境定义的方法。什么是`JSValue`呢？点击查看这个它的定义:

>A JSValue is a reference to a value within the JavaScript object space of a
 JSVirtualMachine. All instances of JSValue originate from a JSContext and
 hold a strong reference to this JSContext.
 
从上面我们知道它在JVM环境中，它代表任何从JSContext获取的实例，无论整型，字符型，数组还是方法。跟上面那样的获取，返回值都是一个`JSValue`

* 最后，调用`callWithArguments`这个方法，用数组的方式将参数h包装起来并发送过去。点击进入该方法所在的头文件`JSValue.h`中，你发现很多关于方法声明，参数都是通过数组的方式包装过去的。要记得，`参数传递的数组要跟JS方法定义的列表保持一致的`。

继续修改index.html文件的传递参数的方法:

```
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>hello</title>
</head>
<body>
	hello,I am webView!you can alert or write something here!
    <script type="text/javascript">
    	function callJSFunction () {
            <!--    如果调用成功就输出弹框        -->
    		 alert("原生调用JS方法成功！！");
    	}
    	function callJSFunctionWithParam (param) {
            <!--  如果调用成功就会输出传递的参数内容          -->
            alert("your parma is: " + param[0] + " " + param[1] + "" + param[2])
    	}
	</script>
</body>
</html>
```

OK，继续运行这个工程并点击按钮看一下输出:

![](https://github.com/zhaiyjgithub/article/raw/master/titlePic/js4.png)

yeah!的确输出我们想要的效果。

另外，在JSValue.h文件找到了一些`toArray`,`toString`...等方法，究竟有什么用呢？
	如果将index.html文件中JS方法添加返回值，类型假如是String类型。那么在native端的事件点击事件方法中修改为：
	
	```
	    JSValue * jsReturnValue =  [jsFunction callWithArguments:@[params]];
    NSLog(@"js return value:%@",[jsReturnValue toString]);
	```

上面提到的方法就是这个作用了。到此，native调用JS端方法也是完成了。
接下来我们开始继续解决JS端调用native端方法。

### JS端调用native端
我们要知道的是:

* 可以通过KVC方法获取当前webView的JavaScript环境并执行JavaScript环境定义的方法。
* 另外，还可以通过`注入`的方式往当前JavaScript环境写入一个方法，相当于在index.html文件中定义方法。

``` 
NSString *jsFunctionText =
    @"var injectFunction = function() {"
    "douctment.write(\"方法注入成功\")"
    "}";
    [context evaluateScript:jsFunctionText];
```
同样可以KVC从当前JSContext中获取当前方法并执行。

* 如何让JavaScript的环境中可以`见到(调用到)`原生端的定义的对象呢？答案就是当对象遵守了`JSExport`协议即可。


下面，我们定义一个`Student`对象。
`Student.h`文件

```
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>
#import <JavaScriptCore/JavaScriptCore.h>

@protocol StudentJS <JSExport>

- (void)takePhoto;

@end

@interface Student : NSObject<StudentJS>
@property(nonatomic,strong)UIViewController * viewController;

@end

```

* 上面定义了一个`StudentJS`协议，它遵循`JSExport`协议。并添加一个协议方法`takePhoto`.
* 然后下面这个`Student`对象遵循`StudentJS`协议。

然后在`Student.m`实现这个方法。

```
#import "Student.h"
#import "PhotoPickerTool.h"
#import "ViewController.h"

@implementation Student

- (void)takePhoto{
    NSLog(@"add a student");
    [[PhotoPickerTool sharedPhotoPickerTool] showOnPickerViewControllerSourceType:(UIImagePickerControllerSourceTypeSavedPhotosAlbum) onViewController:self.viewController compled:^(UIImage *image, NSDictionary *editingInfo) {
        NSLog(@"make photo");
        ViewController * VC =  (ViewController *)(self.viewController);
        VC.summerImageView.image = image;
    }];
}

@end

```

调用了这个方法就会打开摄像头拍照。

接下里然后在`viewDidLoad()`方法末尾添加一个`imageView`控件，用于显示拍照后的图片显示

```
    self.summerImageView = [[UIImageView alloc] initWithFrame:CGRectMake(120, 20, 80, 80)];
    [self.view addSubview:self.summerImageView];
```

回到index.html文件中，添加一个按钮，按钮的id=`pid`。需要知道的是在JavaScript中可以通过`id`的方式来获取`目的标签`并使用它。这个跟iOS端使用`tag`方式获取目的控件一样的原理。

```
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>hello</title>
</head>
<body>
	hello,I am webView!you can alert or write something here!
    <script type="text/javascript">
    	function callJSFunction () {
            <!--    如果调用成功就输出弹框        -->
    		 alert("原生调用JS方法成功！！");
    	}
    	function callJSFunctionWithParam (param) {
            <!--  如果调用成功就会输出传递的参数内容          -->
            alert("your parma is: " + param[0] + " " + param[1] + " " + param[2])
    	}
	</script>
    <!-- 添加一个按钮，id = "pid" -->
    <button id="pid">click me</button>
</body>
</html>
```

接下来回到`viewController.m`文件中添加代码:

```
- (void)webViewDidFinishLoad:(UIWebView *)webView{
    NSLog(@"webView finsh load");

    JSContext *context = [webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
    
    [context setExceptionHandler:^(JSContext *context, JSValue *value) {
        NSLog(@"WEB JS: %@", value);
    }];
    
    Student * kson = [[Student alloc] init];
    kson.viewController = self;
    context[@"myStudent"] = kson;
    
    NSString * str =
    @"function spring () {"
    "   myStudent.takePhoto();}"
    "var btn = document.getElementById(\"pid\");"
    "btn.addEventListener('click', spring);";
    
    [context evaluateScript:str];
    
}

```

* 首先获取当前webView的JavaScript环境
* 设置JS代码语法运行handler block
* 定义一个`Student`带对象，并将其`注册`到JavaScript环境中。因为`Student`对象已经遵守了`JSExport`协议，因此该类型对象在JavaScript环境是`可见(可以被访问)`。
* 定义一个JS的方法，名称叫`spring`。作用就是先调用上面注册的原生对象`myStudent`的方法`takePhoto()`.然后获取HTML文件中定义的按钮，并为这个按钮添加`点击`的监听事件，事件就是`spring`。最后将其拼接成为一个脚步字符串。
* 最后context将上面脚本同样注册到JS环境中。

接下来，run一下工程，可以看到webView上面已经多了一个title叫`click me`的按钮。

![](https://github.com/zhaiyjgithub/article/raw/master/titlePic/js5.png)

然后点击`click me`按钮，就会弹出手机相册，选择其中任意一张图片后就会在原来界面顶部显示刚才选择的照片。

![](https://github.com/zhaiyjgithub/article/raw/master/titlePic/js7.png)

![](https://github.com/zhaiyjgithub/article/raw/master/titlePic/js6.png)

同样地，如果要在JS端传值到native端。只需要在`takePhoto`这些遵循了`JSExport`协议的方法添加参数，然后在JS端的方法传递参数即可。你会发现，JS端的方法的变量定义跟`swfit`有些相似的。

OK，到此为止，JS端调用native端的问题同样也得到了解决了。不过，事情还没有到此结束的。当时我朋友跟我说，他的iOS程序猿是用`swift`编写项目的。刚开始我还以为很简单地转换一下就好了，最后发现的确是简单转换一下，但是还是遇到了一些`坎(并不是坑，因为我的swift语言基础仍然是很渣渣，所以是坎)`。

### swfit版

index.html文件内容跟OC测试环境一样。先看下webView的delegate方法

```
    func webViewDidFinishLoad(webView: UIWebView) {
        print("finshed load")
        let context = webView.valueForKeyPath("documentView.webView.mainFrame.javaScriptContext") as!  JSContext
        
        let kson = Student()
        kson.delegate = self
        
        context.setObject(kson, forKeyedSubscript:"Student")
        context.exceptionHandler = { context, exception in
            print("JS Error: \(exception)")
        }
        
        let script = "function spring () {document.write(\"kson\");Student.takePhoto();}var btn = document.getElementById(\"pid\");btn.addEventListener('click', spring);";
        context.evaluateScript(script)
    }
```

要修该的地方是，`context.setObject(kson, forKeyedSubscript:"Student")`,只能通过这样的方式来`注册native对象`

接下来也就是最重要的地方就是在`Student.h`文件中，`StudentJS`代理协议的定义，前面必须要加上`@Objc`，否则native对象在JavaScript环境中不可见。

```
@objc protocol StudentJS:JSExport{
    func takePhoto()
}
```

其他地方保持一致即可，继续运行工程，效果跟之前的一样。

###结束语
OK，关于webView与JavaScriptCore的实践过程到此结束。通过这一个总结，让自己对这个实践过程更加地深刻。同时，希望也可以给你带来一些帮助。好吧，接下来继续学习一下前端开发，努力再努力。




