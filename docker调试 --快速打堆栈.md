docker deamon 在启动时，注册监听`sigusr1`信号的打印堆栈回调函数，dockerd接受到`sigusr1`信号后，打印当前堆栈。
docker deamon 快速打印堆栈方法：
```kill -SIGUSR1 <dockerd-pid>```

docker12.6 版本，堆栈调用链被打印在log日志中，docker-ce-17版本后，调用堆栈打印到单独文件，放置于`/var/run/docker`目录下。
