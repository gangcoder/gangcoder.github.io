<!-- ---
title: nsq apps
date: 2018-07-31 16:32:13
category: language, go, nsq
--- -->

nsq apps

- nsqd nsqd 程序
- nsqlookupd nsqlookupd 程序
- to_nsq 一个生产者示例，从终端获取内容写入topic
- nsq_tail 一个消费者实例，从nsqd 接收消息写入终端
- nsqadmin
- nsq_stat
- nsq_to_file
- nsq_to_http
- nsq_to_nsq


## to_nsq

将终端输入信息写入nsq

```go
//初始化 producers
producers := make(map[string]*nsq.Producer)
for _, addr := range destNsqdTCPAddrs {
    producer, err := nsq.NewProducer(addr, cfg)
    producers[addr] = producer
}

//初始化读取 buffer
r := bufio.NewReader(os.Stdin)
delim := (*delimiter)[0]
go func() {
    for {
        err = readAndPublish(r, delim, producers)
    }
}()

//读取终端内容并且推送到 producer
func readAndPublish(r *bufio.Reader, delim byte, producers map[string]*nsq.Producer) error {
    //读取
    line, readErr := r.ReadBytes(delim)

    //推送
    for _, producer := range producers {
        err := producer.Publish(*topic, line)
    }

    return readErr
}
```


## nsq_tail

打印 topic 的消息内容

```
consumers := []*nsq.Consumer{}
for i := 0; i < len(topics); i += 1 {
    //创建topic consumer
    consumer, err := nsq.NewConsumer(topics[i], *channel, cfg)
    
    //添加处理handler
    consumer.AddHandler(&TailHandler{topicName: topics[i], totalMessages: *totalMessages})

    //连接nsqd
    err = consumer.ConnectToNSQDs(nsqdTCPAddrs)
    err = consumer.ConnectToNSQLookupds(lookupdHTTPAddrs)
    
    consumers = append(consumers, consumer)
}

//消息处理struct
type TailHandler struct {
	topicName     string
	totalMessages int
	messagesShown int
}

//消息处理
func (th *TailHandler) HandleMessage(m *nsq.Message) error {
	th.messagesShown++
    _, err := os.Stdout.WriteString(th.topicName)
	_, err = os.Stdout.WriteString(" | ")

    //打印msg
	_, err := os.Stdout.Write(m.Body)

	_, err = os.Stdout.WriteString("\n")
}
```


## nsq_stat

统计 topic / channel 信息

statLoop

```go
//获取集群信息
ci := clusterinfo.New(nil, http_api.NewClient(nil, connectTimeout, requestTimeout))

//channel 结构
var o *clusterinfo.ChannelStats

for i := 0; !countNum.isSet || countNum.value >= i; i++ {
    //producer 结构
    var producers clusterinfo.Producers

    //获取topic 的producer
    if len(lookupdHTTPAddrs) != 0 {
        producers, err = ci.GetLookupdTopicProducers(topic, lookupdHTTPAddrs)
    } else {
        producers, err = ci.GetNSQDTopicProducers(topic, nsqdHTTPAddrs)
    }

    //获取topic 的channel 信息
    _, channelStats, err := ci.GetNSQDStats(producers, topic, channel)

    c, ok := channelStats[channel]

    //每25 行打印一次表头，方便对照统计信息
    if i%25 == 0 {
        fmt.Printf("%s+%s+%s\n",
            "------rate------",
            "----------------depth----------------",
            "--------------metadata---------------")
        fmt.Printf("%7s %7s | %7s %7s %7s %5s %5s | %7s %7s %12s %7s\n",
            "ingress", "egress",
            "total", "mem", "disk", "inflt",
            "def", "req", "t-o", "msgs", "clients")
    }

    // 打印统计信息
    fmt.Printf("%7d %7d | %7d %7d %7d %5d %5d | %7d %7d %12d %7d\n",
        int64(float64(c.MessageCount-o.MessageCount)/interval.Seconds()),
        int64(float64(c.MessageCount-o.MessageCount-(c.Depth-o.Depth))/interval.Seconds()),
        c.Depth,
        c.MemoryDepth,
        c.BackendDepth,
        c.InFlightCount,
        c.DeferredCount,
        c.RequeueCount,
        c.TimeoutCount,
        c.MessageCount,
        c.ClientCount)

    o = c
    time.Sleep(interval)
}
```


