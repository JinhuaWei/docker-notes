## 整体框架分析
dockerd 底层运行容器需要依赖多个二级制组件：docker daemon, containerd, container-shim, runC，
代码实现上，containerd包含了container-shim代码。同一份代码，通过Makefile编译控制，编译成两个二级制文件。
### 组件间通信
概括图
!["底层通信概括图"][01]

通信流程：
1. docker daemon 模块通过 grpc 和 containerd模块通信：dockerd 由`libcontainerd`负责和`containerd`模块进行交换， dockerd 和 containerd 通信socket文件：`docker-containerd.sock`
2. containerd 在dockerd 启动时被启动，启动时，启动grpc请求监听。containerd处理grpc请求，根据请求做相应动作；
3. 若是start或是exec 容器，containerd 拉起一个container-shim , 并通过exit 、control 文件（每个容器独有）通信；
4. container-shim别拉起后，start/exec/create拉起runC进程，通过exit、control文件和containerd通信，通过父子进程关系和SIGCHLD监控容器中进程状态；
5. 若是top等命令，containerd通过runC二级制组件直接和容器交换；
6. 在整个容器生命周期中，containerd通过 **epoll 监控容器文件，监控容器的OOM等事件**；

NOTE：
**containerd,container-shim 组件本质上runC 和dockerd 间的adapter中间件**，容器本身有runC单独完成 --- 使用runC可以单独完成一个容器部署。

## 组件通信详细分析
### DOCKER DEAMON ---> LIBCONTAINERD 组件分析

LIBCONTAINERD 部分主要作用：1，赋值和CONTAINERD进程通信 2，监控CONTAINERD进程状态

LIBCONTAINERD 随DOCKERD启动，启动过程中启动协程监控：1. 和CONTAINERD间的grpc链接情况 2. 监听由CONTAINERD发送过来的消息
**启动监控消息协程**
``` go
//libcontainerd/remote_linux.go
//starEventMonitor() ---> handleEventStream()

func (r *remote) handleEventStream(events containerd.API_EventsClient) {
	for {
		e, err := events.Recv()   // ---> 此为阻塞方法：等待CONTAINERD发送 gRPC消息
...
		if err := container.handleEvent(e); err != nil {
			logrus.Errorf("libcontainerd: error processing state change for %s: %v", e.Id, err)
		}
		...
}
```
上面主要代码主要功能就是接收gRPC返回的事件，并调用各自容器来的handleEvent对事件进行处理
```
//libcontainerd/container_linux.go
func (ctr *container) handleEvent(e *containerd.Event) error {
	ctr.client.lock(ctr.containerID)
	defer ctr.client.unlock(ctr.containerID)
	switch e.Type {
	case StateExit, StatePause, StateResume, StateOOM:
		st := StateInfo{
			CommonStateInfo: CommonStateInfo{
				State:    e.Type,
				ExitCode: e.Status,
			},
			OOMKilled: e.Type == StateExit && ctr.oom,
		}
		if e.Type == StateOOM {
			ctr.oom = true
		}
		if e.Type == StateExit && e.Pid != InitFriendlyName {
			st.ProcessID = e.Pid
			st.State = StateExitProcess
		}

		// Remove process from list if we have exited
		switch st.State {
		case StateExit:
			ctr.clean()
			ctr.client.deleteContainer(e.Id)
		case StateExitProcess:
			ctr.cleanProcess(st.ProcessID)
		}
		ctr.client.q.append(e.Id, func() {
			if err := ctr.client.backend.StateChanged(e.Id, st); err != nil {
				logrus.Errorf("libcontainerd: backend.StateChanged(): %v", err)
			}

			if e.Type == StatePause || e.Type == StateResume {
				ctr.pauseMonitor.handle(e.Type)
			}
			if e.Type == StateExit {
				if en := ctr.client.getExitNotifier(e.Id); en != nil {
					en.close()
				}
			}
		})

	default:
		logrus.Debugf("libcontainerd: event unhandled: %+v", e)
	}
	return nil
}
```
handleEvent 函数处理event事件，并将event放入client的队列，等待被调度；
`remote.q`的目的：保证来自于容一个id容器的grpc包，能安装先进先出的顺序处理。

**启动监控gRPC链接协程**
``` go
//libcontainerd/remote_linux.go
// New()//New creates a fresh instance of libcontainerd remote -> handleConnectionChange()
```
handleConnectionChange()主要功能：根据gRPC状态判断、确定CONTAINERD的状态，并做相应的处理

### CONTAINERD 组件分析

