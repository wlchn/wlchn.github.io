---
title: "Mac解决too-many-open-files问题"
date: 2022-08-10T00:00:00+08:00
---

查看 maxfiles 设置

```
launchctl limit
```

设置一个较大的 maxfiles

```
sudo launchctl limit maxfiles 102400 102400
```

查看应用资源情况

```
lsof -n | awk '{print $1}' | uniq -c | sort -rn | head
```
