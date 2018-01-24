# swarm 代码流程分析
## swarm node 创建流程
```
docerkclient->swarm init
	dockerd/daemon/cluster/cluster.go: New
		startNewNode
			swarmagent.NewNode
				runManager -> manager.New 创建node manager,如果非mangaer -> 不创建manager结构即为worker
```
NOTE: 

1. swarm node节点毋容置疑都会dockerd创建的。swarm 第一个运行的结构为`agent/node.go`，在Node对象方法：start中，启动runManager和runAgent函数，并监控Node节点角色变化(manger -> worker 或 woker -> manager);
2. 一个NODE即有可能是manager也有可能是worker;(agent->run函数)
3. manager负责管理，worker负责运行task

<font color=red>weijinhua</font>


## docker swarm 异常分析
```
Error response from daemon: rpc error: code = 4 desc = context deadline exceeded
```
1. stub存根时间过长：此stub只能被使用X时长，此后将不能再进行请求，应该被释放。 -> 此tcp连接时间过长，强制断开

   >  根据官网对swarm说明：当swarm集群leave的manger节点必须小于：(N/2-1)，若当前集群有3个manager节点，将2个节点降级为workder后，操作docker node rm 等管理集群操作，报错：`Error response from daemon: rpc error: code = 4 desc = context deadline exceeded`，所以理论上这个错误是比较正常的，符合swarm设计。

2. ​

## 集群状态
```
const (
        // LocalNodeStateInactive INACTIVE
        LocalNodeStateInactive LocalNodeState = "inactive"
        // LocalNodeStatePending PENDING
        LocalNodeStatePending LocalNodeState = "pending"
        // LocalNodeStateActive ACTIVE
        LocalNodeStateActive LocalNodeState = "active"
        // LocalNodeStateError ERROR
        LocalNodeStateError LocalNodeState = "error"
)
```
1. 集群中没有任何NODE节点：INACTIVE
2. 集群cluster.cancelDelay 失败：ERROR
3. 集群中有NODE节点、但是集群的ready==false：pending
4. 集群中有NODE节点、但是集群的ready==true：active



## swarm service创建流程



## swarm 连接时安全证书
docker swarm manager NODE: `${docker_root}/swarm/certificates`


## swarm 持久化存储
在manager主机的host端存储：/var/lib/docker/swarm



