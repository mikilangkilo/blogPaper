---
title: "安卓基础-webview学习"
date: 2018-03-20 10:36:19
tag : "webview"
category : "安卓基础"
description : "本文主要是学习webview的一些内容"
---

项目中用到了js连调的技术，之前有使用过，但是没有全面的了解过，这次做个复习和深入学习。

# js与native交互

js与native交互总共有四种方式

## JavascriptInterface

首先Java代码要实现这么一个类，它的作用是提供给Javascript调用。

```
public class JavascriptInterface {

    @JavascriptInterface
    public void showToast(String toast) {
        Toast.makeText(MainActivity.this, toast, Toast.LENGTH_SHORT).show();
    }
}
```

然后把这个类添加到WebView的JavascriptInterface中。

```
webView.addJavascriptInterface(new JavascriptInterface(), "javascriptInterface");
```

在Javascript代码中就能直接通过“javascriptInterface”直接调用了该Native的类的方法。

```
function showToast(toast) {
    javascript:javascriptInterface.showToast(toast);
}
```


但是这个官方提供的解决方案在Android4.2之前存在安全漏洞。在Android4.2之后，加入了@JavascriptInterface才得到解决。所以考虑到兼容低版本的系统，JavascriptInterface并不适合。

## WebViewClient.shouldOverrideUrlLoading()

这个方法的作用是拦截所有WebView的Url跳转。页面可以构造一个特殊格式的Url跳转，shouldOverrideUrlLoading拦截Url后判断其格式，然后Native就能执行自身的逻辑了。

```
public class CustomWebViewClient extends WebViewClient {

    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        if (isJsBridgeUrl(url)) {
            // JSbridge的处理逻辑
            return true;
        }
        return super.shouldOverrideUrlLoading(view, url);
    }
}
```

## WebChromeClient.onConsoleMessage()

这是Android提供给Javascript调试在Native代码里面打印日志信息的API，同时这也成了其中一种Javascript与Native代码通信的方法。

在Javascript代码中调用console.log('xxx')方法。

```
console.log('log message that is going to native code')
```

就会在Native代码的WebChromeClient.consoleMessage()中得到回调。

consoleMessage.message()获得的正是Javascript代码console.log('xxx')的内容.

```
public class CustomWebChromeClient extends WebChromeClient {

    @Override
    public boolean onConsoleMessage(ConsoleMessage consoleMessage) {
        super.onConsoleMessage(consoleMessage);
        String msg = consoleMessage.message();//Javascript输入的Log内容
    }
}
```

## WebChromeClient.onJsPrompt()

其实除了WebChromeClient.onJsPrompt()，还有WebChromeClient.onJsAlert()和WebChromeClient.onJsConfirm()。顾名思义，这三个Javascript给Native代码的回调接口的作用分别是提示展示提示信息，展示警告信息和展示确认信息。鉴于，alert和confirm在Javascript的使用率很高，所以JSBridge的解决方案中都倾向于选用onJsPrompt()。

Javascript中调用

```
window.prompt(message, value)
```

WebChromeClient.onJsPrompt()就会受到回调。

onJsPrompt()方法的message参数的值正是Javascript的方法window.prompt()的message的值。

```
public class CustomWebChromeClient extends WebChromeClient {

    @Override
    public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result) {
        // 处理JS 的调用逻辑
        result.confirm();
        return true;
    }
}
```


# JsBridge

java 通信js只有这一种方式

- WebView加载html页面

webView.registerHandler("submitFromWeb",...);这是Java层注册了一个叫"submitFromWeb"的接口方法，目的是提供给Javascript来调用。这个"submitFromWeb"的接口方法的回调就是BridgeHandler.handler()。

webView.callHandler("functionInJs", ..., new CallBackFunction());
这是Java层主动调用Javascript的"functionInJs"方法。


```
public class MainActivity extends Activity implements OnClickListener {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        webView = (BridgeWebView) findViewById(R.id.webView);
        webView.loadUrl("file:///Android_asset/demo.html");
        webView.registerHandler("submitFromWeb", new BridgeHandler() {

            @Override
            public void handler(String data, CallBackFunction function) {
                Log.i(TAG, "handler = submitFromWeb, data from web = " + data);
                function.onCallBack("submitFromWeb exe, response data 中文 from Java");
            }

        });

        webView.callHandler("functionInJs", new Gson().toJson(user), new CallBackFunction() {
            @Override
            public void onCallBack(String data) {
                
            }
        });
    }
}
```