CONTTAINERD的主要功能：
1. 和DOCKERD 的LIBCONTAINERD通信 ---> gRPC
2. 和CONTAINER-SHIM 、runC进行通信 ---> 进程返回或epoll系统调用方式
3. 简单的调度和任务分发

CONTAINERD 关键组成部分
```
├── api         //gRPC 事件处理
├── containerd  //CONTAINERD 命令处理 ---> main 主函数
├── runtime     //以容器为单位，处理状态、事件
├── supervisor  //以整体为单位，处理、转发状态、事件
```

#### CONTAINERD 启动
CONTAINERD 是随DOCKERD启动而启动的后台进程。
CONTAINERD 启动过程中，主要工作：
1. **启动gRPC服务**
  gRPC启动后，主要监控事件：
```
//containerd/api/grpc/types/api.proto
service API {
	rpc GetServerVersion(GetServerVersionRequest) returns (GetServerVersionResponse) {}
	rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse) {}
	rpc UpdateContainer(UpdateContainerRequest) returns (UpdateContainerResponse) {}
	rpc Signal(SignalRequest) returns (SignalResponse) {}
	rpc UpdateProcess(UpdateProcessRequest) returns (UpdateProcessResponse) {}
	rpc AddProcess(AddProcessRequest) returns (AddProcessResponse) {}
	rpc CreateCheckpoint(CreateCheckpointRequest) returns (CreateCheckpointResponse) {}
	rpc DeleteCheckpoint(DeleteCheckpointRequest) returns (DeleteCheckpointResponse) {}
	rpc ListCheckpoint(ListCheckpointRequest) returns (ListCheckpointResponse) {}
	rpc State(StateRequest) returns (StateResponse) {}
	rpc Events(EventsRequest) returns (stream Event) {}
	rpc Stats(StatsRequest) returns (StatsResponse) {}
}
```
处理事件代码：`//containerd/api/grpc/server/server.go`

2. **启动supervisor.Supervisor 从整体上监控事件：grpc事件、容器OOM事件、容器进程退出事件**
  Supervisor的New函数：
```
//Superviso New函数
...
	monitor, err := NewMonitor()
...
	go s.exitHandler()      //启动协程监控全部的Exit事件，并分发
	go s.oomHandler()       //启动协程监控全部的OOM事件，并分发
	if err := s.restore(); err != nil { //重启以后的容器 --->CONTAINERD进程被杀死，DOCKERD进程重新拉起CONTAINERD场景下生效
		return nil, err
	}
...
```

exitHandle() 和comHandler()代码
```
//containerd/supervisor/supervisor.go
func (s *Supervisor) exitHandler() {
	for p := range s.monitor.Exits() {
		e := &ExitTask{
			Process: p,
		}
		e.WithContext(context.Background())
		s.SendTask(e)
	}
}

func (s *Supervisor) oomHandler() {
	for id := range s.monitor.OOMs() {
		e := &OOMTask{
			ID: id,
		}
		e.WithContext(context.Background())
		s.SendTask(e)
	}
}
```
这两个函数监控，Exit和OOM事件，并通过SendTask发送事件

Supervisor的Start函数干的主要事情：启动一个协程循环等待地处理tasks中的事件
```
func (s *Supervisor) Start() error {
...
	go func() {
		for i := range s.tasks {
			s.handleTask(i)
		}
	}()
	return nil
}
```

上述提到SendTask函数：发送事件给s.tasks
```
func (s *Supervisor) SendTask(evt Task) {
	select {
	case <-evt.Ctx().Done():
		evt.ErrorCh() <- evt.Ctx().Err()
		close(evt.ErrorCh())
	case s.tasks <- evt:
		TasksCounter.Inc(1)
	}
}
```
Sendtask在api的gRPC事件监控和OOM等事件监控, 改函数将上次事件源，发送给Supervisor协程，Supervisor处理事件: gRPC等事件 ---> sendTask ---> Supervisor handleTask

handleTask处理不同事件
```

func (s *Supervisor) handleTask(i Task) {
	var err error
	switch t := i.(type) {
	case *AddProcessTask:
		err = s.addProcess(t)
	case *CreateCheckpointTask:
		err = s.createCheckpoint(t)
	case *DeleteCheckpointTask:
		err = s.deleteCheckpoint(t)
	case *StartTask:
		err = s.start(t)
	case *DeleteTask:
		err = s.delete(t)
	case *ExitTask:
		err = s.exit(t)
	case *GetContainersTask:
		err = s.getContainers(t)
	case *SignalTask:
		err = s.signal(t)
	case *StatsTask:
		err = s.stats(t)
	case *UpdateTask:
		err = s.updateContainer(t)
	case *UpdateProcessTask:
		err = s.updateProcess(t)
	case *OOMTask:
		err = s.oom(t)
	default:
		err = ErrUnknownTask
	}
	if err != errDeferredResponse {
		i.ErrorCh() <- err
		close(i.ErrorCh())
	}
}
```

