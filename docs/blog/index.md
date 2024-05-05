# Squarehuang's 简介

之前博客的一些搬运工作，现在已经基本完成了，接下来会继续更新一些新的内容。

毕竟现在越来越不想折腾了，不过折腾还是很有意思的，不是吗(x，所以可能还是会继续折腾的。

``` py title="bubble_sort.py"
    def bubble_sort(items):
        for i in range(len(items)):
            for j in range(len(items) - 1 - i):
                if items[j] > items[j + 1]:
                    items[j], items[j + 1] = items[j + 1], items[j]
```