## nsq_to_file

nsq 消息写入文件


- `nsq_to_file.go` 主函数
- `topic_discoverer.go` 获取最新topic 列表逻辑
- `consumer_file_logger.go` 创建消费者和处理函数
- `file_logger.go` 消息写入文件逻辑

`nsq_to_file.go` 主函数

```go
//获取最新topics 列表
discoverer := newTopicDiscoverer(cfg, hupChan, termChan, *httpConnectTimeout, *httpRequestTimeout)
//更新当前topics 列表
discoverer.updateTopics(topics, *topicPattern)
//周期性更新topics 列表
discoverer.poller(lookupdHTTPAddrs, len(topics) == 0, *topicPattern)
```


`topic_discoverer.go` 获取最新topic 列表逻辑

- `updateTopics` 更新当前topics map 的处理consumer
- `poller` 定时从集群中获取最新topics 列表，并且更新topics map

```go
type TopicDiscoverer struct {
	ci       *clusterinfo.ClusterInfo
	topics   map[string]*ConsumerFileLogger
	cfg      *nsq.Config
}

//更新当前topics map
func (t *TopicDiscoverer) updateTopics(topics []string, pattern string) {
	for _, topic := range topics {
		//初始化topic 处理consumer
		cfl, err := newConsumerFileLogger(topic, t.cfg)
		t.topics[topic] = cfl

        //开启goroutine 处理
		t.wg.Add(1)
		go func(cfl *ConsumerFileLogger) {
            //处理各类消息
			cfl.F.router(cfl.C)
			t.wg.Done()
		}(cfl)
	}
}

//定时从集群中获取最新topics 列表
func (t *TopicDiscoverer) poller(addrs []string, sync bool, pattern string) {
	var ticker <-chan time.Time
	ticker = time.Tick(*topicPollRate)
    
	for {
		select {
		case <-ticker:
			//获取集群最新topics
            newTopics, err := t.ci.GetLookupdTopics(addrs)
            
            //更新topics 处理consumer
			t.updateTopics(newTopics, pattern)
		}
	}
	t.wg.Wait()
}
```


`consumer_file_logger.go` 创建消费者和处理函数

```go
type ConsumerFileLogger struct {
	F *FileLogger
	C *nsq.Consumer
}

//创建消费者和处理函数
func newConsumerFileLogger(topic string, cfg *nsq.Config) (*ConsumerFileLogger, error) {
	//创建处理函数struct
    f, err := NewFileLogger(*gzipEnabled, *gzipLevel, *filenameFormat, topic)
	
    //创建消费者
	c, err := nsq.NewConsumer(topic, *channel, cfg)	
	c.AddHandler(f)

	err = c.ConnectToNSQDs(nsqdTCPAddrs)
	err = c.ConnectToNSQLookupds(lookupdHTTPAddrs)

    return &ConsumerFileLogger{
		C: c,
		F: f,
	}, nil
}
```


`file_logger.go` 消息写入文件逻辑

```go
type FileLogger struct {
	out              *os.File
	writer           io.Writer
	//文件轮转
	lastFilename string
}

//消息写入文件
func (f *FileLogger) router(r *nsq.Consumer) {
	pos := 0
	output := make([]*nsq.Message, *maxInFlight)
	sync := false
	ticker := time.NewTicker(time.Duration(30) * time.Second)
	
	for {
		select {
        case <-ticker.C:
            //周期性地轮转文件
			if f.needsFileRotate() {
                f.updateFile()
			}
			sync = true
		case m := <-f.logChan: //HandleMessage 将获取的消息写入channel
            //消息写入文件
			_, err := f.writer.Write(m.Body)
			_, err = f.writer.Write([]byte("\n"))
			output[pos] = m
			pos++
			if pos == cap(output) {
				sync = true
			}
		}

		if closing || sync || r.IsStarved() {
            //文件缓存刷入文件
            err := f.Sync()
        
            //消息返回 finish
            for pos > 0 {
                pos--
                m := output[pos]
                m.Finish()
                output[pos] = nil
            }
                
			sync = false
		}
	}
}
```

