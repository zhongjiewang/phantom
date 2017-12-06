

1.初识 phantomjs  onInitialized

```js
onInitialized 的回调 为 page.open('') 所触发, 无 open 无 回调

page.onInitialized = function() {
  page.evaluate(function() {
    document.addEventListener('DOMContentLoaded', function() {
      console.log('DOM content has loaded.');
    }, false);
  });
};
```
2. Phantomjs 环境下  此种方法修改成功 
```js
page.onInitialized = function () {
    page.evaluate(function () {
        //spoof navigator plugins
        var oldNavigator = navigator;
        var oldPlugins = oldNavigator.plugins;
        var plugins = {};
        plugins.length = 1;
        plugins.__proto__ = oldPlugins;

        window.navigator = {plugins: plugins};
        window.navigator.__proto__ = oldNavigator;

        //spoof callPhantom and _phantom
        
        var p = window.callPhantom;
        delete window._phantom;
        delete window.callPhantom;
        Object.defineProperty(window, "myCall", {
            get: function () {
                return p;
            },
            set: function () {
            }, enumerable: false});
        setTimeout(function () {
            window.myCall();
        }, 1000);
    });
};

page.open('./allDetect.html', function (status) {
    console.log('status-------',status);
});
```

但当 page.evaluate在 onInitialized 回调之外 则 测试失败,
代码如下所示
```js
page.onInitialized = function () {
    console.log('======================');
};

page.evaluate(function () {
    //spoof navigator plugins
    var oldNavigator = navigator;
    var oldPlugins = oldNavigator.plugins;
    var plugins = {};
    plugins.length = 1;
    plugins.__proto__ = oldPlugins;

    window.navigator = {plugins: plugins};
    window.navigator.__proto__ = oldNavigator;

    //spoof callPhantom and _phantom
      // 暂时不用
    var p = window.callPhantom;
    delete window._phantom;
    delete window.callPhantom;
    Object.defineProperty(window, "myCall", {
        get: function () {
            return p;
        },
        set: function () {
        }, enumerable: false});
    setTimeout(function () {
        window.myCall();
    }, 1000);
});
```

3. 基于上述事实,我们在node 环境下运行如下代码
```js
    const phantom = require('phantom'); 
    const instance = await phantom.create(); 
    const page = await instance.createPage(); 
    await page.on('onInitialized', async function(msg) { 
        console.log('--------we received  page  onInitialized callback-------');
        page.evaluate(function () {
           var oldNavigator = navigator; 
           var oldPlugins = oldNavigator.plugins; 
           var plugins = {}; 
           plugins.length = 1; 
           plugins.proto = oldPlugins;

          window.navigator = {plugins: plugins};
          window.navigator.__proto__ = oldNavigator;

          //spoof callPhantom
           var p = window.callPhantom;
           delete window._phantom;
           delete window.callPhantom;
           Object.defineProperty(window, "myCallPhantom", {
               get: function () {
                   return p;
               },
               set: function () {
               }, enumerable: false});
           setTimeout(function () {
               window.myCallPhantom();
           }, 1000);   
        }); 
    }); 
    const status = await page.open('./test/allDetect.html');
    console.log('--open allDetect ---status-----------',status);
``` 

如下是 phantom debug模式得到的信息
```js
xiangrikuid: zhongjie$ DEBUG=true node server.js 
debug: Starting /usr/local/bin/phantomjs /Users/zhongjie/node_modules/phantom/lib/shim/index.js
debug: Sending: {"id":1,"target":"phantom","name":"createPage","params":[],"deferred":null}
debug: Sending NOOP command.
info: command id  1
debug: Parsing: {"id":1,"target":"phantom","name":"createPage","params":[],"deferred":null,"response":{"pageId":1}}
debug: Received NOOP command.
debug: Sending: {"id":2,"target":"page$1","name":"addEvent","params":[{"type":"onInitialized"}],"deferred":null}
debug: Sending NOOP command.
debug: Parsing: {"id":2,"target":"page$1","name":"addEvent","params":[{"type":"onInitialized"}],"deferred":null}
debug: Sending: {"id":3,"target":"page$1","name":"invokeAsyncMethod","params":["open","./test/allDetect.html"],"deferred":null}
debug: Received NOOP command.
debug: Sending NOOP command.
debug: Parsing: {"target":"page$1","type":"onInitialized","args":[]}
--------we received  page  onInitialized callback-------
debug: Sending: {"id":4,"target":"page$1","name":"invokeMethod","params":["evaluate","function () {\n var oldNavigator = navigator;\n   var oldPlugins = oldNavigator.plugins;\n   var plugins = {};\n      plugins.length = 1;\n       plugins.proto = oldPlugins;\n\n                          window.navigator = { plugins: plugins };\n       window.navigator.__proto__ = oldNavigator;\n\n                          // //spoof callPhantom\n                          // var p = window.callPhantom;\n                          // delete window._phantom;\n                          // delete window.callPhantom;\n                          // Object.defineProperty(window, \"myCallPhantom\", {\n                          //     get: function () {\n                          //         return p;\n                          //     },\n                          //     set: function () {\n                          //     }, enumerable: false});\n                          // setTimeout(function () {\n                          //     window.myCallPhantom();\n                          // }, 1000);   \n                        }"],"deferred":null}
debug: Received NOOP command.
debug: Parsing: {"id":3,"target":"page$1","name":"invokeAsyncMethod","params":["open","./test/allDetect.html"],"deferred":null,"response":"success"}
--open allDetect ---status----------- success
debug: Sending NOOP command.
debug: Parsing: {"id":4,"target":"page$1","name":"invokeMethod","params":["evaluate",null],"deferred":null,"response":null}
debug: Received
```
然后请注意 
#### 一个重要基本事实 并认同: 我们应用的这个node操作 phantomjs 的模块, 其实现原理是把node进程下的代码转化成自定义的消息, 然后在phantomjs进程中 通过不断的调用read函数,把来自node进程的 消息 转化成 phantomjs 环境下的代码

因此可得原因:
1. 我们在 node 的进程下的 代码 
```js
    await page.on('onInitialized', async function(msg) { 
        console.log('--------we received  page  onInitialized callback-------');
        page.evaluate(function () { 
        
        });
    });
```
经过 
```js
debug: Sending: {"id":2,"target":"page$1","name":"addEvent","params":[{"type":"onInitialized"}],"deferred":null}
```
#### 请注意 这个消息中 只有 addEvent onInitialized , 并没有onInitialized 回调中的代码片段,只是单独 添加了这个事件,因此这个消息 仅仅转化了成了如下代码
```js
page.onInitialized = function () {

};
```

然后再收到page  onInitialized 回调之后
```json
--------we received  page  onInitialized callback-------
debug: Sending: {"id":4,"target":"page$1","name":"invokeMethod","params":["evaluate","function () { .... }

此时才向 phantomjs 进程发送 执行 evaluate 的消息,这个消息中有 evaluate callback 中的代码
因此 phantomjs 环境下的代码便是
```
```js
page.onInitialized = function () {

};
page.open('....'); //运行page.open()之后 才 触发 onInitialized 的回调函数 

page.evaluate(function(){
    
})
```
因此这样 是 不可能 修改成功的;














