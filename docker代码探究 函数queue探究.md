
>  今天探究 DOCKER 的源码，发现了一段有意思代码，探究了一会，将结果记录下来

# 代码目的
docker daemon 接受来自containerd 的grpc消息，并针对依次处理。这段代码出自依次处理过程；

**代码：**
```
//libcontainer/container_linux.go
type queue struct {
	sync.Mutex
	fns map[string]chan struct{}
}

func (q *queue) append(id string, f func()) {
	q.Lock()
	defer q.Unlock()

	if q.fns == nil {
		q.fns = make(map[string]chan struct{})
	}

	done := make(chan struct{})

	fn, ok := q.fns[id]
	q.fns[id] = done
	go func() {
		if ok {
			<-fn //等待对端关闭channel
		}
		f()
		close(done)//关闭channel后，在另外一端的读操作，不在阻塞
	}()
}
```
上述queue的append的代码设计目的：
1. 使用q的sync.Mutex属性，实现互斥，避免竞争
2. 在处理func()函数时，启用了协程加快func() 处理速度，避免func() 业务复杂时，阻塞于此处
3. 在go中，协程的执行顺序没有先后顺序，所以添加了channel来保证同 id 的协程按照**先进先执行**的顺序执行。

