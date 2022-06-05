![avatar](cyclops.ico)**Cyclops is a web browser with XSS detection feature**   

📝[中文说明](https://github.com/v8blink/Chromium-based-XSS-Taint-Tracking/blob/main/README.md)   

The Cyclops's binary code can be directly [downloaded here](https://github.com/v8blink/Chromium-based-XSS-Taint-Tracking/releases); It's source code is not provided now.    

Cyclops is open evaluation for Microsoft Windows 10; Versions for Linux and Mac will be published soon.

# User Manual
Cyclops is still in development. If any problem occurs, please contact us.  
When using Cyclops, arg "**--no-sandbox**" is needed. If any suspicious XSS is detected, a log file named "SourceSink.txt" will be generated in running directory (Default:C:\Users\UserName\AppData\Local\Chromium\Application\97.0.4684.0\). The format of SourceSink.txt will be introduced below.  

[DEMO video 1](https://www.zhihu.com/zvideo/1505471657166282752)，[DEMO video 2](https://www.zhihu.com/zvideo/1505847898797969409)          
**Note** Cyclops updates frequently so newly-developed features may not be included in the videos above.

# To do list   
Ordered by priorities  
**0. id field must be included in sink info.**    
E.g. "Sink:HTMLLIElement.className" without id field will display as Element type, and it must display its id if it is not null. This feature is in progress...
**1.source and sink**
Currently, the supported sources and sinks are as follows:  
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

**2.crash error fix**    
Our test cases only cover limited scenarios, so we will be appreciated if you raise an issue when encountering a crash.   
**3.optimization for SourceSink.txt**   
The current format of SourceSink.txt is temporary. We intend to output it as JSON or redirect it to a specific port. But limited by our finite ability and the complexity of Chromium's architecture, we have not solve the crash problem yet.   
We are trying to implement our thoughts.   

# Description of SourceSink.txt  
Each line of the file is a record of sink, with three parts divided by comma. The first part is sink info; The second part is a test ID; The third part is the source of the data, with embeddable units enclosed with [].
Here is an example listed below:
>[Sink:alert:qerwr , 910226, [ENURI:[Substring:[Location.search]]]]


"Sink:alert:qerwr", divided by colon, with "alert" as its destination method name, and its parameter is string "qerwr";  
910226 represents to a test ID;    
[ENURI:[Substring:[Location.search]]] describes the data source of the parameter of method "alert". alert(EncodeURI(location.search.substring()));

>[Sink:HTMLLIElement.className ,910226, [Trim:[ADD:[ADD:[ADD:,[HTMLLIElement.className]],],]]]
  
Sink:HTMLLIElement.className  divided by colon, represents the termination of the data is a className property of an Element.    
910226 represents to a test ID;    
[Trim:[ADD:[ADD:[ADD:,[HTMLLIElement.className]],],]]] ADD means operator +; If an operand does not contain tainted info, it will be an empty string "".
 
**Note** Test ID 910226 must be with data source. A test id without data source or data source without test id is a bug. Bug report around this issue will be appreciated.
"Trim" and "ENURI" in the examples above are abbreviation marks used in Sink data flow. Full abbreivation mark table is listed below.

|Mark|JavaScript function|Direction|Illustration|
|----|----|----|----|
|A|string.anchor|[A:strX]|`<a name=undefined>strX</a>`|
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
|Geval|eval method|
|Deval|eval method|
|ADD|addition|[ADD:strX,strY]|missing operand(s) means no tainted info|
|Pad:,E=|string.padEnd|[Pad:strx,E=strY]|b=strX.padend(num,strY), it will be recorded if the value of b differs from that of strX|  
|Rep:rec=,s=|strin.replace|[Rep:rec=hello,s=world]|strX.replace("hello","world"), find the "hello" substring from strX and replace it with substring "world"|  
|Repeat|string.repeat|[Repeat:strX]|Repeat times are currently not recorded. It will be updated soon|
|Slice|string.slice|[Slice:strX]|b=strX.slice(i,j)，when the length of b is greater than zero and less than strX, it will be recorded|
|Substr|string.substr|[Substr:strX]|refer to the above example| 
|Trim|string.trim|Does not distinguish trimStart, trimEnd, trimLeft and trimRight   
|Concat|string.concat|[Concat:strX,strY] or [Concat:strX,strY,strZ...]|b=strX.concat(strY),b=strX.concat(strY,strZ...)       
|gAttr|element.getAttribute()|[gAttr:name:value]|e.g. `<input id=name value='huidou'/>`, method "input.getAttribute('value')" generates the record, represents that the data comes from a "value" property of an Element, whose id is "name"。
|sAttr|element.setAttribute()|[sAttr:name:value]|same as the above example, represents to a write action to the "value" property of an Element, whose "id" is name
|F:P=,B=|Function() constructor|[F:P=p1,p2,B=evil]|P represents parameter, p1 and p2 represents data source of the two parameters;B represents the data source of the function body. Generates this record if P or B is attacker-controlled.   

**Note** common properties/methods such as location.href, settimeout and alert are not listed above because they just simply use their full names.  
# Disclaimer  
The Cyclops can only be used in the safety construction of enterprises with sufficient legal authorization. During the use of the Cyclops, you should ensure that all your actions comply with local laws and regulations. If you have any illegal behavior in the process of using this tool, you will bear all the consequences yourself, and all developers and all contributors of this tool will not bear any legal and joint liability.