我们一层层深入callHandler()方法的实现。这其中会调用到doSend()方法，这里想解释下callbackId。

callbackId生成后不仅仅会被传到Javascript，而且会以key-value对的形式和responseCallback配对保存到responseCallbacks这个Map里面。

它的目的，就是为了等Javascript把处理结果回调给Java层后，Java层能根据callbackId找到对应的responseCallback，做后续的回调处理。

```
private void doSend(String handlerName, String data, CallBackFunction responseCallback) {
        Message m = new Message();
        if (!TextUtils.isEmpty(data)) {
            m.setData(data);
        }
        if (responseCallback != null) {
            String callbackStr = String.format(BridgeUtil.CALLBACK_ID_FORMAT, ++uniqueId + (BridgeUtil.UNDERLINE_STR + SystemClock.currentThreadTimeMillis()));
            responseCallbacks.put(callbackStr, responseCallback);
            m.setCallbackId(callbackStr);
        }
        if (!TextUtils.isEmpty(handlerName)) {
            m.setHandlerName(handlerName);
        }
        queueMessage(m);
    }
```

最终可以看到是BridgeWebView.dispatchMessage(Message m)方法调用的是this.loadUrl()，调用了_handleMessageFromNative这个Javascript方法。那这个Javascript的方法是哪里来的呢？

```
final static String JS_HANDLE_MESSAGE_FROM_JAVA = "javascript:WebViewJavascriptBridge._handleMessageFromNative('%s');";

void dispatchMessage(Message m) {
        String messageJson = m.toJson();
        //escape special characters for json string
        messageJson = messageJson.replaceAll("(\\\\)([^utrn])", "\\\\\\\\$1$2");
        messageJson = messageJson.replaceAll("(?<=[^\\\\])(\")", "\\\\\"");
        String javascriptCommand = String.format(BridgeUtil.JS_HANDLE_MESSAGE_FROM_JAVA, messageJson);
        if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
            this.loadUrl(javascriptCommand);
        }
    }

```

- 页面加载完成后会加在一段Javascript。

在WebViewClient.onPageFinished()里面的BridgeUtil.webViewLoadLocalJs(view, BridgeWebView.toLoadJs)。正是把保存在assert/WebViewJavascriptBridge.js加载到WebView中。

```
public class BridgeWebViewClient extends WebViewClient {
　　
    @Override
    public void onPageFinished(WebView view, String url) {
        super.onPageFinished(view, url);

        if (BridgeWebView.toLoadJs != null) {
            BridgeUtil.webViewLoadLocalJs(view, BridgeWebView.toLoadJs);
        }

        //
        if (webView.getStartupMessage() != null) {
            for (Message m : webView.getStartupMessage()) {
                webView.dispatchMessage(m);
            }
            webView.setStartupMessage(null);
        }
    }
}
```

我们看看WebViewJavascriptBridge.js的代码，就能找到function _handleMessageFromNative()这个Javascript方法了。

- WebViewJavascriptBridge.js

_handleMessageFromNative()方法里面会调用_dispatchMessageFromNative()方法。

当处理来自Java层的主动调用时候会走“直接发送”的else分支。

message.callbackId会被取出来，实例化一个responseCallback，而它是用来Javascript处理完成后把结果数据回调给Java层代码的。

接着会根据message.handleName（在这个分析例子中，handleName的值就是"functionInJs"）在messageHandlers这个Map去获取handler，最后交给handler去处理。

```
function _dispatchMessageFromNative(messageJSON) {
    setTimeout(function() {
        var message = JSON.parse(messageJSON);
        var responseCallback;
        //java call finished, now need to call js callback function
        if (message.responseId) {
            ...
        } else {
            //直接发送
            if (message.callbackId) {
                var callbackResponseId = message.callbackId;
                responseCallback = function(responseData) {
                    _doSend({
                        responseId: callbackResponseId,
                        responseData: responseData
                    });
                };
            }

            var handler = WebViewJavascriptBridge._messageHandler;
            if (message.handlerName) {
                handler = messageHandlers[message.handlerName];
            }
            //查找指定handler
            try {
                handler(message.data, responseCallback);
            } catch (exception) {
                if (typeof console != 'undefined') {
                    console.log("WebViewJavascriptBridge: WARNING: javascript handler threw.", message, exception);
                }
            }
        }
    });
}
```