3. **启动多个Supervisor.work协程，并发处理容器创建任务**；
  在CONTAINER启动时，创建了十个work协程池
```
//containerd/main.go ---> damon()函数
	wg := &sync.WaitGroup{}
	for i := 0; i < 10; i++ {
		wg.Add(1)
		w := supervisor.NewWorker(sv, wg)
		go w.Start()
	}
```
containerd/supervisor/work.go ---> Start()函数主要功能：循环等待监控 w.s.startTasks CHANNEL ，创建相应的容器，并将创建结构以 Event方式通知 gRPC server。

#### 其他
CONTAINERD 的runtime部分，包括container ---> 运行时，containerd中代码的容器信息实体 ； Process --->运行时容器中的进程信息

### CONTAINERD-SHIM 分析
CONTAINERD-SHIM的代码和业务逻辑都很简单；其主要实现目的：
1. 通过runC命令可以启动、执行容器、进程；
2. 监控容器进程状态，当容器执行完成后，通过exit fifo文件报告容器进程结束状态；
3. 当此容器SHIM的第一个实例进程被杀死后，reaper掉所有其子进程；

一个容器可以由多个container-shim进程；container-shim进程由containerd进程拉起，并持续存在到容器实例进程退出为止；

#### 代码流程分析
CONTAINERD-SHIM的代码在containerd/container-shim/目录下；主要包含main.go和process.go；
主要启动业务逻辑：
```
main()
------>start()
------------>(p *process) create() //主要功能启动runC进程
------>start函数中，监控runC进程状态
```

