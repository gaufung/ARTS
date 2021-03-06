---
date: 2018-09-14
status: public
tags: Tools
title: SICP(01) 准备工作
---
# 1 Emacs 初体验
## 1.1 配置
`Emacs`包含了两个重要的控制键：`Control`键和`Meta`键。MBP键盘并不包含`Meta`，因此需要做一些配置修改，打开`terminal`的偏好设置，设置如下：
![](./_image/meta.jpg)
将`option`键设置为`meta`键.
# 2.2 Tutorial
阅读`emacs`中自带的`tutorial`，熟悉基本操作方法和逻辑。
![](./_image/Emacs.png)
# 2 Scheme 开发配置
Scheme为解释型语言，需要解析器执行代码，然后输出结果。在进入`scheme`交互式环境后，需要输入`(load "~/home/root/fib.scm" )` 每次脚本做出修改后都需要进行上述操作，费时费力，并不友好。因此我们希望借助`Emacs`能够帮助我们做到这一点。

## 2.1 设置`.emacs.d`文件
```lisp:n
;;; Always do syntax highlighting
(global-font-lock-mode 1)
;;; Also highlight parens
(setq show-paren-delay 0
      show-paren-style 'parenthesis)
(show-paren-mode 1)
;;; This is the binary name of my scheme implementation
(setq scheme-program-name "/usr/bin/scheme")
```
第`8`行用来配置`scheme`本地解释器的位置，可以使用`which scheme`命令查看。

## 2.2 使用`Emacs`
- 首先打开需要编写的`scheme`文件，输入`C-x 2`打开第二个窗口，并`C-x o`跳转到该窗口中；
- 输入`M-x run-scheme` ，该窗口进入`scheme`交互执行环境；
- 输入`C-x o`跳转到需要编辑的`scheme`文件，输入相关代码，输入`C-x C-e`执行光标所在的`S`表达式；
- 输入`C-x h C-c C-r`执行全部`scheme`文件代码；