# go event
## 实现原理
1. 使用一个队列保存events事件，先进入到队列的事件先得到处理
2. 开启一个协程，循环检测队列中是否有事件
3. 队列事件的写入必须在另外的协程中，所以必须使用锁保护队列events数据
4. 设计 sink 装载events,为保证设计的兼容性，sink设计为interface
5. sink 配合channel可以实现多协程间事件通知

## 代码分析
**核心代码文件：**
go-events/event.go
go-events/queue.go

```
// Event marks items that can be sent as events.
type Event interface{} //事件：声明为interface{}可兼容所有数据类型

// 槽的兼容性考虑
type Sink interface {
	// Write an event to the Sink. If no error is returned, the caller will
	// assume that all events have been committed to the sink. If an error is
	// received, the caller may retry sending the event.
	Write(event Event) error

	// Close the sink, possibly waiting for pending events to flush.
	Close() error
}
```
Event 为事件原型， 实现了Sink的方法都可以作为sink

Queue类型声明
```
type Queue struct {
	dst    Sink //处理事件或转载事件的槽
	events *list.List //转载事件，
	cond   *sync.Cond //
	mu     sync.Mutex //List的数据涉及到多协程改动，使用“锁”保护数据完整性
	closed bool
}
```
新建事件监控时，会启动一个协程循环检测List（实现为队列）中的事件
```
func (eq *Queue) run() {
	for {
		event := eq.next() //获取LIST中的事件，若LIST没有事件将阻塞

		if event == nil {
			return // nil block means event queue is closed.
		}

		if err := eq.dst.Write(event); err != nil { //调用sink的write方法，处理event
			logrus.WithFields(logrus.Fields{
				"event": event,
				"sink":  eq.dst,
			}).WithError(err).Debug("eventqueue: dropped event")
		}
	}
}
```
Quene 的write方法，由其他协程调用，向List中写入event数据；
最后达到的效果：有一个协程从始至终检测events事件，当另外的协程调用Queue的write方法写入事件时，检测到有event事件来，使用sink（具体上次实现）write方法处理事件；

# docker watch机制
docker 实现了watch机制，等待事件或数据
## watch使用
```
//监控queue事件队列上的EventUpdataTask事件
Watch(queue, EventUpdateTask{})
```
