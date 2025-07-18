+++
title = "Rust 学习笔记"
template = "series.html"
sort_by = "slug"
transparent = true
paginate_by = 10

[extra]
series = true
post_listing_index_reversed = false

[extra.series_intro_templates]
default = """
本文属于我的 $SERIES_HTML_LINK 系列。

Rust 入门学习笔记以实际例子为主，讲解部分不是从零开始的，所以不建议纯萌新观看，读者最好拥有任意一种面向对象语言的基础，然后自己多多少少看过 Rust 的基本语法，刷过一点 [rustlings](https://github.com/SandmeyerX/rustlings-zh-cn)。

来源：原子之音。当然也包含个人的一些补充。
[视频](https://www.bilibili.com/video/BV15y421h7j7/)
[代码](https://gitlab.com/yzzy/rust_project/)

Rust 进阶学习笔记以及实战的来源则五花八门，将会标注在下一行。
"""

[extra.series_outro_templates]
next_only = """
📝 系列导航
- 下一篇: $NEXT_HTML_LINK
- <a href=\"$SERIES_PERMALINK\">合集列表</a>
"""

middle = """
---
📝 系列导航
- 上一篇: $PREV_HTML_LINK
- 下一篇: $NEXT_HTML_LINK
- <a href=\"$SERIES_PERMALINK\">合集列表</a>
"""

prev_only = """
📝 系列导航
- 上一篇: $PREV_HTML_LINK
- <a href=\"$SERIES_PERMALINK\">合集列表</a>
"""
+++
