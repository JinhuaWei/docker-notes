# go event
## 实现原理
1. 使用一个队列保存events事件，先进入到队列的事件先得到处理
2. 开启一个协程，循环检测队列中是否有事件
3. 队列事件的写入必须在另外的协程中，所以必须使用锁保护队列events数据
4. 设计 sink 装载events,为保证设计的兼容性，sink设计为interface,sink可理解为：运输船，将event事件运输到相应的协程
5. sink 配合channel可以实现多协程间事件通知
6. golang channel 为一对一方式通知，设计broadcast广播通知所有相关sinks

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
Watch函数源码（state/watch.go）
```
func Watch(queue *watch.Queue, specifiers ...Event) (eventq chan events.Event, cancel func()) {
	if len(specifiers) == 0 {
		return queue.Watch() // queue 默认watch函数
	}
	return queue.CallbackWatch(events.MatcherFunc(func(event events.Event) bool {
		for _, s := range specifiers {
			if s.matches(event) {
				return true
			}
		}
		return false
	}))
}
```
向CallbackWatch传入了一个匿名事件匹配函数
CallbackWatch函数(state/watch/watch.go)
```
func (q *Queue) CallbackWatch(matcher events.Matcher) (eventq chan events.Event, cancel func()) {
	ch := events.NewChannel(0)              //event使用channel 封装，为了通知其他协程
	sink := events.Sink(events.NewQueue(ch))// 封装为sink

	if matcher != nil {
		sink = events.NewFilter(sink, matcher) //使用Filter实现的sink：实现filter方法，检测事件筛选
	}

	q.broadcast.Add(sink)  //将sink添加到 broadcast
	return ch.C, func() {
		q.broadcast.Remove(sink)
		ch.Close()
		sink.Close()
	}
}
```

`events.NewChannel` `events.NewFilter` `events.Broadcast` 都是sink的实现特殊功能的封装类，具体实现参考源码：go-events目录下 filter.go broadcast.go channel.go

`events.Broadcast`特殊说明下：**golang 的channel相当于一个管道，只能实现协程间一对一的通信；为了实现一对多的情况，go-events实现了broadcast机制:`list中有事件后，通知broadcast，再由broadcast遍历其中所有的sink，调用sink的write处理方法，通知相应的协程（一般情况是channel信号，协程根据channel执行相应events动作）`**

通知Watch向注册监控事件，并获取返回的channel，并在当前协程阻塞、监控channel,当事件发生时，当前协程可通过channel获取事件类型作出相应的动作。

