---
title: "Mac修改esc键位映射"
date: 2018-09-14T00:00:00+08:00
---

MacBook Pro 的 touch bar 上的 esc 体验很差，尤其在频繁使用 vim 的时候，esc 不能给任何反馈，用着很不舒服。下面介绍两种方式用其它按键映射 esc(escape).

## 方法一 通过 Mac 自带方式

System Preferences - Keyboard - Modifier Keys
通过上面操作可以修改其它按键为 escape 按键。
此方法的缺点是不太灵活，例如选择将 option 改为 escape 时，不能选择左右，变更为 escape 后，则不能使用 option。

## 方法二 使用 Karabiner(KeyRemap4MacBook)

下载链接： https://pqrs.org/osx/karabiner/
将“右 option”改为“escape”(如图)，这样使用 escape(右 option)的同时，还保留了 option(左)自身的功能。
![karabiner.webp](/images/karabiner.webp)
