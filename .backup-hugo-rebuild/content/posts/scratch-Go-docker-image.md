---
title: "使用scratch构建最小化Go程序的docker-image"
date: 2019-11-22T00:00:00+08:00
---

由于 Golang 编译之后的文件是二进制，而 scratch 是 docker 最基础的空 image，所以可以使用 scratch 来构建 Go 程序的 docker image，使得最终构建的 image 最小化.

构建 image 过程分为两步：

- 1. 在 Go 基础 image 中 build.
- 2. 将 build 好的二进制文件拷贝到 scratch image 中。

### 无需 cgo 的程序

对于无需 cgo 交叉编译的程序，使用 scratch 来作为最终运行的基础 image 非常合适。

首先，选择合适版本的 golang 基础 image 来 build，这里没有必要选择更小的 golang alpine，build 过程中 pull 一般会有缓存所以 pull 速度差别不大，此外 alpine 中没有 git 和 ssl，我们在构建 image 过程中都有可能用到，况且 alpine 也不会影响最终 image 大小。

```Dockerfile
FROM golang:1.13 AS builder
```

禁掉 cgo 交叉编译，我们服务器一般为 linux amd64，build 二进制文件。

```Dockerfile
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-w -s" -o /bin/appmain main.go
```

对于绝大多数 go 程序而言，是无需 root 来运行，根据 docker best practice，使用 non-root 来运行程序能够带来更好的安全性，所以我们使用 non-root 用户来运行，创建一个 appuser，之后再拷贝到 scratch 运行 image 中。（scratch 是空 image，所以在 builder 中创建 user，再拷贝。）

```Dockerfile
# 创建appuser
RUN groupadd -r appuser && useradd --no-log-init -r -g appuser appuser
...
# 拷贝appuser到scratch
COPY --from=builder /etc/passwd /etc/passwd
...
# 选择appuser为默认程序运行用户
USER appuser
```

多数程序可能会用到 ssl，我们将 builder 中的 crt 拷贝一下即可。（如果 builder 是 alpine，不能拷贝，需要在 alpine 中 apk 先预装一下。）

```Dockerfile
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
```

完整版 Dockerfile

```Dockerfile
FROM golang:1.13 AS builder
COPY . /app
WORKDIR /app
RUN groupadd -r appuser && useradd --no-log-init -r -g appuser appuser
RUN go mod download
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-w -s" -o /bin/appmain main.go

FROM scratch
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /bin/appmain /bin/appmain
USER appuser
CMD [ "/bin/appmain" ]
```

### 需要 cgo 的程序

有些 Go 程序是需要 cgo 交叉编译的，例如 ethereum. 对于需要 cgo 的程序，相对于 scratch，更推荐使用 alpine 来作为基础 image，原因是 alpine 中带有 libc，并且体积也才 2MB 多。而 scratch 中没有，当然也可以在 builder 中 ldd 依赖并拷贝到 scratch 中。只是用 alpine 会更方便一些。

在 alpine 中只要软链接一下就可以使用。

```Dockerfile
RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2
```

此外，创建 non-root 用户的步骤也没有必要在 builder 中进行了，可以直接在 alpine 中创建。

```Dockerfile
RUN addgroup -S appuser && adduser -S -G appuser appuser
```

完整版 Dockerfile

```Dockerfile
FROM golang:1.13 AS builder
COPY . /app
WORKDIR /app
RUN go mod download
RUN CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -ldflags="-w -s" -o /bin/appmain main.go

FROM alpine:3.10
RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /bin/appmain /bin/appmain
RUN addgroup -S appuser && adduser -S -G appuser appuser
USER appuser
CMD [ "/bin/appmain" ]
```
