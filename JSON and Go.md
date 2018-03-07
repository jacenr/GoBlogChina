# JSON and Go  
25 January 2011  
## 介绍
JSON(JaveScript对象表示法)是一个简单的数据交换格式。
在语法上它类似于JavaScript的对象和列表。它经常用于web后端与运行于浏览器中的JavaScript程序交流数据，但是它也被用于其他很多地方。在JSON的[官网](http://json.org/)上，提供了一个非常清晰和简洁的标准定义。

[json package](https://golang.org/pkg/encoding/json/)，这个包提供了一些读取和写入JSON数据的有用的东东，可以在你的go程序中调用。

## 编码
