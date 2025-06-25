+++
title = "Rust 学习笔记"
template = "series.html"
sort_by = "slug"
transparent = true

[extra]
series = true

[extra.series_intro_templates]
default = """
本文属于我的 $SERIES_HTML_LINK 系列。来源：原子之音。
[视频](https://www.bilibili.com/video/BV15y421h7j7/)
[代码](https://gitlab.com/yzzy/rust_project/)
"""

[extra.series_outro_templates]
next_only = """
📝 导航
- 下一篇: $NEXT_HTML_LINK
- [合集列表]($SERIES_PERMALINK)
"""

middle = """
---
📝 导航
- 上一篇: $PREV_HTML_LINK
- 下一篇: $NEXT_HTML_LINK
- [合集列表]($SERIES_PERMALINK)
"""

prev_only = """
📝 导航
- 上一篇: $PREV_HTML_LINK
- [合集列表]($SERIES_PERMALINK)
"""
+++