### runC 简要分析
runC 是一套符合OCI标准的容器引擎；其是一个非常独立，真正创建容器的组件，runC可以独立于dockerd创建、操作容器，其功能和LXC等容器化工具类似；使用runC主要功能：创建、并启动标准化容器。
容器使用的主要技术(Linux)：namespace 资源隔离，cgroup 资源限制
关于容器的资源和好文：
1. OCI标准化 [runtime-spec](https://github.com/opencontainers/runtime-spec)
2. namespace技术说明 ---> [Namespace资源隔离](http://www.infoq.com/cn/articles/docker-kernel-knowledge-namespace-resource-isolation)
3. [容器标准化和 docker](http://cizixs.com/2017/11/05/oci-and-runc)
4. cgroup 实现原理与使用 --->[Linux Cgroups 详解](https://files.cnblogs.com/files/lisperl/cgroups%E4%BB%8B%E7%BB%8D.pdf)


runC源码分析：网上有较多的runC源码分析，所以不做过多介绍 ---> [docker RunC Create 源码简单分析](https://blog.csdn.net/Daniel_greenspan/article/details/78742039)


## dockerd 启动一个容器流程分析
dockerd 端函数调用链：

containerStart()  --->  (clnt *client) Create()  ---> (ctr *container) start ---> ctr.client.remote.apiClient.CreateContainer ---> 向containerd发送gRPC信号


containerd端的函数执行流程：

gRPC接收到来自于dockerd的请求 ---> (s *apiServer) CreateContainer --->s.sv.SendTask(e) ---> 向supervisor.Supervisor发送函数创建容器事件


supervisor.Supervisor 处理协程收到容器创建事件 ---> handleTask() ---> s.start(t) ---> s.startTasks <- task: 将创建容器事件从新包装后发送给work的处理事件协程


supervisor.work处理事件协程收到创建容器事件（work.go/Start(): 在此函数中操作创建、运行容器，此函数主要工作：
1. 创建容器：t.Container.Start:(c *container) Start ---> (c *container) createCmd：使用shim create 进程方式创建容器（异步方式） ---> 启动shim进程，使用runC create 方式创建容器（创建但不运行容器进程）（异步方式）--->  容器创建成功，此时容器状态为created,并卡在/proc/self/exe init进程中 ---> shim启动runC完后，循环监控容器状态，直至此容器进程实例结束
2. 调用w.s.monitor.MonitorOOM(t.Container)函数，使用epoll方式监控，容器的OOM流程
3. 调用w.s.monitorProcess(process)函数，使用epoll方式监控，容器进程实例退出事件
4. 调用process.Start启动容器进程实例：(p *process) Start() ---> 使用runC start <container-id>方式启动容器进程实例（此<container-id>为第一步骤同个container-id）
5. 调用w.s.notifySubscribers 将容器创建完成事件，通知给api的事件监控模块


containerd/api/grpc/server.go/Events函数 ---> for 循环监听到容器创建完成事件 ---> 封装gRPC的Event事件 --->通过types.API_EventsServer send()方法发送Event


dockerd的监控消息协程将接收并处理容器创建成功的事件，更新dockerd端信息。--->前面已经分析


## 多个组件间通信分析
### dockerd 与 containerd间通信
dockerd和containerd之间通过 gRPC 通信；
方法定义：./vendor/src/github.com/docker/containerd/api/grpc/types/api.proto

### containerd 获取容器实例进程退出、OOM状态
在 supervisor.Supervisor 启动时，会启动两个协程监控监控退出和OOM状态，见上文分析；
在supervisor.work启动容器时，将容器状态添加到监控中：
```
//(w *worker) Start()
w.s.monitor.MonitorOOM(t.Container)
w.s.monitorProcess(process)
```
```
//monitorProcess ---> Monitor()
func (m *Monitor) Monitor(p runtime.Process) error {
	m.m.Lock()
	defer m.m.Unlock()
	fd := p.ExitFD()
	event := syscall.EpollEvent{
		Fd:     int32(fd),
		Events: syscall.EPOLLHUP,
	}
	if err := archutils.EpollCtl(m.epollFd, syscall.EPOLL_CTL_ADD, fd, &event); err != nil {
		return err
	}
	EpollFdCounter.Inc(1)
	m.receivers[fd] = p
	return nil
}
```
使用epoll 系统调用，监控exit文件的EPOLLHUP事件：当open exit文件后，在close时刻会触发EPOLLHUP事件

exit文件在什么地方close？ ---> 在shim进程退出时，close 掉exit文件。
```
// containerd-shim/main.go /start()
	f, err := os.OpenFile("exit", syscall.O_WRONLY, 0)
	if err != nil {
		return err
	}
	defer f.Close()
```
shim 进程什么时候退出？
shim for 循环中代码：
```
//start函数
		case s := <-signals:
			switch s {
			case syscall.SIGCHLD://子进程退出信号
				exits, _ := osutils.Reap(false) //杀掉所欲子进程
				for _, e := range exits {
					// check to see if runtime is one of the processes that has exited
					if e.Pid == p.pid() {
						exitShim = true
						writeInt("exitStatus", e.Status)
					}
				}
			}
			// runtime has exited so the shim can also exit
			if exitShim {
				// kill all processes in the container incase it was not running in
				// its own PID namespace
				p.killAll()
				// wait for all the processes and IO to finish
				p.Wait()
				// delete the container from the runtime
				p.delete()
				// the close of the exit fifo will happen when the shim exits
				return nil
			}
```
shim 在其所有子进程（SHIM创建的进程根，其及其子进程创建的进程都为其子进程）都退出后，SHIM进程退出。 
**容器进程退出流程**:(假设为init进程退出)
容器的实例进程完全退出后，shim进程退出  ---> 引发exit 文件的EPOLLHUP事件 ---> supervisor中检测到process退出 --->` *Supervisor) exitHandler()`协程获取到退出事件 --->SendTask：ExitTask --->`(s *Supervisor) Start()`协程处理退出事件 ---> handleTask ---> s.exit(t) ---> s.delete(ne):ne为DeleteTask ---> s.deleteContainer(i.container)删除容器：在containerd内存结构数据 ---> s.notifySubscribers 通知api Event协程，向dockerd发送容器退出状态

退出容器退出过程中，会获取容器退出状态，并将退出码发送给dockerd;
`(s *Supervisor) exit` ---> proc.ExitStatus:获取退出码 ---> handleSigkilledShim 根据Shim的退出状态定义退出码

### containerd 和 shim 交互
containerd 获取 shim退出状态：上已分析
containerd通过control文件控制shim
containerd对代码：
```
func (p *process) Resize(w, h int) error {
	_, err := fmt.Fprintf(p.controlPipe, "%d %d %d\n", 1, w, h)
	return err
}
```

shim端代码：
```
//container-shim/main.go start()函数
	go func() {
		for {
			var m controlMessage
			if _, err := fmt.Fscanf(control, "%d %d %d\n", &m.Type, &m.Width, &m.Height); err != nil {
				continue
			}
			msgC <- m
		}
	}()
```

### containerd 和 runC 交互
containerd 通过runC命令，获取命令返回输出，获取runC状态：如`(c *container) Status()`函数

[01]: ./docker-通信.png "docker底层通信图"
