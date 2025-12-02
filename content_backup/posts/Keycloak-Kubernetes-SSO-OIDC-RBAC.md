---
title: "利用Keycloak实现Kubernetes单点登录与权限验证（SSO,OIDC,RBAC）"
date: 2020-02-28T00:00:00+08:00
---

本文将详细介绍如何利用 Keycloak 配置 Kubernetes 登录验证，以及 RBAC 管理。
本文全部为测试环境工具。
流程示意
![image.webp](/images/keycloak1.webp)

## 配置 Keycloak

##### 安装配置

生产环境不建议使用 docker 直接部署 Keycloak。如果使用 nginx 需要将`PROXY_ADDRESS_FORWARDING`配置成`true`.

```shell
sudo docker run -e KEYCLOAK_USER=wanglei -e KEYCLOAK_PASSWORD=your_password -e PROXY_ADDRESS_FORWARDING=true -p 8081:8080 -d jboss/keycloak
```

##### 配置 nginx

将 server_name 配置成你的域名,代理 Keycloak 端口，注意上面`PROXY_ADDRESS_FORWARDING=true`

```nginx
server {
    listen 80;
    server_name key.wanglei.me;

    location / {
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   Host      $http_host;
        proxy_pass         http://localhost:8081;
    }
}
```

##### 在 Clients 中创建 kubernetes

设置`Client ID`为`kubernetes`，`Client Protocol`选择`openid-connect`即 OIDC 协议。
![image.webp](/images/keycloak2.webp)

##### 设置 Client

将`Access Type`设置为`confidential`, `Valid Redirect URIs`根据需要设置，本文后面用到 kubelogin,需要设置为`http://localhost:8000`
![image.webp](/images/keycloak3.webp)

##### 记录 Client Secret

在 Credentials 中找到 Secret，后面需要用到。
![image.webp](/images/keycloak4.webp)

##### 新建 Mapper

选择`Mapper Type`为`User Attribute`,选择`Claim JSON Type`为`String`,下面开关一律选`on`(可根据需求更改)。此处`Token Claim Name`字段设置为`groups`,后面配置 kube-apiserver 会用到。
![image.webp](/images/keycloak5.webp)

##### 为 User 添加 Attribute

添加 key `groups`, value `admin`,然后点击 add，再 save。
![image.webp](/images/keycloak6.webp)

## 配置 kubernetes

此处使用 kind(kubernetes in docker) https://github.com/kubernetes-sigs/kind#installation-and-usage 为例。
安装 kind（mac 为例）

```shell
brew install kind
```

##### 配置 kube-apiserver

config.yaml

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    metadata:
      name: config
    apiServer:
      extraArgs:
        oidc-issuer-url: "https://key.wanglei.me/auth/realms/master"
        oidc-client-id: "kubernetes"
        oidc-username-claim: "preferred_username"
        oidc-username-prefix: "-"
        oidc-groups-claim: "groups"
```

创建本地 kubernetes 集群。

```shell
kind create cluster --config=./config.yaml
```

查看 apiserver log

```
kubectl logs -n kube-system --tail=10 -f kube-apiserver-kind-control-plane
```

##### 配置 RBAC

RBAC 介绍[https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

创建一个 ClusterRoleBinding, 文中使用系统自带的 cluster-admin 为例，也可以创建符合要求的 role。
创建 cluster-role-binding.yaml

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: keycloak-admin-group
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: Group
    name: admin
    apiGroup: rbac.authorization.k8s.io
```

kubectl apply

```shell
kubectl apply -f cluster-role-binding.yaml
```

##### 配置 kubeconfig

安装 kubelogin，[https://github.com/int128/kubelogin](https://github.com/int128/kubelogin)

```
brew install int128/kubelogin/kubelogin
```

kubelogin 示意图
![image.webp](/images/keycloak7.webp)

编辑 ~/.kube/config,里面为 kind 生成的配置，添加一个名为 oidc 的 user，并配置到 kind-kind 的 context

```yaml
apiVersion: v1
clusters:
  - cluster:
      insecure-skip-tls-verify: true
      server: https://localhost:6443
    name: docker-for-desktop-cluster
  - cluster:
      certificate-authority-data: ...
      server: https://127.0.0.1:32773
    name: kind-kind
contexts:
  - context:
      cluster: docker-for-desktop-cluster
      user: docker-for-desktop
    name: docker-for-desktop
  - context:
      cluster: kind-kind
      user: oidc
    name: kind-kind
current-context: kind-kind
kind: Config
preferences: {}
users:
  - name: docker-for-desktop
    user:
      client-certificate-data: ...
      client-key-data: ...
  - name: kind-kind
    user:
      client-certificate-data: ...
      client-key-data: ...
  - name: oidc
    user:
      auth-provider:
        config:
          client-id: kubernetes
          client-secret: your-secret
          idp-issuer-url: https://key.wanglei.me/auth/realms/master
        name: oidc
```

运行 kubelogin,出现如下提示即登录成功。
![image.webp](/images/keycloak8.webp)

试一下
![image.webp](/images/keycloak9.webp)

此时再看~/.kube/config,可以看到多了两行`id-token`和`refresh-token`

```yaml
- name: oidc
  user:
    auth-provider:
      config:
        client-id: kubernetes
        client-secret: 50cc10ed-1859-4042-9305-c913470ac6e2
        id-token: ...
        idp-issuer-url: https://key.wanglei.me/auth/realms/master
        refresh-token: ...
      name: oidc
```

其中`id-token`可以解析为如下 json
![image.webp](/images/keycloak10.webp)

同理，如需要多种权限验证皆可以通过 RBAC 和 Keycloak 很好的实现。
参考：
[https://kubernetes.io/docs/reference/access-authn-authz/authentication/](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
[https://octopus.com/blog/kubernetes-oauth](https://octopus.com/blog/kubernetes-oauth)
[https://kind.sigs.k8s.io/docs/user/quick-start/#enable-feature-gates-in-your-cluster](https://kind.sigs.k8s.io/docs/user/quick-start/#enable-feature-gates-in-your-cluster)
[https://github.com/int128/kubelogin/blob/master/docs/setup.md#keycloak](https://github.com/int128/kubelogin/blob/master/docs/setup.md#keycloak)
[https://blog.hdls.me/15647317755993.html](https://blog.hdls.me/15647317755993.html)
