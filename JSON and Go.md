# JSON and Go
25 January 2011  
（译者注：“json package”或者“json包”指的是Golang的json标准库，“JSON”表示的是编码后的json数据。）
## 介绍
JSON(JaveScript对象表示法)是一个简单的数据交换格式。
在语法上它类似于JavaScript的对象和列表。它经常用于web后端与运行于浏览器中的JavaScript程序交流数据，但是它也被用于其他很多地方。在JSON的[官网](http://json.org/)上，提供了一个非常清晰和简洁的标准定义。

[json package](https://golang.org/pkg/encoding/json/)，这个包提供了一些读取和写入JSON数据的有用的东东，可以在你的go程序中调用。

## 编码
我们使用Marshal函数编码JSON数据。

```
func Marshal(v interface{}) ([]byte, error)
```
我们后面的代码使用这样一个struct，Message：

```
type Message struct {
    Name string
    Body string
    Time int64
}
```
Message的一个实例：

```
m := Message{"Alice", "Hello", 1294706395881547000}
```
我们可以使用json.Marshal对m进行编码：

```
b, err := json.Marshal(m)
```
如果一切正常的话，err的值会是nil，b的值会是一个包含了JSON数据的[]byte：

```
b == []byte(`{"Name":"Alice","Body":"Hello","Time":1294706395881547000}`)
```
- 只有那些表现可用的JSON数据结构才会被编码：
- JSON对象仅支持字符串作为key，可编码为JSON数据的Go map类型必须这样一种形式map[string]T（T是任何被json包支持的Go类型）；
- channel，complex，和function类型不能被编码；
- 环回结构体不被支持，它们会使Marshal陷入无尽的循环；
- 指针会被编码为它们所指向的值（nil指针会被编码为“null”）。

因为json包仅能访问导出的结构字段（以大写字符开始的字段），所有只有那些导出的结构字段才会出现在JSON编码输出中。

## 解码
我们使用Unmarshal函数对JSON数据进行解码。

```
func Unmarshal(data []byte, v interface{}) error
```
我们必须创建一个变量，用于存储解码后的数据。

```
var m Message
```
然后调用json.Unmarshal，同时把[]byte类型的JSON数据和m的指针传给这个函数：

```
err := json.Unmarshal(b, &m)
```
如果b包含可用的JSON数据，并与m中的字段相符合，err的值将会是nil，b的值将会被存储到m中，就好像做了如下的赋值：

```
m = Message{
    Name: "Alice",
    Body: "Hello",
    Time: 1294706395881547000,
}
```
Unmarshal是如何知道把解码后的数据存储进哪些结构字段的？比如有这么一个json key：“Foo”，Unmarshal将会按照下面的顺序查找目标结构体中的字段（优先级顺序）：
- 可导出的字段并且具有这样一个tag：“Foo”（有关结构tag更多信息请参考[Go Spec](https://golang.org/ref/spec#Struct_types)）
- 可导出的字段名为“Foo”，或者
- 可导出的字段名为“FOO”或者“FoO”或者其他的“Foo”的大小写的匹配

当JSON数据结构没有找到匹配的Go类型时会发生什么？

```
b := []byte(`{"Name":"Bob","Food":"Pickle"}`)
var m Message
err := json.Unmarshal(b, &m)
```
Unmarshal仅解码在目标类型中找到匹配的字段。以上面的变量为例，m的“Name”字段将被解码填充，“Food”字段将被忽略。这种现象特别适用于仅仅希望从一个大的JSON数据中提取少量字段的情况。并且目标结构体中的非导出字段是不会被Unmarshal影响到的。

但是如果你事先不知道JSON数据的结构该怎么办呢？

## 对JSON数据通用的interface{}
interface{}类型（空接口）描述的是一个具有0个方法的接口。每个Go类型实现了至少0个方法并且满足这个空接口。

空接口被当做一种通用的容器来使用：

```
var i interface{}
i = "a string"
i = 2011
i = 2.777
```
类型断言可以访问接口底层的类型：

```
r := i.(float64)
fmt.Println("the circle's area", math.Pi*r*r)
```
或者，如果底层类型时未知的，可以通过type switch判断其底层的类型：

```
switch v := i.(type) {
case int:
    fmt.Println("twice i is", v*2)
case float64:
    fmt.Println("the reciprocal of i is", 1/v)
case string:
    h := len(v) / 2
    fmt.Println("i swapped by halves is", v[h:]+v[:h])
default:
    // i isn't one of the types above
}
```
json包使用map[string]interface{}和[]interface{}类型的值存储任意的JSON对象和数组；json包非常乐意将任何可用的JSON二进制序列编码到一个interface{}类型的值中。默认的具体Go类型有：
- bool对应JSON booleans
- float64对应JSON numbers
- string对应JSON strings，以及
- nil对应JSON null。

## 解码任意数据
考虑下这个存储在变量b中的JSON数据：

```
b := []byte(`{"Name":"Wednesday","Age":6,"Parents":["Gomez","Morticia"]}`)
```
无须知道JSON数据的结构，我们也可用使用Unmarshal将JSON数据解码至interface{}值中：

```
var f interface{}
err := json.Unmarshal(b, &f)
```
依上面的代码来看来，存储在f中的将是一个map，这个map的key是字符串，key的值是存储在空接口中的值：

```
f = map[string]interface{}{
    "Name": "Wednesday",
    "Age":  6,
    "Parents": []interface{}{
        "Gomez",
        "Morticia",
    },
}
```
要访问这样的数据，我们可以使用类型断言访问f底层的map[string]interface{}：

```
m := f.(map[string]interface{})
```
我们使用一个range语句遍历这个map，然后使用type switch访问其实际类型的值：

```
for k, v := range m {
    switch vv := v.(type) {
    case string:
        fmt.Println(k, "is string", vv)
    case float64:
        fmt.Println(k, "is float64", vv)
    case []interface{}:
        fmt.Println(k, "is an array:")
        for i, u := range vv {
            fmt.Println(i, u)
        }
    default:
        fmt.Println(k, "is of a type I don't know how to handle")
    }
}
```
通过这种方式我们可以应对未知结构的JSON数据，同时安全的体验type的好处。
## 引用类型
让我们定义一个新的Go type存储上面例子中的数据：

```
type FamilyMember struct {
    Name    string
    Age     int
    Parents []string
}

    var m FamilyMember
    err := json.Unmarshal(b, &m)
```
正如我们期望的那样，JSON数据被完美的解码到了FamilyMember类型的值中，但是如果我们自己观察的话，有一个值得注意的事情发生了。伴随着var关键字，我们分配了一个FamilyMember类型的变量，同时提供了此值的指针用于解码，但是同时m的Parents字段的值是个nil切片值。为了填充Parents字段，Unmarshal暗中分配了一个新的切片。这就是典型的Unmarshal如何处理支持的引用类型（pointers, slices, and maps）的情况。

考虑解码数据到如下结构的情况：

```
type Foo struct {
    Bar *Bar
}
```
如果JSON对象中存在一个Bar字段，Unmarshal将分配一个新的Bar并填充它。如果没有，Bar将仍旧是一个指向nil的指针。

从这个有用的特性我们可以延伸下：如果你有一个应用，并且只接收少量的可区分的消息类型，你可以定义这样一个“receiver（可以接收所有消息的接收器）”的结构：

```
type IncomingMessage struct {
    Cmd *Command
    Msg *Message
}
```
（未完待续）