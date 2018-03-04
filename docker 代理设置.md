# 为什么需要设置代理
在运行docker run 时，首先会检查当前环境是否有对应的镜像，若没有将去`docker hub`上下载。由于国内长城，基本上不可能直接访问到国外的`docker hub`。
如下运行`hello-world`报错：
```
[root@localhost ~]# docker run hello-world
Unable to find image 'hello-world:latest' locally
docker: Error response from daemon: Get https://registry-1.docker.io/v2/: dial tcp: lookup registry-1.docker.io on [::1]:53: read udp [::1]:39795->[::1]:53: read: connection refused.
See 'docker run --help'.
```
需为docker设置海外代理，访问`docker hub`。

# 为docker 设置代理步骤

1. 创建docker http-proxy.conf文件
```
mkdir -p /etc/systemd/system/docker.service.d
touch /etc/systemd/system/docker.service.d/http-proxy.conf
```

2. 在http-proxy.conf代理配置文件中添加代理设置
```
[Service]
Environment="HTTP_PROXY=http://x.x.x.x:x/"

[Service]
Environment="HTTPS_PROXY=https://x.x.x.x:x/"
```

3. 重新加载docker配置
```
systemctl daemon-reload
```

4. 重启docker
```
service docker restart
```

# 验证代理设置
```
[root@localhost home]# docker run hello-world

Hello from Docker!

This message shows that your installation appears to be working correctly.
To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide
```
