---
layout: post
title:  "Learn Emacs in Y Minutes"
date:   2020-06-29
categories: jekyll update
---

[GNU Emacs Reference Card](https://www.gnu.org/software/emacs/refcards/pdf/refcard.pdf)

同时按下 `control` 和 `chr` 记作 `C-<chr>`（一般用于字符、行），同时按下 `meta` (`alt`, `option`) 和 `chr` 记作 `M-<chr>`（一般用于单词、句子、段落）。

### 启动/关闭/模式

- 按 `q` 关闭欢迎界面
- 打开内置教程 `C-h t`
- 关闭 `C-x C-c`
- 挂起 `C-z`
- 终止执行、清除输入 `C-g`
- 有些命令需要二次确认，按 `n` 取消，按 `<SPC>`（空格）继续
- 使用人类语言的文本模式 `M-x text-mode`
- 显示当前模式文档 `C-h m`
- 在新窗口中显示命令文档 `C-h k ...`
- 命令的简短描述 `C-h c ...`
- 以关键字搜索命令 `C-h a ...`

### 窗口/光标移动

- 下移一屏 `C-v`（上移一屏 `M-v`）
- 光标所在行移至屏幕中间 `C-l`，再按一次至屏幕顶端，再按一次至屏幕底端
- 上移（Previous）`C-p`
- 下移（Next）`C-n`
- 左移（Backward）`C-b`（按词左移 `M-b`）
- 右移（Forward）`C-f`（按词左移 `M-f`）
- 移至行首 `C-a`（句首 `M-a`）
- 移至行尾 `C-e`（句末 `M-e`）
- 移至最首行 `M-<`
- 移至最末行 `M->`
- 重复命令 `C-u n ...`
- 关闭其他窗口 `C-x 1`
- 设置换行宽度 `C-x f ...`
- 前向搜索 `C-s ...`（后向 `C-r ...`）
- 窗口一分为二 `C-x 2`
- 在新窗口打开文件 `C-x 4 C-f ...`
- 副窗口下移一屏 `C-M-v`
- 进入其他窗口 `C-x o`
- 选中 `C-@`

### 编辑

- 删除 `<DEL>`（按词删除`M-<DEL>`）
- 向右删除字符 `C-d`（按词向右删除 `M-d`）
- 删至行尾 `C-k`（句尾 `M-k`）
- 粘贴最近删除内容（yank）`C-y`（替换为更早删除 `M-y`）
- 撤销 `C-/`
- 大写光标至单词结束的字符 `M-u`（小写 `M-l`，首字母大写 `M-c`）
- 交换左右单词 `M-t`（交换左右字符 `C-t`）
- 替换 `M-x replace-string ... ...`
- 剪切 `C-w`
- 复制 `M-w`

### 文件/缓存

- 打开文件 `C-x C-f ...`
- 保存 `C-x C-s`
- 显示缓存列表 `C-x C-b`
- 打开缓存 `C-x b ...`
- 保存缓存 `C-x s`
- 恢复自动保存文件 `M-x recover-file`
