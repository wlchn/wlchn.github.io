---
title: "不同域名之间共享localStorage-sessionStorage"
date: 2019-10-17T00:00:00+08:00
---

### 问题

两个不同的域名的 localStorage 不能直接互相访问。那么如何在`aaa.com`中如何调用`bbb.com`的 localStorage?

### 实现原理

1.在`aaa.com`的页面中，在页面中嵌入一个 src 为`bbb.com`的`iframe`，此时这个`iframe`里可以调用`bbb.com`的 localStorage。 2.用`postMessage`方法实现页面与`iframe`之间的通信。
综合 1、2 便可以实现`aaa.com`中调用`bbb.com`的 localStorage。

### 优化 iframe

我们可以在`bbb.com`中写一个专门负责共享 localStorage 的页面，例如叫做`page1.html`，这样可以防止无用的资源加载到`iframe`中。

### 示例

以在`aaa.com`中读取`bbb.com`中的 localStorage 的`item1`为例，写同理：
`bbb.com`中`page1.html`，监听`aaa.com`通过`postMessage`传来的信息，读取 localStorage，然后再使用`postMessage`方法传给`aaa.com`的接收者。

```
<!DOCTYPE html>
<html lang="en-US">
<head>
<script type="text/javascript">
    window.addEventListener('message', function(event) {
        if (event.origin === 'https://aaa.com') {
          const { key } = event.data;
          const value = localStorage.getItem(key);
          event.source.postMessage({wallets: wallets}, event.origin);
        }
    }, false);
</script>
</head>
<body>
  This page is for sharing localstorage.
</body>
</html>
```

在`aaa.com`的页面中加入一个 src 为`bbb.com/page1.html`隐藏的`iframe`。

```
<iframe id="bbb-iframe" src="https://bbb.com/page1.html" style="display:none;"></iframe>
```

在`aaa.com`的页面中加入下面 script 标签。在页面加载完毕时通过`postMessage`告诉`iframe`中监听者，读取`item1`。监听`bbb.com`传回的`item1`的值并输出。

```
<script type="text/javascript">
  window.onload = function() {
      const bbbIframe = document.getElementById('bbb-iframe');
      bbbIframe.contentWindow.postMessage({key: 'item1'}, 'https://bbb.com');
  }
  window.addEventListener('message', function(event) {
      if (event.origin === 'https://bbb.com') {
          console.log(event.data);
      }
  }, false);
</script>
```
