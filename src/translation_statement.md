# 翻译说明

本书是续写版本，续写的版本由 Veykril 撰稿，续作对原作有补充和删改。

- 原作：[repo](https://github.com/DanielKeep/tlborm) | [渲染版](https://danielkeep.github.io/tlborm/)
- 原作中文翻译：[repo](https://github.com/DaseinPhaos/tlborm) | [渲染版](https://www.bookstack.cn/read/DaseinPhaos-tlborm-chinese/README.md)

原作及其翻译渲染版本没有使用 mdbook 构建，而是使用 py 来生成 HTML。
在发布文档和运行样板代码方面诸多不便。

而且由于原作在 2016 年没再更新，其内容基于 Rust 2015 版本，
续写的版本也只是把过时的细节更新至 2018 之后的版本。

我认为这本书在阐述 **声明宏** 方面搭建了一个很小巧精美的骨架，过程宏的资料比较丰富，而且过程宏生态主要围绕[第三方库](proc-macros/third-party-crates.md)开展。

续作及本翻译渲染版本使用 mdbook 构建：
- 续作：[repo](https://github.com/veykril/tlborm) | [渲染版](https://veykril.github.io/tlborm/)
- 续作中文翻译：[repo](https://github.com/zjp-CN/tlborm) | [渲染版](https://zjp-cn.github.io/tlborm)

另外，此翻译版本提供的阅读功能：
1. 行间代码块大部分可以点击右上角按钮运行，有些可以 **编辑** 和运行
（目的是快速而方便地验证读者思考的代码能否编译通过）。
只用于展示说明、或者不适合运行的代码只有复制按钮。

> 区分能编辑代码块的方法：光标能够在代码块中停留和闪动；有同级竖线；右上角有
> undo 图标；选中代码时背景色较浅；看代码块的主题颜色。

2. 每个页面右侧都有本章节的 **大纲目录** ，可以点击跳转。
如果大纲目录显示不完整，可以缩小浏览器页面；或者收起左侧的章节目录。
大纲目录仅在电脑网页版生效，移动端网页不会显示。

3. 所有 [`code`](#) 蓝色样式、光标移上去有下划线的内容（普通正文或者行内代码）都是链接，可以跳转。
无链接的行内代码样式是这样的：`code` 。

4. 翻译专有名词时，给出原英文，因为我认为那些词语是初次阅读英文时的障碍，
所以当读者查阅其他英文资料时，就不会感到陌生了。

# 更新日志

## 2022.04

主要补充的部分在于：
- [元变量表达式](./decl-macros/minutiae/metavar-expr.html)
- [macro 2.0](./decl-macros/macros2.html)
- [模式：TT “撕咬机”](./decl-macros/patterns/tt-muncher.html#性能建议) 之类的“模式”章节添加了“性能建议”
- [过程宏](./proc-macros.html)

## 2021.06

主要补充的部分在于：

- [元变量 (metavariables)](./decl-macros/macros-methodical.html#元变量)
- [片段分类符 (fragment-specifiers)](./decl-macros/minutiae/fragment-specifiers.md)
- [调试](./decl-macros/minutiae/debugging.md)
- [作用域](./decl-macros/minutiae/scoping.md)
- [导入/导出宏](./decl-macros/minutiae/import-export.md)
- [计数：bit twiddling](./decl-macros/building-blocks/counting.md#bit-twiddling)
- 译者补充：[算盘游戏](./decl-macros/building-blocks/abacus-counting.md#算盘游戏)
- [构件：解析](./decl-macros/building-blocks/parsing.md)

