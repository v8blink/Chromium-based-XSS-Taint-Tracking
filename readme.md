# 基于 Chromium 的 XSS 检测工具   
![avatar](cover.jpg)     
# 1 介绍  
XSS 是网络中最常见的漏洞，再配合其它的攻击手段往往能轻易敲开服务器的大门。在各大漏洞平台中，XSS 漏洞也是数量最多的。本文介绍基于 Chromium 源码实现的 XSS 检测工具，该工具采用动态污染追踪（Taint Tracking）技术，可以监测 source 与 sink 之间的数据流动过程，为人工验证提供详实可靠的依据。  
# 2 工具演示视频   
**（1）** 我的工具如何检测 XSS？Burp Suite 靶场实践 (1) https://www.zhihu.com/zvideo/1505471657166282752   
**（2）** 我的工具如何检测 XSS？Burp Suite 靶场实践 (2) https://www.zhihu.com/zvideo/1505847898797969409      
视频还在持续更新中...    

# 3 原理介绍  
污点分析技术（taint analysis, 又被称作信息流跟踪技术）是自动检测 XSS 的理论基础，它是信息流分析技术的一种实践方法, 该技术通过标记系统中的敏感数据, 进而跟踪‘标记数据’在程序中的传播过程, 以检测系统安全问题。
# 4 敏感数据 
在污点分析技术中，最重要的两个概念分别是敏感数据和传播记录。首先引入敏感数据的定义：由 source 开始，经过传播后进入 sink 并引发“数据与指令的错用”的数据被定义为敏感数据。其中 source 为污染源点，也就是数据的生产者。sink 为污染终点，也就是数据的消费者。
攻击者可以控制的输入被称作污染源点，常规的污染源点如下：  
//Source 代表 document、location、element  
Source.baseURI  
Source.cookie   
Source.documentURI  
Source.domain  
Source.URL  
Source.referrer   
Source.textContent   
Source.title  
Source.hash   
Source.hostname  
Source.href  
Source.pathname  
Source.search   
Source.className   
Source.innerHTML   
Source.namespaceURI       

漏洞的本质是数据与指令的错用，所以能消费敏感数据的方法（函数或属性）都是污染终点，常规的污染终点如下：   
//以下是全局方法  
setTimeout()  
setInterval()  
Function()  
alert()  
eval()   
window.postMessage()   
//Sink 代表 document、location、element  
Sink.write()  
Sink.writeln()  
Sink.createComment()  
Sink.createTextNode()  
Sink.createElement()  
Sink.innerHTML  
Sink.className  
Sink.innerText  
Sink.textContent  
Sink.title  
Sink.href  
 
# 5 传播记录  
接下来引入传播记录的定义：被 sink 消费的数据的来源以及变化过程称为传播记录。  
传播记录描述了数据从 source 到 sink 的变化过程，传播记录的完整性直接影响 XSS 的检测结果。衡量传播记录的指标主要有：  
**（1）** 传播记录是否全面、完整地描述了数据的变化过程；  
**（2）** 传播记录是否可以做到字符级别的描述。 

众所周知，触发 XSS 漏洞的字符串往往是来自于不同 source 的字符的组合，如果不能完整记录数据变化的过程或做不到字符级别的描述，那将会产生大量的漏报。通过分析 ES 和 HTML 规范可以得知：以下的 API 可以修改字符串。做好以下这些 API 的记录就可以最大程度保证传播记录的完整性。
* String.prototype.anchor 用于创建 a 标签，样例代码如下：  
```c++
var myString = "Table of Contents";
document.body.innerHTML = myString.anchor("contents_anchor");
//输出.............
<a name="contents_anchor">Table of Contents</a>
```  
1. String.prototype.big 创建带有 big 标签的字符串，样例代码如下：  
```c++
var worldString = 'Hello, world';
console.log(worldString.big());       // <big>Hello, world</big>
```  
3. String.prototype.blink 创建带有 blink 标签的字符串，样例代码如下：  
```c++
var worldString = 'Hello, world';
console.log(worldString.blink());   // <blink>Hello, world</blink>
```
4. String.prototype.charAt  
5. String.prototype.charCodeAt，charCodeAt() 方法返回指定位置的字符的 Unicode 编码。返回值是 0 - 65535 之间的整数。  
6. String.prototype.codePointAt()   
7. String.prototype.concat，concat() 方法拼接两个或多个字符串并返回一个新的字符串  
8. String.prototype.fixed，样例代码如下：
```c++
var worldString = 'Hello, world';
console.log(worldString.fixed()); // "<tt>Hello, world</tt>"
```
9. String.prototype.fontcolor，样例代码如下：  
```c++
var worldString = 'Hello, world';
console.log(worldString.fontcolor('red') +  ' is red in this line');
// '<font color="red">Hello, world</font> is red in this line'
``` 
10. String.prototype.fontsize  
11. String.prototype.link，样例代码如下：
```c++
var hotText = 'MDN';
var url = 'https://developer.mozilla.org/';
console.log('Click to return to ' + hotText.link(url));
// Click to return to <a href="https://developer.mozilla.org/">MDN</a>
```  
12. String.prototype.italics，样例代码如下：
```c++
var worldString = 'Hello, world'; console.log(worldString.blink());
//"<i>str</i>"
```
13. String.prototype.match 
14. String.prototype.search  
15. String.prototype.matchAll  
16. String.prototype.normalize
17. String.prototype.padEnd，样例代码如下：
```c++
const str1 = 'Breaded Mushrooms';

console.log(str1.padEnd(25, '.'));
// expected output: "Breaded Mushrooms........"
```  
18. String.prototype.padStart   
19. String.prototype.repeat   
20. String.prototype.replace  
21. String.prototype.slice  
22. String.prototype.small，样例代码如下：  
```c++
var worldString = 'Hello, world';
console.log(worldString.small());     // <small>Hello, world</small>
```
22. String.prototype.split
23. String.prototype.strike  
24. String.prototype.sub，样例代码如下：  
```c++
var worldString = 'Hello, world';
console.log(worldString.sub());     // <sub>Hello, world</sub>
```
25. String.prototype.substr
26. String.prototype.substring
27. String.prototype.sup 
28. String.prototype.toLocaleLowerCase
29. String.prototype.toLocaleUpperCase
30. String.prototype.toLowerCase
31. String.prototype.toUpperCase
32. String.prototype.toString
33. String.prototype.trim  The trim() method removes whitespace from both ends of a string and returns a new string, without modifying the original string.  
34. String.prototype.trimEnd  
35. String.prototype.trimStart  
36. String.prototype.valueOf
37. RegExp.prototype.exec
38. document.write  
39. document.writeln
40. decodeURI  encodeURI  decodeURIComponent  encodeURIComponent  unescape  escape  
41. postMessage
42. '+' 加法    


总结，检测 XSS 就是记录数据从生产者到消费者的传播过程，这正污点分析技术的应有之义。  

**个人能力有限，有不足与纰漏，欢迎批评指正**  
**微信：qq9123013  备注：v8交流    邮箱：v8blink@outlook.com**
