![avatar](cyclops.ico)**Cyclops 是一款具有 XSS 检测功能的浏览器**    

📝[Englist Document](https://github.com/v8blink/Chromium-based-XSS-Taint-Tracking/blob/main/README-en.md)   

Cyclops 暂时不开源，直接下载构建的二进制文件即可 [下载地址](https://github.com/v8blink/Chromium-based-XSS-Taint-Tracking/releases)  

现在仅有 Win 10 版，Linux 和 Mac 版即将发布。   


# 使用说明
Cyclops 仍处于开发阶段，有问题及时沟通。  
使用 Cyclops 访问目标网站时需要 **--no-sandbox** 参数，发现可疑 XSS 时，会在当前运行目录下生成 SourceSink.txt文件（默认位置：C:\Users\UserName\AppData\Local\Chromium\Application\97.0.4684.0\），[SourceSink格式说明见下文](#sourcesink-文件格式说明)。   

[使用演示视频1](https://www.zhihu.com/zvideo/1505471657166282752) ，[使用演示视频2](https://www.zhihu.com/zvideo/1505847898797969409)      
**注意** 软件版本更新较快，新功能不一定包括在视频中  

# 更新计划    
按优先级排序：  
**0.sink 信息必须要包含 id**  
例如：“Sink:HTMLLIElement.className”，没有 id 时会显示 Element 类型，有 id 时一定会显示 id。正功能在完善中...  

**1.更新 source 和 sink**  
目前的 source 和 sink 如下：
|source|sink|
|----|----|
|document.baseURI|element.innerHTML  
|document.URL|document.cookie   
|document.referrer|element.innerText  
|location.href/a.href|element.className     
|location.hash|element.href   
|location.host|element.src  
|location.hostname|document.write()  
|location.pathname|document.writeln()  
|location.search|window.alert()  
|document.domain|window.setTimeout()  
|document.cookie|window.setInterval()
|document.documentURI|window.postMessage
|element.textContent|document.createElement()
|document.title|
|element.classname|
|element.namespaceURI|
|element.src|

大家多多提供修改意见，尽早完善 Cyclops。  

**2.修复 crash 问题**  
我的测试范围有限，大家发现 crash 时请提 issue 或微信：qq9123013，感谢。   
**3.优化 SourceSink.txt 格式**    
目前这样的 SourceSink.txt 是临时版本。我的本意是采用 JSON 格式输出，或转发 SourceSink 结果到指定端口。但 Chromium 源码耦合度很高，源码又十分复杂，我的改动稍有差池，crash满天飞。   
我正在努力实现上述想法。  

# SourceSink 文件格式说明  
文件中每行是一条 sink 记录，由逗号分为三部分。第一部分是 sink 信息；第二部分 910226 是测试标记；第三部分是数据来源，其中 每对 [] 是一个单元，单元可以嵌套，从里向外看。举例如下：  
>[Sink:alert:qerwr , 910226, [ENURI:[Substring:[Location.search]]]]

 
Sink:alert:qerwr  冒号分隔，数据的终点是 alert 方法，进入 alert 的数据是 "qerwr"；  
910226, 测试标记；  
[ENURI:[Substring:[Location.search]]] 说明 alert 数据来源。首先 x = location.search，然后是 y=x.substring()，z=EncodeURI(y)，最后是 alert(z); 

>[Sink:HTMLLIElement.className ,910226, [Trim:[ADD:[ADD:[ADD:,[HTMLLIElement.className]],],]]]

 
Sink:HTMLLIElement.className  冒号分隔，数据的终点是某个 Element 的 className属性；  
910226, 测试标记；   
[Trim:[ADD:[ADD:[ADD:,[HTMLLIElement.className]],],]]]， 首先是x= Element.className，然后 x 参与了加法(ADD)运算，注意：[ADD:,x]，正常的加法需要两个操作数，但这里只有一个右操作，说明它的左操作数没有污染信息。Trim 代表 Trim 方法。  
**注意** 测试标记 910226 和数据流来源一定同时存在，否则是 bug，请通知我，谢谢。    
上面两个例子中的 "Trim" 和 "ENURI" 是 Sink 数据流中使用的简写标记，下表给出所有的简写标记。

|标记|JavaScript 方法|示例|备注|
|----|----|----|----|
|A|string.anchor|[A:strX]|`<a name=undefined>strX</a>`，使用字符串strX创建A标签|
|B|string.big|[B:strX]|`<big>strX</big>`||
|Blink|string.blink|[Blink:strX]|`<blink>strX</blink>`||
|Bold|string.bold|[Blod:strX]|`<b>strX</b>`||
|Fontcolor|string.fontcolor
|Fontsize|string.fontsize
|Fixed|string.fixed
|Italics|string.Italics
|Link|string.link  
|Small|string.small
|Strike|string.strike
|Sub|string.sub|
|Sup|string.sup|
|Pad:,S=|string.padStart|  
|LC|string.toLowerCase|
|Substring|string.substring|
|DURI|decodeURI|  
|DURIC|decodeURIComponent|  
|ENURI|encodeURI|
|ENURIC|encodeURIComponent|
|Escape|escape|
|UnEscape|Unescape|
|Geval|eval方法|
|Deval|eval方法|
|ADD|加法|[ADD:strX,strY]|缺少的操作数即不存在污染信息|
|Pad:,E=|string.padEnd|[Pad:strx,E=strY]|b=strX.padend(num,strY), b 的值与 strX 不同时，产生此记录。| 
|Rep:rec=,s=|strin.replace|[Rep:rec=hello,s=world]|strX.replace("hello","world"), 查找字符串内的"hello"并替换为"world"|  
|Repeat|string.repeat|[Repeat:strX]|重复次数暂时没记录，后续更新|
|Slice|string.slice|[Slice:strX]|b=strX.slice(i,j)，b的长度大于零且小于strX长度时，产生此记录|
|Substr|string.substr|[Substr:strX]|参照上一条|
|Trim|string.trim|不区分trimStart、trimEnd、trimLeft、trimRight  
|Concat|string.concat|[Concat:strX,strY]或[Concat:strX,strY,strZ...]|b=strX.concat(strY),b=strX.concat(strY,strZ...)      
|gAttr|element.getAttribute()|[gAttr:name:value]|例如`<input id=name value='huidou'/>`，input.getAttribute('value') 会产生此记录,说明此条数据来自某个 Element 的 value 属性，该标签 id 是 name。  
|sAttr|element.setAttribute()|[sAttr:name:value]|同上例，说明向某 Element 的 value 属性写值，其 id 是 name。
|F:P=,B=|Function()构造方法|[F:P=p1,p2,B=evil]|P 表示参数，p1，p2 是两个参数的数据源；B 表示函数主体的数据来源。P 或 B 中的一个项是攻击者可控时产生此条记录。

**注意**如 location.href、settimeout、alert 等没有使用简写标记，而是使用全称表示的方法，没在表格中列出。  


# 🍺赞赏    
如果对你有帮助，请打赏豆豆以资鼓励🥂   
 <img src="https://github.com/v8blink/Chromium-based-XSS-Taint-Tracking/blob/main/Donate.jpg" width = "300" height = "300" alt="图片名称" align=center />     


# 免责声明

如您在使用本工具的过程中存在任何非法行为，您将自行承担所有后果，本工具所有开发者和所有贡献者不承担任何法律及连带责任。
除非您已充分阅读、完全理解并接受本协议所有条款，否则，请您不要安装并使用本工具。
