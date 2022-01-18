# Lab1: MapReduce

- [Coordinator](#Coordinator)
    - [核心结构体](#核心结构体)
    - [RPC handler](#RPC-handler)
    - [核心逻辑](#核心逻辑)
- [Worker](#Worker)



在本次lab中我们的任务是实现一个分布式的MapReduce，它由两个程序组成，Coordinator和Worker。只有一个Coordinator，一个或多个Worker并行执行。

每个Worker将通过RPC与Coordinator通信以请求一个Map或Reduce任务，之后从一个或多个文件中读取任务的输入，执行任务，并将任务的输出写入一个或多个文件。

Coordinator应注意到Worker是否在合理的时间内（10s）完成了任务，如果没有则将相同的任务交给另一个Worker。

## Coordinator

写这个lab的时候刚学go语言不久，觉得channel这个东西很帅，就使用了很多channel实现了一个lock-free版本的Coordinator，实践了一下csp。

### 核心结构体

Coordinator维护每一个Map和Reduce任务的状态，这样就不用维护每一个worker的状态，这也利于worker的弹性伸缩。

xxxidCh用于在获取任务编号并发放给worker，xxxDoneCh和xxxUndoneCh用于获取完成或未完成的任务编号修改任务状态。

```go
type Coordinator struct {
	files          []string
	nMap           int
	nReduce        int
	mapidCh        chan int
	reduceidCh     chan int
	mapStatus      []Task
	reduceStatus   []Task

	heartbeatCh    chan heartbeatMsg
	reportCh       chan reportMsg
	stateCh        chan getStateMsg

	mapDoneCh      chan Execution
	reduceDoneCh   chan Execution
	mapUndoneCh    chan Execution
	reduceUndoneCh chan Execution
	
	mapComplete    bool
	reduceComplete bool
	mapRemain      int
	reduceRemain   int
}
```

每个任务的状态有3种，每个任务被初始化时都是UnStarted，被分配给Worker之后转换为Processing，收到Report完成转为Done，未完成转为UnStarted。

结构体Task用term和任务状态共同表示一个任务的信息，term代表该任务被分配给worker执行的次数。
```go
type TaskStatus int
const (
	UnStarted TaskStatus = iota
	Processing
	Done
)

type Task struct {
	term int
	TaskStatus
}
```

### RPC-handler

Coordinator接收到RPC之后，包装出一个xxxMsg结构，传入RPC对应的channel中。

Done在这里作用类似于一个回调。Coordinator在启动时会在后台启动一个goroutine，不断监控 heartbeatCh 和 reportCh 中的Msg并处理，处理完成后执行msg.Done <- struct{}{}。在RPC handler中只需要等待Done这个channel返回。

```go
type heartbeatMsg struct {
	response *HeartbeatResponse
	Done       chan struct{}
}

type reportMsg struct {
	request *ReportRequest
	Done      chan struct{}
}

func (c *Coordinator) Heartbeat(request *HeartbeatRequest, response *HeartbeatResponse) error {
	log.Println("[Coordinator] receive a request from worker")
	msg := heartbeatMsg{
		response: response,
		Done:       make(chan struct{}),
	}
	c.heartbeatCh <- msg
	<-msg.Done

	log.Printf("[Coordinator] run heartbeat [%s] for worker", response)
	return nil
}

func (c *Coordinator) Report(request *ReportRequest, response *ReportResponse) error {
	log.Printf("[Coordinator] receive worker's report [%s]", request)
	msg := reportMsg{
		request: request,
		Done:      make(chan struct{}),
	}

	c.reportCh <- msg
	<-msg.Done

	log.Println("[Coordinator] finish dealing with the report from worker")
	return nil
}
```

`handleHeartbeatMsg`函数中处理心跳，根据当前Map和Reduce任务的状态给Worker分配一个任务、让worker等待或是告知所有任务已经完成。任务的id从mapidCh或reduceidCh两个channel中读出，在response中还要加上任务的term，每次分配该任务前需要对term自增以在不同的执行者之间区分。

那么任务的id是什么时候写入channel中的呢？Coordinator在初始化时先将所有Map任务的id写入mapidCh，在所有Map任务都完成后将所有Reduce任务的id写入reduceidCh。

需要注意一点，每个任务在分配之后10s内如果没有收到Report，则应该默认任务失败。这需要另起一个goroutine来判断，直接sleep 10s之后将id写入Undone channel即可，让run函数去判断。

在`handleReportMsg`函数中处理worker的返回任务结果，根据结构类型将任务的Execution写入对应的Done/Undone channel。我将任务的term和id包装成一个Execution结构表示任务的一次执行，使得某次任务失败是超时还是worker返回失败这两种情况可以被区分。


```go
type Execution struct {
	term int
	id   int
}
```

### 核心逻辑

run函数是Coordinator的核心，它作为一个后台运行的goroutine在不断的循环中监听各个channel并执行对应的操作。由于所有的数据都在这一个goroutine中修改，避免了data-race。

Coordinator真正处理worker上报的任务的完成情况是由run函数在select中同时监听这4个channel，再根据任务id来执行对应逻辑。因此`handleReportMsg`函数可以另起一个goroutine来执行，这4个channel的容量也只需设置为1。

从4个channel读出任务id后要注意，只有在对应的状态、Execution中term和本地任务的term一致时才能执行逻辑。

例如某个MapFailed消息在10s之后到达，这可能是因为网络拥塞或是worker执行任务太慢，这个map任务已经被重新分配给了另一个worker，此时状态是仍是Processing。但这时term不一致应该放弃处理这个MapFailed消息。

```go
func (c *Coordinator) run() error {
	for {
		select {
		case hbMsg := <-c.heartbeatCh:
			c.handleHeartbeatMsg(hbMsg)

		case rpMsg := <-c.reportCh:
			go c.handleReportMsg(rpMsg)

		case e := <-c.mapDoneCh:
			if c.mapStatus[e.id].TaskStatus == Processing && c.mapStatus[e.id].term == e.term {
				···
			}

		case e := <-c.reduceDoneCh:
			if c.reduceStatus[e.id].TaskStatus == Processing && c.reduceStatus[e.id].term == e.term {
				···
			}

		case e := <-c.mapUndoneCh:
			if c.mapStatus[e.id].TaskStatus == Processing && c.mapStatus[e.id].term == e.term {
				···
			}

		case e := <-c.reduceUndoneCh:
			if c.reduceStatus[e.id].TaskStatus == Processing && c.reduceStatus[e.id].term == e.term {
				···
			}

		case stMsg := <-c.stateCh:
			stMsg.state <- c.reduceComplete

		}
	}
}
```

## Worker

worker的实现比较简单，只需要循环向coordinator请求任务执行。
```go
func Worker(mapf func(string, string) []KeyValue,
	reducef func(string, []string) string) {

	// Your worker implementation here.
	for {
		response := doHeartbeat()
		log.Printf("[Worker] receive coordinator's heartbeat [%s]", response)
		switch response.Type {
		case Map:
			doMapTask(mapf, response.Id, response.Term, response.NReduce, response.Name)
		case Reduce:
			doReduceTask(reducef, response.Id,response.Term, response.NMap)
		case Wait:
			time.Sleep(1 * time.Second)
		case Completed:
			return
		default:
			panic(fmt.Sprintf("[Worker] unexpected jobType %v", response.Type))
		}

	}
}
```

执行Map任务时，只需将mapf函数产生的中间文件kv pair按照ihash(kv.Key)%nReduce的余数写入不同的文件等待Reduce即可。写入的文件要先调用ioutil.TempFile("", "temp")生成再调用os.Rename()改为mr-i-j。

执行Reduce任务时，先建立一个kv数组，再将所有中间文件中的kv pair append到数组中再排序，将相同key对应的所有value append到一个string数组中，喂给reducef函数执行。看起来非常暴力，在工业界应该不可行，但通过本次lab的测试足够了。
