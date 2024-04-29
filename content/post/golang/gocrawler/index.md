---
title: golang分布式爬虫设计
description: 
slug: 
date: 2024-02-21
categories:
    - golang
tags:
    - 爬虫框架
weight: 1       
---

> 关联项目: [https://github.com/superjcd/gocrawler](https://github.com/superjcd/gocrawler)

## 缘起
我自己是一个python的重度使用者， python中有大量的爬虫相关的包， 也不乏类似scrapy这种非常有知名度的包， 结合redis的scrapy-redis似乎也能在一定程度上解决分布式爬虫的需求。
但是这些依然还是不够， 不管是使用用基于线程的爬虫框架（scrapy）， 还是基于协程的爬虫(ruia)， 它们还是不够高效， 当然一部分的原因是在于python语言本身。python的优势是灵活性， 以及丰富的生态， 但是如果要追求极致的运行效率， 那么go其实是一个更好的选择
## 核心特性
个人向来推崇极简主义， 爬虫框架gocrawler的开发也不例外， 对我个人而言， 这个框架最最重要的其实就是一个任务调度器(scheduler)：
```go
package scheduler

import "github.com/superjcd/gocrawler/request"

type Scheduler interface {
	Pull() *request.Request
	Push(typ int, reqs ...*request.Request)
	Schedule()
}
```
scheduler需要实现的方法无非就是：

- Pull, 获取一个request对象
- Push, 提交若干request对象
- Schedule， 调度器的入口， 启动并持续运行调度器

同样的, 以数据的存储模块store为例， store只需要实现一个Save方法就可以了
```go
package store

import (
	"github.com/superjcd/gocrawler/parser"
)

type Storage interface {
	Save(ctx context.Context, datas ...parser.ParseItem) error
}
```
其它组件， 再比如Fetcher(请求数据模块)也不过只是一个仅仅有Fetch方法的interface。
```go
type Fetcher interface {
	Fetch(ctx context.Context, req *request.Request) (*http.Response, error)
}
```

当然gocrawler会为用户提供一些开箱可用的调度器以及存储组件等等， 但所有这些构成爬虫的组成部分， 不过就是一个个的接口， 用户可以使用gocrawler提供的模块， 也可以使用自己的， 所以gocrawler天然的秉承了极简主义和面向接口编程的哲学。
> 极简和可拓展性才是gocrawler最最核心的feature


## 构建请求对象
Request请求对象是一个结构体， 定义如下：
```go
type Request struct
	URL    string
	Method string
	Retry  int
	Data   map[string]string
}

type RequestDataCtxKey struct{}

```
Request对象字段说明：
- URL定义了请求地址
- Method定义了具体的请求方法， 比如GET, POST等
- Retry定义了重试的次数， 如果worker定义了全局最大retry数， 爬虫引擎会在关键节点(比如决定是否重试)的时候查看这个值
- Data是用户自定义的一些元数据， 这些元数据会通过context.WithValue的方式注入到爬虫上下文中， 注入的键为上面的RequestDataCtxKey类型

为了实现爬虫的去重， 一种实现的方式是：不希望相同的Request在一定的时间内被访问两次， 首先需要去hash这个请求

```golang
func (r *Request) Hash(hashFields ...string) string {
	components := make([][]byte, 2+len(hashFields))
	components[0] = []byte(r.URL)
	components[1] = []byte(r.Method)

	for i, field := range hashFields {
		if fieldValue, ok := r.Data[field]; ok {
			components[i+2] = []byte(fieldValue)
		} else {
			panic(fmt.Errorf("field not in request.Data"))
		}
	}

	hash := md5.Sum(bytes.Join(components, []byte(":")))
	return string(hash[:])
}
```
默认的Hash实现比较简单， 这里使用md5对指定的字段进行摘要；  
当然， 需要注意的一点是：什么时候设置已请求已访问很关键， 假设在获得请求的时候就进行设置， 那么如果请求失败没有入库， 依然会被认为是访问过的，所以在gocrawler中， 设置去重项的时间节点是在数据导入之后


## 实现调度器
调度器是gocrawler的核心，因为需要实现分布式调度， 所以类似scrapy的那种基于线程和内存队列的方式是不合适的, 所以需要选择一个高吞吐的存储介质； 消息队列是很容易想到的选项， kafaka, rabbit mq, rocket mq这些其实都可以， 但是作为爬虫worker的消息队列， 不需要太强的一致性保证，所以gocrawler默认提供了一个基于nsq的实现：
```go
package nsq

import (
	"encoding/json"
	"log"

	"github.com/nsqio/go-nsq"
	"github.com/superjcd/gocrawler/request"
	"github.com/superjcd/gocrawler/scheduler"
)

const (
	DIRECT_PUSH = iota
	NSQ_PUSH
)

type nsqScheduler struct {
	workerCh       chan *request.Request
	nsqLookupdAddr string
	topicName      string
	channelName    string
	nsqConsumer    *nsq.Consumer
	nsqProducer    *nsq.Producer
}

type nsqMessageHandler struct {
	s *nsqScheduler
}

func (h *nsqMessageHandler) HandleMessage(m *nsq.Message) error {
	var err error
	if len(m.Body) == 0 {
		return nil
	}

	processMessage := func(mb []byte) error {
		var req request.Request
		if err = json.Unmarshal(mb, &req); err != nil {
			return err

		}
		h.s.Push(DIRECT_PUSH, &req)
		return nil
	}

	err = processMessage(m.Body)

	return err

}

var _ scheduler.Scheduler = (*nsqScheduler)(nil)

func NewNsqScheduler(topicName, channelName, nsqAddr, nsqLookupdAddr string) *nsqScheduler {

	nsqConfig := nsq.NewConfig()

	nsqConsumer, err := nsq.NewConsumer(topicName, channelName, nsqConfig)

	if err != nil {
		log.Fatal(err)
	}

	nsqProducer, err := nsq.NewProducer(nsqAddr, nsqConfig)

	if err != nil {
		log.Fatal(err)
	}

	workerCh := make(chan *request.Request)
    
	return &nsqScheduler{workerCh: workerCh,
		topicName:      topicName,
		channelName:    channelName,
		nsqLookupdAddr: nsqLookupdAddr,
		nsqConsumer:    nsqConsumer,
		nsqProducer:    nsqProducer,
	}
}

func (s *nsqScheduler) Pull() *request.Request {
	req := <-s.workerCh
	return req
}

func (s *nsqScheduler) Push(typ int, reqs ...*request.Request) {
	switch typ {
	case DIRECT_PUSH:
		for _, req := range reqs {
			s.workerCh <- req
		}
	case NSQ_PUSH:
		for _, req := range reqs {
			msg, err := json.Marshal(req)
			if err != nil {
				log.Printf("push msg to nsq failed")
			}
			s.nsqProducer.Publish(s.topicName, msg)

		}
	default:
		log.Fatal("wrong push type")
	}

}

func (s *nsqScheduler) Schedule() {
	s.nsqConsumer.AddHandler(&nsqMessageHandler{s: s})
	if err := s.nsqConsumer.ConnectToNSQLookupd(s.nsqLookupdAddr); err != nil {
		log.Fatal(err)
	}

}

func (s *nsqScheduler) Stop() {
	s.nsqConsumer.Stop()
}

```
因为调度器需要同时实现Push和Pull的操作， 所以nsqScheduler同时具有一个nsq.Consumer(实现pull)和nsq.Producer(实现push)的指针.
Pull的逻辑很简单， 直接从workerChan获取一个Request对象就好
Push则会根据类型选择DIRECT_PUSH直接往workerChan放一个Request， 如果是NSQ_PUSH则会是将Request通过nsqProducer发布到nsq里面
最后Schedule的实现很简单， 就是连接nsqlookupd, 开起监听， 当监听一个到message时就会触发HandleMessage回调函数， 而HandleMessage的主要逻辑就把获取的Request对象push到workerCh中
> 由于nsq本身并[不能保证顺序](https://nsq.io/overview/features_and_guarantees.html), 所以这个Scheduler不是严格意义上的FIFO, 当然对于爬虫来说绝对的先入先出不是特别的重要

## 实现存储器
通常我们会使用mysql之类的关系性数据库作为存储器， 无它， 比较便利。但是考虑到爬虫任务的特殊性， 我个人其实会更倾向于使用文档性数据库， 因为页面的变动是常有的事， 而且，爬取得的数据不见得都是规则的，所以有严格的shema的关系型数据库显然不是最适合的选择
> 虽然mysql现在也支持存储json， 但是查询性能无法与原生的文档数据库相媲美。当然使用gocrawler, 因为只是依赖接口， 所以实现一个支持Save的mysql存储器也是可以的。

gocrawler自带基于mongo的存储器：

- 无缓存mongo存储器
- 带缓存的mongo存储器

无缓存实现：
```go
type mongoStorage struct {
	Cli *qmgo.QmgoClient
}

var _ store.Storage = (*mongoStorage)(nil)

func NewMongoStorage(uri, database, collection string) *mongoStorage {
	ctx := context.Background()
	cli, err := qmgo.Open(ctx, &qmgo.Config{Uri: uri,
		Database: database,
		Coll:     collection}) // counter
	if err != nil {
		panic(err)
	}

	return &mongoStorage{Cli: cli}
}

func (s *mongoStorage) Save(items ...parser.ParseItem) error {
	var result *qmgo.InsertOneResult
	var err error
	for _, item := range items {
		result, err = s.Cli.Collection.InsertOne(context.Background(), item)
		if err == nil {
			log.Println("[store]insert one ok")
		}
	}
	if err != nil {
		return err
	}
	_ = result
	return nil
}

```
无缓存的mongoStorage， 会在每次调用Save方法的时候， 依次将数据插入到mongodb中。
很显然这种实现只适合对爬虫速度要求不高的场景， 更合理的方式是使用缓存， 下面是带缓存的存储器实现(部分代码):
```go
type bufferedMongoStorage struct {
	L            *sync.Mutex
	Cli          *qmgo.QmgoClient
	buf          []parser.ParseItem
	bufSize      int
	counter      counter.Counter
	taskKeyField string
}


// 略

func (s *bufferedMongoStorage) Save(items ...parser.ParseItem) error {
	s.L.Lock()
	defer s.L.Unlock()

	for {
		if len(items) > s.bufSize {
			return fmt.Errorf("number of items too large(larger than the max bufSize), either increase storage bufSize or decrease number of items")
		}

		if len(items) > (s.bufSize - len(s.buf)) {
			if err := s.flush(); err != nil {
				return err
			}

		} else {
			s.buf = append(s.buf, items...)
			break
		}
	}

	return nil
}

// 略
```
 bufferedMongoStorage会对将要导入的数据进行判断：

- 如果要导入的数据量大于最大缓存， 那么直接返会错误
- 如果要导入的数据量已经大于缓存的剩余空间， 那么直接促发flush， flush会将缓存buf中数据批量导入到mongodb， flush完成后buf会被清空
- 否则， 直接追加到缓存中并退出循环
> 之所以使用for循环的意义在于， 上诉的第二步只是把缓存中的数据写到mongo， items还没有被写入， 所以需要执行下次操作， 将它们加入到缓存中

由于flush的触发条件是缓存已满或者将满的情况， 那么如果没有新的数据导入到缓存中， 那么缓存中的数据就会一直驻留， 所以我们需要某种机制来解决数据驻留的情况：
```go

func NewBufferedMongoStorage(...) *bufferedMongoStorage {
    (略)
	ticker := time.NewTicker(autoFlushInterval)

	go func() {
		for t := range ticker.C {
			log.Printf("auto flush triggered at %v", t)
			store.flush()
		}

	}()

	return store

}
```
解决的方案也很简单， 利用定时器定时触发flush即可
## 实现分布式任务计数
以python的scrapy为例， 他会在将request加入队列的时候统计任务数量：
```python
    def enqueue_request(self, request: Request) -> bool:
        if not request.dont_filter and self.df.request_seen(request):
            self.df.log(request, self.spider)
            return False
        dqok = self._dqpush(request)
        assert self.stats is not None
        if dqok:
            self.stats.inc_value("scheduler/enqueued/disk", spider=self.spider)
        else:
            self._mqpush(request)
            self.stats.inc_value("scheduler/enqueued/memory", spider=self.spider)
        self.stats.inc_value("scheduler/enqueued", spider=self.spider)
        return True

```
然后stats是存储在内存中的一个字典， 这种实现方式也存在以下问题：
- 相比于请求数， 其实我更关心完成数量， 特别是当我需要重试请求， 或者有带缓存的存储器的时候；请求数量不是一个核心指标
- 更重要的一点是， 在分布式场景下 ，基于worker的内存计数显然是不行的
顺带提一嘴，scrapy的计数函数inc_value并没有锁， 当然原因在于它的schduler的入队操作并不是并发实现的
```python
    def inc_value(
        self, key: str, count: int = 1, start: int = 0, spider: Optional[Spider] = None
    ) -> None:
        d = self._stats
        d[key] = d.setdefault(key, start) + count

```
Ok, 言归正转，总而言之我需要一个可以在分布式场景下完成任务完成计数的功能， 一个很容易想到的解决方案就是redis的[INCR操作](), incr会以并发安全的方式为key加上1， 这样结合gocrawler的worker的AfterSave生命周期函数（添加一个调用redis的incr函数的hook）， 我就可以实现一个不错的分布式计数功能。  
但问题是， 如果我用的是带缓存的存储器， 然后实现的是批量存储呢？如果存储不会失败， 那么使用生命周期函数统计数量， 然后多次调用INCR似乎也不是不行， 但是不管是存储不会失败还是多次调用INCR都不是鲁棒性很高的操作，理想情况是在发生flush的情况下， 对flush的数量进行单次计数才是最完美的。但是redis除了incr是并发安全之外，常规的修改计数的方法一定会涉及get和put操作， 所以它不是并发安全的， 所以需要使用redis的事务：
> 这里我直接用了官方示例， 稍微调整一下来实现了这个功能

```go
...

func (c *redisTaskCounters) Incr(key string, increment int64) {
    // transaction
    key = c.prefix + key
    txf := func(tx *redis.Tx) error {
        // Get the current value or zero.
        n, err := tx.Get(key).Int64()
        if err != nil && err != redis.Nil {
            return err
        }

        // Actual operation (local in optimistic lock).
        atomic.AddInt64(&n, increment)

        // Operation is commited only if the watched keys remain unchanged.
        _, err = tx.Pipelined(func(pipe redis.Pipeliner) error {
            pipe.Set(key, n, c.TTL) // time
            return nil
        })
        return err
    }

    // Retry if the key has been changed.
    for i := 0; i < maxRetries; i++ {
        err := c.RCli.Watch(txf, key)
        if err == nil {
            // Success.
            return
        }
        if err == redis.TxFailedErr {
            // Optimistic lock lost. Retry.
            continue
        }
        // TODO: igonore any other error.
        return
    }

}

```
既然这个计数功能需要统计flush的数据量， 那么我们就把它写在flush函数里面
```go
func (s *bufferedMongoStorage) flush() error {
	if len(s.buf) == 0 {
		return nil
	}
	err := s.insertManyTOMongo(s.buf...)
	if err != nil {
		return err
	}

	if s.counter != nil {
		tc := s.collectTaskCounts(s.buf)
		s.count(tc)
	}
	// update buffer to an empty buffer
	s.buf = make([]parser.ParseItem, 0, s.bufSize)
	log.Printf("Flushed")
	return nil
}

func (s *bufferedMongoStorage) count(tc map[string]int64) {
	for k, v := range tc {
		s.counter.Incr(k, v)
	}
}
```
collectTaskCounts函数是考虑到缓存队列中可能同时存在多个任务的数据，所以考虑先统计了每个任务对应的数据量
## 生命周期函数
目前gocrawler的worker支持的生命周期函数有：

- BeforeRequest
- AfterRequest
- BeforeSave
- AfterSave

它们的应用场景是怎样的呢？这里以BeforeRequest为例， 
```go
type BeforeRequestHook func(context.Context, *request.Request) (Signal, error)

func (w *worker) BeforeRequest(ctx context.Context, req *request.Request) (Signal, error) {
	var sig Signal
	if w.BeforeRequestHook != nil {
		return w.BeforeRequestHook(ctx, req)
	}
	sig |= DummySignal
	return sig, nil
}
```
BeforeRequest会在worker调用Fetch之前被调用：
```go
...
w.BeforeRequest(context.Background(), req)
resp, err := w.Fetcher.Fetch(req)
...
```

因为BeforeRequestHook有指向Request的指针， 意味着我们可以在worker发起网络请求之前修改Request的参数， 比如Request的Url， 爬虫任务的目标url通常是规律的， 比如站点网址+ 某种ID或者页码数，而站点网址通常是固定的， 如此一来， 我们在提交任务的时候， 不用写入完整的url， 把补全Url的功能写成一个BeforeRequestHook提交给worker即可， 这样可以在很大程度减少任务队列的存储消耗

### 关于Signal
生命周期函数的第一个返回值是Signal, Siganl是一个8位的flag， 方便用户告诉worker的主流程， 接下来该如何操作
通过signal, 用户可以控制worker选择直接进行一下轮循环， 或者直接忽略错误， 再或者直接panic错误  
> 生命周期函数是用户外部注入到框架的，所以通过Signal作为桥梁， 让用户拥有控制worker运行流程的能力是很重要的


## 其他组件
其他诸如网页解析器， 代理获取， cookie获取， 请求头获取等组件的实现也并不困难, gocrawler同样也提供了相应的接口。
当然有一个组件其实非常重要，那就是进行网络请求的Fetcher组件
```go
type Fetcher interface {
	Fetch(req *request.Request) (*http.Response, error)
}

type fectcher struct {
	Cli          *http.Client
	CookieGetter cookie.CoookieGetter
	UaGetter     ua.UaGetter
}

func NewFectcher(timeOut time.Duration, proxyGetter proxy.ProxyGetter, cookieGetter cookie.CoookieGetter, uaGetter ua.UaGetter) *fectcher {
	tr := http.DefaultTransport.(*http.Transport)
	tr.Proxy = proxyGetter.Get
	tr.DisableKeepAlives = true
	client := &http.Client{Transport: tr, Timeout: timeOut}

	f := &fectcher{
		Cli:          client,
		CookieGetter: cookieGetter,
		UaGetter:     uaGetter,
	}

	return f
}

func (f *fectcher) Fetch(r *request.Request) (resp *http.Response, err error) {
	jar, err := f.CookieGetter.Get()
	if err != nil {
		return nil, err
	}
	f.Cli.Jar = jar
	req, err := http.NewRequest(r.Method, r.URL, nil)
	if err != nil {
		return nil, fmt.Errorf("get url failed: %w", err)
	}
	ua, err := f.UaGetter.Get()

	if err != nil {
		return nil, fmt.Errorf("get ua failed: %w", err)
	}
	req.Header.Set("User-Agent", ua)

	resp, err = f.Cli.Do(req)

	if err != nil {
		return nil, err
	}

	return
}
```
 gocrawler自带了一个基础的默认fetcher；可以看到NewFectcher的依赖基本上都是接口， 意味着用户可以自行替换所需要的依赖， 增加了框架的灵活性
默认fetcher会调用net/http中的请求函数来发起网络请求， 不过官方库的请求函数为了追求性能， 会复用连接， 但是很多爬虫场景下， 我们会希望我们每次的请求都是一个新的连接，这样就可以每次调用不同的代理，  所以在实例fetcher的时候， 我们把.DisableKeepAlives设置成了false
## worker完整工作流
有了前面的铺垫， 我们来看一下worker的Run入口函数的实现：
```go
func (w *worker) Run() {
	ctx, cancel := context.WithTimeout(
		context.Background(),
		w.MaxRunTime,
	)
	defer cancel()
	for i := 0; i < w.Workers; i++ {
		go singleRun(w)
	}

	<-ctx.Done()
}

```
Run函数其实就是并发开启多个singleRun来执行具体的爬虫流程， 同时通过带TimeOut的上下文， 我们会让worker在MaxRunTime之后，退出主程序；最后附上singleRun的实现， 整体的逻辑还是比较简单的
```go
func singleRun(w *worker) {
	for {
		w.Limiter.Wait(context.TODO())
		req := w.Scheduler.Pull()
		if req == nil {
			continue
		}
		var reqKey string
		if w.UseVisit {
			if w.AddtionalHashKeys == nil {
				reqKey = req.Hash()
			} else {
				reqKey = req.Hash(w.AddtionalHashKeys...)
			}

			if w.Vister.IsVisited(reqKey) {
				continue
			}
		}

		originReq := req

		// Fetch
		w.BeforeRequest(context.Background(), req)
		resp, err := w.Fetcher.Fetch(req)

		if err != nil {
			log.Printf("request failed: %v", err)
			if req.Retry < w.MaxRetries {
				originReq.Retry += 1
				w.Scheduler.Push(nsq.NSQ_PUSH, originReq)
			} else {
				log.Printf("too many fetch failures for request:%s, exceed max retries: %d", req.URL, w.MaxRetries)
			}
			continue
		}

		if resp.StatusCode != http.StatusOK {
			originReq.Retry += 1
			w.Scheduler.Push(nsq.NSQ_PUSH, originReq)
			continue
		}

		// Parse
		parseResult, err := w.Parser.Parse(resp)
		if err != nil {
			log.Printf("parse failed for request: %s, error: %v", req.URL, err)
			originReq.Retry += 1
			w.Scheduler.Push(nsq.NSQ_PUSH, originReq)
			continue
		}

		// New Requests
		if parseResult.Requests != nil && len(parseResult.Requests) > 0 {
			for _, req := range parseResult.Requests {
				w.Scheduler.Push(nsq.NSQ_PUSH, req)
			}
		}

		// Save
		if parseResult.Items != nil && len(parseResult.Items) > 0 {
			if w.SaveRequestData {
				for _, p_item := range parseResult.Items {
					for dk, dv := range req.Data {
						p_item[dk] = dv
					}
				}
			}

			w.BeforeSave(context.Background(), parseResult)

			if err := w.Store.Save(parseResult.Items...); err != nil {
				log.Printf("item saved failed err: %v;items: ", err)
				continue
			}
            w.AfterSave(context.Background(), parseResult)

		}
		if w.UseVisit {
			w.Vister.SetVisitted(reqKey, w.VisterTTL)
		}

	}
}
```
这里还需要追加提到的一点是， 如果用户决定使用过滤器（在一定时间周期内， 不想反复地抓取相同数据）， 那么可以通过下面的Option函数添加过滤器
```go
func WithVisiter(v visit.Visit, ttl time.Duration) Option {
	return func(opts *options) {
		opts.Vister = v
		opts.UseVisit = true
		opts.VisiterTTL = ttl

	}
}
```
> worker的所有外部依赖都是基于Option模式添加的
然后如果UseVisit是true的话， worker会在发起请求前先判断Request是不是已经Visit过了, 如果开启UseVisit其有访问过， 那么就会直接进入下轮循环
> gocralwer提供了一个基于redis的Vister的实现，默认实现并没有使用BloomFilter， 如果数据量非常大， 且不太在意部分Request会被误判为访问过(hash冲突)的话， 也可以自己实现一个基于BloomFilter(redis也是支持的)