## docker swarm 网址
[官方网址]((https://docs.docker.com/engine/swarm/admin_guide/#forcibly-remove-a-node)


## docker swarm session和grpc关系
docker session 初始化：
```
//agent.go -> run -> new
	var (
		backoff    time.Duration
		session    = newSession(ctx, a, backoff) // start the initial session
		registered = session.registered
		ready      = a.ready // first session ready
		sessionq   chan sessionOperation
	)
```

docker swarm session 数据结构定义：
```
// session encapsulates one round of registration with the manager. session
// starts the registration and heartbeat control cycle. Any failure will result
// in a complete shutdown of the session and it must be reestablished.
//
// All communication with the master is done through session.  Changes that
// flow into the agent, such as task assignment, are called back into the
// agent through errs, messages and tasks.
type session struct {
	conn *grpc.ClientConn //保护grpc.ClientConn连接
	addr string //session对端地址

	agent     *Agent  //session所属Agent
	sessionID string  //session ID 唯一
	session   api.Dispatcher_SessionClient
	errs      chan error
	messages  chan *api.SessionMessage
	tasks     chan *api.TasksMessage

	registered chan struct{} // closed registration
	closed     chan struct{}
	closeOnce  sync.Once
}
```

从上面结构看出，在docker swarm中，session属于grpc连接的上层结构。
```
//newSession函数调用：
func (s *session) run(ctx context.Context, delay time.Duration) {
	time.Sleep(delay) // delay before registering.

	if err := s.start(ctx); err != nil { //启动session监控
		select {
		case s.errs <- err:
		case <-s.closed:
		case <-ctx.Done():
		}
		return
	}

	ctx = log.WithLogger(ctx, log.G(ctx).WithField("session.id", s.sessionID))

	go runctx(ctx, s.closed, s.errs, s.heartbeat) //使用 grpc 连接检测心跳
	go runctx(ctx, s.closed, s.errs, s.watch)
	go runctx(ctx, s.closed, s.errs, s.listen)

	close(s.registered)
}
```

## docker swarm 状态分析
```
docker info
root@host-5 ~]# docker info
...
Swarm: pending
 NodeID: 4lhcdq2tt4ic4umbjzo1k9240
 Error: rpc error: code = 2 desc = raft: no elected cluster leader
 Is Manager: true
```

`docker info`分析：
```
//daemon/cluster/cluster.go
func (c *Cluster) Info() types.Info {
	info := types.Info{
		NodeAddr: c.GetAdvertiseAddress(),
	}
	c.RLock()
	defer c.RUnlock()

	if c.node == nil {
		info.LocalNodeState = types.LocalNodeStateInactive
		if c.cancelDelay != nil {
			info.LocalNodeState = types.LocalNodeStateError
		}
	} else {
		info.LocalNodeState = types.LocalNodeStatePending
		if c.ready == true {
			info.LocalNodeState = types.LocalNodeStateActive
		}
	}
```


## grpc 错误码
// OK is returned on success.
	OK Code = 0

	// Canceled indicates the operation was cancelled (typically by the caller).
	Canceled Code = 1
	
	// Unknown error.  An example of where this error may be returned is
	// if a Status value received from another address space belongs to
	// an error-space that is not known in this address space.  Also
	// errors raised by APIs that do not return enough error information
	// may be converted to this error.
	Unknown Code = 2
	
	// InvalidArgument indicates client specified an invalid argument.
	// Note that this differs from FailedPrecondition. It indicates arguments
	// that are problematic regardless of the state of the system
	// (e.g., a malformed file name).
	InvalidArgument Code = 3
	
	// DeadlineExceeded means operation expired before completion.
	// For operations that change the state of the system, this error may be
	// returned even if the operation has completed successfully. For
	// example, a successful response from a server could have been delayed
	// long enough for the deadline to expire.
	DeadlineExceeded Code = 4
	
	// NotFound means some requested entity (e.g., file or directory) was
	// not found.
	NotFound Code = 5
	
	// AlreadyExists means an attempt to create an entity failed because one
	// already exists.
	AlreadyExists Code = 6
	
	// PermissionDenied indicates the caller does not have permission to
	// execute the specified operation. It must not be used for rejections
	// caused by exhausting some resource (use ResourceExhausted
	// instead for those errors).  It must not be
	// used if the caller cannot be identified (use Unauthenticated
	// instead for those errors).
	PermissionDenied Code = 7
	
	// Unauthenticated indicates the request does not have valid
	// authentication credentials for the operation.
	Unauthenticated Code = 16
	
	// ResourceExhausted indicates some resource has been exhausted, perhaps
	// a per-user quota, or perhaps the entire file system is out of space.
	ResourceExhausted Code = 8
	
	// FailedPrecondition indicates operation was rejected because the
	// system is not in a state required for the operation's execution.
	// For example, directory to be deleted may be non-empty, an rmdir
	// operation is applied to a non-directory, etc.
	//
	// A litmus test that may help a service implementor in deciding
	// between FailedPrecondition, Aborted, and Unavailable:
	//  (a) Use Unavailable if the client can retry just the failing call.
	//  (b) Use Aborted if the client should retry at a higher-level
	//      (e.g., restarting a read-modify-write sequence).
	//  (c) Use FailedPrecondition if the client should not retry until
	//      the system state has been explicitly fixed.  E.g., if an "rmdir"
	//      fails because the directory is non-empty, FailedPrecondition
	//      should be returned since the client should not retry unless
	//      they have first fixed up the directory by deleting files from it.
	//  (d) Use FailedPrecondition if the client performs conditional
	//      REST Get/Update/Delete on a resource and the resource on the
	//      server does not match the condition. E.g., conflicting
	//      read-modify-write on the same resource.
	FailedPrecondition Code = 9
	
	// Aborted indicates the operation was aborted, typically due to a
	// concurrency issue like sequencer check failures, transaction aborts,
	// etc.
	//
	// See litmus test above for deciding between FailedPrecondition,
	// Aborted, and Unavailable.
	Aborted Code = 10
	
	// OutOfRange means operation was attempted past the valid range.
	// E.g., seeking or reading past end of file.
	//
	// Unlike InvalidArgument, this error indicates a problem that may
	// be fixed if the system state changes. For example, a 32-bit file
	// system will generate InvalidArgument if asked to read at an
	// offset that is not in the range [0,2^32-1], but it will generate
	// OutOfRange if asked to read from an offset past the current
	// file size.
	//
	// There is a fair bit of overlap between FailedPrecondition and
	// OutOfRange.  We recommend using OutOfRange (the more specific
	// error) when it applies so that callers who are iterating through
	// a space can easily look for an OutOfRange error to detect when
	// they are done.
	OutOfRange Code = 11
	
	// Unimplemented indicates operation is not implemented or not
	// supported/enabled in this service.
	Unimplemented Code = 12
	
	// Internal errors.  Means some invariants expected by underlying
	// system has been broken.  If you see one of these errors,
	// something is very broken.
	Internal Code = 13
	
	// Unavailable indicates the service is currently unavailable.
	// This is a most likely a transient condition and may be corrected
	// by retrying with a backoff.
	//
	// See litmus test above for deciding between FailedPrecondition,
	// Aborted, and Unavailable.
	Unavailable Code = 14
	
	// DataLoss indicates unrecoverable data loss or corruption.
	DataLoss Code = 15