## nsq_to_http

读取nsq msg 并且发送到指定http 接口

```go
var publisher Publisher
var addresses app.StringArray
var selectedMode int

//创建consumer
consumer, err := nsq.NewConsumer(*topic, *channel, cfg)

//创建handler struct
hostPool := hostpool.New(addresses)
handler := &PublishHandler{
    Publisher:        publisher,
    addresses:        addresses,
    mode:             selectedMode,
    hostPool:         hostPool,
    perAddressStatus: perAddressStatus,
    timermetrics:     timer_metrics.NewTimerMetrics(*statusEvery, "[aggregate]:"),
}

//consumer 添加handler
consumer.AddConcurrentHandlers(handler, *numPublishers)

//连接nsq
err = consumer.ConnectToNSQDs(nsqdTCPAddrs)
err = consumer.ConnectToNSQLookupds(lookupdHTTPAddrs)

//handler struct
type PublishHandler struct {
	Publisher //内嵌信息发送逻辑
	hostPool  hostpool.HostPool
}

func (ph *PublishHandler) HandleMessage(m *nsq.Message) error {
    for _, addr := range ph.addresses {
        st := time.Now()
        err := ph.Publish(addr, m.Body)
        if err != nil {
            return err
        }
        ph.perAddressStatus[addr].Status(st)
    }
	return nil
}

//消息发送逻辑
type PostPublisher struct{}
func (p *PostPublisher) Publish(addr string, msg []byte) error {
	buf := bytes.NewBuffer(msg)
    
    //http 发送消息
    resp, err := HTTPPost(addr, buf)
	resp.Body.Close()
	return nil
}
```

## nsq_to_nsq

将一处nsql 的msg 写入另一个nsq

```go
//1. 创建目的producer 
producers := make(map[string]*nsq.Producer)
for _, addr := range destNsqdTCPAddrs {
    producer, err := nsq.NewProducer(addr, pCfg)
    producers[addr] = producer
}

//2. 创建msg consumer
var consumerList []*nsq.Consumer

//消息发送实例
publisher := &PublishHandler{
    addresses:        destNsqdTCPAddrs,
    producers:        producers,
}

//每个consumer 添加handler
for _, topic := range topics {
    consumer, err := nsq.NewConsumer(topic, *channel, cCfg)
    consumerList = append(consumerList, consumer)
    
    publishTopic := topic
    if *destTopic != "" {
        publishTopic = *destTopic
    }

    //msg handler
    topicHandler := &TopicHandler{
        publishHandler:   publisher,
        destinationTopic: publishTopic,
    }
    consumer.AddConcurrentHandlers(topicHandler, len(destNsqdTCPAddrs))
}

//消息处理完成后响应处理结果
for i := 0; i < len(destNsqdTCPAddrs); i++ {
    go publisher.responder()
}

//consumer 连接nsq
for _, consumer := range consumerList {
    err := consumer.ConnectToNSQDs(nsqdTCPAddrs)
}
for _, consumer := range consumerList {
    err := consumer.ConnectToNSQLookupds(lookupdHTTPAddrs)
}

//处理消息
func (ph *PublishHandler) HandleMessage(m *nsq.Message, destinationTopic string) error {
	var err error
	msgBody := m.Body

    //发送消息
    counter := atomic.AddUint64(&ph.counter, 1)
    idx := counter % uint64(len(ph.addresses))
    addr := ph.addresses[idx]
    p := ph.producers[addr]
    err = p.PublishAsync(destinationTopic, msgBody, ph.respChan, m, startTime, addr)

	m.DisableAutoResponse()
	return nil
}

//消息处理完成后响应处理成功消息
func (ph *PublishHandler) responder() {	
    msg.Finish()
}
```

## 参考资料

- [nsq apps](https://nsq.io/components/utilities.html)