- 页面注册的"functionInJs"方法，提供给Java调用Javascript的。

延续上面的分析，messageHandler是哪里设置的呢。答案就在当初webView.loadUrl("file:///Android_asset/demo.html");加载的这个demo.html中。

bridge.registerHandler("functionInJs", ...)这里注册了"functionInJs"。

```
<html>
    <head>
    ...
    </head>
    <body>
    ...
    </body>
    <script>
        ...

        connectWebViewJavascriptBridge(function(bridge) {
            bridge.init(function(message, responseCallback) {
                console.log('JS got a message', message);
                var data = {
                    'Javascript Responds': '测试中文!'
                };
                console.log('JS responding with', data);
                responseCallback(data);
            });

            bridge.registerHandler("functionInJs", function(data, responseCallback) {
                document.getElementById("show").innerHTML = ("data from Java: = " + data);
                var responseData = "Javascript Says Right back aka!";
                responseCallback(responseData);
            });
        })
    </script>
</html>

```


- "functionInJs"执行完毕把结果回传给Java

"funciontInJs"执行完毕后调用的responseCallback正是_dispatchMessageFromNative()实例化的，而它实际会调用_doSend()方法。

_doSend()方法会先把Message推送到sendMessageQueue中。

然后修改messagingIframe.src，这里会出发Java层的WebViewClient.shouldOverrideUrlLoading()的回调。

```
function _doSend(message, responseCallback) {
    if (responseCallback) {
        var callbackId = 'cb_' + (uniqueId++) + '_' + new Date().getTime();
        responseCallbacks[callbackId] = responseCallback;
        message.callbackId = callbackId;
    }

    sendMessageQueue.push(message);
    messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
}
```

 在BridgeWebViewClient.shouldOverrideUrlLoading()里面，会先执行webView.flushMessageQueue()的分支。

```
@Override
public boolean shouldOverrideUrlLoading(WebView view, String url) {
    try {
        url = URLDecoder.decode(url, "UTF-8");
    } catch (UnsupportedEncodingException e) {
        e.printStackTrace();
    }

    if (url.startsWith(BridgeUtil.YY_RETURN_DATA)) { // 如果是返回数据
        webView.handlerReturnData(url);
        return true;
    } else if (url.startsWith(BridgeUtil.YY_OVERRIDE_SCHEMA)) { //
        webView.flushMessageQueue();
        return true;
    } else {
        return super.shouldOverrideUrlLoading(view, url);
    }
}
```

webView.flushMessageQueue()首先去执行Javascript的_flushQueue()方法，并附带着CallBackFunction。

Javascript的_flushQueue()方法会把sendMessageQueue中的所有message都回传给Java层。

CallBackFunction就是把messageQueue解析出来后一个一个Message在for循环中处理，也正是在for循环中，"functionInJs"的Java层回调方法被执行了。

```
void flushMessageQueue() {
    if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
        loadUrl(BridgeUtil.JS_FETCH_QUEUE_FROM_JAVA, new CallBackFunction() {

            @Override
            public void onCallBack(String data) {
                // deserializeMessage
                List<Message> list = null;
                try {
                    list = Message.toArrayList(data);
                } catch (Exception e) {
                    e.printStackTrace();
                    return;
                }
                if (list == null || list.size() == 0) {
                    return;
                }
                for (int i = 0; i < list.size(); i++) {
                    ...
                }
            }
        });
    }
}
```

到此，JsBridge的调用流程就分析完毕了。虽然JsBridge使用了MessageQueue后，分析起来有点绕，但原理是不变的。

Javascript调用Java是通过WebViewClient.shouldOverrideUrlLoading()。当然，还有在文章开头介绍另外3种方式。

Java调用Javascript是通过WebView.loadUrl("javascript:xxxx")。




