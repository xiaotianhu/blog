Date: 2018-01-23
Title:  Chrome Extension扩展开发之复制粘贴
Tags:  chrome extension 粘贴
Toc:no
Status: public
Position: 1

最近在研究Chrome Extension扩展开发.想实现一个功能,选中文本后写入到插件弹层的div中.  
一开始想的方案,用Selection来实现.钻研了好几天的Selection,总是有莫名其妙的问题,在某些页面选中文本后Selection对象里面无法获取到文本内容,感觉跟过于复杂的DOM结构有点关系.灵机一动,用复制粘贴不就解决了嘛,鼠标选中文本后自动复制,然后在弹出的div层中把文本粘贴回来就行了,如此简单的方案 岂不美滋滋.说干就干

翻了翻方案,读写clipboard两条命令搞定:
```
document.execcommand('copy')
document.execcommand('paste')
```
然后在onmouseup里面调用一下copy,复制就搞定了,美滋滋  
然而,paste却死活不行.调用document.execcommand('paste'),返回值总是false,死活不行.查了半天,Chrome因为安全问题禁用了读取剪切板内容...这不坑爹了

不信邪 一顿操作猛如虎,找到两个solution
```
https://stackoverflow.com/questions/7144702/the-proper-use-of-execcommandpaste-in-a-chrome-extension

https://gist.github.com/srsudar/e9a41228f06f32f272a2
```

stackoverflow讲的不是很明白,但是信息是明确的:Chrome的Extension里面是可以调用这个方法的,这个路子肯定是没问题的.大方向没问题,就开始折腾.

开始在Content Script(就是注入到页面的js)各种折腾,各种失败.继续搜了半天,只有在background.js里面才有权限.

那么,思路基本上是:必须在background.js里面调用paste拿到剪贴板内容,然后传给页面里的JS.
开始在页面js里面调用chrome.extension.getBackgroundPage(),死活也不行.翻官方手册,貌似可以用传Message的方式进行通讯.一顿尝试,最终成功,必须记录一下了.

background.js:
```
chrome.runtime.onMessage.addListener(
    function(request, sender, sendResponse){
        var result = '';
        var sandbox = document.getElementById('sandbox');
        sandbox.value = '';
        sandbox.select();
        if (document.execCommand('paste')) {
            result = sandbox.value;
        }
        sandbox.value = '';
        sendResponse({clipboard: result});
    }
)
```

background.html
```
<!DOCTYPE html>
<html>

    <head>
        <script src="background.js"></script>
    </head>

    <body>
        <textarea id="sandbox"></textarea>
    </body>

</html>
```

mainfest.json
```
...
"background": {
    "persistent": true,
    "page": "background.html"
  },
 ...
 
 权限我加入了"clipboardRead", "clipboardWrite","background"这几个,
 web_accessible_resources加入"background.html","background.js" 懒得去测每个值的影响了,反正都加上了
 
 ```
 
 在需要获取剪切板内容的方法里发消息:
 ```
 chrome.runtime.sendMessage({}, function (response){
     console.log(response);
 });
           
 ```
 response中就可以获取到剪切板的内容了,搞定!
 
 PS:
 调试background.html的时候,这里面引入的js的debug信息和console信息,是不会在普通的tab页面中显示的.导致我摸黑调试了半天,浪费了很多时间.background相关的调试信息,需要在chrome://extensions/里面打开 develop mode,然后在插件下面点击"Inspect views: background.html"在弹出的inspector里面就可以看到debug信息了.有了信息在debug就方便多了.


