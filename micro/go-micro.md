# go-micro

#### 熔断器
熔断关闭状态（Closed）：服务没有故障时，熔断器所处的状态，对调用方的调用不做任何限制（Interval close状态下周期性的晴空掉用计数）。    
熔断开启状态（Open）：在固定时间窗口内（Hystrix默认是10秒），接口调用出错比率达到一个阈值（Hystrix默认为50%），会进入熔断开启状态。进入熔断状态后，后续对该服务接口的调用不再经过网络，直接执行本地的fallback方法。   
半熔断状态（Half-Open）：在进入熔断开启状态一段时间之后（Hystrix默认是5秒，通过参数timeout配置），熔断器会进入半熔断状态。所谓半熔断就是尝试恢复服务调用，允许有限的流量调用该服务（MaxRequests限制，最大的掉用服务次数），并监控调用成功率。如果成功率达到预期，则说明服务已恢复，进入熔断关闭状态；如果成功率仍旧很低，则重新进入熔断关闭状态。    
ReadyToTrip: 自定义函数判断是否进入Half-Open
```
ReadyToTrip: func(counts gobreaker.Counts) bool {
  fmt.Println(counts.TotalSuccesses, counts.TotalFailures)
  return counts.TotalFailures >= 1
},
```
OnStateChange: 状态改变时掉用此方法，一般是触发告警使用
```
OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
  fmt.Println("name", name)
  fmt.Println("from", from)
  fmt.Println("to", to)
},
```

#### 限流器（令牌桶）
```
type Bucket struct {
	clock Clock

	startTime time.Time // 开始时间

	capacity int64 // 最大容量

	quantum int64  // 每个周期增加的令牌数量
  
  fillInterval time.Duration // 周期持续时间
  
	mu sync.Mutex // 加锁操作

	availableTokens int64 // 可用令牌数量

	latestTick int64 // 上个周期最后时刻
}
```

获取当前是第几个周期
```
func (tb *Bucket) currentTick(now time.Time) int64 {
	return int64(now.Sub(tb.startTime) / tb.fillInterval)
}
```

通过当前周期是第几个周期
```
func (tb *Bucket) adjustavailableTokens(tick int64) {
	if tb.availableTokens >= tb.capacity {
		return
	}
	tb.availableTokens += (tick - tb.latestTick) * tb.quantum
	if tb.availableTokens > tb.capacity {
		tb.availableTokens = tb.capacity
	}
	tb.latestTick = tick
	return
}
```

通过周期，判断还有多少可用token
```
func (tb *Bucket) adjustavailableTokens(tick int64) {
	if tb.availableTokens >= tb.capacity {
		return
	}
	tb.availableTokens += (tick - tb.latestTick) * tb.quantum
	if tb.availableTokens > tb.capacity {
		tb.availableTokens = tb.capacity
	}
	tb.latestTick = tick
	return
}
```

获取token流程
```
func (tb *Bucket) take(now time.Time, count int64, maxWait time.Duration) (time.Duration, bool) {
	if count <= 0 {
		return 0, true
	}

	tick := tb.currentTick(now)
	tb.adjustavailableTokens(tick)
	avail := tb.availableTokens - count
	if avail >= 0 {
		tb.availableTokens = avail
		return 0, true
	}
	// Round up the missing tokens to the nearest multiple
	// of quantum - the tokens won't be available until
	// that tick.

	// endTick holds the tick when all the requested tokens will
	// become available.
	endTick := tick + (-avail+tb.quantum-1)/tb.quantum
	endTime := tb.startTime.Add(time.Duration(endTick) * tb.fillInterval)
	waitTime := endTime.Sub(now)
	if waitTime > maxWait {
		return 0, false
	}
	tb.availableTokens = avail
	return waitTime, true
}
```
#### jaeger介绍

