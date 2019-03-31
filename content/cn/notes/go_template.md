---
title: "深入理解GoTemplate"
date: 2018-08-24
type: "notes"
draft: false
---

虽然近几年Restful架构的盛行推动前后端分离大行其道，模板渲染的地方由后端转移到了前端，类似于JSP和PHP的传统技术没多少人在用了。但是在Go语言中，模板技术不只局限于服务端HTML页面的渲染，可以实现文本化的输出，对于某些云计算的场景十分友好。今天，我们就来详细聊一聊Go Template的技术细节。

## 运行机制

模板的渲染技术本质上都是一样的，一句话来说明就是**字串模板和结构化数据的结合**。再详细的讲就是将定义好的模板应用于结构化的数据，使用注解语法引用数据结构中的元素（例如Struct中的特定feild，Map中的key）并显示它们的值。模板在执行过程中遍历数据结构并且设置当前光标（`"."`表示当前的作用域）标识当前位置的元素。

类似于Python的[jinja](http://jinja.pocoo.org/)，Node的[jade](http://jade-lang.com/)等模版引擎，Go语言模板引擎的运行机制也是类似：

1. 创建模板对象
2. 解析模板字串
3. 加载数据渲染模板

![](https://i.loli.net/2019/03/31/5ca036206c6f3.jpg)

## Warning Up

Go提供了两个标准库用来处理模板`text/template`和`html/template`，它们的接口基本一样，其中`text/template`用来处理普通文本的模板渲染，而`html/template`专门用来格式化`html`字符串。

下面的例子我们使用`text/template`来处理普通文本模板的渲染：

```
package main

import (
	"os"
        "text/template"
)

type Student struct {
    ID      uint
    Name    string
}

func main() {
    stu := Student{0, "jason"}
    tmpl, err := template.New("test").Parse("The name for student {{.ID}} is {{.Name}}")
    if err != nil { panic(err) }
    err = tmpl.Execute(os.Stdout, stu)
    if err != nil { panic(err) }
}
```

上述代码第4行引入`text/template`来处理普通文本模板渲染，第14行定义一个模板对象`test`来解析变量`"The name for student {{.ID}} is {{.Name}}"`模板字符串，第16行使用定义好的结构化数据来渲染模版到标准输出。

> Note: 要引用的模板数据一定是export出来的，也就是说对应的字段必须以大写字母开头，比如例子中的`ID`, `Name`。

我们再来看一个HTML字符串默板渲染的例子：

```
func templateHandler(w http.ResponseWriter, r *http.Request){
    tmpl := `<!DOCTYPE html>
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8"> <title>Go Template Demo</title>
    </head>
    <body>
        {{ . }}
    </body>
</html>`
    
    t := template.New("hello.html")
    t, _ = t.Parse(tmpl)
    t.Execute(w, "Hello, Go Template!")
}
```

> Note: 在Go语言中不倾向于使用单引号来表示字符串，请根据需要使用双引号或反引号。另外，Go语言的字符串类型在本质上就与其他语言的字符串类型不同。Java的`String`、C++的`std::string`以及python3的`str`类型都只是**定宽字符序列**，而Go语言的字符串是一个用`UTF-8`编码的**变宽字符序列**，也就是说，它的每一个字符都用一个或多个字节表示。
> Go语言中的字符串字面量使用双引号或反引号("`")来创建：
> - 双引号用来创建可解析的字符串字面量 (支持转义，但不能用来引用多行)；
> - 反引号用来创建原生(raw)字符串字面量，这些字符串可能由多行组成(不支持任何转义序列)，原生的字符串字面量多用于书写多行消息、HTML以及正则表达式。

本地部署执行：

```
curl -i http://127.0.0.1:8080/
HTTP/1.1 200 OK
Date: Fri, 09 Dec 2016 09:04:36 GMT
Content-Length: 223
Content-Type: text/html; charset=utf-8

<!DOCTYPE html>
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8"> <title>Go Template Demo</title>
    </head>
    <body>
        Hello, Go Template!
    </body>
</html>
```

go不仅可以直接解析模板字串，也可以使用[ParseFile](https://golang.org/pkg/html/template/#ParseFiles)解析模板文件，还是就是标准的处理流程：**创建-加载-渲染**。

## 模板命名

之前的例子模板对象是有名字的，可以在创建模板对象的时候显示命名，也可以让Go Template自动命名。但是如果涉及到嵌套模板的时候，该如何命名模板呢，这种情况下，模板文件有好多个！

Go Template提供了[ExecuteTemplate方法](https://golang.org/pkg/html/template/#Template.ExecuteTemplate)，用于执行指定名字的Go模板。比如加载`hello`模板的时候，可以指定layout.html

```
tmplstr := `{{ define "stu_info" }}
The name for student {{.ID}} is {{.Name}}
{{ end }}
{{ define "stu_name" }}
Student name is {{.Name}}
{{ end }}
`
stu := Student{0, "jason"}
tmpl, err := template.New("test").Parse(tmplstr)
if err != nil { panic(err) }
err = tmpl.ExecuteTemplate(os.Stdout, "stu_info", stu)
if err != nil { panic(err) }
}
```

在模板字符串中，使用了`define`这个action定义了两个命名模版`stu_info`，`stu_name`。这是虽然`Parse`方法返回的模板对象里面包含两个模板名， 但是`ExecuteTemplate`执行的模板还是`stu_info`。

不仅可以通过`define`定义模板，还可以通过`template`引入定义好的模板，类似jinja的`include`特性：

```
tmplstr := `{{ define "stu_name" }}
Student name is {{.Name}}
{{ end }}
{{ define "stu_info" }}
{{ template "stu_name" . }}
{{ end }}
`
stu := Student{0, "jason"}
tmpl, err := template.New("test").Parse(tmplstr)
if err != nil { panic(err) }
err = tmpl.ExecuteTemplate(os.Stdout, "stu_info", stu)
if err != nil { panic(err) }
}
```

上面的例子当中我们在`stu_into`模板中使用`template`引入了`stu_name`模板，同时传给`stu_name`模板当前作用域的数据(`.`)，第三个参数是可选的，如果为空，则表示传给嵌套模版的数据为`nil`。

总而言之，创建模板对象后和加载多个模板文件，执行模板文件的时候需要指定base模板（`stu_info`），在base模板中可以引入其他命名的模板。无论点`.`，`define`，`template`这些在双花括号的关键字其实都是Go Template的action（模板标签）。

## Action

Go Template的action是用于动态执行一些逻辑和展示数据的形式，大致分为下面几类：

- 条件语句
- 迭代
- 封装
- 引用

我们在之前的例子中看到了`define`以及`template`的用法，下面再看看其他的action怎么使用：

#### 条件判断

条件判断的语法很简单：

```
{{if pipeline}} T1 {{end}}
	If the value of the pipeline is empty, no output is generated;
	otherwise, T1 is executed. The empty values are false, 0, any
	nil pointer or interface value, and any array, slice, map, or
	string of length zero.
	Dot is unaffected.

{{if pipeline}} T1 {{else}} T0 {{end}}
	If the value of the pipeline is empty, T0 is executed;
	otherwise, T1 is executed. Dot is unaffected.

{{if pipeline}} T1 {{else if pipeline}} T0 {{end}}
	To simplify the appearance of if-else chains, the else action
	of an if may include another if directly; the effect is exactly
	the same as writing
		{{if pipeline}} T1 {{else}}{{if pipeline}} T0 {{end}}{{end}}
```

#### 迭代

对于一些数组，Slice或者是Map，可以使用迭代的action，与Go语言本身的迭代类似，使用`range`进行处理：

详细规范如下：

```
{{range pipeline}} T1 {{end}}
	The value of the pipeline must be an array, slice, map, or channel.
	If the value of the pipeline has length zero, nothing is output;
	otherwise, dot is set to the successive elements of the array,
	slice, or map and T1 is executed. If the value is a map and the
	keys are of basic type with a defined order ("comparable"), the
	elements will be visited in sorted key order.

{{range pipeline}} T1 {{else}} T0 {{end}}
	The value of the pipeline must be an array, slice, map, or channel.
	If the value of the pipeline has length zero, dot is unaffected and
	T0 is executed; otherwise, dot is set to the successive elements
	of the array, slice, or map and T1 is executed.
```

```
{{ range . }}
    <li>{{ . }}</li>
{{ else }}
 empty
{{ end }}
```

当`range`的结构为空的时候，则会执行`else`分支的逻辑。

#### with封装

`with`语言在Python中可以开启一个上下文环境。对于Go Template，`with`语句类似，其含义就是创建一个封闭的作用域，在其范围内，可以使用`.action`，而与外面的`.`无关，只与`with`的参数有关：

详细规范如下：

```
{{with pipeline}} T1 {{end}}
	If the value of the pipeline is empty, no output is generated;
	otherwise, dot is set to the value of the pipeline and T1 is
	executed.

{{with pipeline}} T1 {{else}} T0 {{end}}
	If the value of the pipeline is empty, dot is unaffected and T0
	is executed; otherwise, dot is set to the value of the pipeline
	and T1 is executed.
```

举个例子：

```
{{ with arg }}
    .
{{ end }}
```

在上面`with`里面的`.`代表with新开辟的作用域，而不是with外面的作用域。`with`语句的`.`与其外面的`.`是两个不相关的对象。`with`语句也可以有`else`。`else`中的`.`则和`with`外面的`.`一样，毕竟只有with语句内才有封闭的上下文：

```
{{ with ""}}
 Now the dot is set to {{ . }}
{{ else }}
 {{ . }}
{{ end }}
```

#### 参数和管道

前面我们提到过`template` action可以有可选的参数来传递给内部嵌套模版。实际上，引用除了模板的include，还包括参数的传递。

模板的参数可以是Go中的基本数据类型，如数字，布尔值，字符串，数组切片或者一个结构体。在模板中设置变量可以使用：

```
$variable := value
```

Go还有一个特性就是模板的管道函数，熟悉django和jinja的开发者应该很熟悉这种手法。通过定义函数过滤器，实现模板的一些简单格式化处理。并且通过管道哲学，这样的处理方式可以连成一起。

例如，模板内置了一些函数，比如格式化输出：

```
{{ 3.1415926 | printf "%.3f" }}
```

#### 函数

既然管道符可以成为模板中的过滤器，那么除了内建的函数，Go Template还支持自定义函数可以扩展模板的功能：

在Go Template中定义一个函数分两步：

1. 创建一个`FuncMap`类型的map，key是模板函数的名字，value是其函数的定义
2. 将`FuncMap`注入到模板中


```
funcMap := template.FuncMap{"fdate": formDate}
tmpl := template.New("test").Funcs(funcMap)
tmpl = template.Must(t.Parse(`test data: {{.}}`))
tmpl.ExecuteTemplate(os.Stdout, "test", time.Now())
```

在模板中可以使用`{{ . | fdate }}`，当然也可以不用管道过滤器，而是使用正常的函数调用形式`{{ fdate . }}`。

## Summary

Go Template经常用来处理譬如插入特定数据的文本转化等，虽然没有正则表达式那么灵活，但是渲染速度超过正则表达式，而且使用起来也更简单。
