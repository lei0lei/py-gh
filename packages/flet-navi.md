---
layout: default
title: navigation
permalink: /py/flet/navigation
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

# page views
flet的page不再是一个单独的page,而是View的容器，像一个三明治一样一层层的.

Page具有`page.views`属性来访问view集合。view列表的最后一层会显示在一个页面上，这个list最少有一个元素(root view).

要模拟页面间的奇幻，更改`page.route`并在`page.view`列表的最后添加一个新的View.

在view集合的最后弹出view然后在page.on_view_pop事件句柄中更改route到前一个来返回。

# route更改后构建views

要构建可靠的导航在程序的某个地方需要根据当前的route构建view list.导航的history stack必须是路由的一个函数。下面是一个在两个页面中导航的例子:
```python
import flet as ft

def main(page: ft.Page):
    page.title = "Routes Example"

    def route_change(route):
        page.views.clear()
        page.views.append(
            ft.View(
                "/",
                [
                    ft.AppBar(title=ft.Text("Flet app"), bgcolor=ft.colors.SURFACE_VARIANT),
                    ft.ElevatedButton("Visit Store", on_click=lambda _: page.go("/store")),
                ],
            )
        )
        if page.route == "/store":
            page.views.append(
                ft.View(
                    "/store",
                    [
                        ft.AppBar(title=ft.Text("Store"), bgcolor=ft.colors.SURFACE_VARIANT),
                        ft.ElevatedButton("Go Home", on_click=lambda _: page.go("/")),
                    ],
                )
            )
        page.update()

    def view_pop(view):
        page.views.pop()
        top_view = page.views[-1]
        page.go(top_view.route)

    page.on_route_change = route_change
    page.on_view_pop = view_pop
    page.go(page.route)


ft.app(target=main, view=ft.WEB_BROWSER)
```



