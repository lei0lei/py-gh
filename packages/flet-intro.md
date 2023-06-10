---
layout: default
title: fletintro
permalink: /py/flet/intro
parent: flet
grand_parent: packages
has_toc: true
---
<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

一个最小的flet app有如下结构:
```python
import flet as ft

def main(page: ft.Page):
    # add/update controls on Page
    pass

ft.app(target=main)
```

常见的Flet程序以调用`flet.app()`为结束，app启动后会等待新用户的会话。main()函数是Flet应用的入口函数，在每个带有Page实例的用户会话传递给main函数的时候会在新线程上进行调用。如果在浏览器中运行Flet app对于每个新打开的tab或者页面会启动一个新的用户会话。如果是桌面app只会创建一个会话。

`Page`是某个用户的canvas，这是一个用户状态的可视化显示。在构建应用的界面的时候会给一个`page`添加或者移除`controls`并更新它们的属性。上面的最小例子会显示一个空白的页面。 

默认情况下Flet应用会启动一个本地的系统窗口，像下面这样更改对`flet.app()`的调用可以改成在浏览器窗口中打开:

```python
ft.app(target=main, view=ft.WEB_BROWSER)
```

{: .note }
每个Flet app都是一个web app，即使实在本地的系统窗口中打开在后台也会启动一个内建的web服务器。这个服务器叫做`Fletd`,会监听在一个随机的TCP端口上，可以自定义一个TCP端口然后在浏览器中打开app.

# controls

用户接口由`Controls`构成(在flutter中这叫做widgets)，controls被嵌套在其他controls或者添加到`Page`中时才能让用户见到，`Page`是最顶层的control,以`Page`为根，嵌套的controls可以表示为一颗树。


`Controls`是python类，通过带参数的构造器创建control的实例然后把属性传递进入，比如: 
```python
t = ft.Text(value="Hello, world!", color="green")
```

将control添加到一个`Page`的control list中然后调用`page.update()`将更改发送给浏览器或者客户端可以显示这个control:
```python
import flet as ft

def main(page: ft.Page):
    t = ft.Text(value="Hello, world!", color="green")
    page.controls.append(t)
    page.update()

ft.app(target=main)
```
通过更改control的属性在下次`page.update()`的时候就可以更新ui的显示。

```python
t = ft.Text()
page.add(t) # it's a shortcut for page.controls.append(t) and then page.update()

for i in range(10):
    t.value = f"Step {i}"
    page.update()
    time.sleep(1)
```
一些controls是容器类的controls比如`Page`,可以包含其他的controls.`Row` control可以将其他controls组织成一行行的：
```python
page.add(
    ft.Row(controls=[
        ft.Text("A"),
        ft.Text("B"),
        ft.Text("C")
    ])
)
```
也可以添加`TextField`和`ElevatedButton`：
```python
page.add(
    ft.Row(controls=[
        ft.TextField(label="Your name"),
        ft.ElevatedButton(text="Say my name!")
    ])
)
```
`page.update()`可以智能的在上次调用之后反应control的改变，因此可以给page添加一些新的controls,移除controls，更改control的属性然后调用`page.update`进行批量的更新，比如:

```python
for i in range(10):
    page.controls.append(ft.Text(f"Line {i}"))
    if i > 4:
        page.controls.pop(0)
    page.update()
    time.sleep(0.3)
```

一些controls比如buttons具有event handler响应用户的输入,比如`ElevatedButton.on_click`:
```python
def button_clicked(e):
    page.add(ft.Text("Clicked!"))

page.add(ft.ElevatedButton(text="Click me", on_click=button_clicked))
```

下面是一个更高级的例子：
```python

import flet as ft

def main(page):
    def add_clicked(e):
        page.add(ft.Checkbox(label=new_task.value))
        new_task.value = ""
        new_task.focus()
        new_task.update()

    new_task = ft.TextField(hint_text="Whats needs to be done?", width=300)
    page.add(ft.Row([new_task, ft.ElevatedButton("Add", on_click=add_clicked)]))

ft.app(target=main)
```

## `visible`属性
每个control默认都有个`visible`属性，即默认上control会在page上渲染。将visible设置成false会禁止control在页面上渲染。

## `disabled`属性

每个control还有个disabled属性默认上是false，即control和所有的子control都被启用，disabled属性常用于数据入口controls比如TextField，Dropdown，Checkbox,button.disabled可以设置给父contorl并将值迭代的给所有孩子。

比如有个form有多个entry control,可以单独给每个control设置disabled属性。

```python
first_name = ft.TextField()
last_name = ft.TextField()
first_name.disabled = True
last_name.disabled = True
page.add(first_name, last_name)
```

或者可以吧form control交给container,比如Column：
```python
first_name = ft.TextField()
last_name = ft.TextField()
c = ft.Column(controls=[
    first_name,
    last_name
])
c.disabled = True
page.add(c)
```
# Control Refs
flet的controls是对象，要访问属性需要保持对这些对象的引用。
```python
import flet as ft

def main(page):

    first_name = ft.TextField(label="First name", autofocus=True)
    last_name = ft.TextField(label="Last name")
    greetings = ft.Column()

    def btn_click(e):
        greetings.controls.append(ft.Text(f"Hello, {first_name.value} {last_name.value}!"))
        first_name.value = ""
        last_name.value = ""
        page.update()
        first_name.focus()

    page.add(
        first_name,
        last_name,
        ft.ElevatedButton("Say hello!", on_click=btn_click),
        greetings,
    )

ft.app(target=main)
```

在main方法的开始创建了三个controls，我们会在`on_click`句柄中用到，2个`TextField`用于输入名和姓，基于他们的属性来创建controls,在最后进行更新。

随着controls和事件句柄的增加，将所有的contorl定义放在一起很难,他们会分布在main()函数体的各个部分，查看page.add()参数很难想象界面的样子,比如:
```python
    page.add(
        first_name,
        last_name,
        ft.ElevatedButton("Say hello!", on_click=btn_click),
        greetings,
    )
```
first_name是个TextField，它有没有autofocus集,greetings是Row还是Column?

flet 有一个Ref功能类可以给control定义一个reference,在句柄中使用然后将reference设置给真正的control.

定义提一个新类型的control ref:

```python
first_name = ft.Ref[ft.TextField]()
```
进行访问:
```python
# empty first name
first_name.current.value = ""
```
要把contorl赋值给ref,设置Control.ref属性:
```python
page.add(
    ft.TextField(ref=first_name, label="First name", autofocus=True)
)
```

所有的controls都有ref属性:
```python
import flet as ft


def main(page):

    first_name = ft.Ref[ft.TextField]()
    last_name = ft.Ref[ft.TextField]()
    greetings = ft.Ref[ft.Column]()

    def btn_click(e):
        greetings.current.controls.append(
            ft.Text(f"Hello, {first_name.current.value} {last_name.current.value}!")
        )
        first_name.current.value = ""
        last_name.current.value = ""
        page.update()
        first_name.current.focus()

    page.add(
        ft.TextField(ref=first_name, label="First name", autofocus=True),
        ft.TextField(ref=last_name, label="Last name"),
        ft.ElevatedButton("Say hello!", on_click=btn_click),
        ft.Column(ref=greetings),
    )

ft.app(target=main)
```