#### opencensus是google服务追踪和监控的框架
```
package main

import (
	"context"
	"contrib.go.opencensus.io/exporter/jaeger"
	"contrib.go.opencensus.io/exporter/prometheus"
	"contrib.go.opencensus.io/exporter/zipkin"
	"fmt"
	"github.com/micro/go-micro"
	"github.com/micro/go-micro/client"
	"github.com/micro/go-micro/registry"
	"github.com/micro/go-plugins/registry/etcdv3"
	"github.com/micro/go-plugins/wrapper/trace/opencensus"
	openzipkin "github.com/openzipkin/zipkin-go"
	zipkinHTTP "github.com/openzipkin/zipkin-go/reporter/http"
	"go.opencensus.io/stats/view"
	"go.opencensus.io/trace"
	"log"
	"net/http"
	"proto"
	"time"
)

var service micro.Service

func main() {

	registrys := etcdv3.NewRegistry(
		registry.Addrs("192.168.1.1:2379", "192.168.1.1:2379"),
		registry.Timeout(10*time.Second),
	)

	client.DefaultClient = client.NewClient(
		client.Registry(registrys),
		client.Wrap(opencensus.NewClientWrapper()),
	)

	service = micro.NewService(
		micro.RegisterTTL(5*time.Second),
		micro.RegisterInterval(4*time.Second),
		//micro.Client(client.DefaultClient),
		micro.Registry(registrys),
		micro.Name("greeter1"),
		micro.WrapHandler(opencensus.NewHandlerWrapper()),
		//micro.WrapClient(opencensus.NewClientWrapper()),
	)

	micro.RegisterHandler(service.Server(), new(HelloHandeler))

	service.Init()

	initZipkin()
	initJaeger()

	go prometheusTest()

	if err := service.Run(); err != nil {
		log.Fatal(err)
	}

}

func initZipkin() {
	localEndpoint, err := openzipkin.NewEndpoint("go.micro.srv.greeter", "")
	if err != nil {
		log.Fatalf("Failed to create the local zipkinEndpoint: %v", err)
	}
	reporter := zipkinHTTP.NewReporter("http://192.168.3.106:9411/api/v2/spans", zipkinHTTP.BatchInterval(1*time.Second))
	ze := zipkin.NewExporter(reporter, localEndpoint)
	trace.RegisterExporter(ze)
	trace.ApplyConfig(trace.Config{DefaultSampler: trace.ProbabilitySampler(0.9)})

}

func initJaeger() {
	exporter, err := jaeger.NewExporter(jaeger.Options{
		CollectorEndpoint: "http://192.168.3.168:14268/api/traces",
		Process: jaeger.Process{
			ServiceName: "trace-demo",
		},
	})
	if err != nil {
		log.Fatal(err)
	}
	trace.RegisterExporter(exporter)
}

func prometheusTest() {

	exporter, err := prometheus.NewExporter(prometheus.Options{})
	if err != nil {
		log.Fatal(err)
	}
	view.RegisterExporter(exporter)

	// Create view to see the number of processed videos cumulatively.
	// Create view to see the amount of video processed
	// Subscribe will allow view data to be exported.
	// Once no longer needed, you can unsubscribe from the view.
	if err = view.Register(append(opencensus.DefaultServerViews, opencensus.DefaultClientViews...)...); err != nil {
		log.Fatalf("Cannot register the view: %v", err)
	}

	// Set reporting period to report data at every second.
	view.SetReportingPeriod(1 * time.Second)

	addr := ":9000"
	log.Printf("Serving at %s", addr)
	http.Handle("/metrics", exporter)
	log.Fatal(http.ListenAndServe(addr, nil))
}

type HelloHandeler struct{}

func (p *HelloHandeler) Hello(ctx context.Context, req *proto.String, rsp *proto.String) error {

	// Register to all RPC client views.
	//if err := view.Register(opencensus.DefaultClientViews...); err != nil {
	//	log.Fatal(err)
	//}

	return call(ctx)
}

func (p *HelloHandeler) Test(ctx context.Context, req *proto.String, rsp *proto.String) error {
	time.Sleep(100 * time.Millisecond)
	return nil
}

func call(ctx context.Context) error {
	var method string
	//if i%2 == 0 {
	//	method = "HelloHandeler.Hello"
	//} else {
	method = "HelloHandeler.Test"
	//}

	response := new(proto.String)
	request := client.NewRequest("greeter2", method, &proto.String{})
	err := client.Call(ctx, request, response)

	fmt.Println("response", response)
	fmt.Println("err", err)
	return err

}

```
