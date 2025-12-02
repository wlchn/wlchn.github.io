---
title: "为React单页应用提供Runtime运行时环境变量"
date: 2019-03-27T00:00:00+08:00
---

使用 create-react-app 创建的单页应用（SPA）是在 build 时注入环境变量的。一旦 build 成静态文件便不能动态提供环境变量了。
比如 build 一个单页应用的 docker image，可以在 build 时提供环境变量。但是已经 build 完成，使用 docker run 运行的时候不能再传递环境变量。
本文主要解决在运行时提供环境变量的问题。
原理：通过一段 shell 脚本将指定将 env 转换为 config.js 文件，该文件和 build 好的 static 文件 serve 在同一目录下，在应用中引用 config.js 文件，通过 window.\_env 获取环境变量。shell 脚本通过 docker image 的 entrypoint 执行。服务使用 go 构建的一个简单服务器，比 nginx 轻量很多。

1. 将如下`env.sh`文件拷问到项目目录下，在 entrypoint（docker run）时执行，作用是将环境变量转出 config.js

```shell
#!/bin/sh
if [ $CONFIG_VARS ]; then
  # clear
  echo -n > ${CONFIG_FILE_PATH}/config.js

  SPLIT=$(echo $CONFIG_VARS | tr "," "\n")
  echo "window._env = {" >> ${CONFIG_FILE_PATH}/config.js

  for VAR in ${SPLIT}; do
      VALUE=$(printenv ${VAR})
      echo "  ${VAR}: \"${VALUE}\"," >> ${CONFIG_FILE_PATH}/config.js
  done

  echo "}" >> ${CONFIG_FILE_PATH}/config.js
fi

# disable broswer cache
sed -i "s/config.js?v=[0-9]*/config.js?v=$(date +'%s')/g" /srv/http/index.html
# for macOS
# sed -i "" "s/config.js?v=[0-9]*/config.js?v=$(date +'%s')/g" /srv/http/index.html

# exec CMD
exec "$@"
```

例如

```
# 传入
CONFIG_VARS=ABC,XYZ
ABC=helloabc
XYZ=HELLOXYZ

# 转换为
window._env = {
  ABC: "helloabc",
  XYZ: "HELLOXYZ",
}
```

2. 编写 Dockerfile，使用 goStatic 当做静态文件服务器（比 nginx 轻量）。build 完成只有几 MB.

```
FROM node:11 AS builder
COPY . /app
WORKDIR /app
RUN yarn install
RUN yarn run build

FROM wlchn/gostatic:latest
ENV CONFIG_FILE_PATH /srv/http
COPY --from=builder /app/build /srv/http
COPY ./env.sh /env.sh
# Ensure convert envs to window._env
ENTRYPOINT ["sh", "/env.sh"]
# start server. listen on 8043(in container) by default.
CMD ["/goStatic"]
```

3. 在项目中引用 config.js

```
<script type="text/javascript" src="/config.js?v="></script>
```

4. 使用环境变量，通过`window._env`获取

```
let _env = process.env;
// check if there is env exits in local, otherwise using window._env
if (_env.ENV_ABC) {
  _env = window._env;
}
// so you can use env like _env.ENV_ABC
```

5. 转换示例

```
# pass envs like:
CONFIG_VARS=ABC,XYZ
ABC=helloabc
XYZ=HELLOXYZ

# you coud get config.js
window._env = {
  ABC: "helloabc",
  XYZ: "HELLOXYZ",
}

# in your app, you can use like
console.log(_env.ABC)
console.log(_env.XYZ)
```

## GitHub

- [https://github.com/wlchn/docker-spa-env](https://github.com/wlchn/docker-spa-env)

## reference

- [https://github.com/SocialEngine/docker-nginx-spa](https://github.com/SocialEngine/docker-nginx-spa)
- [https://medium.freecodecamp.org/how-to-implement-runtime-environment-variables-with-create-react-app-docker-and-nginx-7f9d42a91d70](https://medium.freecodecamp.org/how-to-implement-runtime-environment-variables-with-create-react-app-docker-and-nginx-7f9d42a91d70)
