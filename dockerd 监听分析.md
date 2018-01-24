### 监听端口使用

如果docker 命令若没有指定 -H 使用 特定docker host server, 那么默认将命令发送到 unix:///var/run/docker.sock 


### 监听端口创建


### 监听端口释放
通过`kill 9 ${dockerd-id}`命令，发送SIGTERM 给dockerd进程。dockerd在初始化时注册了SIGTERM信号处理函数；
注册信号处理代码：
```
//cmd/dockerd/daemon.go
//start() 函数
    cli.api = api
    signal.Trap(func() {
        cli.stop()
        <-stopc // wait for daemonCli.start() to return
    })
```
信号处理函数代码：
```
//api/server/server.go
// Close closes servers and thus stop receiving requests
func (s *Server) Close() {
    for _, srv := range s.servers {
        if err := srv.Close(); err != nil {
            logrus.Error(err)
        }
    }
}
```
在srv.Close()函数中调用`s.l.Close()`释放监听端口；

端口释放、服务停止后;`cmd/dockerd/daemon.go -> start()函数`
```
...
    serveAPIWait := make(chan error)
    go api.Wait(serveAPIWait) 
    // after the daemon is done setting up we can notify systemd api
    notifySystem()

    // Daemon is fully initialized and handling API traffic
    // Wait for serve API to complete
    errAPI := <-serveAPIWait //等待server退出
    c.Cleanup()
    shutdownDaemon(d, 15)
    containerdRemote.Cleanup()
    if errAPI != nil {
        return fmt.Errorf("Shutting down due to ServeAPI error: %v", errAPI)
    }
```
serverAPIWait收到信号，继续向下执行，清理dockerd进程环境、释放资源（比如container容器等）

### 监听端口故障
**故障描述**：
    dockerd重启时，dockerd启动失败，异常现象
`can't create unix socket /var/run/docker.sock: is a directory`
**故障原因分析**：
    dockerd 和 docker client通过默认通过/var/run/docker.sock这个socket文件进行通信。dockerd启动和docker命令时若不使用`-H`特殊设置监听使用端口，默认使用/var/run/docker.sock文件进行监听。
    启动时在start()函数调用Listern.init函数初始化socket通道，初始化service后等待。当dockerd接受到终止信号后，signal.go -> trap()函数处理信号，调用server.Close()终止服务、释放socket函数后，start()函数继续执行清理dockerd资源、释放正在运行的容器(向容器发送SIGTERM或SIGKILL信号)。
    所以socket文件在释放容器前先释放，在某些竞争场景下，容器会创建/var/run/docker.sock目录。导致下次启动dockerd时，发现/var/run/docker.sock为目录而失败。

**解决方法**
​	暂时没有找到具体哪些场景下的竞争会导致，创建/var/run/docker.sock目录。
**规避方法**：

​	每次启动dockerd时，创建socket文件前，检测/var/run/docker.sock，若发现其存在且为目录，删除目录。

