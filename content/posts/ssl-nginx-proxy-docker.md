---
title: "SSL双向验证-Nginx反向代理docker-sock"
date: 2018-07-31T00:00:00+08:00
---

## 不做验证的代理（不安全、可内网使用）

nginx 的用户 www-data 在组 www-data 中，而 docker.sock 的用户在 docker 组中，所以需要将 www-data 添加到 docker 组中

```
sudo usermod -a -G www-data,docker www-data
```

查看用户以及组

```
compgen -u
compgen -g
```

nginx 的配置

```
server {
    listen 23750;

    location / {
        proxy_pass http://unix:/var/run/docker.sock:/;
    }
}
```

重启 nginx 即可。

## 做验证的代理（外网暴露安全）

按照https://docs.docker.com/engine/security/https/#create-a-ca-server-and-client-keys-with-openssl生成ca证书以及RSA证书

```
ca-key.pem  ca.pem  cert.pem  key.pem  server-cert.pem  server-key.pem
```

配置 nginx 证书，ssl_verify_client 与 ssl_client_certificate 实现验证客户端，如客户端没有自签私钥则不能访问。

```
server {
    listen 23760;

    ssl on;
    ssl_certificate        /home/deployer/docker-ca/server-cert.pem;
    ssl_certificate_key    /home/deployer/docker-ca/server-key.pem;
    ssl_client_certificate /home/deployer/docker-ca/ca.pem;
    ssl_verify_client      on;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 SSLv3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location = / {
        return 502 '';
    }

    location / {
        proxy_pass http://unix:/var/run/docker.sock:/;
    }
}
```

重启 nginx。
在客户端中加入证书验证。

```ruby
Docker.options = {
    client_cert: File.join(cert_path, 'cert.pem'),
    client_key: File.join(cert_path, 'key.pem'),
    ssl_ca_file: File.join(cert_path, 'ca.pem'),
    scheme: 'https',
}
```

即可做验证访问。

## Attach docker websocket

```
server {
    listen 443;
    server_name docker-attach.wanglei.io;

    ssl on;
    ssl_certificate     /home/deployer/docker-ca/wanglei.io.pem;
    ssl_certificate_key /home/deployer/docker-ca/wanglei.io.key;

    location / {
        return 400;
    }

    location ~ws/?$ {
        if ( $query_string !~ "secret=xxxxxxxx" ) { return 400; }

        proxy_pass http://unix:/var/run/docker.sock;
        proxy_http_version 1.1;
        proxy_set_header Upgrade "websocket";
        proxy_set_header Connection "upgrade";
    }
}
```
